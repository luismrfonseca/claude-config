# Monitoring and Alerting

Distilled from *Site Reliability Engineering* ch. 1, 6 and 10. Use when designing what to collect, what to page on, what to put on a dashboard, and how to fix a noisy pager.

## Contents
1. Three outputs: page, ticket, log
2. Symptoms vs causes; black-box vs white-box
3. The four golden signals (with AWS/Node mapping)
4. Instrumentation rules: counters, histograms, resolution
5. Alert design: the five questions and the four properties of a page
6. Alert rule mechanics (ratio + volume + duration; SLO burn-rate alerts)
7. Dashboards
8. Fixing a noisy pager: triage procedure
9. Keep it simple: what to delete

---

## 1. Three valid outputs of monitoring (SRE ch. 1, 6)

| Output | Meaning | Channel | Rule |
|---|---|---|---|
| Page | A human must act now to stop or prevent user-visible damage. | Pager (PagerDuty/Opsgenie via SNS). | Software decides that action is needed; the human never interprets raw data to find out. |
| Ticket | A human must act, but days of delay cause no harm. | Ticket/issue queue, reviewed daily. | Disk 80% full with 3 weeks of headroom; cert expires in 20 days; error budget at 60%. |
| Log / dashboard | Nobody reads it unless something else prompts them. | Logs, metrics, traces. | Everything that is neither of the above. |

Email alerts are not a fourth category; they decay into unread noise. Anything you would send by email belongs on a dashboard of "current sub-critical problems" or in a ticket. A system that needs a human to read a message and decide whether it matters is a monitoring bug.

## 2. Symptoms vs causes; black-box vs white-box (SRE ch. 6)

- Symptom = what is broken from the user's point of view (5xx, slow responses, stale data, wrong data). Cause = why (DB refusing connections, CPU pegged, a push dropped ACLs).
- Page on symptoms. Use cause-oriented signals as debugging aids on dashboards. Causes change as the architecture changes; symptoms do not. Cause-based pages fire when users are fine (noise) and miss failures with a new cause (gap).
- "Symptom" is layered: slow DB reads are a symptom for the DB owner and a cause for the API owner. Each team pages on its own symptoms.
- Black-box (probe from outside, as a user) is symptom-oriented and proves an *active* problem: ideal for the most important pages, useless for imminent problems. White-box (internal metrics, logs) detects imminent failures and failures masked by retries, and is essential for debugging ("how slow does the API think the DB is" vs "how slow does the DB think it is" tells you whether the network between them is the problem).
- Probe both the public entry point (DNS + CloudFront/ALB + app) and the individual targets behind the LB: the first catches what users see, the second isolates which hop failed and lets you suppress the per-target alert when the aggregate is fine.
- Dependency-aware alert hierarchies ("if DB is down, suppress API alerts") rarely pay off in systems under continuous change; keep the few that cover very stable facts ("region drained => don't alert on its latency").

## 3. The four golden signals (SRE ch. 6)

If you can only measure four things, measure these; page when one is bad (or, for saturation, about to be).

| Signal | What | Pitfalls | AWS/Node sources |
|---|---|---|---|
| Latency | Time to serve a request, *split by success vs error*. | A fast 500 pulls the average down and hides a latency problem; a slow error is worse than a fast one, so track error latency separately rather than dropping it. | ALB `TargetResponseTime` (p50/p90/p99), app histogram labelled by route + status class, RDS/Dynamo latency metrics. |
| Traffic | Demand, in the unit that matters: rps, concurrent sessions, messages/s, bytes/s. | Averages over long windows hide bursts. | ALB `RequestCount`, SQS `NumberOfMessagesSent`, Kinesis `IncomingRecords`. |
| Errors | Rate of failed requests: explicit (5xx), implicit (200 with wrong content), by policy (slower than the SLO threshold counts as failed). | LB-level 5xx misses wrong-content failures; only end-to-end checks catch those. Business errors returned as 200 are invisible to the ALB. | ALB `HTTPCode_Target_5XX_Count`, `HTTPCode_ELB_5XX_Count` (LB itself failing), app counters by status, synthetic canaries validating payloads. |
| Saturation | How full the most constrained resource is; degradation starts well before 100%, so define a utilization target. Includes forecasts ("disk full in 4 h"). | For Node: event-loop lag, heap, pool exhaustion (pg/redis client pools), not just CPU. | Event-loop lag gauge (`perf_hooks.monitorEventLoopDelay`), `process.memoryUsage`, DB connection pool wait count, RDS `DatabaseConnections`/`FreeableMemory`, SQS queue age, ECS CPU/memory reservation. p99 latency over a 1-minute window is a leading indicator of saturation. |

