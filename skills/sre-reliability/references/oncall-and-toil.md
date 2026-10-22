# On-Call, Toil and Interrupts

Distilled from *Site Reliability Engineering* ch. 1, 5, 11, 28 and 29. Use when setting up or fixing an on-call rotation, sizing it, deciding whether the team's operational load is sustainable, onboarding someone to on-call, or deciding what work to automate away.

## Contents
1. Toil: definition and the 50% rule
2. On-call numbers: response time, shift load, rotation size
3. Rotation structure and handoffs
4. Feeling safe: what the on-caller needs
5. Operational overload and underload
6. Interrupts: polarize time, ticket handling
7. Onboarding to on-call

---

## 1. Toil (SRE ch. 5)

Toil is operational work that is some combination of: manual (hands-on time running a script is still toil), repetitive, automatable (a machine could do it, or the need could be designed away), tactical/interrupt-driven, without enduring value (the service is in the same state afterwards), and O(n) with service growth. Not toil: overhead (meetings, reviews, hiring) and grungy one-off work that leaves a lasting improvement (cleaning up alert config is not toil).

Classification test for any recurring task:

| Question | Yes => |
|---|---|
| Does the service end up permanently better afterwards? | Engineering, not toil. |
| Would it need doing twice as often if traffic doubled? | Toil; automate or design away before the next doubling. |
| Is human judgment *intrinsically* required, or only because the system was built badly? | If the latter, it is toil until the redesign ships; pages that need "complex judgment" several times a day are a design bug, not a judgment job. |

The 50% rule: cap operational work (pages, tickets, manual ops, interrupts) at 50% of team time averaged over quarters; the rest must be engineering that reduces future toil or improves the service. A team whose toil creeps past 50% scales linearly with the service and ends up as a pure ops team. If the cap is breached: push tickets/bugs back to the dev team, put devs in the pager rotation, add headcount without adding ops scope, or hand the pager back (section 5). Toil floor from on-call alone: a 6-person rotation with 1 week primary + 1 week secondary per cycle puts 2/6 = 33% of each person's time into on-call; 8 people => 25%.

Why it matters beyond morale: toil hides technical debt, sets a precedent that dev teams can offload ops onto you, slows feature velocity, and drives the best engineers out. Measure it (quarterly self-reported toil % is good enough) and act on outliers, not the average.

## 2. On-call numbers (SRE ch. 1, 11)

| Parameter | Book's value | Why |
|---|---|---|
| Page acknowledgement target | 5 min for user-facing/time-critical, 30 min for relaxed SLOs | Response time must fit the error budget: 99.99%/quarter is ~13 min of downtime, so a 30-min ack cannot meet it; engineer automation instead. |
| Incidents per 12-h shift | <= 2 (an incident = related pages sharing a root cause and a postmortem) | Each incident costs ~6 h including root-cause, remediation, postmortem and follow-up. More than 2 and nothing gets investigated properly. |
| Target distribution | Flat, median 0 per day | A component paging daily means the next unrelated failure overflows the shift. |
| Lower bound | >= 1 event per shift on average, or on-call at least once or twice a quarter per engineer | Underload: engineers lose touch with production, confidence (over or under) drifts, gaps appear only during real incidents. |
| On-call time share | <= 25% of an engineer's time (half of the 50% ops cap) | Leaves room for engineering. |
| Minimum rotation size | 8 for one site with primary + secondary and week-long shifts (each person on-call 1 week in 4); 6 per site for a two-site follow-the-sun team | Derived from the 25% rule; also keeps the rotation small enough that everyone stays current. |
| Compensation | Time-off-in-lieu or cash, capped | The cap doubles as a hard limit on how much on-call any individual absorbs. |

For a small team (3-5 engineers) the math says you cannot run sustainable 24/7 on-call alone. Options: share the secondary rotation with a sibling team, put the dev team in the rotation, restrict paging to business hours for non-critical services (and set the SLO accordingly), or buy follow-the-sun coverage. Pretending a 4-person rotation is fine produces burnout and quiet attrition.

## 3. Rotation structure (SRE ch. 11, 29)

- Exactly one primary responds to pages; a single named owner avoids the bystander effect. The primary may escalate but owns the incident until handed off.
- Secondary options: (a) fall-through only if primary misses the page (can be another team); (b) handles tickets and non-urgent production work while primary handles pages; (c) actively assists the primary during high page volume. Pick one and write it down.
- Dev team participates in a 24/7 escalation path. Escalation is the principled response to an outage with unknown scope, not an admission of failure.
- Alert fan-out: aim for 1 alert per incident. Group related alerts, inhibit per-instance alerts when the service-level alert fires, silence duplicates during an incident so the on-caller can work.
- Handoff at the end of each shift: open incidents, ongoing rollouts, silenced alerts and why, tickets in flight. Do the same for the ticket rotation; most teams do page reviews but skip ticket handoffs, so ticket root causes never get fixed.
- Define transferable roles (push manager, ticket handler) so a long-running rollout or ticket does not follow one person past their shift.

