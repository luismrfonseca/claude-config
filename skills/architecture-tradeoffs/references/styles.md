# Architecture styles: topology, scorecard, when to use

Contents: §0 how to read the scorecards · §1 layered · §2 pipeline · §3 microkernel · §4 service-based · §5 event-driven · §6 space-based · §7 orchestration-driven SOA · §8 microservices · §9 choosing a style (decision table).

## 0. How to read the scorecards (Fundamentals Part II)

- ★ = the style actively works against this characteristic; ★★★★★ = one of its defining strengths. Ratings are for the *default* topology; variants shift them (noted per style).
- **Partitioning**: technical (layers, filter types, event processors) vs domain (bounded contexts). Domain partitioning is the one that migrates and maps to teams (modularity.md §5).
- **Quanta**: always 1 for monoliths; "1 to many" for distributed styles, but only *many* when databases and UIs are also split. Sharing one database collapses everything to one quantum regardless of how many services you deploy.
- A style is a named bundle of trade-offs; a *pattern* (CQRS, saga, cache-aside) is a tactic inside a style.

Characteristics abbreviations used below: Deploy = deployability, Elast = elasticity, Evol = evolutionary, FT = fault tolerance, Mod = modularity, Cost = overall cost (more stars = cheaper), Perf = performance, Rel = reliability, Scale = scalability, Simp = simplicity, Test = testability.

## 1. Layered (n-tier) (Fundamentals ch. 10)

Topology: horizontal layers (presentation → business → persistence → database); technical partitioning; 1 quantum. Physical variants: all-in-one; UI separate from business+persistence; everything incl. embedded DB in one unit (on-prem products).

| Deploy | Elast | Evol | FT | Mod | Cost | Perf | Rel | Scale | Simp | Test |
|---|---|---|---|---|---|---|---|---|---|---|
| ★ | ★ | ★ | ★ | ★ | ★★★★★ | ★★ | ★★★ | ★ | ★★★★★ | ★★ |

Key mechanics:
- **Closed vs open layers.** Closed = a request may not skip the layer below. Closed layers on the main request path give *layers of isolation*: swap the UI framework or the persistence tech without touching other layers, provided contracts hold. Opening layers (presentation calling persistence directly, the old "fast-lane reader") makes every layer depend on every lower layer; the app becomes brittle.
- Adding a layer is how you enforce a rule: want to forbid the presentation layer from using business-layer utilities? Move them to a *services* layer below business, mark it open so business can skip it, keep business closed. Document which layers are open/closed and **why**, or the rule evaporates.
- **Architecture sinkhole anti-pattern**: requests pass through layers that add nothing (controller → service → repository → SQL and straight back). Some sinkholes are normal; measure the share. ~20 % is fine; ~80 % means the style is wrong for this domain (or open all layers and accept the coupling).
- Deploy/test are ★–★★ because a three-line change redeploys and re-tests the whole unit, usually bundled with dozens of unrelated changes. FT is ★ because an OOM anywhere kills everything and MTTR (restart 2–15 min) is high.

Use when: small/simple app, tight budget or deadline, team familiarity, or you have not yet decided the real style and need to start coding (keep reuse low and inheritance shallow so you can leave later). Avoid when: the system is growing, testability/deployability/agility are top-3, or the domain is smeared across layers and every feature touches every team.

## 2. Pipeline (pipes and filters) (Fundamentals ch. 11)

Topology: unidirectional point-to-point pipes between stateless single-purpose filters. Filter kinds: producer (source), transformer (map), tester (filter/reduce), consumer (sink). Technical partitioning; 1 quantum.

| Deploy | Elast | Evol | FT | Mod | Cost | Perf | Rel | Scale | Simp | Test |
|---|---|---|---|---|---|---|---|---|---|---|
| ★★ | ★ | ★★★ | ★ | ★★★ | ★★★★★ | ★★ | ★★★ | ★ | ★★★★★ | ★★★ |

Use when: one-way processing (ETL, EDI, telemetry enrichment, log/metric pipelines, integration flows like Camel). Mod/Test beat layered because a filter can be changed or replaced in isolation. Still a monolith: deployment ceremony and shared fate remain. In Node this is a stream/`pipeline()` of small stages or a chain of worker steps; keep payloads small, filters stateless.

## 3. Microkernel (plug-in) (Fundamentals ch. 12)