Supplement saturation with a headroom estimate: "can this service take 2x, 10% more, or less than today?" Get it from load tests, re-validate after major changes.

## 4. Instrumentation rules (SRE ch. 6, 10)

- Prefer counters to gauges for events. A counter sampled every 15 s never loses events that happened between samples; a gauge does. Derive rates at query time (`rate()` in Prometheus; `Sum`/period in CloudWatch).
- Compute sums of rates, not rates of sums, across instances. It survives counter resets on restart and missing scrapes.
- Export latency as histograms with roughly exponential bucket boundaries (e.g. 10, 30, 100, 300, 1000, 3000 ms); it lets you read any percentile and the shape of the tail from cheap counters. Never alert on mean latency: at 1,000 rps and a 100 ms mean, 1% of requests can easily take 5 s, and one backend's p99 becomes the frontend's median once a page fans out to several backends.
- Resolution should match the SLO. Probing a 99.9% service for HTTP 200 more than once or twice a minute buys nothing; sampling CPU per second then aggregating into buckets per minute lets you see short spikes without storing per-second data. Choose scrape intervals by what you need to detect in time, not by what is cheap or default.
- Label by what you need to slice: route/operation, status class, region/AZ, instance, shard/tenant class. Use the same labels for aggregation (service-wide) and isolation (which instance/AZ is sick).
- Alert rules should require the condition to hold for at least two evaluation cycles (`for: 2m` style) so that one missed scrape never pages.

## 5. Alert design (SRE ch. 6)

Before adding or keeping any alert, answer:

