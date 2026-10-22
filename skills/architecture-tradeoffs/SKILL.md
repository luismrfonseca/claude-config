---
name: architecture-tradeoffs
description: Trade-off analysis toolkit from "Fundamentals of Software Architecture" and "Solution Architecture Patterns for Enterprise". Use whenever a backend design choice is on the table, even if "architecture" is never said - choosing an architecture style (modular monolith, service-based, event-driven, microservices, microkernel, space-based), splitting a monolith or asking "should this be a separate service", drawing module/service boundaries, sync vs async, orchestration vs choreography, defining architecture characteristics/NFRs and fitness functions, writing or reviewing an ADR, evaluating a design proposal, architecture risk assessment, or planning a Kubernetes/multi-cloud deployment. Brings style scorecards, connascence, the quantum test, distributed fallacies, anti-patterns and a risk matrix so the answer is a reasoned trade-off, not an opinion. For code changes inside domain-service's layers use domain-architect.
---

# Architecture trade-offs

Two laws frame every answer (Fundamentals ch. 1-2):

1. **Everything is a trade-off.** If a proposal looks free of downsides, the downside has not been found yet. Name it before recommending anything.
2. **Why beats how.** A topology without the reasoning behind it cannot be evolved safely. Record the why (ADR), and write the how only as far as needed.

Corollaries that shape the answer: there are no right answers, only expensive ones; aim for the *least worst* architecture, not the best; prefer designs that are cheap to iterate over designs that try to be right first time; "it depends" is honest only when you then say what it depends on.

## Workflow

Run these steps in order; skip a step only when the user has already done it and says so.

1. **Identify the driving characteristics** (`references/characteristics.md`). Translate domain language into -ilities ("thousands of users, maybe millions" = scalability; "mealtime rush / ticket drop" = elasticity; "regulatory deadline" = performance + availability + reliability + recoverability + auditability). Keep ≤ 7 candidates, add the implicit ones (availability, reliability, security baseline, modularity), then get the **top 3 in any order**. Apply the three-part test (non-domain, structural impact, critical) before promoting anything; customization that fits in a design pattern is design, not architecture.
2. **Determine the quanta** (`references/modularity.md` §4). Group candidate components by the operational characteristics they need and the database they share. One quantum → monolith (modular, domain-partitioned). Several quanta with genuinely different characteristics → distributed. A shared database collapses everything into one quantum regardless of how many deployables exist.
3. **Partition and find components** (`references/modularity.md` §5-6). Domain partitioning by default (migratable, maps to teams); technical partitioning only for small, simple or throwaway systems. Discover components with actor/actions, event storming or workflows, never one-component-per-entity. Split a component whenever two parts need different characteristics even if they do the same thing functionally.
4. **Pick the style with the scorecard** (`references/styles.md`). Match the top-3 characteristics against the table below, check domain/architecture isomorphism (customization → microkernel; bursty > 10 k users → space-based; coupled multi-step workflow → service-based or modular monolith), then read the "avoid when" line for the candidate. Decide sync vs async last: **synchronous by default, asynchronous where a specific characteristic demands it**.
5. **Record the decision as an ADR** (`references/decisions-and-risk.md` §3). Context with alternatives, decision in imperative voice with technical and business justification, consequences including the trade-offs already argued, compliance via a fitness function.
6. **Identify risk** (`references/decisions-and-risk.md` §4-5). Score impact × likelihood per area and dimension; unknown technology is always a 9; propose mitigations with a cheaper fallback for the stakeholder conversation.
7. **Protect the result** (`references/characteristics.md` §5). Every characteristic that drove the decision gets a fitness function in CI, or it will erode.

When the user brings an existing system, start at step 1 anyway: most "should we split this" questions are really "we never agreed on the characteristics".

## Decision rules

- **Split test.** Pain inside one deployable (tangled modules, slow tests, every change touches everything) is a modularity problem: fix boundaries with domain modules and dependency fitness functions. Only genuinely different operational characteristics (scale, availability, security zone, release cadence) justify a new quantum. Ask: which characteristic differs, and would the two halves still share a database?
- **Transactions mark boundaries.** Entities that must change atomically belong in one service. If the design needs sagas everywhere, the boundaries are wrong; a saga or two is a tool, not a style.
- **Connascence across the wire must be weak** (`references/modularity.md` §3). Moving code further apart (module → service) requires weaker coupling at the same time: name/type-level contracts, never shared algorithms, value coupling or identity coupling across a network.
- **Reuse is coupling.** A canonical shared model library across services recreates orchestration-driven SOA; prefer duplication to coupling for domain code, and share only operational concerns (sidecar/mesh).
- **Responsiveness is not performance.** An async ack that returns in 25 ms instead of 3 s changes what the user waits for, not the work done. Valid only when nothing is needed back and errors have a way back to the user.
- **Know your p99, not your average**, before any chained synchronous design: ten 100 ms hops is a second; the long tail kills distributed performance (`references/distributed.md` §2).
- **Broker vs mediator.** Choreography for decoupling, scale and extensibility; orchestration when you need workflow state, error handling and restart. Orchestrate the critical business transaction, choreograph the side effects.
- **A style assumes practices.** Microservices without automated provisioning, CI/CD and observability fail on process, not on design. Check team and ops maturity before the scorecard.
- **Never present a decision without its business justification** (cost, time to market, user satisfaction, strategic position); without one it is probably not worth making.

## Style selection table (Fundamentals ch. 10-18)

★ = works against it, ★★★★★ = defining strength. Cost: more stars = cheaper. Quanta: shared DB or UI collapses "many" to 1.