Topology: minimal **core system** (the happy path / minimal runnable product) plus independent **plug-in components** holding volatile, custom or client-specific logic; a **registry** (map, config, Consul/ZooKeeper) that tells the core what plug-ins exist and how to reach them; a standard **contract** per plug-in domain (adapter when a third-party plug-in dictates its own). Core can itself be layered (technical) or modular (domain). Only style that is both domain- and technically partitioned. 1 quantum, even when plug-ins are remote: every request passes through the core.

| Deploy | Elast | Evol | FT | Mod | Cost | Perf | Rel | Scale | Simp | Test |
|---|---|---|---|---|---|---|---|---|---|---|
| ★★★ | ★ | ★★★ | ★ | ★★★ | ★★★★★ | ★★★ | ★★★ | ★ | ★★★★ | ★★★ |

Rules:
- Plug-ins are independent of each other; no plug-in-to-plug-in dependencies.
- Plug-ins do not touch the shared DB; the core passes them data. A plug-in may own a private store (rules, config).
- Compile-time plug-ins (packages/namespaces like `app.plugin.<domain>.<context>`) are simple but redeploy the monolith; runtime plug-ins (OSGi-style, dynamic import, separate processes) can be hot-swapped at the cost of framework complexity. Remote plug-ins (REST/messaging) give decoupling and async but turn the style into a distributed one with all fallacies, and an unreachable plug-in now fails the request.
- Move `if (type === 'x') ... else if ...` chains with high cyclomatic complexity out of the core and into one plug-in per case; the core becomes `registry.get(type).run()`.

Use when: product-style software (IDEs, CI servers, browsers), per-jurisdiction/per-tenant/per-client rules (insurance claims per state, tax forms, device assessment rules), configurable SaaS with customer-specific behavior. Avoid when: scalability/elasticity/FT are top-3 or the "plug-ins" actually need to call each other.

## 4. Service-based (Fundamentals ch. 13)

Topology: separately deployed UI, 4–12 coarse-grained **domain services** (avg ~7), one shared monolithic database. Services are deployed like small monoliths (containers optional); UI usually calls services directly or via gateway. Domain partitioning; quanta 1 to many (only >1 when DB and UI are federated).

| Deploy | Elast | Evol | FT | Mod | Cost | Perf | Rel | Scale | Simp | Test |
|---|---|---|---|---|---|---|---|---|---|---|
| ★★★★ | ★★ | ★★★ | ★★★★ | ★★★★ | ★★★★ | ★★★ | ★★★★ | ★★★ | ★★★ | ★★★★ |

Variants: split the UI per domain or per service; split the DB per domain (only if no other service needs that data — never introduce inter-service calls to compensate); add an API gateway for cross-cutting concerns or external exposure. Each domain service is internally layered (API façade + business + persistence) or domain-partitioned; the façade orchestrates the business request *in-process*, so one request = one ACID transaction (checkout with expired card simply rolls back). That is the defining difference from microservices, where the same request spans several services and needs sagas/BASE.

Shared-database discipline (the main risk): never one `entities` library for the whole schema (any table change redeploys every service). Partition the schema logically by domain and ship one entity library per partition (`invoicing_entities`, `order_entities`, …) so a change touches only the services that use that partition. Keep a small `common` partition locked to a data team. Make partitions as fine as still makes a coherent data domain.

Use when: you want most of the benefits of distribution (independent deploy, fault isolation, team ownership, ACID preserved) without the cost; DDD with coarse bounded contexts; modularity matters but you want to avoid granularity/orchestration problems; "a Ferrari for the commute" is a waste. Avoid when: you need elastic fine-grained scaling of individual capabilities or services truly need independent data.

**For a NestJS monolith under pressure this is usually the first real target**: extract 4–8 coarse domain services that keep the shared Postgres, no inter-service calls, each one still a plain Nest app.

## 5. Event-driven (Fundamentals ch. 14)

Request-based model (orchestrator directs synchronous processors; deterministic, data-driven: "get my order history") vs event-based model (system reacts to things that happened: "bid placed"). EDA is a standalone style or a hybrid ingredient (event-driven microservices, event-driven microkernel). Technical partitioning (a domain is spread across processors, queues, mediators); quanta 1 to many — shared DB or request-reply between processors collapses them into one quantum.