1. Does it detect an otherwise undetected condition that is urgent, actionable and (actively or imminently) user-visible? N+0 redundancy and "nearly full" count as imminent.
2. Will there be times I can safely ignore it because I know it is benign? When, why, and how do I stop that case from firing (drained traffic, test deployments, maintenance windows)?
3. Does it definitely mean users are hurt? Which detectable non-harm cases should be filtered out?
4. Can I act on it? Must the action happen now or can it wait until morning (=> ticket)? Could the action be automated (=> then do that, and don't page)? Is the action a real fix or a workaround?
5. Is someone else already paged for the same underlying problem? Then one of the pages is redundant.

Every page must satisfy all four:

| Property | Meaning | Violation smells |
|---|---|---|
| Urgent | You can react with urgency only a few times a day before fatigue. | "I'll look at it in the morning" pages. |
| Actionable | There is something a human can do right now. | Pages whose runbook says "wait and see" or "known issue". |
| Needs intelligence | The response requires judgment. | Pages with a rote response ("restart the pod", "clear the queue"): automate the response and delete the page. Unwillingness to automate such pages is a sign the team doesn't trust itself to pay down debt; escalate it. |
| Novel | It reports a problem not already known/handled. | Duplicate pages for one incident; recurring pages for a known defect that should be in the ticket queue with an owner. |

Corollary: once a page passes these tests, it doesn't matter if it is black-box or white-box. Spend most effort on catching symptoms; of causes, page only on the very definite and very imminent ones (disk filling, cert expiring, loss of redundancy).

Target page volume: at most ~2 incidents per 8-12 h shift (each incident may be several related pages). Above that, incidents stop being investigated properly and the team stops learning. Below ~1 per shift, the rotation is wasted effort. Report incidents-per-shift to management quarterly.

## 6. Alert rule mechanics (SRE ch. 10)

Pattern for an error-ratio page, expressed generically:

```
page when
  errors_ratio_10m > 1%            -- symptom, relative to traffic
  and errors_rate_10m > 1/s        -- absolute floor: don't page on 1 error out of 10 requests at 3 am
  for 2 evaluation cycles          -- anti-flap, tolerate a missed scrape
route by severity label: page | ticket | dashboard-only
```

The absolute floor matters: ratio-only rules page at low traffic on trivial counts; count-only rules miss a 2% error rate during peak.

Prometheus equivalent:

```
sum(rate(http_requests_total{status=~"5.."}[10m])) / sum(rate(http_requests_total[10m])) > 0.01
  and sum(rate(http_requests_total{status=~"5.."}[10m])) > 1
```

CloudWatch equivalent: metric math `m5xx / mTotal > 0.01` combined with `m5xx > 600` per 10 min, `DatapointsToAlarm: 2 of 2`, treat missing data as `notBreaching` for ratio alarms on low-traffic services (and have a separate "no traffic at all" symptom alarm if silence is itself a failure).

SLO burn-rate alerting (the natural extension of the ratio rule when you have an SLO): `burn_rate = error_ratio / (1 - SLO)`. Two tiers:

| Tier | Burn rate | Window | Meaning |
|---|---|---|---|
| Page | ~14.4x | 1 h (confirm with 5 min) | 2% of a 30-day budget gone in an hour; a real outage. |
| Page | ~6x | 6 h (confirm with 30 min) | 5% of budget in 6 h; serious degradation. |
| Ticket | ~1-3x | 1-3 days | Slow bleed; fix within the week. |

Burn-rate alerts page on budget consumption, so the same rule works for latency SLOs (bad = slower than threshold) and availability SLOs. They replace most hand-tuned thresholds.

Alert routing (Alertmanager / PagerDuty rules): deduplicate identical label sets, inhibit per-instance alerts when the service-level alert is firing, group related alerts into one notification. Page-worthy alerts go to the on-call rotation; important-but-not-urgent go to the ticket queue; everything else stays on dashboards.

Keep rule definitions separate from target lists so one rule set applies to every service/environment; put alert rules in version control with tests that replay synthetic series (Prometheus `promtool test rules`).

## 7. Dashboards (SRE ch. 6)

- Top of the primary dashboard: the four golden signals for the service as a whole, plus SLO attainment and remaining error budget.
- Then cause-oriented panels in the order you would debug: upstream dependencies' latency/error as seen from this service, saturation (event-loop lag, pool wait, CPU/mem), recent deploys/config changes as annotations (because ~70% of outages follow a change).
- A "current sub-critical problems" view replaces email alerts: everything in ticket state, visible in one place.
- Nobody should be expected to stare at a dashboard to find problems; dashboards serve debugging and trend analysis, alerts serve detection.

## 8. Fixing a noisy pager

1. Export the last 30-90 days of pages. Group by alert name and by underlying incident. Compute incidents per shift and the fraction of pages with no action taken.
2. For each alert name, apply section 5. Classify: keep as page / demote to ticket / automate the response and delete / delete outright / tighten (add absolute floor, longer `for`, aggregate across instances instead of per-instance).
3. Per-instance resource alerts (CPU, memory, disk on one node) are almost always causes, not symptoms. Replace with service-level symptom alerts (error ratio, latency SLO burn) and keep the resource metrics on the dashboard; keep only cause alerts that are definite and imminent (disk full in < N hours, N+0 redundancy).
4. If the SLO itself is unattainable with the current system, temporarily loosen the alerting target (the book's example: alert on p75 instead of the tail until the storage layer was fixed) and disable the flood, so the team gets time to make the long-term fix. A controlled short-term availability hit for long-term stability is a legitimate trade; sustained heroics are not.
5. Every page that remains must have an owner and either a runbook entry or an automation ticket. Pages with rote responses are a red flag that should be escalated, not tolerated.
6. Cut over in this order: run new alerts in shadow (no paging) and back-test against history → switch paging to the new set and *disable* (not delete) the old alarms → re-measure pages per shift for two rotation cycles and confirm no real incident was missed → only then delete the old alarms. Deleting at cutover throws away the rollback. Review pager stats quarterly with management.
7. CloudWatch specifics (practice, not from book): `PR(:t)` / `TC(:t)` statistics give "fraction of requests under t" on `TargetResponseTime`; metric math has no time-shift/offset function, so week-over-week comparison needs anomaly detection or a Prometheus `offset`; `RejectedConnectionCount` means the ALB hit its max connections, not empty target groups (those produce ELB 503s).

## 9. Keep it simple (SRE ch. 6)

- The rules that catch real incidents most often must be the simplest and most reliable ones in the system.
- Collection, aggregation or alert config that hasn't been exercised in a quarter is a removal candidate.
- Metrics collected but neither on a prebuilt dashboard nor used by an alert are removal candidates.
- Avoid "magic" (auto-learned thresholds, automatic causality). Simple anomaly rules on end-user request rate are the acceptable exception because they detect one specific, severe condition quickly.
- Keep monitoring separate from profiling, crash tracking, log analytics and load testing; integrate through stable, simple interfaces rather than one fragile mega-system.
- The path from symptom to page to triage must be understandable by every engineer on the rotation, not just the one who built it.