## 4. What an on-caller needs to feel safe (SRE ch. 11)

Stress pushes people from deliberate reasoning into fast heuristics (confirmation bias: "the fourth page this week, must be the same upstream problem"). Reduce the stress structurally:

- Clear escalation paths with names and response expectations.
- A written incident-management protocol to fall back on when the scope is unknown or multiple teams are involved (see incidents reference).
- A blameless postmortem culture, so the on-caller is not deciding under fear.
- Playbooks for the pages that exist: the book reports ~3x faster MTTR with a playbook than when improvising. Every page should link to its entry.
- Explicit permission to roll back, drain traffic, and take other mitigations without seeking approval first.

## 5. Operational overload and underload (SRE ch. 11)

Overload signals, make them measurable: pages per shift > 2, daily tickets above an agreed number, toil self-reports > 50%, postmortems not getting written, action items not closing.

Standard causes and fixes:

| Cause | Fix |
|---|---|
| Misconfigured monitoring (cause-based, per-instance, non-actionable pages) | Re-align paging with SLO-threatening symptoms; demote the rest to tickets or dashboards. See monitoring reference, section 8. |
| Alert fan-out (many alerts per incident) | Grouping/inhibition rules; silence duplicates during incidents. |
| Dev team shipping noisy or unreliable changes | Joint goals with the dev team; error budget policy; put devs in the rotation. |
| Architecture needs quarters of work to become operable | Temporarily route some or all pages to the dev on-call ("give back the pager") until the service meets the onboarding bar; loan an experienced engineer to drive the recovery. Write concrete recovery objectives into quarterly planning. |

Underload is also a failure mode: a quiet service with a large rotation means each person is on-call rarely and forgets how production works. Shrink the rotation, run disaster role-play exercises, and run periodic game days.

## 6. Interrupts (SRE ch. 29)

Interrupts come in three classes: pages (urgent), tickets (non-urgent, must be done), and ongoing responsibilities (pushes, flag flips, long rollouts). The human cost of an interrupt is the context switch, not the task: a 20-minute interruption costs a couple of hours of real project work.

Rules the book recommends to managers:

- Polarize time. A person should know, for the whole day (ideally the whole week), whether they are on project work or on interrupts. Never both.
- The primary on-call does on-call work only. Write the on-call week off for project purposes. If someone's project cannot slip a week, swap their shift; do not expect progress on both.
- If the interrupt volume is too high for one person, add a second person to that interrupt role. Do not spread it across the whole team.
- Stop random auto-assignment of tickets to the team. Tickets are a dedicated role for a bounded period, handled by the primary/secondary or a named ticket person.
- People not on interrupts should not pick up tickets "to look busy": it hides the true load and breaks the queue metrics.
- Define roles anyone can pick up (push manager, ticket duty) with a formal handover, so responsibilities do not stick to individuals.
- Regularly scrub pages and tickets by class to find root causes. If a root cause is fixable within a reasonable time, silence that class of interrupt until the fix is due; the silence is both relief and a deadline.
- Use policy as a tool: require requesters to prepare the change/config and submit it for review instead of doing it for them; push back on support for a low-value flaky component (deprecate, replace, or hand it back). Your team sets the service level it offers.

## 7. Onboarding to on-call (SRE ch. 28)

Recommended vs anti-patterns:

| Do | Don't |
|---|---|
| Structured, cumulative learning path: how a request enters, each hop, where state lives, how it fails | Trial by fire; training via alert/ticket triage |
| Teach reverse engineering and statistical thinking from fundamentals | Train only on checklists and playbooks |
| Curate and share "teachable" postmortems; run postmortem reading sessions | Treat outages as secrets |
| Contained but realistic breakages on a staging/diverted instance under synthetic load; "burn a cluster down" exercises with predictions first, then validation | First hands-on failure only happens once already on-call |
| Weekly disaster role-play ("Wheel of Misfortune"): game master runs a real past or hypothetical incident; primary + secondary talk through actions, GM describes what dashboards show | Knowledge siloed in a few experts |
| Shadow on-call early (copy pages to the trainee during business hours), then reverse-shadow (trainee is primary, senior lurks and validates) | Primary on-call before holistic understanding |
| On-call learning checklist as a social contract; trainee refreshes the most outdated sections, reviewed by the named experts | Static training docs only experts may edit |
| Real, scoped project work with partial ownership from the start | Juniors get the scraps |

Gate on-call readiness with the checklist (plus optionally a final exercise), celebrate it, then keep learning continuous: regular sessions where engineers shepherding changes present what is about to change in the stack, recorded for future trainees. When a trainee co-authors a postmortem, pair them with the on-caller; never dump the write-up on the most junior person, or they learn postmortems are grunt work.
