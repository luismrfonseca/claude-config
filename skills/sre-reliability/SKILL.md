---
name: sre-reliability
description: Applies the concrete rules, formulas, checklists and templates from Google's Site Reliability Engineering book to production services (Node.js/NestJS on AWS by default, but principles are stack-agnostic). Use whenever the task touches reliability engineering - defining SLIs/SLOs/error budgets or deciding "how many 9s" a service needs; deciding what to alert on, fixing a noisy pager or designing dashboards and monitoring; setting up or sizing on-call; running or reviewing an incident; writing a postmortem; handling overload, retries, timeouts/deadlines, load shedding, retry storms or cascading failures; a production readiness / launch review ("is this ready for prod?"); deciding how much reliability to invest in versus features; or reducing toil. Trigger on phrases like "our alerts are noisy", "we had an outage", "how many 9s", "retry storm", "error budget", "SLO", "is this ready for prod", "pager fatigue", "thundering herd", "rate limit this", even when the user does not say "SRE".
---

# SRE Reliability

Distilled from Google's *Site Reliability Engineering* (cited as `SRE ch. N`). This skill supplies the specific criteria, numbers and templates; it assumes you already know how to run a generic incident or write a runbook.

The `SRE ch. N` markers are for your navigation and for when the user asks where a rule comes from. Keep them, and any "per the skill / the skill's template" phrasing, out of postmortems, alert designs and other deliverables — the user sees a document, not a skill, and citations to a book they never mentioned read as noise.

## Core principles

- Reliability is a product feature with a cost curve: each extra nine costs roughly 10x and users cannot perceive reliability beyond their own device/network. Set the target from user need and revenue, not from current performance, and treat it as both a floor and a ceiling (SRE ch. 3).
- The error budget (`1 - SLO`) is the contract between feature velocity and reliability. Product sets the SLO, monitoring measures it, the remaining budget decides whether releases continue. It removes the politics from "can we ship this" (SRE ch. 1, 3).
- Monitor symptoms, debug with causes. Page only on user-visible or imminent pain; every page must be urgent, actionable, require judgment, and be novel. Email alerts are `/dev/null` with extra steps (SRE ch. 6).
- Humans are the slowest component. Playbooks cut MTTR ~3x; automation of rote responses beats paging; ~70% of outages follow a change, so progressive rollout + fast rollback is the single biggest lever (SRE ch. 1).
- Toil (manual, repetitive, automatable, O(n) with growth) is capped at 50% of team time; past that, hand work back to the dev team or hand back the pager (SRE ch. 5, 11).
- Simplicity is a reliability feature: delete dead code, minimal APIs, one change per release (SRE ch. 9).

## Workflows

### Define SLIs, SLOs and an error budget policy

1. Classify the service and pick 2-4 SLIs: request/response -> availability, latency (percentiles of successful requests), throughput as capacity signal; storage -> latency, availability, durability; pipeline/async -> throughput, end-to-end freshness, correctness. Define exclusions (health checks, client 4xx, load tests) in writing.
2. Measure as close to the user as possible (ALB/CloudFront or synthetic probe over app metrics), as distributions not means, with 1-minute windows for alerting SLIs.
3. Write each SLO as `X% of <events> over <window> <condition>, measured at <where>, excluding <…>`. Use two latency targets (e.g. 99% < 300 ms, 99.9% < 1 s) when the curve shape matters. Start loose and tighten; keep an internal SLO tighter than any external SLA.
4. Convert to a budget: `allowed failures = (1 - SLO) x expected events per window`. Write the budget policy table (release cadence and engineering priority at >50%, 10-50%, <10%, exhausted) and get product + engineering sign-off before the first breach.
5. Report burn weekly; `burn_rate = observed error ratio / (1 - SLO)`.

Details, nines table, templates: `references/slos-and-error-budgets.md`.

### Design alerting or fix a noisy pager

