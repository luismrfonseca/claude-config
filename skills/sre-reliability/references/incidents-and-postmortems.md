# Incidents, Troubleshooting and Postmortems

Distilled from *Site Reliability Engineering* ch. 12, 13, 14, 15, 16 and Appendices C, D. This file holds the concrete criteria, roles, templates and reasoning rules; a generic incident workflow is assumed to exist elsewhere.

## Contents
1. When to declare an incident
2. Incident command roles and protocol (ICS-derived)
3. Live incident state document template
4. Troubleshooting method (hypothetico-deductive)
5. Emergency response lessons
6. Postmortem triggers, blamelessness, review
7. Postmortem template (from Appendix D)
8. Tracking outages: what to count

---

## 1. When to declare an incident (SRE ch. 14)

Declare early; closing an incident that turned out to be trivial costs minutes, spinning up coordination hours into a spiralling outage costs the outage. Any one of these means it is an incident:

- A second team is needed to fix it.
- The outage is visible to customers.
- One hour of focused analysis has not resolved it.

Keep the skill alive by running the same protocol for planned cross-team/cross-timezone changes and for DR exercises; it atrophies when used only in real emergencies.

## 2. Incident command roles and protocol (SRE ch. 14)

The three failure modes of an unmanaged incident: the responder is too deep in the technical problem to see the whole picture; nobody communicates, so leadership interrupts the responders and helpers duplicate work; well-meaning "freelancers" change production without coordination and make it worse.

Roles (recursive: any role can delegate and create sub-incidents when overloaded; the commander holds every role not yet delegated):

| Role | Owns | Does not |
|---|---|---|
| Incident Commander (IC) | High-level state, staffing, assigning work by priority, removing roadblocks, keeping the live state document current, explicit handoff. | Debug. The IC who starts typing commands has stopped commanding. |
| Operations lead | All changes to the system; coordinates the people doing hands-on mitigation. The ops group is the only group modifying production. | Communicate to stakeholders. |
| Communications lead | Periodic updates to stakeholders and the response team (fixed cadence, e.g. every 30 min or on state change), user-facing messaging, keeping the summary at the top of the state doc accurate. | Make technical decisions. |
| Planning lead | Longer-horizon support: filing bugs, tracking every deviation from normal config so it can be reverted, arranging handoffs and relief, food/rest. | |

Protocol:
- Recognized command post: one channel (Slack/IRC) that is the log of record; bots post alerts and deploys into it. Chat logs feed the postmortem timeline.
- Anyone helping reports to the IC/ops lead and takes tasks from them; "just looking" is fine, changing state without coordination is not.
- Handoff is explicit and verbal: "You are now incident commander, confirm?" and the outgoing IC stays until acknowledged; announce it to the channel. Plan relief before people are exhausted (start looking for replacements at least an hour before end of day).
- Do not depend on the system you are fixing for your incident tooling (state doc, chat, pager). Keep an out-of-band channel and tested CLI/alternate access paths.

Priorities, in order: stop the bleeding, restore service, preserve evidence for root-causing. Then trust (full autonomy within role), introspect (if you feel panic, ask for more people), reconsider alternatives periodically, practice, and rotate roles between incidents.

## 3. Live incident state document (SRE App. C)

Editable concurrently; most important info at top; keep it for the postmortem.

```
# <Service> <short problem>: <date>          incident #<n>   status: active|mitigated|closed
Summary (comms lead keeps current): one or two sentences.
Command post: #channel
Roles: IC: <name>  Ops: <name>  Comms: <name>  Planning: <name>  Next IC: <name/tbd>

Detailed status (update at least every 4 h and at every comms handoff; stamp time+author)
Exit criteria:
  - [ ] <fix in place>
  - [ ] within SLO (availability x, latency y) for 30+ minutes
TODO / bugs filed:
  - [x] ... DONE   - [ ] ... (ticket link)

Timeline (most recent first, UTC, author per line)
  hh:mm <who>  <action or observation>
```

## 4. Troubleshooting method (SRE ch. 12)

Model: problem report -> triage -> examine -> diagnose -> test/treat -> cure, iterating hypothesis/experiment until the cause is found; re-triage whenever the situation changes.

