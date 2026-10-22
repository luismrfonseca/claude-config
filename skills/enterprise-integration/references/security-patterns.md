# Security Patterns for Enterprise Platforms (SolArch ch. 6)

Synthesis of chapter 6, limited to the concepts that drive architecture decisions. Vendor-neutral. Assumes you know OAuth2/OIDC/SAML/JWT/mTLS; this file tells you *which topology to pick*, not what they are.

## Contents

1. [Concepts that drive the patterns](#1-concepts-that-drive-the-patterns)
2. [IAM use cases the platform must serve](#2-iam-use-cases)
3. [Pattern: Centralized IAM](#3-pattern-centralized-iam)
4. [Pattern: Cloud application security](#4-pattern-cloud-application-security)
5. [Pattern: Security federation](#5-pattern-security-federation)
6. [Pattern: API security](#6-pattern-api-security)
7. [Pattern: Microservices (east-west) security](#7-pattern-microservices-east-west-security)
8. [Decision rules and checklists](#8-decision-rules-and-checklists)

---

## 1. Concepts that drive the patterns

**Scope:** application-layer security (not network/physical). Two targets: **data** (at rest: encryption, tokenization of sensitive fields, strong auth on stores; in transit: TLS everywhere with real cert/key management; privacy/consent per jurisdiction, e.g. GDPR) and **applications** (identity, authz, secure SDLC, incident readiness).

**Identity foundation = three functions, always in this order:**

| Function | Question | Mechanisms |
|---|---|---|
| Authentication | Who is this? | password, JWT/access token, hardware key, biometrics, OTP/TOTP; protocols SAML, OIDC (on OAuth2), LDAP/Kerberos for stores and legacy |
| Authorization | What may they do? | RBAC (coarse, group → permissions), ABAC (fine, per-attribute from the auth response/claims) |
| User management | Lifecycle | provisioning (bulk import, self-sign-up with validation workflow, JIT), role assignment, approvals/audit, self-service profile + consent portal |

**Credential/factor classes:** something you are (biometric), something you have (device, key, card), something you hold digitally (password, token, key). Human → combine classes (MFA); machine-to-machine → digital token only, since there is no person to challenge.

**Zero trust (the book's framing):** no implicit trust inside the LAN/DMZ; all entities untrusted by default; least privilege; comprehensive monitoring. Apply to users (MFA, least access, device posture), applications (authn on every inter-service call — no "trusted subsystem"), infrastructure (secured node/network comms).

**Principles the book sets:** build a strong identity foundation; adopt and automate security practices (secure SDLC with design review, code scanning, SAST/DAST/dependency scanning before release; vulnerability intake → classify → workaround → ETA → fix; disclosure/reward program); zero trust; prepare for incidents (roles, process, monitoring/alerting tools, pre-granted break-glass access, forensics).

**Token shape matters for topology:**

| Token | Validation | Latency | Revocation |
|---|---|---|---|
| Opaque (API key, opaque access token) | Introspection call to IAM on every hop | +1 network call per hop | Immediate |
| Self-contained JWT | Local signature + claims check (JWKS cached) | Local | Only via short TTL / denylist |

---

## 2. IAM use cases

| Use case | Who | Optimise for | Typical features |
|---|---|---|---|
| **B2C / CIAM** (and G2C) | customers, citizens | UX *and* security; highest risk surface | SSO, social/BYOI login, multi-option login, passwordless, MFA, adaptive (risk-based) authn, self-service consent |
| **B2E / workforce** | employees | security over convenience; trained users | hardware keys, VPN/device restriction, SSO for call-centre style multi-app roles |
| **B2B / organisation management** | partner companies with their own staff | delegated admin | hierarchical tenants: partner admins create users/roles without your security team |
| **B2B2C** | customers of your partners | trust partner authn | federation; map partner-asserted identity to your permissions |

**Common IAM feature catalogue (what a centralized platform gives you):**
- SSO across heterogeneous protocols (SAML, OIDC, Kerberos/RADIUS/LDAP for legacy): IAM is the single auth server; browser session cookie at the IAM → second app's redirect is satisfied without a login.
- Strong/MFA and adaptive authn: step 1 credentials (possibly multi-option), step 2 single stricter factor (TOTP, OTP, hardware key, biometric, challenge-response; security questions are weak). Adaptive = step 2 only on risk signals (location, device). Required for FedRAMP/FISMA/ISO-type compliance.
- Multi-option authn: social/external IdPs on step 1; keep step 2 single-option.
- Federated user provisioning (JIT): trusted external IdP authenticates; on first success the enterprise IAM creates the local user (optionally asks for a new password + consent); subsequent logins are local. Doubles as a zero-downtime migration path.
- Delegated access control (OAuth2): client obtains user-approved token; resource server (often the gateway) validates locally or via IAM; refresh keeps sessions alive without re-prompting.
- User-management workflows: multi-step approvals for user/group create/update/delete and role assignment; audit trail.

---

## 3. Pattern: Centralized IAM

**Problem:** N applications (ERP, CRM-SaaS, in-house, legacy) each with its own user store and auth scheme → one person has N profiles, N passwords, N logins; CRUD on a user touches N stores; SSO/MFA/social login would require changing every app.

**Solution:** one IAM platform owns authentication, authorization policy and user management; apps become service providers speaking a standard protocol to it. Same move as the ESB for integration: centralize the cross-cutting concern.

```
                 ┌──────────────┐  SAML   ┌──────────────┐
   user ──SSO──► │ In-house app │◄───────►│              │   LDAP/JDBC   ┌──────────────┐
                 └──────────────┘         │ Centralized  │◄─────────────►│ Enterprise   │
                 ┌──────────────┐  OIDC   │ IAM platform │               │ user store(s)│
                 │ SaaS CRM     │◄───────►│  authn       │               └──────────────┘
                 └──────────────┘         │  authz policy│  federation   ┌──────────────┐
                 ┌──────────────┐ Kerberos│  user mgmt   │◄─────────────►│ External IdP │
                 │ ERP          │◄───────►│  MFA/adaptive│  outbound     └──────────────┘
                 └──────────────┘         └──────┬───────┘  provisioning ──► SaaS user APIs
                                                 └── single profile per person
```

| Component | Responsibility |
|---|---|
| IAM platform | Login UI/session, protocol endpoints (SAML/OIDC/OAuth2), MFA orchestration, federation (inbound/outbound), provisioning (SCIM-style outbound, JIT inbound), workflows, audit |
| User store(s) | Credentials/profiles; LDAP/AD or DB; can be several stores by population (customers vs employees) behind one IAM |
| Service providers (apps) | Delegate authn; consume assertions/ID tokens; enforce authz locally from claims/roles |
| Extension points | Adapters for apps with non-standard auth so nothing is left outside |

**Fits when:** more than a handful of apps; need SSO/MFA/social login/compliance (GDPR, FISMA); mixed B2C/B2E/B2B populations; legacy modernization (see `api-management.md` §5).

**Avoid / defer when:** single app with built-in auth and no roadmap for more; or where the dominant population lives in a partner IdP (use federation, §5).

**Trade-offs:** IAM becomes tier-0 (HA, DR, latency budget); protocol adapters for legacy cost effort; user migration into the consolidated store needs a plan (bulk import + forced reset, or JIT on first login).

---

## 4. Pattern: Cloud application security

**Problem:** migration brings many SaaS apps, each with its own login, MFA level and user profile; users accumulate credentials; profile management fragments.

**Solution:** make every SaaS app a *service provider* of an enterprise-controlled IdP (cloud-hosted or on-prem IAM — the choice is yours, the pattern is identical). On each SaaS app: configure the enterprise IAM as the (federated) IdP. On the IAM: register each SaaS app as an SP with its own policy (MFA, adaptive, allowed login options). Users sign in with enterprise credentials; SSO spans all SaaS; user lifecycle is managed once (and pushed outbound where the SaaS supports provisioning).

```
 user ──► SaaS A ─┐ (SAML/OIDC)   ┌──────────────────┐
 user ──► SaaS B ─┼──────────────►│ Enterprise IAM   │──► enterprise user store
 user ──► SaaS C ─┘               │ (per-SP policy)  │
                                  └──────────────────┘
```

**Fits when:** onboarding SaaS at pace; compliance needs one place to prove who can access what. **Avoid when:** SaaS app lacks SAML/OIDC support (then front it with a gateway or accept a local account + stricter controls).

---

## 5. Pattern: Security federation

**Problem:** partners, acquisitions or large external populations have their own IdP; importing thousands/millions of users bloats and slows your store and breaks their existing credentials.

**Solution:** chained trust. App → enterprise IAM (as its IdP) → external IdP (as a federated IdP registered under that SP) → back. User never sees the enterprise IAM hop; keeps their existing credentials.

**Three-sided configuration:**
1. App: enterprise IAM is the IdP (SAML/OIDC request goes there).
2. Enterprise IAM: app registered as SP; external IdP registered as federated IdP for that SP (optionally with attribute mapping and home-realm discovery).
3. External IdP: enterprise IAM registered as an SP so it accepts the relayed request.

```
 partner user ──► Enterprise app ──► Enterprise IAM ──(relay)──► Partner/Acquired IdP
                                           ▲                            │
                                           └──── assertion ◄────────────┘
                        (no local user created, or JIT if you need one)
```

**Choose between federation and JIT migration:**

| Need | Pattern |
|---|---|
| Users stay partner-managed; you only grant app access | Pure federation (no local record) |
| Users must eventually live in your store (M&A integration) | Federation + JIT provisioning; optionally cut over to local login later |
| Partner has many staff and needs self-admin | B2B organisation management (hierarchical tenant admin) + federation |

**Fits when:** M&A, dealer/partner networks, B2B2C. **Avoid when:** the partner cannot run a standards-based IdP (fall back to inviting users into a partner tenant in your IAM).

**Trade-offs:** your security posture depends on the partner's authn strength — enforce step-up MFA on your side for sensitive apps; attribute/role mapping becomes a governance item; outages at the partner IdP lock their users out.

---

## 6. Pattern: API security

**Problem:** APIs exposed via a gateway to internal and external apps; implementing security per API is wasteful; a gateway alone can't do authn/authz/user management — it needs an IAM behind it.

**Mechanisms (north-south, at the gateway):**

| Mechanism | How | Pick when | Risks / notes |
|---|---|---|---|
| **Basic auth** | Client sends user credentials; gateway validates against IAM | Internal tooling inside the perimeter only | Apps hold user passwords → leak risk; no MFA; avoid for anything public |
| **API key** | Opaque random key issued by API owner/IAM to the *application*; sent as header; gateway introspects with IAM | Identify/throttle a client app; low-risk read APIs; partner server-to-server with no user context | Key identifies the app, not a user; static secret in config → rotate, scope, rate-limit; leak = access until revoked |
| **OAuth2 access token (delegated)** | User authenticates at IAM (MFA possible); client obtains token via the grant that fits it; gateway validates (introspect opaque / verify JWT locally) | Default for anything user-facing or external; any API needing user context | Short TTL, refresh tokens, revocation, scope → gateway authz; grant choice: auth-code (+PKCE) for user apps, client-credentials for M2M (`(practice, not from book)`: PKCE) |

```
 user ──login──► IAM (MFA, consent) ──► client app gets access token
 client ──Authorization: Bearer──► API Gateway ──validate──► IAM (opaque) / local JWKS (JWT)
                                        │ scopes → policy
                                        ▼
                                  backend service
```

**Gateway responsibilities:** token validation, scope/role check, rate limiting per app/token, request logging for audit, stripping/forwarding identity to backends. IAM responsibilities: issue/refresh/revoke tokens and keys, authenticate users, hold policy data.

**Decision:** evaluate the *client*: can it do OAuth2 flows? → OAuth2. Pure server-to-server with no user? → client-credentials (preferred) or API key. Only when neither is possible and the caller is internal → basic auth, and plan to replace it.

---

## 7. Pattern: Microservices (east-west) security

**Problem:** gateway secures the edge; zero trust says service-to-service calls (HTTP/gRPC sync, Kafka/NATS async) must be authenticated and authorized too, without re-implementing security in every service. Tension: IAM is centralized, microservices are decentralized.

### 7a. Centralized IAM, token propagation

```
 client ─JWT─► Gateway ─mTLS + JWT─► svc1 ─mTLS + JWT (same or re-issued)─► svc2
                 │ validate            │ validate (IAM call or local)         │ validate
                 ▼                     ▼                                       ▼
                IAM ◄──────────────────┴───────────────────────────────────────┘
```

- mTLS between gateway ↔ service and service ↔ service: proves *which workload* is calling.
- JWT carries *user context*; propagate the same token or exchange it for a narrower one (token exchange) at each hop.
- Validation: call IAM (opaque, simpler, slower) or verify self-contained JWT locally (faster, more code per service).

### 7b. Service mesh / sidecar offload

Same flow, but the sidecar proxy terminates mTLS and validates JWTs per policy pushed from the mesh control plane. Services stay security-agnostic. Adds mesh operations cost; pick when you already run Kubernetes at scale or have many polyglot services.

### 7c. Decentralized policy (policy agent per service)

Authn still issues the JWT (IAM/AD), but *validation and authorization decisions* run locally in a policy agent next to each service (sidecar + agent in the same pod, or agent embedded as a library). Policies and data live outside the service code and are updated via the agent's API/CLI. Removes the IAM round-trip from the hot path; fits fine-grained ABAC per service. Library mode drops the sidecar for simpler deployment.

| Variant | Latency | Ops complexity | Service code impact | Pick when |
|---|---|---|---|---|
| Central IAM, opaque tokens | highest (IAM call per hop) | low | low | few services, simple |
| Central IAM, self-contained JWT | low | low | medium (validation lib per service) | most NestJS-style platforms |
| Mesh sidecar | low | high (mesh) | none | K8s at scale, polyglot |
| Policy agent sidecar | low | medium-high | none | fine-grained authz, K8s |
| Policy agent library | lowest | medium | medium | fine-grained authz, no mesh |

**Async traffic:** the same principle applies to broker clients — authenticate producers/consumers to the broker (mTLS/SASL) and carry user/tenant context in event headers for downstream authz (`(practice, not from book)`).

---

## 8. Decision rules and checklists

### Which IAM / token topology

| Situation | Pattern |
|---|---|
| Many apps, own users, want SSO/MFA | Centralized IAM (§3) |
| Mostly SaaS estate | Cloud application security = centralized IAM with SaaS as SPs (§4) |
| Partner/acquired users with their own IdP, no need to own them | Federation (§5) |
| Need to absorb those users eventually | Federation + JIT provisioning (§5) |
| Partner companies managing their own staff access | B2B organisation management (hierarchical admin) (§2) |
| Public/mobile/third-party API consumers | OAuth2 at gateway; auth-code for user apps, client-credentials for M2M (§6) |
| Internal server-to-server, no user | client-credentials, or API key if the client can't do OAuth2 (§6) |
| Service-to-service inside the platform | mTLS + JWT propagation; self-contained JWT unless revocation latency matters (§7a) |
| Many polyglot services on K8s | mesh sidecar or policy-agent sidecar (§7b/7c) |
| Fine-grained, data-dependent authz per service | policy agent (library or sidecar) with externalized policies (§7c) |

### Design checklist

- [ ] User populations classified (B2C/B2E/B2B/B2B2C) and each mapped to a store, authn policy and admin model.
- [ ] Every app integrates via SAML/OIDC; non-standard apps have an adapter plan.
- [ ] MFA policy: which population, which factor, adaptive triggers; step 2 is single-option.
- [ ] Federation trust documented per partner IdP: attribute mapping, step-up rules, JIT yes/no.
- [ ] API mechanism per client type chosen; basic auth confined to internal and time-boxed.
- [ ] Token policy: TTLs, refresh, revocation path, JWT vs opaque per hop, JWKS caching.
- [ ] East-west: mTLS everywhere; user context propagated; validation location decided (service lib / sidecar / agent).
- [ ] Authz model: RBAC for coarse, ABAC/policy for fine; policies versioned outside code.
- [ ] User-management workflows with approvals and audit for privileged changes.
- [ ] Consent/privacy: PII consent capture and revocation → deletion path.
- [ ] Secure SDLC gates (design review, SAST/DAST/dependency scan) and vulnerability intake process defined.
- [ ] Incident readiness: roles, runbook, alerting, pre-granted break-glass access, forensics logging.
