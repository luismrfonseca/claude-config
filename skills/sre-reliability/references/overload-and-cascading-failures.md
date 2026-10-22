# Overload and Cascading Failures

Distilled from *Site Reliability Engineering* ch. 21 and 22. Use when designing rate limits, retries, timeouts, load shedding and degraded modes, when reviewing a service for cascade risk, and when a service is currently crash-looping under load.

## Contents
1. Principles
2. Measuring capacity and load
3. Per-customer quotas
4. Client-side adaptive throttling (formula)
5. Criticality tiers
6. Server-side overload protection and load shedding
7. Retries: budgets and rules
8. Deadlines, propagation, cancellation, bimodal latency
9. Cascade mechanics: resource exhaustion chains, slow start, cold caches, intra-layer calls
10. Prevention checklist (priority order)
11. Testing for cascades
12. Immediate steps during a cascading failure
13. Node.js / AWS translation table

---

## 1. Principles (SRE ch. 21, 22)

- Some part of the system will eventually be overloaded regardless of load balancing. The goal is not "never overloaded" but "an overloaded instance keeps doing useful work at roughly its provisioned rate, with modest latency, no matter how much excess is thrown at it (target: up to 2x-10x), and never crashes".
- An overloaded backend should not stop accepting traffic; it should accept what it can process and reject the rest cheaply and quickly, so the client can retry elsewhere or degrade.
- A cascading failure is positive feedback: one instance fails from overload, its share lands on the others, they fail. Everything below is about breaking that loop. Beware that changes which improve the steady state (retries, caches, evicting unhealthy instances, failover proxies) often widen the cascade path; evaluate each one for "what does this do when everything is already on fire".

## 2. Measuring capacity and load (SRE ch. 21)

Requests per second is a poor capacity unit: cost per request varies by client, endpoint, time of day and software version, and the ratio shifts without warning. Model capacity in resources (CPU-seconds/s, memory) and request *cost* as normalized CPU time. CPU alone is usually sufficient because memory pressure in GC runtimes shows up as CPU, and other resources can be over-provisioned relative to CPU.

For Node: the equivalent of "active threads vs. cores" is event-loop utilization/lag plus the depth of pending work (in-flight requests, pool wait queues). Use `perf_hooks.monitorEventLoopDelay` / `performance.eventLoopUtilization()` smoothed with exponential decay as the per-instance utilization signal; start rejecting as it stays above threshold, so short fan-out bursts do not trigger rejection but sustained saturation does.

Also count connection load: many idle clients doing health checks or a batch job opening thousands of connections at once can cost more than the requests. Mitigate with connection reuse (keep-alive agents), lower health-check frequency, or a batch proxy tier that absorbs batch clients and funnels controlled traffic to the real backend (acts as a fuse).

## 3. Per-customer quotas (SRE ch. 21)

When a shared backend is globally overloaded, only the misbehaving client(s) should see errors. Provision from negotiated usage, set per-client limits in resource units (e.g. "client A: 4,000 CPU-s/s"), and allow the sum of limits to exceed capacity on the assumption that not all clients peak together. Enforce by aggregating usage across instances and pushing effective per-instance limits down (or approximate with a shared token-bucket in Redis/ElastiCache keyed by client ID, API key or tenant).

Rejecting out-of-quota requests must be cheaper than serving them; where it is not (a cheap in-memory lookup), rejection alone cannot save the backend and clients must throttle themselves (section 4).

## 4. Client-side adaptive throttling (SRE ch. 21)

Each client keeps a 2-minute sliding window of:
- `requests` = attempts made by the application layer
- `accepts` = requests the backend accepted (not rejected with an overload/quota error)

Reject locally, before the network, with probability:

```
P(reject) = max(0, (requests - K * accepts) / (requests + 1))
```

- K = 2 (default): the client sends until it is attempting twice what the backend accepts; the backend wastes ~1 rejection per accepted request, but recovery propagates to clients fast.
- K = 1.1: ~1 rejection per 10 accepts; use when rejecting costs almost as much as serving.
- Locally rejected requests still count in `requests`, so the drop probability keeps rising while the app keeps hammering. This is intended.
- Keep separate windows per criticality. Does not work for clients that send very sporadically (window too empty to judge).
- Decision uses only local state: no extra dependency, no latency.