1. Instrument the four golden signals per service: latency (split success/error), traffic, errors (explicit, implicit, by policy), saturation (event-loop lag, pool wait, queue age; p99 latency as leading indicator).
2. For each alert ask: does it catch an otherwise undetected, urgent, actionable, user-visible condition? Can it be safely ignored sometimes? Is the action automatable? Is someone else already paged? Fail any test -> demote to ticket, automate, or delete.
3. Build pages from symptoms: ratio threshold AND absolute floor AND `for` duration (two evaluation cycles), or SLO burn-rate alerts (14.4x over 1 h, 6x over 6 h page; 1-3x over days ticket). Per-instance CPU/memory/disk alerts become dashboard panels; keep only definite, imminent cause alerts (disk full in N hours, N+0 redundancy).
4. Group/inhibit so one incident produces ~1 page. Target <= 2 incidents per 12 h shift; report pages per shift quarterly.
5. If the SLO is unattainable today, loosen the alerting target temporarily and fix the system; sustained heroics are not a strategy.

Rule mechanics, CloudWatch/Prometheus examples, dashboard layout, noisy-pager triage procedure: `references/monitoring-and-alerting.md`.

### Run an incident (what this skill adds to a generic workflow)

- Declare when any of: a second team is needed, customers can see it, one hour of focused work has not resolved it. Declare early.
- Roles: Incident Commander (state, staffing, the live doc, never hands-on), Ops lead (the only people changing production), Comms lead (fixed-cadence updates, user messaging), Planning (bugs, tracking divergences from normal to revert later, relief). Explicit verbal handoff: "you are now IC, confirm".
- Priority order: stop the bleeding (shift traffic, shed load, roll back, disable subsystems) -> restore service -> preserve evidence -> root cause. Fly the plane first.
- Troubleshoot hypothetico-deductively: what changed last; divide/bisect the stack; inject known inputs at boundaries; test hypotheses in order of likelihood and inverse risk; keep timestamped notes; beware correlation and "it was X last time".

Live incident doc template, troubleshooting method, emergency-response lessons: `references/incidents-and-postmortems.md`.

### Write a postmortem

- Trigger on: user-visible impact beyond threshold, any data loss, on-call intervention (rollback, reroute), resolution time above threshold, monitoring failure. Postmortems for incidents that did not page are the most valuable.
- Blameless: assume good intent with the information available; fix systems and information flow, not people.
- Headings: Summary, Impact (quantified, incl. error budget consumed), Root causes (plural, five-whys depth), Trigger, Resolution, Detection, Action items (table: action / type mitigate-prevent-process-detect / owner / ticket / status, including the error-budget consequence), Lessons learned (went well / went wrong / where we got lucky), Timeline (UTC screenplay with OUTAGE BEGINS, INCIDENT DECLARED, MITIGATED, ENDS), Supporting info.
- Review by senior engineers against the checklist (data captured, impact complete, root cause deep enough, action items prioritized and owned, shared) before publishing; unreviewed postmortems do not count.

Full template derived from the book's example: `references/incidents-and-postmortems.md`.

### Handle overload, retries and deadlines

- Capacity in resources (CPU-seconds, event-loop utilization), not rps. Per-client quotas in those units, enforced at the backend; sum of quotas may exceed capacity.
- Client-side adaptive throttling: track 2-minute `requests` and `accepts`; reject locally with `P = max(0, (requests - K*accepts) / (requests + 1))`, K = 2 by default, 1.1 when rejection is nearly as expensive as serving.
- Four criticality tiers (CRITICAL_PLUS, CRITICAL, SHEDDABLE_PLUS, SHEDDABLE), set near the user, propagated on every call, shed lowest first; provision for the critical tiers only.
- Retries: max 3 attempts per request; per-client retry budget ~10% of requests; jittered exponential backoff always; retry at one layer only (3 retries at 3 layers = 64 attempts); distinguish retriable overload, "overloaded, do not retry", and permanent errors.
- Deadlines: always set one; propagate the remaining time down the stack; cancel the call tree on abandonment; never work on a request whose deadline passed; deadlines orders of magnitude above typical latency create bimodal-latency collapse.
- Servers: cap in-flight requests and return 503 early; short queues; LIFO/CoDel under overload; degraded modes that are exercised continuously and have a kill switch; process health and service health are different checks.

