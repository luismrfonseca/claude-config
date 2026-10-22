# Launch Readiness, Testing and Release

Distilled from *Site Reliability Engineering* ch. 8, 9, 17, 27, Appendices B and E, with a few lines from ch. 33. Use for production readiness reviews, launch checklists, deciding what tests a reliability change needs, and release/rollout policy.

## Contents
1. Production readiness / launch checklist
2. Launch techniques: staged rollouts, feature flags, client behaviour, load tests
3. Testing for reliability: the hierarchy and where to start
4. Production tests: config tests, stress tests, canaries, probes
5. Release engineering principles
6. Simplicity rules
7. Fail-sane rules for config and input
8. Lessons from other industries

---

## 1. Production readiness / launch checklist (SRE ch. 27, App. B, App. E)

Design principles for the checklist itself: lightweight (engineers route around a heavy process), catches the obvious, consistent, scalable (fast path for common launch classes, deep path for novel ones). Every item must be justified by a past incident, and every action item must be concrete and doable. Review the whole list once or twice a year and delete the obsolete.

Use the questions; the AWS/Node column suggests what "done" looks like.

| Theme | Questions | Typical action items |
|---|---|---|
| Architecture and dependencies | Request flow user -> edge -> services -> data? Which request types have different latency needs? Does one page view fan out to many calls? Which shared infra and which owners become stakeholders? | Separate user-facing from non-user-facing traffic (different services/target groups/queues). Validate volume assumptions with fan-out multipliers. List every dependency with its SLO. |
| Capacity | Tied to a press release/campaign? Expected traffic and growth during and after launch? Resources acquired, including lead times? Spike up to 10-15x the estimate has happened; traffic *mix* at launch differs from steady state. | Load test with a launch-shaped mix. N+2: peak served by N units while the two largest are out (AZ loss + deploy). Launch in one region/country first. Service quotas raised (ALB, Lambda concurrency, RDS connections, SES/SNS limits). Don't mistake day-one load for steady state. |
| Failure modes | Single points of failure? What happens when an instance, AZ, region, or the network between them fails? Each dependency: how do we detect it dying, what do we do, can we start without it, can we serve degraded without it? Bad input? DoS? | Request deadlines everywhere; load shedding; circuit breakers/fail-fast on dead backends; degraded mode for each non-critical dependency; input validation that keeps the last good state. |
| Client behaviour | Any auto-save, polling, heartbeat, sync, auto-refresh? Mobile clients you cannot patch quickly? | Exponential backoff with jitter on every client retry; random spread for periodic jobs (never "everyone at 02:00"); server-controlled client config (poll interval, retry policy, kill switches) so behaviour can be changed without an app release; don't retry 4xx. |
| Processes and automation | Which manual processes keep it running (release, failover, restore, rotate secrets, scale)? | Document each so any team member can execute it at 03:00; automate build and release; test rollback and restore before launch. |
| Development process | All code and config in version control? Releases cut from branches with cherry-picks, not from an ever-moving main? | Config changes reviewed and deployed like code; release branch per release. |
| External dependencies | Third-party code/services/data? Partners depending on you who need notice? Hard external deadline and what if the vendor slips? | Proxies/caches/transcoding in front of vendors; vendor quotas and rate limits understood; contingency plan (backup announcement, partial launch). |
| Monitoring | Golden signals, SLO dashboards, black-box probes on the public endpoint, internal-state metrics, "monitoring the monitoring" (dead-man's switch), financially important alerts and audit logs? Alerts routed (page vs ticket)? | See monitoring reference. Alert on absence of data (heartbeat) for critical pipelines. Don't send alert email from inside the serving path. |
| Security | Security design review, code audit, authn/authz, TLS, abuse/spam, pre-launch access control, block lists? | WAF rules, secrets in a manager not env files, least-privilege IAM, pre-launch gated behind allow-lists. |
| Growth | Spare capacity for 10x? Where does scaling stop being linear (DB writes, single-tenant hot keys, cache)? Sharding/resharding plan? Growth alerts? | Ticket-level alerts on capacity trends (disk at 70%, connections at 60%), documented bottleneck ladder: "at ~N rps we need X". |
| Rollout plan | Steps, owners, order of dependencies (feature flags across several services take hours), risk per step, contingency if a step fails, no Friday or launch-day-eve deployments, SOPs for this and affected services. | Written launch plan with named owners per step, go/no-go criteria, rollback owner and tested rollback. |
| Data | Persistent data? Backups implemented, restore tested, retention, PII handling? | Automated backups with periodic restore drills; data-integrity reconciliation jobs. |
| SLOs and error budget | SLIs defined and measured from the user's side, SLO and budget policy agreed, on-call owner and escalation path named? | Launch blocked if the budget is exhausted unless an explicit exception is granted. |

## 2. Launch techniques (SRE ch. 27, App. B)

Gradual rollouts: essentially nothing user-facing ships "push-button" to everyone. Pattern: a few instances in one region, observe for a defined bake period, then the region, then global; different stages in different geographies catch diurnal/traffic-mix issues. Automation observes the new instances and rolls back on crash or metric regression. If anything unexpected appears, roll back first, diagnose after (MTTR first). Also applies to app updates (percentage rollout in app stores) and to sign-ups (rate-limited invites to grow load gradually). AWS: CodeDeploy canary/linear with CloudWatch alarms as rollback triggers, weighted target groups on the ALB, Lambda aliases with weighted routing.

Feature flag framework requirements: many changes in flight at once, each scoped to a few servers/users/entities/regions; ramps from 1-10% upward; routing by user/session/object/location; new code paths fail safe by design without affecting users; each flag independently and instantly revertible; measure impact. Stateless UI changes: payload rewriting keyed by cookie hash range. Stateful logic: route a subset of user/entity IDs to the new path. Flags beat parallel release tracks: ship dormant code, enable later, kill instantly.

Abusive first-party clients: a sync interval change from 600 s to 60 s is a 10x load increase with no user growth. Enforce backoff + jitter, classify retriable errors, randomize scheduled client work, keep server-side control over client behaviour.

Overload behaviour and load tests: services are nonlinear near saturation (caches make light load cheap; logging of errors can cost more than serving; GC thrashing). You cannot predict the knee from first principles, so load testing is required for most launches, including the behaviour past the knee (see overload reference, section 11).

## 3. Testing for reliability (SRE ch. 17)

Testing quantifies confidence that behaviour after a change equals behaviour before; monitoring quantifies past reliability. Each passing test before and after a change reduces the uncertainty you must budget for. Tests cannot prove reliability, but failing tests prove its absence. A test that catches what monitoring would have caught, before the push, is a bug fixed with zero MTTR, which is the cheapest reliability there is.

Hierarchy and what each is for:

| Level | Purpose | Reliability-specific notes |
|---|---|---|
| Unit | Correctness of a function/class; executable spec. | Cheap; milliseconds. Needed for the libraries other layers assume work (a mocked dependency with untested behaviour makes the test meaningless). |
| Integration | Assembled components with injected fakes for stateful dependencies. | Replace the DB with a precisely specified fake; test retry/timeout/backoff logic here. |
| System: smoke | Simple, critical behaviour; short-circuits expensive suites. | First thing to add to a codebase with no tests; run on every release. |
| System: performance | Catch creeping regressions in latency/memory before users do. | Track p99 and RSS per release; fail on drift beyond a threshold. |
| System: regression | A gallery of past bugs as tests. | Convert every reported bug into a failing test first; outage-causing queries ("queries of death") become regression tests. |

Where to start when coverage is poor: stack-rank components by business criticality (billing, auth, APIs other teams integrate against), write smoke tests for every release, turn bugs into tests, set explicit coverage goals with deadlines rather than "we need more tests". Keep main green: a broken build is a drop-everything priority because later changes bury the defect, everyone works around it, and emergency releases (security fix) become impossible.

Test flakiness arithmetic: to reject a good patch less than 1% of the time across a suite of N tests run before and after, each test must pass with probability 0.99^(1/2N); for 21,000 tests that is 99.9999%. Flaky tests are a reliability problem of the test system; quarantine and fix them.

Test the tools too: automation and repair tools act outside the normal API; test that the system's observable invariants hold before/after the tool runs, build barriers so risky tools can only touch replicas that are already out of service, and test restart/checkpoint semantics of tools that may be moved by other automation.

## 4. Production tests (SRE ch. 17)

- Configuration tests: compare what is actually deployed (query the running service for its effective config) with what is in version control; the pass/fail pattern across the fleet shows how far production lags the repo and flags unsafe version combinations during a rollout.
- Stress tests: find the limits (how full can the DB get before writes fail, how many rps before the service degrades). Inputs to capacity planning and to the overload work.
- Canary: not a test but structured user acceptance. Upgrade a subset, bake, watch for variances, roll back on any. Exponential rollout: with a fault of order U and traffic growing by e every K, cumulative variance reports C = R*K/U; most bugs are order 1 (linear in traffic). Keep K small (ramp in many small steps) to minimise users exposed while still learning quickly. Higher-order bugs (state corruption visible to later requests) only show in canaries, never in replayed-request regression tests, so catch them at release time.
- Production probes: maintain three request banks: known-bad (must error), known-good replayable in production, known-good not replayable. Use them as integration tests, release tests and as continuous black-box probes. Probes exercise the real LB+frontend+backend combination, which the hermetic release test never did; failing probes mean prod and release environments differ, i.e. the site is probably broken. Roll probes out alongside the app so old/new probe x old/new app combinations are all exercised; a readiness check that includes the probes makes a bad rollout stall safely without taking traffic.
- Break-glass pushes: allow emergency pushes before tests finish, but keep the tests running, link the push to the pending test run, back-annotate failures, and file a bug so each break-glass creates pressure for a better fix. Disabling tests on break-glass means the first signal is user impact.
- Config files: classify each as either "changed only during incidents to keep MTTR low" (rare, risky manual edits) or "changes more often than the app" (needs test and monitoring coverage *better* than the app, or it dominates reliability negatively). Prefer schema-checked formats with bounded parse time (JSON schema, protobuf) over executable config; validate semantics as well as syntax.

## 5. Release engineering principles (SRE ch. 8)

- Self-service: teams run their own releases with shared tooling; the process requires humans only when something fails.
- High velocity, small batches: frequent releases mean fewer changes per release, easier testing and troubleshooting ("push on green" where tests gate the deploy). Fit the cadence to risk: hourly automatic in pre-prod; for user-facing services start in one unit and expand exponentially; for sensitive infra spread over days and interleave regions.
- Hermetic, reproducible builds: pinned toolchain and dependencies (lockfiles, pinned base images, pinned Node version), no network at build time, the same revision builds the same artifact anywhere. Build tools versioned with the project so rebuilding last month's release uses last month's compiler.
- Release branches with cherry-picks from main, so each release's exact content is known and bug fixes don't drag in unrelated changes; re-run the tests on the release branch because its content may exist nowhere on main.
- Artifacts are named, hashed, signed, and labelled with their stage (dev/canary/prod); a label moves to the newest artifact carrying it. Every binary can report build ID, revision and date.
- Gates: who may approve code, cut a release, approve cherry-picks, deploy, change build config. Every release ships with an automatically generated changelog archived with the artifact, which is what the on-caller reads first when a release misbehaves.
- Configuration is code: in version control, reviewed, and either packaged with the binary, packaged separately but versioned against it (so a flag fix is a config rebuild, not a binary rebuild), or served from a dynamic store for values that must change at runtime. Avoid skew between checked-in and running config.
- Budget release engineering from day one; retrofitting is more expensive.

## 6. Simplicity (SRE ch. 9)

- Stability and agility are not opposites in practice: reliable, fast rollouts make changes visible and bugs cheap to find.
- Boring is good. Distinguish essential complexity from accidental; push back on accidental complexity entering a system you are on-call for, and keep removing it from systems you inherit.
- Every line of code is a liability. Delete dead code; do not comment it out or gate it behind a permanently-off flag (a flag flip later executes untested code: the Knight Capital pattern). Celebrate negative lines of code.
- Minimal APIs: fewer methods and arguments, each done well. A small API usually means a well-understood problem.
- Modularity: loose coupling between services and between code and config, versioned APIs so release cadences can differ, no "misc/util" services with unrelated responsibilities, backward- and forward-compatible wire formats.
- Release simplicity: one change per release is easiest to attribute; batches of 100 unrelated changes make performance regressions unattributable. Small steps, measured each time.

## 7. Fail sanely (SRE App. B)

Validate configuration syntax and semantics; treat empty, truncated or implausibly different input (N% smaller than last version, a pattern that matches everything) as invalid; keep running on the last good state *and* alert; expire-sensitive data should alert well before it expires. Two canonical failures: an empty DNS config served NXDOMAIN for everything for six minutes; a malware list replaced by "/" flagged the whole web. Both are caught by "is this plausibly different from the last version?" checks and by "does the new config still contain the things we know must be there?".

## 8. Lessons from other industries (SRE ch. 33)

Nuclear, aviation, medicine and manufacturing converge on the same practices: rehearsed disaster drills and live simulations, swing capacity, defence in depth, training and certification before taking the console, blameless post-incident review, relentless automation of repetitive work, and structured decision-making under pressure. The differences are cost of failure and change velocity; the practices transfer.