Node implementation sketch: wrap the HTTP client (axios/undici interceptor) with a ring-buffer counter per upstream; treat 429/503 with an `overloaded` marker as "not accepted"; throw a local `ThrottledError` with `P(reject)`; expose `requests`, `accepts`, local rejections as metrics.

## 5. Criticality tiers (SRE ch. 21)

Four values suffice; more adds operational cost without adding decision power.

| Tier | Meaning | Default for | Provisioning expectation |
|---|---|---|---|
| CRITICAL_PLUS | Failure has serious user-visible impact | Checkout, auth, core write paths | Fully provisioned, rejected last |
| CRITICAL | Default for production serving traffic; user-visible but less severe | Interactive reads | Fully provisioned |
| SHEDDABLE_PLUS | Partial unavailability expected; retry minutes/hours later | Batch, async workers, prefetch | Not provisioned for peak |
| SHEDDABLE | Frequent partial and occasional full unavailability OK | Typeahead, analytics, recommendations | Best effort |

Rules: an overloaded instance rejects lower tiers first; per-customer quotas can be set per tier; a backend only rejects tier N once it is rejecting all of N-1 and below. Criticality is orthogonal to latency sensitivity (typeahead is sheddable but latency-critical). Set it as close to the user as possible (BFF/API gateway) and propagate it automatically on every outgoing call (a header such as `x-criticality`), overriding only at specific points (e.g. a background refresh inside a critical request is sheddable).

## 6. Server-side overload protection and load shedding (SRE ch. 22)

Queue management: with steady traffic, queues should be short relative to worker capacity (<= 50%) so the server rejects early instead of accumulating latency; a queue 10x the pool depth with 100 ms service time turns a request into 1.1 s mostly spent waiting. Queueless-with-failover is a valid design. Bursty traffic may justify a queue sized from current concurrency, service time and burst shape. Prefer LIFO or CoDel-style dropping over FIFO under overload: the oldest queued request is the one the user has already abandoned.

Load shedding = drop a fraction of traffic as overload approaches, to stay below the memory/latency/health-check cliff while still doing maximal useful work. Simplest effective rule: cap in-flight requests per instance and return 503 beyond the cap. Richer: shed by criticality, by client, by endpoint cost.

Graceful degradation = do less work per request: search a cached subset, use a cheaper ranking, omit images/recommendations/personalization, serve stale data from cache.

Design questions to answer before building either: which signal triggers it (CPU, event-loop lag, in-flight count, queue depth, latency) and is entry/exit automatic; what exactly changes in degraded mode; at which layer (one choke point at the edge is often enough; per-layer is for shared services). Then: degraded mode should trigger rarely (capacity planning failure or load shift), keep it simple, exercise it continuously (run a small slice of instances near overload, or a scheduled canary) because unused code paths rot, alert when too many instances are degraded, and have a kill switch/tunable that does not itself depend on the overloaded path.

Separate process health ("is the binary alive?", for the scheduler/ECS) from service health ("can it serve this class of request now?", for the load balancer). An ALB health check that fails under load causes the LB to pull capacity exactly when it is most needed, and ECS replacing "unhealthy" tasks under overload turns a slowdown into a crash loop.

## 7. Retries (SRE ch. 21, 22)

How a retry storm forms: backend limit 10,000 rps; client sends 10,100; 100 are rejected and retried next second on top of new traffic => 10,200, 200 rejected => ... until the backend spends most of its capacity rejecting, crashes, and the load redistributes to the survivors. Reducing offered load back below the original limit often does *not* recover it, because rejections cost resources and retries keep the pressure on. Browser JS, mobile sync protocols and queue consumers all do this, not just service-to-service RPC.

Rules:

