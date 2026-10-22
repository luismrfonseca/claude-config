# Distributed architectures: fallacies, trade-offs, patterns

Contents: §1 monolith vs distributed · §2 the eight fallacies with counter-measures · §3 other distributed costs · §4 microservices reference architecture, service mesh, governance (SolArch) · §5 event-driven patterns, CDC, async APIs · §6 sync vs async, orchestration vs choreography, sagas.

## 1. Monolith vs distributed (Fundamentals ch. 9)

Monolithic styles (layered, pipeline, microkernel) share one deployment unit; distributed styles (service-based, event-driven, space-based, orchestration-driven SOA, microservices) buy performance, scalability and availability at the price of every problem below. The decision input is the quantum count (modularity.md §4): one quantum → monolith, several with different operational characteristics → distributed.

Big Ball of Mud is the absence of a style: handlers wired straight to the DB, global state, every change ripples unpredictably. It arrives through lack of governance, not intent; the defense is fitness functions (characteristics.md §5), not a rewrite.

## 2. The eight fallacies (Fundamentals ch. 9)

| # | Fallacy | What actually happens | Counter-measure to design in |
|---|---|---|---|
| 1 | The network is reliable | Healthy callee, unreachable; request sent, response lost | Timeouts, retries with idempotency keys, circuit breakers, bulkheads; the more hops per request, the lower the composite reliability |
| 2 | Latency is zero | In-process call: µs. Remote call: ms. Ten chained calls at 100 ms = 1 s added | Know your production round-trip latency, **p95/p99 not average** (avg 60 ms can hide p95 400 ms). Cap synchronous call depth; aggregate at the edge; prefer async for fan-out |
| 3 | Bandwidth is infinite | Stamp coupling: callee returns 500 KB when 200 B are needed, 2 000×/s = 1 Gb/s for one call | Purpose-built endpoints, field selectors / sparse fieldsets, GraphQL, consumer-driven contracts, internal messaging with minimal payloads |
| 4 | The network is secure | Every endpoint, including service-to-service, is attack surface | mTLS / service mesh, authn+authz on internal calls, policy agents; accept the latency cost of per-hop security |
| 5 | Topology never changes | A "minor" 2 a.m. network change invalidates all latency assumptions and trips circuit breakers | Standing channel with ops/network; latency assumptions written down and tested (fitness function) |
| 6 | There is only one administrator | Dozens of admins and teams own parts of the path | Know who to ask about latency and topology; the coordination cost is real and recurring |
| 7 | Transport cost is zero | Not latency: **money**. Gateways, proxies, subnets, firewalls, extra instances, egress | Cost the target topology before committing; distributed is always more expensive to run than the same system as a monolith |
| 8 | The network is homogeneous | Mixed vendors, mixed configs, occasional packet loss | Feeds back into 1–3; design for loss, never for the lab network |

## 3. Other distributed costs (Fundamentals ch. 9)

- **Distributed logging and tracing**: one request crosses dozens of logs with different formats. Correlation ids and a central log/trace stack are table stakes, not optional.
- **Distributed transactions**: ACID is gone across quanta. You get eventual consistency (BASE: basic availability, soft state, eventual consistency). Use sagas (compensating actions, state machine or event-sourced) and accept that consistency windows are a product decision. If a workflow needs atomicity across two candidate services, that is evidence they are one quantum.
- **Contract maintenance and versioning**: decoupled services owned by different teams; deprecation needs a communication model, not just a `v2` prefix. Consumer-driven contract tests catch breakage before deployment.

## 4. Microservices in a brown-field enterprise (SolArch ch. 3)

Most enterprises are brown-field: existing DBs, COTS/SaaS core apps, integration layer, API layer, security, monitoring, infra. Not everything should become a microservice. Which components to split and which to keep central:

| Component | Treatment | Why |
|---|---|---|
| In-house applications | Microservices via **strangler**: put an API gateway in front of the monolith first, move one domain at a time behind it, clients never notice; keep the gateway afterwards | Phased, reversible, no big-bang; needs exec support because it threatens CoE teams |
| Integration flows (Camel/Spring Integration/Synapse-style) | Split into coarse integration runtimes per high-level domain (products runtime; payments+orders runtime talking to ERP/CRM) | Integration services touch several domains, so boundaries are coarser than business services |
| Message broker | **Keep central** (one clustered Kafka/Rabbit/NATS) | It is shared infrastructure; splitting it buys nothing |
| API software | Split the **data plane** (gateways/microgateways) per client type or per service (even as a sidecar next to the service); keep the **control plane** and developer portal central (~1 control plane per ~10 gateways) | Gateways scale with traffic and client variety; control/portal are management concerns |
| Security | External (north-south): API gateway with OAuth2/OIDC. Internal (east-west): policy agent (OPA + Rego + data) running in the same pod as the service, policies pushed by a controller | Service code does not change when policies change; no central authz hop |
| Databases, COTS, SaaS, monitoring, infra | Stay as they are | Not yours to decompose |

Service sizing is relative to the technology (lines of code are a poor measure); what must hold is a clearly defined, relatively small domain that you are allowed to re-cut later (fixed deposits → foreign-currency / senior / local deposits). Start with a workable boundary, iterate; waiting for the perfect DDD cut means never shipping. Two-pizza cross-functional teams, polyglot allowed but governed, deployment automated end to end in containers with an orchestrator, otherwise independent deploy is fiction.

### Service mesh vs message broker for inter-service communication

| | Service mesh (sidecar proxy + control plane) | Message broker (topics + reply topics) |
|---|---|---|
| Communication | Sync HTTP/gRPC/TCP through local proxies; mTLS, routing, LB, retries, discovery, telemetry handled in the proxy | Async publish/subscribe; service is a "smart client" of the broker, knows nothing about other services |
| Operational coupling | Centralized in the control plane, zero code in services | Each service implements broker protocol and delivery semantics |
| Delivery guarantees | Request/response; failure surfaces immediately | At-most/at-least/exactly-once, durable subscriptions, offline consumers catch up |
| Complexity | A proxy per pod, a control plane to run, latency per hop | One central broker cluster; simpler topology |
| Pick when | Request/response workflows dominate, you need uniform security/observability on sync traffic | Temporal decoupling, buffering, fan-out, guaranteed delivery matter more than immediate responses |

Both can coexist: mesh for sync edges, broker for async flows. The mesh is the Fundamentals ch. 17 sidecar idea made concrete.

### Governance for polyglot microservices

Freedom per team must still produce one coherent system. Minimum governance capabilities: reusability (discoverable services), people allocation across teams, **standards** (one API description format, e.g. OAS for REST and AsyncAPI for events, so a common portal is possible), policies, extensibility, integration interfaces, **life-cycle management** (create → implement → test → publish → deprecate, with the core service life cycle in a governance tool and the published API life cycle in the API manager, linked). A dependency graph of core → integration → API services is what lets you change a service without breaking consumers.

## 5. Event-driven patterns, CDC, async APIs (SolArch ch. 5; Fundamentals ch. 14)