| Deploy | Elast | Evol | FT | Mod | Cost | Perf | Rel | Scale | Simp | Test |
|---|---|---|---|---|---|---|---|---|---|---|
| ★★★ | ★★★ | ★★★★★ | ★★★★★ | ★★★★ | ★★★ | ★★★★★ | ★★★ | ★★★★★ | ★ | ★★ |

### Broker topology (choreography)

Initiating event → broker (topics, pub/sub, usually federated per domain) → a processor acts and **always publishes what it did** as a processing event, even if nobody listens today (that is the extensibility hook: a future "email analytics" processor subscribes with zero changes elsewhere). Relay race: once handed off, the processor is free.

| Advantages | Disadvantages |
|---|---|
| Highly decoupled processors, high scale, responsiveness, performance, fault tolerance; topics provide back-pressure | No one knows when the business transaction is done; no workflow control; if `Payment` crashes nobody notices and `Inventory` keeps decrementing; no recoverability/restart (no owner of the state); data inconsistency |

### Mediator topology (orchestration)

Initiating event → initiating queue → **mediator** that knows the steps, sends *commands* (not events) to point-to-point queues, waits for acks, runs steps serially and their sub-tasks in parallel, persists workflow state, and can stop/resume (payment declined → pause at step 3, resume when payment arrives). Usually several mediators per domain to avoid a SPOF. Classify events simple / hard / complex and route from a code-level mediator (Camel-class, or in Node: a state machine/Temporal/Step Functions) to a BPEL/BPM engine only when human-in-the-loop or very dynamic branching really exists.

| Advantages | Disadvantages |
|---|---|
| Workflow control, error handling, recoverability, restart, better consistency | More coupling, mediator must scale too (bottleneck), lower performance/FT/scalability, hard to model truly dynamic flows (hybrids are common) |

Commands must be processed; events may be ignored. Choosing: workflow control + error handling (mediator) vs raw performance + scale + decoupling (broker).

### Mechanics worth remembering

- **Responsiveness ≠ performance.** Async ack in 25 ms instead of 3 100 ms sync changes nothing about the 3 000 ms of work; it changes what the user waits for. Only valid when the user needs nothing back except an acknowledgment, and you have a channel to report later failures.
- **Workflow event pattern** for async error handling: consumer delegates the bad message to a workflow processor and moves on (keeps throughput); the processor repairs programmatically and resubmits, or parks it on a human dashboard queue. Consequence: repaired messages are out of order; if ordering matters per key (account), park subsequent messages for that key until the repaired one is processed.
- **Three data-loss points and fixes**: producer → broker (persistent queues + synchronous send, i.e. wait for broker persistence ack); broker → consumer crash (client-acknowledge mode, not auto-ack); consumer → DB (ACID commit, then ack: "last participant support"). In SQS/Rabbit/Kafka terms: durable topics, manual ack after commit, idempotent consumers.
- **Request-reply** over messaging: request queue + reply queue; prefer correlation id (message selector on reply queue) over temporary reply queues, which hammer the broker at volume.
- **Broadcast** is the highest form of decoupling (publisher knows nothing about subscribers); foundation of eventual consistency and CEP.

| Choose event-based when | Choose request-based when |
|---|---|
| Dynamic user actions, high responsiveness and scale, extensibility, real-time reaction | Well-structured data-driven requests, certainty and control over workflow, strong consistency, easy testing |
| Trade-offs: eventual consistency only, less control and certainty of outcome, hard to test/debug ("event tree" explosion) | |

Testability ★★ and simplicity ★ are the honest cost: nondeterministic flows produce hundreds of scenarios nobody can enumerate. Add EDA to any style to remove bottlenecks and add back-pressure, not as a default.

## 6. Space-based (Fundamentals ch. 15)

Problem it solves: in web → app → DB topologies the bottleneck migrates downward until it hits the database, which is the hardest and most expensive tier to scale. Space-based removes the DB as a synchronous constraint: all transactional data lives in replicated in-memory grids inside **processing units**; writes go to the DB asynchronously. Domain and technically partitioned; quanta determined by UI ↔ processing-unit association (the DB is not in the quantum, since nobody calls it synchronously).

Components: processing unit (app code + in-memory grid + replication engine: Hazelcast/Ignite/Coherence, or Redis-backed equivalents); virtualized middleware = messaging grid (request routing/session, HAProxy/nginx), data grid (replication), processing grid (optional orchestration across unit types), deployment manager (starts/stops units on load — the elasticity engine); data pumps (async messaging per domain/cache, persistent FIFO queues, update-only payloads with action); data writers (per domain or per pump); data readers (used only on cold start of all instances, redeploy, or archive reads; load via reverse data pump with one temporary cache owner holding a lock). Readers/writers form a data *abstraction* layer, so grid schema ≠ table schema and DB changes can be buffered.