| Rule | Value / detail |
|---|---|
| Per-request attempt budget | Max 3 attempts; if three overloaded instances in a row rejected it, the fleet is probably overloaded. |
| Per-client retry budget | Retries may be at most ~10% of the client's recent requests. Worst-case amplification drops from ~3x to ~1.1x. |
| Per-process retry budget (alternative) | e.g. 60 retries/min per process; above that, fail fast. |
| Backoff | Always randomized exponential backoff with jitter; synchronized retries after a blip amplify themselves. |
| Retry only at one layer | Retry at the layer directly above the failing one. With 3 retries at each of JS, frontend, backend, one user action can hit the DB 4^3 = 64 times. Deeper layers return "overloaded, do not retry" (or a degraded response) instead of letting callers retry. |
| Attempt counter in metadata | Clients send attempt number; backends keep a histogram; when the histogram shows many retries, they return the non-retriable overload error, because other instances are evidently overloaded too. |
| Distinguish error classes | Retriable transient overload (`503` + `Retry-After`, gRPC `UNAVAILABLE`), non-retriable overload ("don't retry"), permanent errors (4xx, malformed, validation). Never retry the last two. |
| When to retry immediately | Only a small subset of instances is overloaded (LB imperfection, an expensive request landed on one task) -- an immediate retry lands on a different instance and acts as organic load balancing. When most instances are overloaded, do not retry; bubble the error up. |
| Observability | Graph retry rate and attempt histograms; during an incident they look like a symptom but are often the amplifier. |

Mitigation during an incident caused by retries requires either shipping a client fix, cutting load drastically, or blocking traffic entirely until retries stop.

## 8. Deadlines and latency (SRE ch. 22)

- Always set a deadline. No deadline or a very long one lets a problem that has already resolved keep holding resources until restart.
- Short deadlines consistently fail expensive-but-legitimate requests; long ones let lower-layer trouble consume higher-layer resources. Deadlines several orders of magnitude above typical latency are almost always wrong.
- Servers must not work on requests whose client deadline has already passed: check remaining time before each expensive stage (parse, backend call, post-process). Exception: long catch-up jobs that checkpoint; check after each checkpoint.
- Deadline propagation: set one absolute deadline at the top; each hop passes the remaining time, minus a few hundred ms for network and client post-processing. Node: carry `deadlineAt` in AsyncLocalStorage, derive per-call `AbortSignal.timeout(remaining)`; forward as `grpc-timeout`-style header or `x-deadline-ms`. Also cap outgoing deadlines to non-critical backends.
- Cancellation propagation: when the top-level request is abandoned or a critical sub-call fails fatally, cancel the rest of the call tree (abort signals forwarded to all in-flight downstream calls; `request.on('close')` -> `controller.abort()`), so doomed work stops. Needed for hedged requests too.
- Fail fast on known-unavailable backends (circuit breaker open, DNS failure) rather than waiting out the deadline.
- Bimodal latency trap: 1,000 worker capacity, 1,000 rps at 100 ms uses 100 workers. If 5% of requests hang to a 100 s deadline, those 50 rps x 100 s = 5,000 workers' worth of wait; capacity collapses to ~20% and ~80% of *all* requests fail, not 5%. Detect by looking at the latency distribution, not the mean; fix with fail-fast, sane deadlines, and per-keyspace/per-client concurrency caps (e.g. no client may hold more than 25% of workers).

## 9. Cascade mechanics (SRE ch. 22)

Resource exhaustion chains feed each other, so the first symptom you see is rarely the root:

| Exhausted | Downstream effects |
|---|---|
| CPU | Slower requests -> more in flight -> more memory, more open sockets/FDs, more concurrent backend calls; queues saturate; health checks miss; missed client deadlines => wasted work + retries; cache efficiency drops. |
| Memory | OOM kills (ECS task restart); GC death spiral (GC eats CPU, slower requests, more memory, more GC) -- Node heap pressure behaves the same; shrinking app caches => more backend calls => backend overload. |
| Threads/connections/FDs | Pool exhaustion, inability to open new connections, failed health checks. |