| Style | Partition / quanta | Deploy | Elast | Evol | FT | Mod | Scale | Perf | Cost | Simp | Best for | Avoid when |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Layered | technical / 1 | ★ | ★ | ★ | ★ | ★ | ★ | ★★ | ★★★★★ | ★★★★★ | Small apps, tight budget, undecided style, team familiarity | Growth, agility/testability/deployability in top 3, most requests are pass-through sinkholes |
| Pipeline | technical / 1 | ★★ | ★ | ★★★ | ★ | ★★★ | ★ | ★★ | ★★★★★ | ★★★★★ | One-way processing: ETL, EDI, telemetry, integration flows | Interactive request/response domains |
| Microkernel | both / 1 | ★★★ | ★ | ★★★ | ★ | ★★★ | ★ | ★★★ | ★★★★★ | ★★★★ | Per-tenant/jurisdiction/client rules, product-style extensibility | Scale/elasticity/FT in top 3; plug-ins must call each other |
| Service-based | domain / 1-many | ★★★★ | ★★ | ★★★ | ★★★★ | ★★★★ | ★★★ | ★★★ | ★★★★ | ★★★ | 4-12 coarse domain services, ACID preserved, DDD without granularity pain; first target when leaving a monolith | Fine-grained elastic scaling of single capabilities; services need private data |
| Event-driven | technical / 1-many | ★★★ | ★★★ | ★★★★★ | ★★★★★ | ★★★★ | ★★★★★ | ★★★★★ | ★★★ | ★ | Reactive, bursty, extensible flows; hybrid ingredient for back-pressure | Strong consistency, deterministic testing, workflow certainty required |
| Space-based | both / 1-many | ★★★ | ★★★★★ | ★★★ | ★★★ | ★★★ | ★★★★★ | ★★★★★ | ★★ | ★ | > 10 k concurrent, unpredictable spikes (tickets, auctions, flash sales) | Hot data needs strong consistency; cannot test at real scale; thin budget |
| Orchestration SOA | technical / 1 | ★ | ★★★ | ★ | ★★★ | ★★★★ | ★★★ | ★★ | ★ | ★ | Historical; recognize it when "microservices" have a central ESB + canonical model + shared DB | Always, for new systems |
| Microservices | domain / many | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★ | ★★★★★ | ★★★★★ | ★★ | ★ | ★ | Several quanta with different characteristics, many teams, evolvability and elasticity in top 3, mature automation | Semantically coupled domain, small team, immature CI/CD/ops, performance-critical chatty workflows |

Testability and reliability are in the full scorecards in `references/styles.md` (testability: layered ★★, service-based ★★★★, event-driven ★★, space-based ★, microservices ★★★★).

## Shape of a good answer

1. Driving characteristics: the top 3, each with the requirement line or domain fact it came from.
2. Quanta: how many, what differs between them, which data each owns.
3. Two or three candidate styles with the trade-off each one *loses* on, not only what it wins.
4. Recommendation framed as least-worst, plus the first fitness function that protects it.
5. ADR skeleton (title, context, decision, consequences) ready to paste, and the top 2-3 risks scored with the matrix.

Push back when the question presupposes a topology ("how do we split this into microservices?") and the characteristics do not support it; the honest answer is often a modular monolith or service-based with a shared database, and the books give the rating lines to say so.

## ADR template (Fundamentals ch. 19)

```
# NN. <Short imperative title>                     e.g. 42. Use async messaging between Order and Payment
Status: Proposed | Accepted | Superseded by NN | RFC until <date>
Context: What forces this decision. Alternatives in one line each.
Decision: "We will ..." + technical justification + business justification (cost / time to market / user satisfaction / strategy).
Consequences: Impact, good and bad. Trade-offs already argued and accepted, so they are not re-litigated.
Compliance: How it is checked. Fitness function name, where it lives, when it runs, code changes it needs.
Notes: Author, approval date/by, superseded date, last modified by.
```

Self-approval limits to agree up front: cost threshold, cross-team impact, security impact. Store in one system of record; email only the context and a link, only to people it affects.

## Reference map

| Open | When the question is |
|---|---|
| `references/characteristics.md` | Which -ilities matter, how to extract them from requirements, how to measure them, fitness-function examples for Node/NestJS |
| `references/modularity.md` | Cohesion, coupling metrics, connascence ladder and refactoring rules, the quantum test for monolith vs distributed, domain vs technical partitioning, component discovery and the entity trap |
| `references/styles.md` | Per-style topology, variants, full scorecard, anti-patterns (sinkhole, data collisions, front controller), when to use and when not, the choosing-a-style procedure |
| `references/distributed.md` | The eight fallacies with counter-measures, brown-field microservices reference architecture, service mesh vs broker, governance, messaging patterns, CDC, async APIs, sync/async and saga rules |
| `references/decisions-and-risk.md` | Decision anti-patterns, what is architecturally significant, full ADR rules, risk matrix, risk storming, diagramming conventions |
| `references/cloud-native.md` | Kubernetes deployment layout, micro-architecture of gateways/frontends/security, multi-cloud options, cloud migration phases with strangler façade |

The `(Fundamentals ch. N)` / `(SolArch ch. N)` markers in the reference files are for your navigation and for the user when they ask where a rule comes from. Do not carry them into an analysis, review or ADR: a deliverable that cites a book the user never mentioned reads as noise. Same for phrases like "per the skill" or "the skill's template" — the user sees a document, not a skill.

Book vocabulary is a thinking tool, not output vocabulary. In a deliverable, say what you mean in plain engineering terms ("parts that must deploy, scale and fail together" rather than "quantum"; "automated CI check that guards this property" rather than a bare "fitness function"; "coupling where both sides must change together" rather than "connascence of X"). If a term earns its place, define it in one clause the first time. And never state as fact things about the team or system the user did not tell you (experience level, current transaction code, domain) — label them as assumptions.