| Deploy | Elast | Evol | FT | Mod | Cost | Perf | Rel | Scale | Simp | Test |
|---|---|---|---|---|---|---|---|---|---|---|
| ★★★ | ★★★★★ | ★★★ | ★★★ | ★★★ | ★★ | ★★★★★ | ★★★★ | ★★★★★ | ★ | ★ |

Data collisions in active/active replicated caches: `rate ≈ N × (UR² / S) × RL` (N instances, UR updates/s, S rows, RL replication latency). Example: 5 instances, 20 upd/s, 50 k rows, 100 ms → ~14 collisions/hour out of 72 000 updates (0.02 %). Fewer instances, lower latency or bigger cache lowers it; small hot caches with many writers collide. Compute min/normal/peak rates before trusting replication; measure RL in production, 100 ms is the planning default.

| Criterion | Replicated cache | Distributed cache |
|---|---|---|
| Optimizes | Performance, FT (no SPOF) | Consistency (single copy) |
| Size | < 100 MB | > 500 MB |
| Data | Static-ish reference data (product codes, profiles) | Highly dynamic (inventory counts) |
| Update rate | Low | High |
| Cost of a down cache server | None | Units stop working (mirror → consistency risk) |

Use both in the same system per data type. Near-cache (front cache + backing cache) is not recommended: front caches drift between units and make latency inconsistent.

Use when: > ~10 k concurrent users with unpredictable spikes (ticket sales, auctions, flash sales); hybrid cloud (units in cloud, DB on-prem). Avoid when: strong consistency on hot data is required, testing at real scale is impossible for you (Test ★ because load can only be verified in production), or the licensing/infra budget is thin (Cost ★★).

## 7. Orchestration-driven SOA (Fundamentals ch. 16)

Historical style; read it as a catalogue of what not to do. Taxonomy: business services (named by the business, no code: `ExecuteTrade`), enterprise services (fine-grained reusable implementations: `CreateCustomer`), application services (one-off), infrastructure services (logging, auth), all stitched by an **orchestration engine / ESB** that owns workflow, transformation and transaction boundaries, over one or a few shared DBs. Technical partitioning pushed to the extreme; 1 quantum (shared DB + every request goes through the engine, so nothing can have different characteristics than the mediator).

| Deploy | Elast | Evol | FT | Mod | Cost | Perf | Rel | Scale | Simp | Test |
|---|---|---|---|---|---|---|---|---|---|---|
| ★ | ★★★ | ★ | ★★★ | ★★★★ | ★ | ★★ | ★★ | ★★★ | ★ | ★ |