Example chain from the book: badly tuned GC -> CPU exhausted under *expected* load -> slower requests -> more RAM -> less cache -> lower hit rate -> more backend calls -> backend out of CPU -> health checks fail -> cascade. Nobody diagnoses that during the outage, especially across two teams.

Crash-loop arithmetic: healthy at 10,000 rps, cascaded at 11,000; dropping to 9,000 will not recover it because only the fraction of instances that have finished starting and survived are serving. If 10% are healthy, load must drop to ~1,000 rps to stabilize. Load balancers that exclude instances that served errors, and schedulers that restart "unhealthy" tasks, shrink the healthy pool further.

Slow start and cold caches: new instances are slower (connection setup, JIT warm-up) and empty caches make every request expensive. Distinguish a latency cache (service survives expected load with it empty) from a capacity cache (it cannot); a cache added "for performance" silently becomes a hard dependency. Mitigations: provision for cold-cache load or make the cache explicitly a capacity cache with a plan; ramp traffic into new capacity gradually; keep all regions/AZs warm with nominal load; test large simultaneous restarts.

Intra-layer communication (backend proxying to another backend, primary forwarding to a hot standby under load) creates distributed deadlock risk (shared pool waits on a peer whose pool is full) and load-dependent amplification (more proxying exactly when overloaded). Keep the request path strictly downward; have the backend tell the client to retry against the right instance rather than proxying.

Triggers to check first in any cascade: process deaths (query of death), rollouts that take many instances out at once (push off-peak, budget capacity for the rollout, scale rollout concurrency by current load), new binaries/config changing request cost or profile, organic growth without capacity change, planned drains/maintenance of your own or a dependency's capacity, traffic-mix shifts (a frontend moved traffic to your region), and loss of overcommitted slack (noisy neighbour on shared hosts; load-test within committed limits only).

## 10. Prevention checklist (SRE ch. 22, priority order)

1. Load test to the breaking point and beyond, and observe the failure mode (section 11). Everything else depends on knowing the cliff.
2. Serve degraded results under pressure (service-specific).
3. Instrument servers to reject when overloaded: in-flight cap, utilization signal, criticality-aware.
4. Rate-limit above the servers: at the reverse proxy/edge (per-IP, abusive clients: WAF/API Gateway throttles), at the load balancer (global overload: indiscriminate or selective drop -- drop background sync, keep interactive), per instance (against LB fluctuation). Simple rate limits waste capacity and cannot stop a cascade already in motion; they are the outer fence, not the protection.
5. Capacity planning coupled with performance testing: if each cell breaks at 5,000 rps and peak is 19,000, N+2 means six cells. Autoscaling helps but does not replace it; it is too slow for a cascade that develops in minutes.
6. Retries per section 7; deadlines per section 8.
7. No intra-layer calls on the request path.
8. Caches classified as latency vs capacity; cold-start plan.
9. Health checks split into process vs service, with service checks that do not fail merely because the instance is busy.
10. Change logging for every deploy/config/flag change so "what changed" is answerable in one query.

## 11. Testing for cascades (SRE ch. 22)

- Load each component separately until it breaks; the well-designed one rejects a few requests and keeps its throughput, the fragile one collapses or errors massively. Record the breaking point for capacity planning and regression tests.
- Test gradual ramps and impulses (caching makes them differ). Test the return to nominal load after overload: does degraded mode exit on its own? How far must load drop for crashed instances to stabilize?
- For stateful/cached services, track state across interactions and check correctness under load; concurrency bugs surface there.
- In a small production slice, with spare capacity ready for manual failover: shrink task counts fast and slow, remove an AZ's worth of capacity, blackhole a backend.
- Test popular clients: can they queue while you are down, do they use jittered exponential backoff, can an external event (app update clearing caches) make them all hit you at once? Stage failures with your largest internal clients.
- Test non-critical backends being unavailable *and* blackholing (never responding): the frontend must not exhaust resources waiting on a "non-critical" dependency with a long deadline.

## 12. Immediate steps during a cascading failure (SRE ch. 22)

Run your incident protocol, then in roughly this order:

