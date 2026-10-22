# Cloud-native deployment, multi-cloud, migration (SolArch ch. 8)

Secondary source. Use when the trade-off being discussed is *where and how* the chosen architecture runs, not which style it is. Keep style decisions in styles.md; keep the quantum analysis in modularity.md.

## 1. What "cloud-native" buys and costs

Cloud platform properties to exploit: global availability, automatic scale up **and down** (on-prem typically idles at 50–60 % utilization because it is sized for peak), modular managed services, vendor-managed layers, on-demand provisioning. An application is cloud-native only if it actually uses them: containerized, independently scalable components, managed data stores, no pre-allocated capacity.

| Layer you manage | IaaS | PaaS | KaaS (managed Kubernetes) | SaaS |
|---|---|---|---|---|
| Hardware, network, storage, virtualization | vendor | vendor | vendor | vendor |
| OS, middleware, runtime | you | vendor | vendor runs the cluster, you run workloads | vendor |
| Application and data | you | you | you | vendor (config only) |

Deployment models: public (cheapest, shared tenancy), private (data residency, regulation, dedicated, costlier), hybrid (latency-critical or regulated parts close to core systems, the rest public). Containers beat VMs on density and portability (no guest OS per workload) but need an orchestrator once you have more than a handful of nodes; some workloads (ERP, reservation engines, legacy in-house apps) belong on big VMs, not containers. Mix per component; "all-in on containers" is as wrong as "all-in on VMs".

## 2. Micro-architecture pattern

Apply "micro" beyond business services, each layer deployed as independently scalable units:

| Component | Cloud-native shape | Note |
|---|---|---|
| Business services | Microservices in containers; COTS/SaaS as they come; heavy legacy on VM scale sets | Only in-house code is yours to redesign |
| Integrations | Integration microservices per coarse domain instead of one central ESB | See distributed.md §4 |
| API management | **Microgateways** partitioned per client type or per functional group (even per API); a central control plane (publisher, portal, traffic manager, key manager) that needs far less scale | A monolithic gateway means one bad API can take all APIs down and scaling one means scaling all |
| Front-end | Micro-frontends (search / recommendations / checkout) composed by a shell, each owned by the team that owns the backend domain | Fundamentals ch. 17 microfrontends |
| Security | Zero trust: policy agent sidecar per service + security control plane; a secure token service / IdP (or IDaaS) issuing tokens; MFA at the edge | |
| Inter-service communication | Service-mesh sidecar proxy (sync) or message broker client (async) | distributed.md §4 |
| Observability, SCM, CI/CD | Centralized or vendor-provided; only the data-publishing agents live beside services | Without automated build → deploy → test → promote, cloud-native is just hosting |

Data: pick the store per component (relational managed DB for checkout, NoSQL for recommendations/ML); place components in regions near users when performance is a driver.

## 3. Kubernetes deployment pattern

Objects you must reason about: **namespace** (team/environment isolation on a shared cluster), **deployment** (declared pods + replica set, min/max replicas for autoscaling), **pod** (one or more tightly coupled containers sharing network/storage and an IP), **service** (stable cluster IP load-balancing across pod replicas), **ingress** (external entry: LB, TLS termination, host routing) via an ingress controller.

Reference layout: ingress controller + load balancer → per-team/per-env namespaces → one service per component (gateway, business, integration) each backed by an autoscaled deployment. Pod shapes to copy:

| Pod | Containers | Why |
|---|---|---|
| Gateway pod | microgateway only; talks to the API control plane | Stateless, scales with traffic |
| Business pod | business logic + security agent (OPA) + optional app-local data container | Policies change without code changes |
| Integration pod | integration logic + security agent + mesh sidecar proxy | Needs routing/retries/discovery toward many systems |

Outside the cluster but required: private container registry, persistent storage, etcd (possibly dedicated), overlay network, observability stack, SCM, pipeline. Kubernetes gives self-healing, autoscaling, rollouts/rollbacks, discovery, LB, storage orchestration; it does not give you good service boundaries.

## 4. Multi-cloud

Why: availability across providers (degrade, not die), no vendor lock-in, best-fit service per vendor (one for DB, one for ML, one for .NET-friendly messaging), proximity to users, procurement leverage. Cost: every cross-cloud concern (gateway, IAM, observability, load balancing) now needs a strategy.

| Option | Gateway | Marketplace/portal | Trade-off |
|---|---|---|---|
| Vendor-specific gateways + common marketplace | AWS API GW, Apigee, Azure API GW each in their cloud | One external marketplace aggregating all | Best runtime fit and price, three gateway technologies to learn and operate |
| Single cloud-agnostic gateway vendor | Same product deployed in every cloud, one control plane | Vendor's own portal | One skill set and control plane; gateway vendor becomes the lock-in |
| HA replication of the same APIs across clouds | Identical API set in every cloud, all pointing to the same backend | One | Needs a primary deployment for changes and synchronized API config data; only for extreme availability needs |

Always: one IAM/STS trusted by every gateway; a cloud-agnostic global load balancer; a common observability choice (one stack hosted in one cloud or SaaS).

## 5. Cloud migration pattern

Phases: **assessment** (business need, cloud readiness, per-component verdict: rehost/lift-and-shift, re-architect, repurchase as SaaS, retire), **preparation** (teams, skills, order, timelines, success criteria per component), **migration and modernization** (execute, measure, iterate).

Per-component playbook from the worked brown-field SOA example:

| Existing | Target |
|---|---|
| In-house SOAP/REST apps | Re-architect as microservices |
| On-prem relational DB | Repurchase: managed cloud DB |
| ESB | Replatform: hybrid integration platform / integration microservices |
| COTS (CRM, ERP, HR) | Repurchase as SaaS where possible, else VMs |
| Legacy apps | Move functionality to microservices over time, rehost the rest on VMs |
| JSP web app | New SPA + mobile app over the gateway |

Mechanism: **API gateway as strangler façade** first (one-time client change), then repoint routes to new services one at a time; intermediate state with half migrated is normal and must be healthy. Close the loop with metrics (performance, usage, sales) that validate the business case made in assessment. The executive questions that justified it ("how does this scale globally?", "what if the data center fails?", "how long to roll out a service?", "what is the update cycle?") are the same characteristics that go into the ADR context (decisions-and-risk.md §3).
