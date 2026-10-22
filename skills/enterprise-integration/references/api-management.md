# API Management & Hybrid Integration Patterns (SolArch ch. 4)

Synthesis of chapter 4. Vendor-neutral. Audience: senior backend engineer designing an API layer over a mixed estate of in-house services, COTS/legacy and SaaS.

## Contents

1. [Integration surface: what an integration platform must do](#1-integration-surface)
2. [API management platform: components and responsibilities](#2-api-management-platform)
3. [Pattern: API-led connectivity (layered APIs)](#3-pattern-api-led-connectivity)
4. [Pattern: SOA governance + API management](#4-pattern-soa-governance--api-management)
5. [Pattern: Legacy modernization with a hybrid integration platform](#5-pattern-legacy-modernization-with-hybrid-integration-platform)
6. [Decision rules and checklists](#6-decision-rules-and-checklists)

---

## 1. Integration surface

Four integration concerns, each with a different "shape" of solution. Name the concern first; it decides the component.

| Concern | Typical problem | Solution shape | Key capabilities needed |
|---|---|---|---|
| **Data integration** | Many stores (RDBMS, NoSQL, files, lake); apps must not hit stores directly | Data services in front of stores; batch ETL/ELT; streaming ETL via CDC | Transformation, QoS on data access (authz, rate limit, cache), scheduling, CDC |
| **Application integration** | On-prem, SaaS, PaaS and hybrid apps must share data without changing them | Integration platform as hub/translator | Protocol translation, connectors, routing, orchestration, file handling |
| **Business (B2B) integration** | Partner systems you cannot change; proprietary or industry protocols | Integration platform with B2B adapters | EDI (X12, EDIFACT), AS2/AS4, HL7/FHIR, ISO 8583, custom adapters |
| **Consumer integration** | Omni-channel (web, mobile, kiosk/POS, third parties) needs one stable interface | API management on top of the integration layer | Gateway, developer portal, security, analytics, monetization |

Core integration capabilities the platform must cover (the book's checklist): data transformation, protocol translation, application connectors, routing and orchestration, file processing, partner (B2B) engagement, API management.

**Data-services rule:** never give applications (including ML jobs) direct store access across domain boundaries; front stores with a data service so you can apply authz, throttling, caching and hide schema. Prevents one noisy consumer starving others and limits blast radius of leaks.

**ETL vs streaming ETL (CDC):**
- Batch ETL/ELT: dynamic source → mostly-static destination, run off-peak. Fine when destination staleness of hours is acceptable.
- Streaming ETL via CDC: destination must reflect source changes within seconds. See `event-driven.md` §3 for the CDC pattern.

**Hybrid applications** (control plane in cloud, data plane on-prem) are the normal shape for API management when data must stay in your network: gateways run next to the data, portal/designer/analytics run as a managed service.

---

## 2. API management platform

Business framing first (the book insists on it): define KPIs tied to business goals before buying/building — e.g. sales executed via APIs, sign-ups through the API channel, time-to-build a new capability, partner count onboarded, API security incidents, new revenue via APIs. Use them to justify OpEx/CapEx and to measure after rollout.

### Components

```
 End user / client app ──► API Gateway ──► backend services / integration layer
                              │   ▲
                              │   └── token validation ──► API Security (key/token issuer)
                              │
      App developer ──► Developer Portal (discover, subscribe, try-out, register apps)
      API owner      ──► API Governance / control plane (lifecycle, policies, teams)
      Business owner ──► Business Insights (usage, latency, consumers, anomalies)
      (optional)         Monetization (tiers, quotas, billing)
```

| Component | Responsibility | Design notes |
|---|---|---|
| **Gateway** (data plane) | Enforce policies on every call: authn/authz, throttling, mediation/transformation, monitoring; forward to backend | Stateless, horizontally scalable; policy evaluation must add negligible latency; can be deployed close to the backend (on-prem) while control plane is in cloud |
| **Developer portal** | Marketplace: search by name/version/category/tag; try-out; app registration; feedback/ratings | Serves internal and external developers; the "contract" surface of your platform |
| **API security** | Issue and validate keys/tokens; support the client types you really have (client-credentials for M2M, auth-code for user-facing apps) | Gateway calls this component for opaque tokens; self-validates JWTs. Details in `security-patterns.md` |
| **Governance / control plane** | API lifecycle (create → publish → deprecate → retire), policy templates, team isolation, audit | Export/import via CLI so deployment flows through CI/CD, not click-ops |
| **Business insights** | Popular APIs, latency, consumers, error rates; anomaly alerts (fraudulent token use, backend failures) | Feeds observability stack; see `observability-stack.md` |
| **Monetization** | Tiered subscription plans, quota overage behaviour | Only if you sell API access; otherwise skip |

### Minimum viable API platform (decision)

- Need: one stable interface for several client channels, or any third-party consumer → gateway + security + portal.
- Need: only internal service-to-service calls inside one team → a gateway is optional; use service-level auth (mTLS/JWT) and an internal catalogue.
- Need: data as a product sold to third parties → add monetization and strong insights.

---

## 3. Pattern: API-led connectivity

**Problem:** heterogeneous estate (microservices, COTS, mainframe, SaaS, databases) with industry-specific protocols; every new channel re-integrates everything; teams block each other.

**Idea:** layer the system by *ownership and rate of change*, expose each layer through standard APIs (REST/GraphQL/SOAP over HTTP), and confine proprietary protocols to the lowest boundary.

```
┌──────────────────────────────────────────────────────────────┐
│ Consumer-facing apps (web, mobile, kiosk, 3rd party)          │
└───────────────▲──────────────────────────────────────────────┘
                │ standard API (via API Gateway + Developer Portal)
┌───────────────┴──────────────────────────────────────────────┐
│ EXPERIENCE APIs — shaped per channel; aggregation, field      │
│ trimming, channel-specific auth. Owned by product/channel     │
│ teams.                                                        │
└───────────────▲──────────────────────────────────────────────┘
                │ standard API
┌───────────────┴──────────────────────────────────────────────┐
│ PROCESS / BUSINESS APIs — orchestration, transformation,      │
│ routing, cross-system business rules. Runs on the integration │
│ platform (and brokers). Owned by enterprise integration team. │
└───────────────▲───────────────────────────▲──────────────────┘
                │ standard API              │ proprietary (HL7, ISO 8583, AS2...)
┌───────────────┴───────────────┐  ┌────────┴─────────────────┐
│ SYSTEM APIs — thin, fast,     │  │ Integration platform     │
│ in-house services/micro-      │  │ adapters translate to    │
│ services that unlock one      │  │ standard API             │
│ system of record each         │  │                          │
└───────────────▲───────────────┘  └────────▲─────────────────┘
                │ native protocol            │
┌───────────────┴────────────────────────────┴─────────────────┐
│ SYSTEMS OF RECORD — DBs, COTS, mainframe, legacy, SaaS        │
│ (not owned by you; cannot change their interfaces)            │
└──────────────────────────────────────────────────────────────┘
```

**Layer contract (keep it strict):**

| Layer | Owns | Must NOT do | Change frequency |
|---|---|---|---|
| Experience API | Response shape per channel, channel auth, BFF-style aggregation | Business rules; direct DB access | High (per feature) |
| Process API | Orchestration across systems, transactions/sagas, canonical model mapping | Channel-specific formatting | Medium |
| System API | One system of record each; CRUD-ish, performant, stable | Cross-system logic; knowledge of channels | Low |

**Component mapping (book's real-world view):** consumer apps → API gateway + developer portal → integration platform (process layer) → microservices/SOA services (system layer) → systems of record. Where a system of record only speaks a proprietary protocol, the integration platform owns the adapter and the system layer exposes a standard API over it.

**Fits when:** multiple channels; many systems you don't control; industry standards in play (healthcare, finance, logistics); need teams to work in parallel.

**Avoid / simplify when:** single channel and a handful of services you own — three layers is ceremony; collapse to gateway → services. Also avoid making the process layer a "smart ESB" that accumulates business logic nobody owns: process APIs must have a product owner like any service.

**Trade-offs:** extra hops add latency (budget ~1 hop per layer; collocate gateway and process layer); canonical model in the process layer is a governance burden; strict layering can tempt teams into needless pass-through APIs (allow an experience API to call a system API directly when no orchestration is needed — `(practice, not from book)`).

---

## 4. Pattern: SOA governance + API management

**Problem:** brownfield estate where every system offers a "service" (SOAP, REST, integration flows, microservices). Governance was the thing SOA got wrong; APIs over microservices bring the need back.

**Service lifecycle to govern:** requirement → design/review → implement → test → deploy → consume → support → deprecate → retire. Stakeholders span BA/PM, architects, devs, QA, DevOps, support, consumers, business leaders.

**Governance tool capabilities:** policy management (service definitions such as OAS/WSDL, security policies, questionnaires, with access control), dependency management (impact analysis of a contract change), lifecycle management (gated promotion, graceful retirement), SCM integration (commits reflected in lifecycle state), deployment integration (pipeline triggers), runtime governance (environment-aware service discovery: resolve identifiers to URLs per environment).

**Pattern:** treat the *API management control plane* and the *SOA/service governance tool* as one governance solution with a split of responsibility.

```
 Stakeholders ──► Governance solution ─┬─ API management control plane ─► (CLI/CI) ─► API gateways (pre-prod, prod)
                                       └─ Service governance tool ──────► build pipeline ─► service runtimes (pre-prod, prod)
                                                  │
                                          source repository
```

| Governs | Tool |
|---|---|
| Experience/API proxies (lifecycle, subscriptions, policies, deprecation) | API management control plane |
| System and business services (microservices, integration flows): contracts, dependencies, lifecycle | Service governance tool / service catalogue |

**Rules:**
- API management governs **proxies**, not the services behind them. It cannot replace service governance; don't assume buying an API platform gives you service lifecycle control.
- Deploy APIs through the pipeline (control-plane CLI/export), not by hand from the console, so both halves share one promotion flow and audit trail.
- Link the two so a "macro service" (one business capability = N services + M APIs) can be governed as a unit.
- Runtime governance = service discovery; may be a separate tool (registry/DNS/mesh). Design services to reference endpoints by identifier, resolved per environment.

**Fits when:** many teams, regulated domain, long-lived contracts, external consumers. **Avoid when:** small org with a single platform team — a service catalogue + API gateway with PR-based review covers it.

---

## 5. Pattern: Legacy modernization with hybrid integration platform

**Starting point (typical legacy):** web app → LB/proxy/firewall → SOAP/XML services on an ESB → COTS apps (CRM, ERP), legacy core apps, in-house tools, DB; each app with its own user store; everything on-prem.

**Legacy pain to address:** hard/expensive to update or upgrade (vendor-gated, monolithic); slow delivery of new services (cross-team approvals); over-provisioned, non-elastic infrastructure; knowledge locked in few people or the vendor.

**Order of work (book's sequencing — don't do everything at once):**

| Phase | Change | Why first/here |
|---|---|---|
| 1 | **Strangler facade**: put an API gateway between clients and the old system | Zero client disruption: old web app keeps SOAP/XML through the gateway; new REST/JSON/GraphQL/WebSocket APIs appear next to it for new web/mobile clients |
| 2 | **Refactor what you own**: in-house tools → microservices behind the facade; keep COTS/legacy untouched | You control the code; clients don't notice |
| 3 | **Replace the ESB** with a modern integration component (cloud-native deployable, better dev tooling) that does the gateway's transformations and existing integrations | Needed once new protocols and SaaS onboarding arrive |
| 4 | **COTS → SaaS** where ROI justifies; gateway + integration layer absorb the interface change | Business decision; slowest |
| 5 | **Centralize identity**: IAM consolidates per-app user stores into one; apps (legacy, new, SaaS) delegate authn to it → SSO, MFA, social login, risk-based authn become possible | Can't bolt SSO/MFA onto N built-in auth schemes |
| 6 | **Infrastructure**: containers/cloud for gateway, integration, services; managed DB; IAM may stay on VMs | Last; DB is usually the final monolith |

```
 old web (SOAP/XML)   new web / mobile (REST, GraphQL, WS)
          └──────────────┬───────────────┘
                 LB / proxy / firewall
                         │
            API Gateway (strangler facade)
            ├─ legacy functionality (pass-through / transformed)
            └─ new services (innovation)
                         │
              Integration platform (replaces ESB)
            ┌────────────┼─────────────┬────────────┐
        Legacy core   COTS → SaaS   in-house → µservices   DB
                         │
          Centralized IAM (consolidated user store; SSO/MFA)
```

**Decision rules:**
- Pick phase-1 items by "big benefit, small user-visible change". Never start with the DB or the core COTS.
- Half-finished modernizations are worse than none: each phase must leave a stable, runnable system.
- Only refactor components you own; wrap the rest.
- Migrate users with the IAM's migration tooling (just-in-time migration on first login or bulk import) — `(practice, not from book)`.

**Fits when:** stable legacy that must keep serving while new channels launch. **Avoid when:** the legacy is small enough to rewrite behind a feature flag in one go; a strangler facade then adds permanent indirection.

---

## 6. Decision rules and checklists

### Choosing the integration mechanism (ch. 4 scope; events in `event-driven.md`)

| Situation | Use |
|---|---|
| Client needs a stable, channel-shaped interface | Experience API via gateway |
| Multiple backends must be combined/transformed per request | Process API on integration platform |
| One backend with proprietary protocol (HL7, EDI, ISO 8583, mainframe) | Adapter in integration platform exposed as System API |
| Periodic bulk copy, hours of staleness OK | Batch ETL/ELT job |
| Destination must track source within seconds, source can't emit events | CDC / streaming ETL |
| Partner outside your control | B2B adapter (EDI/AS2/AS4) + partner-facing API |

### API-layer design checklist

- [ ] Each API assigned to exactly one layer (experience / process / system) with a named owner.
- [ ] System APIs touch one system of record; no cross-system logic.
- [ ] Proprietary protocols terminate in the integration platform, never in experience APIs or clients.
- [ ] Gateway policies: authn, authz scope, rate limit per app/tier, request/response mediation only where cheap.
- [ ] Portal entry for every published API (OAS, version, lifecycle state, owner, SLA).
- [ ] Deployment of APIs and services through the same pipeline; no console-only changes in prod.
- [ ] KPIs defined before launch (usage, time-to-new-capability, incidents, partners onboarded).
- [ ] Deprecation policy: versioning scheme, sunset headers/notice period, consumer notification via portal.

### Modernization checklist

- [ ] Inventory: owned vs vendor-controlled components; interfaces per component; user stores per app.
- [ ] Gateway in place as facade; old clients verified unchanged.
- [ ] Refactoring targets limited to owned code.
- [ ] ESB replacement scoped to transformations the gateway needs plus existing flows.
- [ ] IAM consolidation plan with user migration strategy and app-by-app cutover.
- [ ] Infra move last; DB as managed service if vendor supports.
