---
name: enterprise-integration
description: Vendor-neutral patterns for connecting systems in an enterprise, from "Solution Architecture Patterns for Enterprise" ch. 4-7. Covers API gateways and API-led layering (system/process/experience), exposing legacy/ERP systems (strangler facade, adapters for HL7/EDI), sync API vs messaging vs events vs CDC, Kafka event-driven flows, change data capture, async APIs (WebSocket/SSE/webhooks), centralized IAM, SSO and federation, API security (API keys vs OAuth2) and service-to-service security (mTLS, JWT, policy agents), and choosing the observability stack (ELK+Jaeger vs Grafana+Loki+Tempo). Use whenever the user asks how to connect system A to B, integrate with an ERP/CRM/hospital or partner, "we need SSO", "which tracing stack", "event or API call?", "how do we modernize this legacy backend", or anything about gateways, brokers, IAM topology or platform telemetry, even without saying "architecture". For code changes inside domain-service's integration module (CDC sync, ERP) use erp-integration.
---

# Enterprise Integration Patterns

Vendor-neutral synthesis of SolArch ch. 4 (API management, hybrid integration), ch. 5 (event-driven), ch. 6 (security), ch. 7 (observability). Audience: senior backend engineers. Give decisions, components, data flows, trade-offs; skip fundamentals.

## How to use

1. Classify the question with the decision workflows below (integration style → API layering → security → observability). Most real questions touch two or three.
2. Open the matching reference file(s); apply the pattern to the user's stack, not a tour of the book.
3. Answer shape for a design question: component diagram (ASCII/Mermaid) for their case → data flow per edge → fits / avoid / trade-offs → checklist items they still have to decide.
4. The `(SolArch ch. N)` citations and `(practice, not from book)` tags inside the reference files exist so *you* can tell book-backed pattern from added practice while reasoning. They are not for the deliverable: a design doc that cites a book the user never mentioned reads as noise and undermines trust. Keep provenance out of the output unless the user asks where a rule comes from.
5. Separate what the user told you from what you infer. Things like "nothing is idempotent today", "the SaaS is rate-limited", "the ERP pushes SIU messages" are assumptions until stated — write them as such ("Assumption: …, verify with …"), never as observed facts about their system.
6. Node.js client choices go stale fast; flag maintenance status when you name one. At time of writing KafkaJS is effectively unmaintained — prefer `@confluentinc/kafka-javascript` (librdkafka-based) for new work and say why (practice, not from book).

## Pattern catalogue

