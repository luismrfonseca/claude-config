# SLIs, SLOs, SLAs and Error Budgets

Distilled from *Site Reliability Engineering* ch. 1, 3, 4 and Appendix A. Use when deciding how reliable a service should be, how to measure it, and how to spend the resulting error budget.

## Contents
1. Terminology and the "what happens if we miss it" test
2. Choosing the reliability target (risk tolerance)
3. Availability math and the nines table
4. Picking SLIs per service type
5. Measuring SLIs well (percentiles, windows, where to measure)
6. Writing SLOs (template, rules of thumb)
7. Error budgets and the error budget policy
8. Anti-patterns

---

## 1. Terminology (SRE ch. 4)

| Term | What it is | Test |
|---|---|---|
| SLI | A quantitative measure of one aspect of service level (latency, error ratio, throughput, availability, durability, freshness, correctness). Usually an aggregate over a window: ratio or percentile. | "Can I compute this number from telemetry every minute?" |
| SLO | A target for an SLI: `SLI <= target` or `lower <= SLI <= upper`, over a stated window, under stated conditions. | "If we miss it, do we change priorities?" If not, drop it. |
| SLA | SLO(s) plus explicit consequences (credits, penalties, contractual remedies). Owned by business/legal; SRE's job is to make the SLIs measurable and say how likely the SLO is to hold. | "Is there a consequence written down?" No consequence => it is an SLO, not an SLA. |
| Error budget | `1 - SLO target`, expressed as allowed bad events (or bad minutes) per window. The quantity dev and ops jointly agree to spend. | |

Most people who say "SLA breach" mean "missed SLO". Keep the vocabulary straight; it determines who owns the conversation.

## 2. Choosing the reliability target (SRE ch. 1, 3)

Reliability is a product decision with a cost curve, not an engineering maximum. Each extra nine typically costs ~10x more (redundancy + the opportunity cost of engineers not shipping features) while users stop being able to perceive the difference once you are more reliable than the path between them and you (device, Wi-Fi, ISP, mobile network). 100% is the wrong target for everything except safety-critical hardware.

Questions to answer with the product owner before setting a number:

| Dimension | Questions |
|---|---|
| User expectation | What do users expect? Paid or free? Consumer or enterprise? What do competitors offer? |
| Revenue link | Is a failed request lost money (checkout, ads) or a mild annoyance (background poll)? |
| Failure shape | Is a constant 0.1% error trickle or a rare full outage worse for this business? Are there failure modes (data leak, wrong charge) where taking the service down is the right move? |
| Alternatives | What do unhappy users do: wait, retry, churn to a competitor? |
| Cost | What does one more nine cost in infra and in features not built? Does it pay back? |
| Other dimensions | Is latency, freshness or throughput more important than availability for this service? Infra services with mixed clients may need two tiers (low-latency vs. throughput) rather than one over-engineered one. |

Back-of-envelope value of a nine: `value = annual revenue through the service x (new_target - old_target)`. If the engineering + infra cost of the extra nine exceeds that, do not buy it. When revenue is not request-linked, use the background error rate of the internet (roughly 0.01%-1% per the book) as a floor: errors below the user's own connection noise are invisible.

Target as both minimum and maximum: aim to meet the SLO, not to exceed it by a lot. Over-delivering trains clients to depend on reliability you never promised (Google deliberately injected outages into an over-reliable lock service to flush out such dependencies). Translation: if you promise 99.9% and run at 99.999%, either tighten the promise or spend the slack on velocity.

Recommended split for a service with an external SLA: external SLA target (e.g. 99.9%) backed by a tighter internal SLO (e.g. 99.95%) so you notice chronic problems before customers can invoke penalties.

## 3. Availability math (SRE ch. 3, App. A)

Two definitions; pick request-based unless the service is literally up or down:

- Time-based: `availability = uptime / (uptime + downtime)`
- Request-based (yield): `availability = successful requests / total requests` over a rolling window. Works for batch and pipelines too (records processed OK / records attempted) and copes with partial outages.

