# Event-Driven Architecture Patterns (SolArch ch. 5)

Synthesis of chapter 5. Vendor-neutral except for Kafka, which the chapter treats as the reference broker. Audience: senior backend engineer deciding sync vs async, designing Kafka-based flows, CDC and async APIs.

## Contents

1. [Sync vs async: the decision](#1-sync-vs-async-the-decision)
2. [Asynchronous messaging models](#2-asynchronous-messaging-models)
3. [Stream processing models](#3-stream-processing-models)
4. [Pattern: Event-driven enterprise reference architecture](#4-pattern-event-driven-enterprise-reference-architecture)
5. [Pattern: Kafka-based EDA (order processing walk-through)](#5-pattern-kafka-based-eda)
6. [Pattern: Change Data Capture](#6-pattern-change-data-capture)
7. [Pattern: Asynchronous APIs](#7-pattern-asynchronous-apis)
8. [Improved EDA: sync + async on one gateway](#8-improved-eda-sync--async-on-one-gateway)
9. [Checklists](#9-checklists)

---

## 1. Sync vs async: the decision

Start from the **value of the message** and the **availability coupling** you can tolerate.

| Question | Sync request/response | Async via broker |
|---|---|---|
| Both sides must be up at the same instant? | Yes — failure if either is down | No — broker buffers |
| Message loss acceptable? | Only with client-side retries (complex clients) | Durable broker gives delivery guarantee cheaply |
| Caller needs the answer to proceed (search, lookup, validation)? | Yes | No — caller gets "accepted", result arrives later |
| Producer and consumer run at different rates? | Back-pressure hits the caller | Broker absorbs the rate difference |
| Need fan-out to N consumers? | N calls or an orchestrator | One publish, N subscriptions |

**Rule:** sync for read/lookup and anything the user waits on; async for state-changing commands whose processing is slow, fan-out, or where loss is unacceptable. Mixed systems are normal — the book's reference architecture keeps both.

**Retry on the client is not a delivery guarantee strategy.** If the message is worth money, put it in a durable broker before acknowledging the client.

---

## 2. Asynchronous messaging models

| Model | Delivery semantics | Use for | Notes |
|---|---|---|---|
| **Topic pub/sub** | Every subscriber gets every message | Broadcast of domain events to independent consumers | Broker retains messages for a configurable window so late subscribers can catch up |
| **Shared subscription** (durable / non-durable) | Within a shared group, one consumer gets each message; non-shared subscribers still get all | Mix broadcast with work-sharing on the same topic | Durable = messages held while group is offline; Kafka consumer group is this model |
| **Queue pub/sub** | Each message to exactly one of N competing consumers | Producer faster than a single consumer; load-balanced workers | Persist while consumers are down, for a bounded time |
| **Store and forward** | Message persisted first, forwarded later by a processor at a configured rate | Target system has a hard throughput ceiling or is often unavailable; rate adaptation (e.g. 200 msg/s in, 100 msg/s out) | Lives in the integration platform; the primary design goal is "target may be down", unlike topic/queue where it's an edge case |

Choosing: broadcast → topic; work distribution → queue or consumer group; rate mismatch with a fragile target → store-and-forward.

AWS mapping and two facts that are commonly misstated (practice, not from book): SQS standard = queue pub/sub; SNS→SQS fan-out = topic pub/sub; Kafka/MSK consumer group = shared subscription. SQS FIFO ordering is per *message group*, but the throughput quota (300 msg/s, or 3,000 with batching; higher in high-throughput mode) is per *queue*, not per group. A message group also does not give you a single active consumer: with several consumers polling, one group's messages are delivered to one consumer at a time, but nothing stops a different consumer taking the next batch — for a true single writer toward a legacy target use a leader lock (e.g. Postgres advisory lock) or one worker.

---

## 3. Stream processing models

| Model | Shape | Examples | Output |
|---|---|---|---|
| **Real-time stream processing** | Continuous related events → filter, cleanse, aggregate, join, window → act | Fraud detection on transactions; IoT min/max/avg to a controller; threshold alerts | DB tables, notifications, messages to other systems, WebSocket push |
| **Batch processing** | Event receiver stores raw events → batch processor computes → processed store → targets pull via REST/GraphQL | Gateway analytics; nightly ETL; ML training | Processed data store |

Rule: if the decision must happen while the event is still "hot" (seconds), stream; if consumers pull at their leisure, batch.

---

## 4. Pattern: Event-driven enterprise reference architecture

**Problem (book's e-commerce framing):** guaranteed delivery from order to delivery, scalability, availability, low latency for users, and analytics on purchasing patterns — all while keeping simple reads synchronous.

```
  Mobile │ Web │ Terminal │ 3rd-party devs
         └──────┬─────────┘      │
          API Gateway ──── Developer Portal
     ┌─────────┼─────────────┐
     ▼         ▼             ▼
 Stream    Message      Integration
 processor  Broker       Platform
     │  ▲      │  ▲          │
     │  └──────┘  └──────────┤
     ▼                       ▼
 ┌────────────────────────────────────┐
 │ Core apps: in-house µservices │ legacy │ SaaS │
 └────────────────────────────────────┘
                     Database(s)
 cross-cutting: Security (IAM) · Observability
```

**Each link and what it carries (the book defines the architecture by its edges):**

| Edge | Style | Carries |
|---|---|---|
| Client ↔ Gateway | Sync HTTP + async (WebSocket/SSE) | Reads/lookups sync; order-status pushes async — avoids client polling |
| Gateway → Integration platform | Sync HTTP | CRUD; also mediates to broker when gateway lacks native broker protocol |
| Gateway → Broker | Async | Commands turned into events (e.g. order placed), ack to client immediately |
| Gateway ← Stream processor | Async (WebSocket) | Stream processor listens to topics/CDC/files and exposes a push-compatible endpoint |
| Broker ↔ Integration platform | Both directions | Platform publishes received requests; consumes to drive legacy/SaaS updates |
| Broker ↔ Stream processor | Both | Consume for analysis; publish results back for fan-out |
| DB → Stream processor | CDC | React to data changes without polling; also store processing results |
| DB ↔ Broker | Persistence | Broker's durable storage (file or DB-backed) |
| DB ↔ Integration platform | Data services | Gateway must never hit DB directly; data services hide credentials and schema |
| Core apps ↔ Broker | Direct where possible | Microservices subscribe natively; legacy/COTS go through the integration platform |
| Core apps ← Stream processor | Push | Alerts (fraud, failures, high-value orders) to the app that owns the reaction |

**Component responsibilities:**

| Component | Owns |
|---|---|
| API gateway | Entry point for sync and async APIs; policies; protocol bridging if it supports broker protocols |
| Message broker | Durability, ordering within partition, fan-out, replay |
| Integration platform | Protocol/format translation to legacy & SaaS; data services; store-and-forward; broker adapter for things that can't speak to it |
| Stream processor | Real-time analysis; CDC ingestion; bridging topics → WebSocket/SSE |
| Core apps | Source of truth and business logic |

**Fits when:** write-heavy flows where loss is unacceptable; multi-step processing (order lifecycle); many consumers of the same fact; clients need live updates.

**Avoid when:** mostly read/query traffic; small system with one consumer per command (a plain sync API with idempotent retries is simpler); team lacks capacity to operate a broker.

**Trade-offs:** eventual consistency surfaces in UX (must design "accepted" states); duplicate delivery must be handled (idempotent consumers — `(practice, not from book)`); more moving parts to observe (correlate IDs across gateway → topic → consumer; see `observability-stack.md`).

---

## 5. Pattern: Kafka-based EDA

**Kafka facts the pattern relies on:** partitioned, replicated topics across nodes; retention window with consumer-controlled offsets (replay, re-read); ordering guaranteed only within a partition; consumer groups = queue semantics, and **consumers in a group ≤ partitions**; multiple groups each get a full copy; four APIs — Producer, Consumer, Connect (DB/file/system connectors), Streams.

**Order-processing walk-through (book's example, vendor names removed):**

```
 clients ──POST /orders──► Gateway ──► Integration platform: "order publishing" REST API
                                              │ produce
                                              ▼
                                      topic: new-orders
                                 ┌────────────┴─────────────┐
                                 ▼                          ▼
                   Order-management µservice        Order-analysis (stream processor)
                   (consumer group A)               (consumer group B)
                       │ via integration platform       │ stats → DB (dashboards)
                       ▼ "order processing" REST API    │ high-value alert → order-mgmt
                   ERP + CRM (SaaS) updates
                       │ status changes (packed, shipped, delivered)
                       ▼
                   topic: order-updates
                       │
                       ▼
                   Order-updates app (stream processor) ──► WebSocket server
                                                               │
                   Gateway "order updates" WebSocket API ◄─────┘ ──► clients
```

**Design notes:**
- Client gets `202 Accepted` the moment the event is durably produced; everything else is downstream.
- Two consumer groups on the same topic: business processing and analytics never contend.
- Legacy/SaaS updates go through the integration platform's REST API, not from the consumer directly, so the adapter logic lives in one place.
- Status feedback is its own topic → push channel; don't make clients poll the order API.
- Partition key = order ID (or customer ID) so an order's events stay ordered — `(practice, not from book)`.
- Cross-cutting: IAM issues the tokens the gateway validates on both REST and WebSocket APIs; observability stack captures gateway, broker and consumer logs/metrics.

**Capacity:** scale by partitions (parallelism ceiling for a group) and by gateway replicas; size retention for replay needs, not just buffering.

---

## 6. Pattern: Change Data Capture

**Problem:** the DB is the source of truth; derived copies (search index, cache, warehouse, other services' read models) drift. Periodic ETL is too slow for indexes/caches; polling is wasteful.

**Three stages:** change detection → change capture → change propagation.

| Detection method | Pros | Cons |
|---|---|---|
| Poll `LAST_UPDATED` column | No DB privileges beyond read; works anywhere | Misses deletes; latency = poll interval; load on DB; needs the column |
| DB triggers | Catches all DML incl. deletes | Writes in the hot path; schema coupling; DBA-hostile |
| Transaction-log mining (e.g. Debezium-style connector) | Near-zero impact on source; complete and ordered | Needs log access/replication privileges; per-engine connector |

Default to log mining; fall back to polling when you can't get log access (`(practice, not from book)`).

**Topologies:**

```
 A) Direct:        DB ──log──► CDC connector ──► topic ──► sink connector ──► warehouse/index
 B) Transforming:  DB ──log──► CDC connector ──► topic ──► stream processor (transform) ──► sink
 C) Broker-less:   DB ──log──► stream processor with built-in CDC connector ──transform──► sink
```

Pick A for 1:1 replication; B when multiple consumers or transformation/enrichment; C when you have a single target and don't want to operate a broker for it. Use the broker whenever more than one target will ever want the change stream.

**Fits when:** you need near-real-time derived data; the source system can't be changed to emit events (legacy, COTS, vendor DB); outbox-style reliable event publishing (`(practice, not from book)`: CDC on an outbox table is the standard way to get exactly-once-ish domain events out of a transactional service).

**Avoid when:** the source system already emits well-formed domain events; batch staleness is acceptable (plain ETL is simpler); the schema changes constantly (CDC events expose row shape — put a transformation stage in or you leak schema to consumers).

**Trade-offs:** row-level events, not domain events (consumers see tables); schema evolution must be handled in the propagation stage; initial snapshot of a large table needs planning; DBA buy-in for log access.

---

## 7. Pattern: Asynchronous APIs

**Problem:** clients (web, mobile, partners) need server-initiated updates (chat, live scores, order status, tickers) and the event-driven core must be reachable from outside the enterprise network with gateway-grade QoS.

**Spec:** AsyncAPI (the OpenAPI equivalent for event-driven interfaces). Publish async APIs in the same portal as REST ones.

**Client-facing protocols:**

| Protocol | Direction | Transport | Pick when | Caveats |
|---|---|---|---|---|
| **WebSocket** | Bidirectional | HTTP upgrade → TCP; subprotocols define payload | Interactive/bidirectional (chat, collaborative UIs); broker bridge for mobile/web | No intermediary; connection state on the server; LB stickiness |
| **SSE** | Server → client only | Plain HTTP, response kept open | One-way feeds (status, tickers, notifications); simplest to proxy | Text only; browsers cap concurrent connections per host |
| **GraphQL subscriptions** | Server → client | WebSocket under a GraphQL schema | Already GraphQL; typed, field-selected push | Same operational profile as WebSocket |
| **WebSub (webhooks via hub)** | Publisher → hub → subscriber `POST` | HTTP callbacks | Server-to-server / partner notifications; no long-lived connections | Subscriber must expose a public endpoint; delivery retries, signature verification needed |
| **Broker protocols** (Kafka, AMQP, MQTT, STOMP, JMS) | Varies | Native | Internal or trusted consumers that can speak the broker | Usually not exposed to browsers; gateway may not support natively → bridge |

```
  AsyncAPI client
        │
   API Gateway (async API: authn, rate limit, monitoring)
        │
   ┌────┴─────────┬───────────┬──────────────┬───────────────┐
 WebSocket      WebSub       SSE         GraphQL         Message broker
 backend        hub          backend     subscriptions   (Kafka/AMQP/...)
                                                               │ if gateway can't speak it
                                                        Stream processor / integration
                                                        platform bridges to WebSocket/SSE
```

**Decision:**
- Browser/mobile, one-way updates → SSE (or WebSocket if you already have it).
- Bidirectional → WebSocket.
- Partner/system callbacks → webhooks (WebSub-style hub if many subscribers); sign payloads, retry with backoff, provide replay (`(practice, not from book)`).
- Consumer inside your trust boundary and able to run a client → give them the topic directly.
- Gateway must support both OAS and AsyncAPI, or you'll end up with an unmanaged side door.

---

## 8. Improved EDA: sync + async on one gateway

Final shape of the chapter: one gateway publishing **synchronous APIs (OAS 3)** and **asynchronous APIs (AsyncAPI 2.x)** to one developer portal; backends (broker, integration platform, stream processor, core apps) expose either style; DB → stream processor via CDC; Security and Observability as shared services.

```
 Mobile │ Web │ Terminal │ 3rd-party
            API Gateway ──── Portal (sync + async catalogue)
     Sync APIs (OAS3)   │   Async APIs (AsyncAPI)
   ┌────────────────────┼──────────────────────┐
 Stream processor   Message broker   Integration platform
   ▲ CDC                                   │
   DB ◄────────────── Core apps (sync/async interfaces) ──┘
   Security (IAM)  ·  Observability
```

Use this as the target picture when the user's platform must offer REST + live updates + reliable background processing.

---

## 9. Checklists

### Choosing the integration style

| If... | Then... |
|---|---|
| Caller needs the result to continue | Sync API (REST/GraphQL/gRPC) |
| Command is slow or must never be lost | Publish to durable topic, ack `202`, push result later |
| Many consumers of one fact | Topic pub/sub; separate consumer groups |
| Target slower than source or flaky | Queue/consumer group for scale-out; store-and-forward for rate-limited or often-down targets |
| Source is a DB you can't change | CDC (log-based) → topic |
| Need near-real-time derived views (index, cache, warehouse) | CDC → (transform) → sink |
| Decision must happen on the event stream | Stream processor (windows, joins) |
| Client needs live updates | Async API: SSE/WebSocket via gateway; webhooks for systems |

### EDA design checklist

- [ ] Every state-changing flow has a durable produce before the client ack.
- [ ] Partition key chosen per aggregate for ordering; partition count ≥ max consumer parallelism needed.
- [ ] One consumer group per independent concern (processing vs analytics vs audit).
- [ ] Consumers idempotent; dedupe key defined (`(practice, not from book)`).
- [ ] Retention set for replay/backfill scenarios, not just buffering.
- [ ] Legacy/SaaS side effects go through the integration platform, not raw consumers.
- [ ] Status feedback channel defined (topic → push API); no client polling.
- [ ] Async APIs described with AsyncAPI and published in the portal; gateway enforces authn/rate limits on them.
- [ ] CDC: detection method chosen, schema-change handling in propagation, initial snapshot plan.
- [ ] Correlation ID flows gateway → event headers → consumer logs/traces.