| Pattern (ref) | Problem it solves | Key components | Pick when | Avoid when |
|---|---|---|---|---|
| **API management platform** (api-management §2) | Many channels/consumers need one stable, governed interface | Gateway, developer portal, API security, control plane, insights, (monetization) | >1 client channel, external consumers, need rate limits/analytics | Single team, internal calls only — service auth + catalogue suffices |
| **API-led connectivity** (api-management §3) | Heterogeneous estate, proprietary protocols, teams blocking each other | Experience / Process / System API layers; integration platform at the bottom boundary | Multi-channel, many systems you don't own, industry standards (HL7, EDI, ISO 8583) | Few owned services, one channel — collapse layers |
| **SOA governance + API mgmt** (api-management §4) | Service lifecycle, contracts and dependencies ungoverned in a brownfield | Service governance tool/catalogue + API control plane, shared pipeline | Many teams, regulated, long-lived contracts | Small org — PR review + catalogue |
| **Legacy modernization / strangler facade** (api-management §5) | Must add channels and refactor without disrupting existing clients | Gateway as facade, modern integration platform, centralized IAM, containers last | Stable legacy that must keep serving | Legacy small enough to rewrite behind a flag |
| **Event-driven reference architecture** (event-driven §4) | Loss-free, scalable, decoupled processing with live client updates | Gateway, broker, integration platform, stream processor, core apps, IAM, observability | Write-heavy flows, fan-out, slow multi-step processing | Mostly reads; one consumer per command |
| **Kafka-based EDA** (event-driven §5) | Concrete realization: order placement decoupled from processing | Topics per lifecycle stage, consumer groups per concern, push channel back to clients | Thousands of commands/min, replay needed | No ops capacity for a broker |
| **Change data capture** (event-driven §6) | Derived data (index, cache, warehouse, other services) drifts; source can't emit events | Log-based detector → CDC connector → topic → (transform) → sink | Source DB you can't change; sub-minute freshness | Source already emits domain events; batch staleness fine |
| **Asynchronous APIs** (event-driven §7) | Clients/partners need server-initiated updates with gateway QoS | AsyncAPI spec, gateway supporting WS/SSE/WebSub, broker bridge | Live status, feeds, partner callbacks | Polling at low frequency is acceptable |
| **Centralized IAM** (security §3) | N apps × N user stores × N logins; can't add SSO/MFA | IAM platform, consolidated user store(s), apps as SAML/OIDC SPs, federation, provisioning | Several apps, SSO/MFA/compliance, mixed B2C/B2E/B2B | Single app with no roadmap |
| **Cloud application security** (security §4) | SaaS sprawl with separate logins | Enterprise IAM as IdP for every SaaS SP, per-SP policy | SaaS-heavy estate | SaaS lacks SAML/OIDC |
| **Security federation** (security §5) | Partner/acquired users have their own IdP; don't import them | App → enterprise IAM → external IdP chain; optional JIT provisioning | M&A, dealer networks, B2B2C | Partner has no standards IdP |
| **API security** (security §6) | Secure gateway-exposed APIs without per-API code | Gateway + IAM; basic auth / API key / OAuth2 per client type | Any exposed API | — (choose mechanism, don't skip) |
| **Microservices security** (security §7) | Zero-trust east-west traffic without re-implementing security per service | mTLS + JWT propagation; or mesh sidecar; or policy agent (sidecar/library) | Any multi-service platform | — (choose variant) |
| **Observability reference arch** (observability §3) | Heterogeneous components, no cross-system root cause | Log path (agents→collector→store), telemetry path (OTLP→collector→backends), exporters, single pane, alerting | Always | — |
| **ELK + Jaeger + Prometheus** (observability §5) | Search-centric observability | Beats→Logstash→Elasticsearch→Kibana; Jaeger on ES; Prometheus | Log analytics, audit queries, existing ES skills | Cost-sensitive high log volume |
| **Grafana + Loki + Tempo + Prometheus** (observability §6) | Correlation-centric, cost-efficient observability | Promtail→Loki; Tempo; Prometheus(+Mimir); Grafana | K8s/Prometheus shops, high volume | Need full-text log analytics |

## Decision workflows

### 1. Integration style (sync API vs messaging vs events vs CDC)

```
Does the caller need the result to continue?            ── yes ─► Sync API (REST/GraphQL/gRPC) through gateway/process API
  │ no
Is the message valuable (must not be lost) or slow?     ── yes ─► Durable topic; ack 202; status via push channel
  │ no / also
More than one consumer of the same fact?                ── yes ─► Topic pub/sub, one consumer group per concern
Target slower or flaky vs source?                       ── yes ─► Queue/consumer group (scale) or store-and-forward (rate-limited target)
Source is a DB or vendor system you can't change?       ── yes ─► CDC (log-based) → topic → sinks
Hours of staleness acceptable, bulk copy?               ── yes ─► Batch ETL/ELT
Proprietary protocol (HL7, EDI, ISO 8583, mainframe)?   ── yes ─► Adapter in integration platform, exposed as System API
Partner outside your control?                           ── yes ─► B2B adapter + partner-facing API; webhooks for callbacks
```

Details: `references/event-driven.md` §1, §9; `references/api-management.md` §6.

### 2. API layering

- Assign every API to exactly one layer: **Experience** (per channel, aggregation, channel auth) → **Process** (orchestration, canonical mapping, cross-system rules) → **System** (one system of record each, stable, fast).
- Proprietary protocols terminate in the integration platform below the System layer. Never in clients or Experience APIs.
- Skip a layer when there's no orchestration (experience → system directly). Collapse to gateway → services for a small, single-channel estate.
- Legacy: put the gateway in as strangler facade first; refactor only what you own; ESB → modern integration platform; IAM consolidation; infra last.

Details: `references/api-management.md` §3, §5.

### 3. Security topology

| Situation | Choose |
|---|---|
| Several apps, own users, want SSO/MFA/social login | Centralized IAM; apps as SAML/OIDC SPs |
| Estate is mostly SaaS | Same, with each SaaS configured as SP of the enterprise IdP |
| Partner/acquired users with own IdP | Federation; add JIT provisioning only if you must own them eventually |
| Partner companies managing their own staff | B2B org management (hierarchical delegated admin) + federation |
| User-facing/mobile/third-party API clients | OAuth2 auth-code (+PKCE) at gateway; scopes drive authz |
| Server-to-server, no user | client-credentials; API key only if the client can't do OAuth2 |
| Internal legacy tool, can't do either | Basic auth, time-boxed |
| Service-to-service inside platform | mTLS + JWT propagation (self-contained JWT unless instant revocation matters) |
| Many polyglot services on K8s | Mesh sidecar handles mTLS/JWT; services stay agnostic |
| Fine-grained, data-dependent authz | Policy agent (library or sidecar) with policies outside code |

Details: `references/security-patterns.md` §8.

### 4. Observability stack

1. Inventory components: owned vs vendor; log format; OTLP support; metrics endpoint/exporter.
2. Logs first (100% coverage via agents), then OTLP traces on owned services and integration flows, then metrics + SLO alerts; exporters for DB/broker/gateway/K8s.
3. One correlation ID (trace context) across gateway → broker headers → services → logs.
4. Pick stack: search/audit-heavy or existing Elasticsearch → ELK+Jaeger+Prometheus; K8s/Prometheus shop, high volume, pivot-driven debugging → Grafana+Loki+Tempo+Prometheus. Managed cloud equivalents fit the same reference architecture if OTLP stays at the app boundary.

Details: `references/observability-stack.md` §4, §7.

## Reference files

| File | Contents |
|---|---|
| `references/api-management.md` | Integration surface, API platform components, API-led layering, governance split, legacy modernization sequence, checklists (ch. 4) |
| `references/event-driven.md` | Sync vs async decision, messaging models, EDA reference architecture with every edge explained, Kafka order flow, CDC topologies, async API protocols, checklists (ch. 5) |
| `references/security-patterns.md` | IAM use cases (B2C/B2E/B2B), centralized IAM, cloud app security, federation, API security mechanisms, east-west security variants, checklists (ch. 6) |
| `references/observability-stack.md` | Pillars and capture paths, reference architecture, per-component capture plan, ELK vs Grafana stacks, selection table, checklists (ch. 7) |