Formulas, Node/AWS mapping: `references/overload-and-cascading-failures.md`.

### Prevent or break a cascading failure

Prevention, in priority order: load test to the breaking point and watch the failure mode; degraded results; server self-protection; rate limits at edge/LB/instance; capacity planning with N+2 from measured breaking points; retry/deadline hygiene; no intra-layer calls on the request path; classify caches as latency vs capacity; split process vs service health checks; change logging.

During one: check and revert recent changes -> add capacity if any is idle -> relax health-check-driven kills so instances can warm -> restart only wedged instances, slowly -> drop sheddable traffic, then if crash-looping cut to ~1%, let instances recover, ramp back gradually -> enter degraded modes -> stop batch load -> block queries of death. Recovery needs load far below the original knee (10% healthy instances means ~10% of the load), and the trigger must be fixed first or the cascade returns.

### Launch / production readiness review

Walk the checklist themes: architecture and dependencies (request flow, fan-out, owners), capacity (launch spike up to 15x, N+2, quotas raised), failure modes (per dependency: detect, degrade, start without it), client behaviour (backoff+jitter, randomized schedules, server-controlled config), manual processes documented, code and config in VCS with release branches, external dependencies and partners, monitoring (golden signals, probes, monitoring-the-monitoring), security, 10x growth bottlenecks, rollout plan with owners and tested rollback, backups with restore drills, SLOs and budget agreed. Every checklist item must trace to a past incident; every action item must be concrete.

Release principles: staged rollouts with bake time and automatic rollback, feature flags that are independently revertible, hermetic builds, release branches, generated changelogs. Checklist table and the testing hierarchy: `references/launch-and-testing.md`.

### On-call and toil

Ack targets 5 min (user-facing) / 30 min (relaxed); <= 2 incidents per 12 h shift (an incident costs ~6 h all-in); on-call <= 25% of a person's time -> 8 people per site for 24/7 with primary+secondary, 6 per site for follow-the-sun; everyone on-call at least once or twice a quarter to avoid underload. One primary owns pages; polarize time (on-call week is written off for projects); never random-assign tickets; scrub pages and tickets by class and silence a class once its root-cause fix is scheduled. Onboard with a learning checklist, curated postmortems, weekly disaster role-play, shadow then reverse-shadow on-call. Details: `references/oncall-and-toil.md`.

## Reference map

| File | Contents |
|---|---|
| `references/slos-and-error-budgets.md` | SLI/SLO/SLA definitions, risk-tolerance questions, nines table, SLI selection per service type, SLO template and rules, error budget policy table, anti-patterns (SRE ch. 1, 3, 4). |
| `references/monitoring-and-alerting.md` | Page/ticket/log outputs, symptoms vs causes, golden signals with AWS/Node sources, instrumentation rules, the alert tests, rule mechanics and burn-rate tiers, dashboards, noisy-pager triage (SRE ch. 6, 10). |
| `references/incidents-and-postmortems.md` | Declaration criteria, ICS roles, live incident doc, troubleshooting method, emergency-response lessons, postmortem triggers/review/template, outage tracking (SRE ch. 12-16, App. C-D). |
| `references/overload-and-cascading-failures.md` | Quotas, adaptive throttling formula, criticality, load shedding, retry budgets, deadlines, cascade mechanics, prevention checklist, testing, immediate steps, Node/AWS translation (SRE ch. 21-22). |
| `references/launch-and-testing.md` | Launch checklist, rollout and feature-flag techniques, test hierarchy for reliability, production tests and probes, release engineering, simplicity, fail-sane config (SRE ch. 8, 9, 17, 27, App. B, E). |
| `references/oncall-and-toil.md` | Toil definition and 50% rule, on-call numbers and rotation sizing, overload/underload, interrupt handling, on-call onboarding (SRE ch. 5, 11, 28, 29). |

When answering, give the specific number, formula or checklist from the reference rather than a general recommendation, and say why the rule exists so the user can adapt it.