Sync communication fails if either side is down at that moment; retries on the client side push complexity to every client. A broker decouples availability and pace; messages become events. Decide per message what its loss costs (a million-euro order vs a bot's spam) before choosing guarantees.

| Pattern | Semantics | Use |
|---|---|---|
| Topic pub/sub | Every subscriber gets every message; broker retains for late subscribers for a window; partitions/replicas/durability per broker | Fan-out, extensibility (new consumers without producer changes), broadcast |
| Shared subscription (durable / non-durable) | A consumer group shares one copy of each message; other, non-shared subscribers still get their own copy; durable keeps messages while the group is offline | Scale one logical consumer horizontally while others still see everything |
| Queue-based pub/sub (competing consumers) | Each message to exactly one subscriber, load balanced; persisted until a subscriber returns | Producer faster than a single consumer; work distribution |
| Store and forward | Message persisted *regardless* of target availability; a processor forwards at a configured rate | Rate mismatch (200 msg/s in, 100 msg/s out), unreliable targets, legacy integration |
| Real-time stream processing | Continuous filtering/cleansing/joining/aggregating on related events before results reach targets | Fraud detection on transactions, IoT stats, threshold alerts (flag orders above X) |
| Batch processing | Collect raw events to storage, process on schedule, expose results via REST/GraphQL | Analytics, monitoring stats, ETL |

Kafka specifics worth remembering when choosing it: ordering only within a partition; consumers ≤ partitions per consumer group; messages are re-readable by offset until retention expires (replay is free); multiple consumer groups each get a full copy; connector API for DB/file sources and sinks; streams API for processing.

### Reference flow for an order system (SolArch ch. 5)

Client → gateway: sync REST for queries (`search products`), async WebSockets for pushes (order status). Gateway → integration/REST API publishes `new-orders` to the broker and returns "accepted" immediately (responsiveness, not performance; see styles.md §5). Order service and an analytics stream job both consume the topic; the order service updates ERP/CRM through the integration layer and publishes `order-updates`; a stream job turns `order-updates` into WebSocket pushes through the gateway. Security (OAuth2/OIDC) and observability are cross-cutting. If the gateway cannot speak the broker's native protocol, the integration platform or stream processor bridges. Data services in the integration layer keep the gateway (and credentials) away from the DB.

### Change data capture

When the same data lives as *source* (orders table) and *derived* copies (search index, cache, warehouse, another service's read model), periodic ETL is too slow for real-time needs and polling is wasteful. CDC = change detection → capture → propagation.

| Detection | Trade-off |
|---|---|
| Poll a `LAST_UPDATED` column | Simple; misses deletes, loads the DB, latency = poll interval |
| DB triggers | Immediate; adds write latency and schema coupling inside the DB |
| Transaction-log mining (Debezium) | Ordered, complete incl. deletes, no app change; needs log access and a connector runtime |

Topologies: DB → CDC connector → topic → sink connector → target (direct); or add a stream processor between topic and sink for transformation; or stream processor with its own CDC connector and no broker at all. For a NestJS service split, CDC on the monolith's tables is the lowest-risk way to feed a new service's read model while the write path is still in the monolith (outbox pattern is the application-level sibling when you own the writes).

### Asynchronous APIs

Server-to-client pushes without a broker in the client path. Describe them with **AsyncAPI** (the OpenAPI of events) so the gateway can apply security, rate limiting and monitoring, and the portal can publish them next to REST APIs.

| Protocol | Shape | Fit |
|---|---|---|
| WebSockets | Bidirectional, HTTP handshake then TCP, subprotocols define payload | Chat, live dashboards, order status |
| WebSub | Publisher → hub → subscriber via HTTP POST callbacks | Server-to-server webhooks with discovery, replaces long polling |
| GraphQL subscriptions | Subscription query over a long-lived WebSocket | Clients already on GraphQL |
| Server-Sent Events | Subscribe-only over plain HTTP, response left open | One-way feeds, simplest option, proxies/CDNs friendly |
| Broker protocols (Kafka, AMQP, STOMP, JMS) | Exposed through a gateway that supports them, else bridged by stream processor/integration layer | Partner/event integrations |

Target end state: one gateway publishing sync APIs (OAS 3) and async APIs (AsyncAPI 2.x) side by side, with the broker, integration platform, stream processor and core services as either kind of backend, CDC feeding the stream processor from the DB.

## 6. Sync vs async, orchestration vs choreography, sagas (Fundamentals ch. 14, 17, 18)

- **Default synchronous; go async only when a characteristic demands it** (burst absorption, decoupling operational characteristics of two quanta, responsiveness for fire-and-forget actions). Async adds eventual consistency, races, deadlocks, debugging pain and error-handling paths that need a way back to the user.
- **Choreography** (broker topology, front-controller when one service ends up coordinating): maximum decoupling, performance and extensibility; no owner of workflow state, weak error handling, no restart. **Orchestration** (mediator, or a dedicated local orchestrator service): workflow control, error handling, recoverability, consistency; more coupling, a component that must scale, lower performance. Most real systems are hybrids: orchestrate the critical business transaction, choreograph the side effects (notifications, analytics, projections).
- **Sagas**: compensations via pending states or do/undo pairs; expensive and complex. A saga or two is a tool; sagas everywhere mean service boundaries cut through a transaction and should be redrawn (modularity.md §4, styles.md §8).
- **Data loss checklist for any async edge**: durable channel + wait for broker ack on publish; manual ack after processing; commit then ack; idempotent consumers; dead-letter/workflow-repair queue; per-key ordering strategy if order matters.