| Step | Rules |
|---|---|
| Problem report | Expected vs actual behaviour, how to reproduce. One ticket per problem even if it arrived by chat, so investigation is logged and goes to whoever is on duty, not to whoever the reporter knows. Make alerts link to the probe result and playbook entry. |
| Triage | Match response to impact. In a serious outage, resist root-causing first: make the system work as well as it can (shift traffic away, drop load to prevent cascade, disable subsystems, roll back) while preserving logs/evidence. "Fly the plane first." If a bug threatens unrecoverable data corruption, freezing the system beats keeping it running. |
| Examine | Metrics (graph and compare time series), logs (structured, with verbosity that can be raised on the fly and sampled at high volume; a selection language: "show ops that took > 10 ms" or "that called X"), and endpoints exposing current state (recent RPCs in/out, per-RPC error/latency histograms, live config). Instrument a client if you need to see what a component returns. |
| Diagnose | Simplify and reduce: components have known input->output transformations, so inject known inputs and check outputs at each boundary. Divide and conquer: walk the stack end to end, or bisect for large systems. Ask what the system is doing, where its resources/output go, and why ("five whys"). What touched it last: ~70% of outages follow a change, so check deploys/config/load-mix changes first; annotate dashboards with deploy windows. |
| Test and treat | Design tests with mutually exclusive outcomes; run in decreasing order of likelihood and increasing order of risk; watch for confounders (a firewall rule that makes a test from your laptop fail where the app's host would succeed) and for side effects (verbose logging worsening a latency problem). Some evidence will only be suggestive (races, deadlocks). Keep timestamped notes of every hypothesis, test and result in a shared doc/chat; return the system to a known config after active experiments. |
| Cure | Multiple contributing factors are normal; reproduction in prod may be impossible, so settle for probable causes with corroborating evidence. Write it up. |

Pitfalls: chasing irrelevant symptoms or misreading metrics; not knowing how to change the system safely to test a hypothesis; improbable theories or "it was X last time so it is X now"; mistaking correlation (two effects of one shared cause) for causation. Prefer simpler explanations but remember that several small ongoing problems may jointly explain the symptoms better than one rare one. Negative results are valuable: record and share what you ruled out and why.

What makes troubleshooting cheap: observability (white-box metrics + structured logs) in every component from day one, observable interfaces between components, a request ID propagated through every hop (`x-request-id` in NestJS middleware -> logs -> downstream calls -> X-Ray/OTel trace), and logged, controlled changes.

## 5. Emergency response lessons (SRE ch. 13)

- Don't panic; pull in more people when overwhelmed, including the person whose action triggered the event: they hold the most state.
- Three archetypes: test-induced (a planned failure test exposes unknown dependencies), change-induced (a global config push triggers a crash loop; rollback within 5 minutes saved it), process-induced (automation with a missing sanity check: an empty filter meant "all machines").
- Lessons that recur: test rollback procedures before any large-scale test; canary every change thoroughly regardless of perceived risk, since the "low-risk" push is the one that skips the canary; alert storms during a wide outage overwhelm responders and comms channels (grouping/inhibition matter); rate-limited propagation of config to clients throttled the crash loop, an accidental but valuable brake; your debugging tools must not share fate with the thing that is down; automation needs sanity checks and bounded blast radius ("zero can mean all").
- Keep an outage history; hold people accountable for closing action items.
- Ask "what if" at scale: region loss, credential compromise, primary datastore gone. Who calls whom, what breaks, can the person next to you do it? Then test it for real on your schedule, with your best people watching, rather than at 02:00 on a weekend.

## 6. Postmortem triggers, blamelessness, review (SRE ch. 15)

Decide triggers before the first incident. Typical: user-visible downtime or degradation beyond a threshold; any data loss; on-call intervention (rollback, traffic reroute); resolution time above a threshold; a monitoring failure (incident found manually). Any stakeholder may request one. Postmortems for incidents that did not page are especially valuable: they expose monitoring gaps.

Blameless means: assume everyone acted with good intent on the information they had; investigate why the system/process let them have incomplete or wrong information; name actions and systems, not people, as the things to fix. You cannot fix people, only the systems around them. Blame drives incidents underground, and the organization loses the information. It also means not stigmatizing the team that produces many postmortems, and visibly rewarding good incident handling and good write-ups.

Review before publishing; an unreviewed postmortem might as well not exist. Reviewer checklist:
- Key incident data captured (timeline, metrics, logs linked).
- Impact assessment complete and quantified (users, requests, revenue, error budget consumed).
- Root cause(s) deep enough (not "human error"; not stopping at the trigger).
- Action items appropriate: owners, priority matching severity, mix of mitigate/prevent/process; not knee-jerk over-specific monitoring where a unit test would catch it earlier.
- Shared with the stakeholders who need it.

Then publish to the widest useful audience (never with end-user-identifying data), add to the repository, and keep it alive: postmortem of the month, reading clubs, re-enactments for new on-callers. Survey the team periodically on whether the process is too much toil.

## 7. Postmortem template (derived from SRE App. D)

```
# <Service> <incident name> postmortem (incident #<n>)
Date:        <incident date>
Authors:     <names>
Status:      Draft | In review | Complete, action items in progress | Complete
Summary:     One sentence: what broke, for how long, for whom.
Impact:      Quantified: duration, requests/users/customers affected, revenue, SLO/error budget consumed, data loss.
Root causes: The conditions that allowed the failure (plural if plural). Use five-whys style; distinguish from trigger.
Trigger:     The event that activated the latent problem (traffic surge, deploy, config push, expiry).
Resolution:  What stopped the bleeding, what fixed it, what is still temporary (extra capacity, feature disabled).
Detection:   How it was noticed (which alert, or a human) and how long after onset.

## Action items
| Action | Type (mitigate / prevent / process / detect) | Priority (P0/P1/P2, matching severity) | Owner | Ticket | Status |
Include: playbook updates, regression tests for the failure mode, monitoring gaps, load shedding/limits,
process changes (game-day test of this scenario), and the error-budget consequence (freeze/exception).

## Lessons learned
What went well:   detection speed, mitigations that worked, prepared tooling.
What went wrong:  skills out of practice, budget blown, slow steps, misleading signals.
Where we got lucky: near misses that would have made it worse (evidence happened to be in logs,
                    someone happened to be watching, the fix data happened to exist).

## Timeline (UTC, screenplay style; start from the incident state doc, add external events)
hh:mm  external/precursor events (news, traffic change, deploy)
hh:mm  OUTAGE BEGINS
hh:mm  pager/alert fires
hh:mm  INCIDENT DECLARED, IC named, channel
hh:mm  each hypothesis, attempted mitigation and its result (including failed ones and arithmetic errors)
hh:mm  OUTAGE MITIGATED
hh:mm  OUTAGE ENDS
hh:mm  INCIDENT ENDS (exit criteria met: N minutes within SLO)

## Supporting information
Dashboards with fixed time range, log queries, chat log export, traces, graphs.
```

Notes from the example: the action items table in the book mixes DONE and TODO with ticket links; the error-budget consequence ("freeze production until <date> or seek exception") is itself an action item with an owner; "where we got lucky" is mandatory thinking, because luck is not a control.

## 8. Tracking outages (SRE ch. 16)

Postmortems cover the big incidents; a lightweight outage tracker covers the frequent small ones and the cross-team patterns. Minimum viable version: every page/alert lands in a store (PagerDuty/Opsgenie export, or a Slack channel hooked to a small DB), alerts are grouped into incidents, and incidents get free-form hierarchical tags (`cause:network`, `cause:deploy`, `action:rollback`, `bogus` for false positives, `bug:1234`).

Questions it should answer: incidents per shift/week/quarter; alerts per incident (target ~1); actionable vs non-actionable ratio; which service or dependency causes the most incidents (interpret with care: it may reflect over-sensitive alerting or a misbehaving client rather than the worst component); trend vs. own history and vs. sibling teams ("third time this week" means nothing without a baseline). Use it for shift handoff summaries, weekly production reviews, and quarterly pager-load reporting to management.