| Target | Downtime / year | / quarter | / 30 days | / day | Allowed failures per 1M requests |
|---|---|---|---|---|---|
| 99% | 3.65 d | 21.9 h | 7.2 h | 14.4 min | 10,000 |
| 99.5% | 1.83 d | 10.9 h | 3.6 h | 7.2 min | 5,000 |
| 99.9% | 8.76 h | 2.19 h | 43.2 min | 1.44 min | 1,000 |
| 99.95% | 4.38 h | 1.09 h | 21.6 min | 43 s | 500 |
| 99.99% | 52.6 min | 13.1 min | 4.3 min | 8.6 s | 100 |
| 99.999% | 5.26 min | 1.31 min | 26 s | 0.86 s | 10 |

Rule: if the human response chain (page -> ack -> diagnose -> mitigate) takes 30 minutes, a 99.99% monthly target leaves you one incident per month. That is the point where you must engineer for automatic failover rather than faster humans. Below 99.9% you can usually get away with a good on-call rotation and rollbacks.

Worked example: 2.5M requests/day at 99.99% => 250 failed requests/day allowed. A single bad deploy returning 500s for 60 s at 30 rps burns 1,800 => a week of budget.

## 4. Picking SLIs per service type (SRE ch. 4)

Start from what users care about, then find a measurable proxy. Do not start from what is easy to collect. A handful (2-4) of SLIs per service is enough; more dilutes attention.

| Service type | Primary SLIs | Node/AWS measurement notes |
|---|---|---|
| Request/response (API, web, BFF) | Availability (ratio of good responses), latency (percentiles of good responses), throughput (as capacity signal, not an SLO you control) | ALB `HTTPCode_Target_5XX_Count / RequestCount`, `TargetResponseTime` percentiles (p50/p90/p99 via CloudWatch extended statistics). App-level histogram (prom-client `Histogram`) is better: ALB can't see application-level errors returned as 200. |
| Storage / DB-backed service | Latency of reads/writes, availability, durability (data still there), consistency/freshness if advertised | RDS/DynamoDB read/write latency + error rates; durability via backup verification jobs, not metrics. |
| Pipeline / batch / async worker | Throughput (items/s), end-to-end freshness (age of oldest unprocessed item), correctness (ratio of records processed without error) | SQS `ApproximateAgeOfOldestMessage`; Kinesis `IteratorAgeMilliseconds`; custom gauge "lag between event time and processed time". |
| All services | Correctness: right answer returned. Often a property of the data, owned by product, but track it. | Canary queries with known answers; reconciliation jobs. |

Decide, per SLI, which events count. Examples of explicit exclusions: health-check traffic, requests rejected as malformed (4xx caused by the client), traffic from load tests. Write these into the SLI definition so nobody argues during an incident.

## 5. Measuring SLIs well (SRE ch. 4)

- Prefer distributions over means. A 50 ms median with a 1 s p95 is a different product from a flat 60 ms. Alert and set SLOs on high percentiles (p99, p99.9) for latency; users prefer slightly slower but consistent.
- Do not assume normality: latency is bounded at 0 and clipped by timeouts; mean and median diverge wildly. Any automation that acts on "outliers" (e.g. restarting slow instances) must be tuned on the real distribution.
- Watch the aggregation window. 200 rps for one second and 0 the next averages to 100 rps but the instantaneous load (and queueing) is 2x. Use 1-minute rates at most for alerting SLIs; shorter for capacity analysis.
- Measure where the user is when you can. Server-side latency misses slow front-end JS, DNS, TLS and ALB queueing. For APIs, the ALB/CloudFront edge is closer to the user than the Node process; a synthetic probe (CloudWatch Synthetics, a Lambda cron hitting the public URL) is closer still.
- Standardize SLI templates so each SLO only states the deltas: measurement interval (e.g. 1 min), scope (all tasks in the region), sampling frequency, which requests count, where measured (server/ALB/client), latency definition (time to last byte).

Latency SLI definition that avoids the classic trap: `good = requests with status < 500 AND duration < threshold`; `SLI = good / total`. Measuring p99 of *all* requests lets fast 500s mask a latency problem and slow timeouts hide an availability problem.

## 6. Writing SLOs (SRE ch. 4)

Template:

```
<percentage>% of <event type> [over <window>] will <condition>, measured at <where>,
excluding <exclusions>.  Valid when <conditions>.
```

Examples (fully specified, then abbreviated once defaults are agreed):

