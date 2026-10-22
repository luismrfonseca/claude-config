# Observability Stack Patterns (SolArch ch. 7)

Synthesis of chapter 7. Open-source tool names are kept where the book's two patterns are defined by them (ELK/Jaeger/Prometheus vs Grafana/Loki/Tempo/Prometheus); everything else is vendor-neutral. Audience: senior backend engineer designing the logs/metrics/traces stack for a platform with in-house services plus vendor components.

## Contents

1. [Concepts that matter for design](#1-concepts-that-matter-for-design)
2. [Three pillars: capture paths](#2-three-pillars-capture-paths)
3. [Observability reference architecture](#3-observability-reference-architecture)
4. [Component-by-component capture strategy](#4-component-by-component-capture-strategy)
5. [Pattern A: ELK + Jaeger + Prometheus](#5-pattern-a-elk--jaeger--prometheus)
6. [Pattern B: Grafana + Loki + Tempo + Prometheus](#6-pattern-b-grafana--loki--tempo--prometheus)
7. [Choosing between A and B](#7-choosing-between-a-and-b)
8. [Checklists](#8-checklists)

---

## 1. Concepts that matter for design

- **Observability** = property of the system: internal state inferable from external outputs (control-theory definition). **Monitoring** = the action of consuming those outputs. Monitoring quality is capped by observability quality; you fix observability in the code, not in the dashboard.
- **Telemetry** = what instrumentation emits: logs, metrics, traces (OTel adds *baggage* — context passed between spans).
- **Four implementation concerns:** instrumentation (emit via open standards), correlation (one request ID/trace context across all components, including vendor ones), automation (threshold alerts, incident routing — nobody watches logs), insights/predictions (capacity, usage, business signals).
- **Functional pipeline:** publish (apps) → collect (agents/collectors) → analyze (index/query) → alert + dashboards → ops team / business.
- **Tracing is the expensive pillar:** span construction and export cost CPU and memory; keep it toggleable/sampled.
- **Three ways to get data out of a component** (this decides everything in the reference architecture):
  1. Vendor-provided log files (always available).
  2. Instrumented telemetry over a standard protocol (OTLP) — only for code you own or vendors who support it.
  3. Adapters/exporters converting proprietary metrics (JMX, REST, DB stats) into a standard scrape format.
- **Why a standard (OpenTelemetry):** per-language logging frameworks never gave cross-service correlation; OTLP gives one wire format for traces/metrics/logs and vendor-portable backends. Instrument once, swap backends later.
- **Trace vocabulary:** span = one operation with name, start/end, structured events, attributes, parent link; trace = tree of spans for one request from root span downwards. Auth → dispatch → data service → DB is the kind of path a trace must show.
- **Metrics model (Prometheus-style):** time series + labels; pull by default from `/metrics` endpoints (static config or service discovery); push gateway for short-lived jobs; PromQL for queries; alert rules → AlertManager → email/pager; exporters for things you can't instrument.

---

## 2. Three pillars: capture paths

| Pillar | Capture mechanism | Agent/collector | Store | Query/visualize | Alerting |
|---|---|---|---|---|---|
| Logs | Tail files (or stdout in containers); normalize; label | Log shipper agent per host/pod | Indexed log store | Log query language + dashboards | Log-pattern rules |
| Traces | OTLP from SDK (auto + manual instrumentation); legacy agent for pre-OTLP protocols | OTLP collector (validate, transform, batch, route) | Trace store (memory, search index, object store, DB) | Trace UI; link to logs by trace ID | Latency/error rules derived from spans |
| Metrics | Client lib exposes endpoint; exporters for third-party systems; push gateway for batch jobs | Scraper (pull) | TSDB (local disk; long-term store for multi-DC/retention) | PromQL + dashboards | Rules → AlertManager |

Logs are the floor: every component produces them, so a log pipeline covers 100% of the estate on day one. Traces and standards-based metrics only reach the components you own or whose vendor supports OTLP/scrape endpoints.

---

## 3. Observability reference architecture

```
                 ┌───────────────────────── enterprise platform ─────────────────────────┐
  Web/Mobile ──► │ Accessibility (LB / proxy / WAF)        logs ─────────────┐           │
                 │ API software (gateway, portal, CP)      logs, OTLP, metrics│           │
                 │ Integration software                    logs, OTLP, metrics│           │
                 │ Core apps: in-house µservices           OTLP + metrics     │           │
                 │            COTS                         logs (+metrics API)│           │
                 │            SaaS                         vendor API only    │           │
                 │ Security software (IdP)                 logs, metrics      │           │
                 │ Databases                               logs + exporters   │           │
                 │ Infra / SDLC tools (K8s, CI, SCM)       logs + exporters   │           │
                 └─────────────────────────────────────────────────────────────┼──────────┘
                                                                               ▼
   ┌──────────────────────────── observability solution ──────────────────────────────────┐
   │ LOG PATH:     log agents ─► log collector/normalizer ─► index/store ─► log analyzer   │
   │ METRIC/TRACE: OTLP SDK / exporters ─► data collector ─► storage backend ─► query      │
   │                                              │                                        │
   │                                       visualization (dashboards) + alerting ──► ops,  │
   │                                       business, on-call                               │
   └───────────────────────────────────────────────────────────────────────────────────────┘
```

Two collection paths, one visualization/alerting layer. Metrics and traces may share a collector when both come over OTLP; split them when you adopt a dedicated metrics scraper (the usual case).

**Responsibilities:**

| Block | Does |
|---|---|
| Log agents | Tail, parse, label, filter at the edge; ship |
| Log collector | Normalize heterogeneous formats into one schema; enrich; route |
| Log store/analyzer | Index, retain, query, correlate by request/trace ID; feed dashboards and alerts |
| Telemetry collector | Receive OTLP; validate/transform/batch; fan out to trace and metric backends |
| Metrics exporter | Adapter for components without native metrics |
| Storage backends | Trace store; TSDB (+ long-term store if retention/multi-DC needed) |
| Visualization | Single pane over logs+metrics+traces; pivot between them by trace ID/labels |
| Alerting | Threshold and pattern rules → notification channels; also business alerts |

---

## 4. Component-by-component capture strategy

| Component | Can you instrument it? | Capture plan |
|---|---|---|
| LB / proxy / WAF | No (vendor) | Access/error logs via agent; metrics via vendor API/JMX → exporter |
| API gateway / control plane / portal | No (vendor) — but choose one that emits OTLP and scrape-able metrics | OTLP traces + metrics if supported; else logs + exporter. Gateway = highest traffic, instrument first; portal = lowest |
| Integration platform | In-house: yes. Vendor: depends | In-house → OTLP SDK + metrics endpoint. Vendor → its OTLP/metrics if available, else logs |
| In-house microservices | Yes | OTLP auto+manual instrumentation, metrics endpoint, structured logs carrying trace ID |
| COTS (ERP, HRIS) | No | Logs; metrics API → exporter where modern |
| SaaS | No raw logs | Pull from vendor observability APIs into your store |
| Security software (IdP) | In-house: yes; vendor: logs/metrics | Audit logs are mandatory inputs; token validation latency as a metric |
| Databases | No | DB logs + DB exporter (connections, slow queries, replication lag) |
| Containers / orchestrator / CI / SCM | No | Platform logs + exporters (node, kube-state, pipeline durations) |

**Vendor selection rule:** when buying gateway/integration/IdP software, require standards-based observability output (OTLP, scrape endpoint, structured logs with correlation IDs) so you are not locked into the vendor's built-in dashboards.

---

## 5. Pattern A: ELK + Jaeger + Prometheus

```
 apps ─ log files ──► Beats (agent) ──► Logstash (input→filter→output) ──► Elasticsearch ◄── Kibana
 apps ─ OTLP spans ─────────────────► Jaeger collector ──────────────────► Elasticsearch (trace store) ◄── Jaeger UI (via Jaeger Query)
 apps ─ /metrics or exporter ──► Prometheus server (scrape → TSDB → PromQL) ──► AlertManager
                                        └── metricbeat ──► Elasticsearch (metrics visible in Kibana)
```

| Pillar | Components | Notes |
|---|---|---|
| Logs | Beats → Logstash → Elasticsearch → Kibana | Logstash pipeline: inputs, optional filters (transform/filter), outputs. Normalization happens here |
| Traces | OTLP SDK → Jaeger collector → Elasticsearch; Jaeger agent only for legacy (pre-OTLP) instrumentation | Using Elasticsearch as trace storage lets Kibana query traces alongside logs |
| Metrics | Prometheus (+ exporters, push gateway) → AlertManager | Optionally ship metrics into Elasticsearch so Kibana is the single pane; keep Prometheus UI/AlertManager for metric-native alerting |
| Single pane | Kibana | Search, dashboards, alerting, anomaly/fraud analysis on logs+traces+metrics |

**Strengths:** full-text search over everything; one store (Elasticsearch) for logs and traces; mature log analytics; strong when log analysis (fraud, audit, compliance queries) is the primary use. **Costs:** Elasticsearch indexes every field — storage/compute heavy at scale; two alerting systems unless consolidated; metrics duplicated if shipped to ES.

---

## 6. Pattern B: Grafana + Loki + Tempo + Prometheus

```
 apps ─ log files ──► Promtail (label/transform/filter) ──► Loki (label-indexed streams; LogQL) ──┐
 apps ─ OTLP spans ──► Grafana agent (batch/route; optional) ──► Tempo (trace store) ─────────────┼──► Grafana (single pane)
 apps ─ /metrics or exporter ──► Prometheus (TSDB) [→ Mimir for long-term / multi-DC] ────────────┘
                                        └──► AlertManager (metrics and Loki log alerts)
```

| Pillar | Components | Notes |
|---|---|---|
| Logs | Promtail → Loki → Grafana (LogQL) | Loki indexes only labels, not message content → cheap storage, cheaper compute; query by labels then grep |
| Traces | OTLP SDK → (agent) → Tempo → Grafana | Agent is optional but recommended for batching and backend routing |
| Metrics | Prometheus (+ exporters) → Grafana; Mimir when disk-based retention is insufficient | Same Prometheus as pattern A |
| Single pane | Grafana | Correlate logs ↔ traces ↔ metrics from any starting point; dashboards for perf, availability, business, predictions; unified alerting |

**Strengths:** cost-efficient logs at volume; designed around correlation pivots (trace ID in logs → trace view → metrics); one alerting surface; same stack whether self-hosted or managed. **Costs:** no full-text indexing — ad-hoc content search over large ranges is slower; log analytics/ML on content weaker than a search engine.

---

## 7. Choosing between A and B

| Decision factor | Favour A (ELK/Jaeger/Prom) | Favour B (Grafana/Loki/Tempo/Prom) |
|---|---|---|
| Primary use of logs | Search-heavy analytics, audit/compliance queries, fraud detection on log content | Operational troubleshooting from dashboards and traces |
| Log volume vs budget | Budget for indexing everything | High volume, cost-sensitive |
| Existing skills/footprint | Team already runs Elasticsearch | Team already on Prometheus/Grafana (typical K8s shops) |
| Correlation workflow | Query-driven in one search index | Pivot-driven (metrics → traces → logs) |
| Metrics long-term retention | Ship to ES, or external TSDB | Mimir |
| Alerting | Kibana + AlertManager (or consolidate) | Unified in Grafana/AlertManager |

Both patterns: instrument in-house code with OTLP, use exporters for vendor components, start with logs for 100% coverage, add traces to owned services, add metrics-based SLO alerts. Cloud-native managed equivalents (`(practice, not from book)`: AWS CloudWatch/X-Ray, etc.) slot into the same reference architecture; keep OTLP at the app boundary so the backend remains swappable.

---

## 8. Checklists

### Design checklist

- [ ] Inventory every component with: owned/vendor, log format, OTLP support, metrics endpoint/exporter availability.
- [ ] One correlation ID strategy: W3C trace context propagated through gateway, integration layer, broker headers and in-house services; logged in every structured log line.
- [ ] Log pipeline covers all components on day one (agents on every host/pod; normalization schema defined).
- [ ] OTLP instrumentation (auto + key manual spans) on all in-house services and integration flows; sampling policy set; tracing toggleable.
- [ ] Metrics: service-level (RED: rate, errors, duration), infra-level, exporters for DB/broker/gateway/K8s; push gateway for batch jobs.
- [ ] Retention: hot/warm/cold for logs; TSDB long-term store if > local disk horizon or multi-DC.
- [ ] Alerting: SLO-based thresholds (latency vs SLA, error rate, saturation), log-pattern alerts (OOM, auth failures), business alerts; routed to on-call and stakeholders.
- [ ] Dashboards: ops (performance, availability), business insights, capacity.
- [ ] Audit/compliance: tracing/log retention window agreed (e.g. months) and access-controlled.
- [ ] Vendor components chosen with standards-based observability output as a selection criterion.

### Minimum viable stack (small platform)

Structured JSON logs to stdout → one log agent → log store; OTLP SDK in every service → one collector → trace store; `/metrics` on every service + exporters for DB/broker → scraper → alerting; one dashboard tool over all three. Add long-term stores and a mesh-level exporter later.