1. Check recent changes; revert anything that touched capacity or request profile.
2. Add resources if idle capacity exists (scale out ECS/ASG, raise task memory). May not suffice once in a death spiral.
3. Stop health-check-driven kills: temporarily relax ECS/ALB health checks so starting instances are not killed while warming, and busy ones are not pulled from the target group.
4. Restart wedged servers (GC spiral, deadlocks, deadline-less requests pinning resources) -- only after identifying the source, slowly and canaried; if the cause is cold caches, restarts make it worse.
5. Drop less important traffic first (sheddable tiers, prefetch, batch), then if the fleet is crash-looping: fix or paper over the trigger (capacity/memory), cut traffic aggressively (e.g. allow 1%), let most instances become healthy, ramp back gradually so caches warm and connections establish. If the trigger is not addressed, the cascade returns as soon as traffic does.
6. Enter degraded modes (must be pre-built).
7. Turn off batch/background load that shares the serving path (index rebuilds, exports, stats jobs, cron consumers).
8. Block bad traffic (queries of death, one abusive tenant) at the edge.

## 13. Node.js / AWS translation table

| Book concept | Practical equivalent |
|---|---|
| Per-task utilization signal | Event-loop lag/utilization + in-flight request gauge per process; ECS CPU/memory as coarse backup. |
| In-flight cap / 503 | Middleware counting active requests; `503` + `Retry-After` beyond cap; `server.maxConnections`; keep the health endpoint outside the cap but make the *service* health check reflect degraded state. |
| Per-customer quota | Token bucket in Redis keyed by tenant/API key, limits per criticality; API Gateway usage plans for external clients. |
| Edge rate limiting | WAF rate rules, CloudFront, API Gateway throttles, ALB (no native rate limit: do it at the app or a sidecar). |
| Criticality propagation | `x-criticality` header via AsyncLocalStorage; SQS consumers default to SHEDDABLE_PLUS; interactive BFF sets CRITICAL. |
| Deadline propagation | `x-deadline-ms` absolute epoch header; `AbortSignal.timeout(remaining - 200)` on undici/axios/pg queries; pg `statement_timeout` derived from remaining. Gotchas (practice, not from book): `SET LOCAL` only works inside a transaction (outside one it is a no-op) and does not accept bind parameters, so use `SELECT set_config('statement_timeout', $1, true)` inside `BEGIN … COMMIT`, or pass `statement_timeout` per pool/connection; `AbortSignal` does not cancel a running pg query by itself — on abort call `pg_cancel_backend(pid)` or rely on the statement timeout. Detect client disconnect with `res.on('close', () => { if (!res.writableFinished) abort() })` — on Node ≥ 16 `req.on('close')` fires as soon as the request body is consumed, so it would abort every POST. |
| ALB/ECS gotchas | ALB `slow_start` is **not supported** with the `least_outstanding_requests` algorithm — pick one (slow start + round robin, or LOR without slow start). Listener rules have no "disabled" state; pre-build the traffic valve as a weighted forward (or fixed-response rule at low priority) you edit, not toggle. In-flight counters: decrement exactly once (guard flag), since `finish` and `close` both fire on a normal response (practice, not from book). |
| Cancellation | `req.on('close')` -> abort controller -> propagated to downstream fetches and DB queries. |
| Retry budget | Token-bucket per upstream in the HTTP client wrapper: refill 10% of successful requests, retry only when a token is available; jittered exponential backoff; honour non-retriable overload marker. |
| Process vs service health | ECS/K8s liveness = process responds; ALB target health = service can serve now (and is not shedding everything). Different endpoints. |
| Latency vs capacity cache | ElastiCache fronting RDS: if RDS cannot take the full read load with cache empty, it is a capacity cache -- provision read replicas or warm before shifting traffic. |
| Batch proxy / fuse | Separate target group / service for batch and internal bulk clients with its own concurrency cap. |
| Load test to breaking point | k6/artillery against a prod-like environment with real-shaped traffic, impulse + ramp profiles, measure throughput of *successful* requests past the knee. |