Lessons that still apply: reuse = coupling (one canonical `Customer` forces driver's-license fields onto the disability-insurance team and makes every change ripple to all consumers); domain concepts ground to dust across tiers ("add an address line" touches dozens of services); the integration team owning the engine becomes a political and deployment bottleneck (Conway); distributed transactions chosen declaratively never find the right granularity. If your "microservices" have a central ESB-like orchestrator with business logic, a shared canonical model library and one DB, you built this.

## 8. Microservices (Fundamentals ch. 17)

Philosophy: physical bounded contexts. Each service owns its code, data schema and database; nothing couples to another context's classes or tables. **Prefer duplication to coupling** (copy the `Address` shape rather than share a package). Domain partitioning; the most quanta of any style. "Micro" is a label contrasting with SOA's giant services, not an instruction to make them tiny.

| Deploy | Elast | Evol | FT | Mod | Cost | Perf | Rel | Scale | Simp | Test |
|---|---|---|---|---|---|---|---|---|---|---|
| ★★★★★ | ★★★★★ | ★★★★★ | ★★★★ | ★★★★★ | ★ | ★★ | ★★★ | ★★★★★ | ★ | ★★★★ |

Granularity heuristics (iterate; nobody gets it right first pass):
- **Purpose**: one significant, functionally cohesive behavior per service.
- **Transactions**: entities that must change atomically belong in one service. "Don't do transactions across services — fix granularity instead." A few sagas are tolerable; if sagas are the dominant feature, the boundaries are wrong.
- **Choreography**: services that must talk constantly to get anything done should be merged back.
- Beware the entity trap (one service per table). Decide per fact where the single source of truth lives, or replicate/cache deliberately.

Structure:
- API layer is optional and must stay dumb (routing, discovery, auth, rate limiting). Orchestration logic in the gateway violates the style; mediators belong to technically partitioned architectures.
- Operational reuse via **sidecar** + **service mesh**: logging, metrics, circuit breakers, mTLS, discovery live in a sidecar owned by a platform team; the mesh gives one console. Domain reuse stays forbidden; operational reuse is encouraged.
- Service discovery is how elasticity is implemented; often hosted in the API layer/mesh.
- Frontends: monolithic UI over the API layer (common) or microfrontends (UI component per service, full-stack team ownership).
- Communication is *protocol-aware heterogeneous interoperability*: pick and standardize protocols (REST level, gRPC, queues), accept polyglot stacks (one pioneer forced different stacks per team so classes could not be shared by accident).
- Choreography (broker EDA) is the natural fit; when a workflow needs coordination, either a *front controller* (first service also mediates — extra complexity in that service) or a dedicated local orchestrator service (coupling concentrated in one place). Performance usually favors choreography.
- Sagas: a mediator calls participants, tracks success/failure, and on failure issues compensations (participants hold `pending` state, or each op ships `do`/`undo`; undo is typically harder than do). Expensive in coordination traffic and in reasoning about requests that arrive while state is pending.

Use when: several quanta with truly different operational characteristics, fast-moving domain with many teams, elasticity/evolvability/deployability in the top 3, and mature automation (CI/CD, provisioning, observability) already exists. Avoid when: the domain is semantically highly coupled (multi-page forms whose steps depend on each other), the org lacks DevOps maturity, performance is a top-3 driver with chatty workflows, or the team is small — the cost is ★ for a reason.

## 9. Choosing a style (Fundamentals ch. 18)

"It depends" is the honest answer, but the inputs are known. Walk in with: the domain (enough to spot operational implications), the characteristics that affect structure, the data architecture (existing schemas, who owns data), organizational factors (cloud cost, M&A plans → favor open/integration-friendly designs), process/team/ops maturity (a style that assumes CD on a team without CI will fail), and **domain/architecture isomorphism** (does the problem's shape match the topology? customization → microkernel; many discrete parallel computations → space-based; highly coupled multi-step workflow → service-based or modular monolith, *not* microservices).

Then make three determinations, in order:

1. **Monolith or distributed?** One quantum → monolith (modular, domain-partitioned; keep tables grouped by domain so a later split stays possible). Several quanta → distributed.
2. **Where does data live?** Monolith: one or a few relational DBs. Distributed: which services persist what, and how data flows to complete workflows.
3. **Sync or async between services?** Sync is simpler to design, implement and debug; async buys performance, scale and decoupling at the cost of data sync, races, deadlocks and debugging pain. **Default to synchronous; go asynchronous only where a specific characteristic demands it** (e.g. a slow `Payment` that would time out under burst, or two services whose operational characteristics must differ).

Outputs: a topology (style + hybrids), ADRs for the decisions that cost the most thought, and fitness functions guarding the characteristics that drove the choice.

Decision table (driver → first candidates):

| Top-3 characteristics look like | Reach for | Reject |
|---|---|---|
| Simplicity, cost, time-to-first-release, small team | Layered or modular monolith | Anything distributed |
| Customizability per tenant/jurisdiction/client | Microkernel (core + plug-ins), possibly inside a monolith | Microservices "per customer" |
| One-way data processing | Pipeline | Event mediator for linear flows |
| Agility, testability, deployability, ACID still required, 4–12 domains | Service-based | Microservices (granularity tax), SOA |
| Performance, scalability, FT, extensibility, async-tolerant domain | Event-driven (broker), or hybrid EDA inside another style | Pure request/response |
| Extreme elasticity, unpredictable spikes, > 10 k concurrent | Space-based | Scaling the RDBMS |
| Elasticity, evolvability, deployability, many teams, different per-part characteristics | Microservices with EDA for cross-service flows | Shared DB, central orchestrator |
| Strong consistency across the whole workflow, highly coupled domain | Modular monolith or service-based | Microservices, space-based |

Hybrids are normal (event-driven microservices, microkernel core with service-based deployment); pick the base style by the quantum analysis and add EDA where back-pressure or responsiveness is needed.