- 99.9% of HTTP requests to `/api/*`, measured at the ALB over a rolling 30 days, return a non-5xx status, excluding health checks.
- 99% of `GET /orders/:id` requests complete in < 300 ms; 99.9% in < 1 s (two targets describe the curve shape).
- 95% of bulk-import jobs' records are processed within 10 min of enqueue; 99.9% within 1 h.
- Separate objectives for distinct workloads: interactive clients get a latency SLO; batch clients get a throughput SLO.

Rules of thumb:

| Rule | Why |
|---|---|
| Don't set the target from current performance | You lock in whatever heroics currently hold it up and can't change the architecture without "breaking" the SLO. Set it from user need, then measure the gap. |
| Keep the SLI simple | Complex aggregations hide regressions and nobody can explain them during an incident. |
| No absolutes ("always", "infinite scale") | Unachievable targets produce expensive, conservative systems and are ignored in practice. |
| As few SLOs as possible | Keep only those you would quote to win a prioritization argument. If an SLO never changes a decision, delete it. |
| Start loose, tighten later | Loosening a published SLO disappoints users; tightening one is free. |
| Publish them | Published SLOs stop both over-reliance (clients assume 100%) and under-reliance (clients avoid you for being "flaky"). |
| Keep a safety margin | Internal SLO stricter than the external one; it buys time to fix chronic drift. |

The SLO control loop: measure SLI -> compare with SLO -> decide if action is needed -> decide *what* action (hypothesis: "CPU-bound, add instances") -> act. Without the SLO you cannot answer "is this bad enough to act on, and when?".

## 7. Error budgets and the error budget policy (SRE ch. 1, 3)

Budget per window: `budget = (1 - SLO) x expected events in window`. Example: SLO 99.9% over 30 days, 50M requests => 50,000 failed requests allowed. Consumption is reported as a percentage: an incident failing 0.0002% of quarterly requests against a 0.001% budget consumed 20% of the quarter's budget.

The budget is the contract between feature velocity and reliability, with monitoring as neutral referee. It must be defined jointly (product owner sets SLO; monitoring measures; the gap is the budget) and the enforcement side must have real authority, otherwise it is theatre.

Error budget policy template (one page, agreed before the first breach):

| Budget remaining (window) | Release policy | Engineering priority |
|---|---|---|
| > 50% | Normal cadence; canaries and progressive rollouts as usual. | Feature work. |
| 10-50% | Releases continue, but risky launches require canary + extended bake; no "Friday pushes". | Allocate time to top reliability items from recent postmortems. |
| < 10% or burn rate trending to exhaust before window end | Feature freeze except reliability fixes and reverts; every change needs explicit owner sign-off. | Reliability work is top priority for the dev team, not just ops. |
| Exhausted | Freeze until budget recovers; postmortem action items blocking. Product may choose to loosen the SLO instead -- that is a legitimate outcome if the velocity loss hurts more than the unreliability. | |

Additional rules worth writing down:
- Infra-caused outages (AZ loss, upstream provider) also consume the budget. Everyone shares the number; it keeps the incentive to build resilience against them.
- Prefer graded responses (slow releases, longer canaries, auto-rollback thresholds) over a pure on/off freeze; bang-bang control causes release pile-ups that are themselves risky.
- Report burn weekly to the team, monthly/quarterly to management. Do not keep it as an SRE-only number.
- Spend the budget deliberately: 1% experiments, phased rollouts, planned chaos tests are legitimate spends. "Zero outages" is not the goal; unspent budget at window end means you were too conservative or the SLO is too loose.

Burn-rate view (useful for alerting; see monitoring reference): `burn_rate = observed error ratio / (1 - SLO)`. Burn rate 1 exhausts the budget exactly at window end; 14.4 exhausts a 30-day budget in ~2 days.

## 8. Anti-patterns

- SLO on a metric you cannot control (inbound QPS). You can set a capacity target for it, not an SLO.
- Averaged latency SLO. Tail latency is where users suffer and where queueing shows up first.
- 100% / "no errors" SLOs. Nobody can meet them, so they are ignored and lose their power as a prioritization lever.
- Dozens of SLOs. If nobody can recite them, they do not drive decisions.
- SLO defined but no budget policy. The first breach becomes a political fight; the point of the budget is to have settled that fight in advance.
- Overachieving silently. Clients will build on the real behaviour; you will own that behaviour forever.
