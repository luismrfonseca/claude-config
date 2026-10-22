# Architecture characteristics ("-ilities")

Contents: definition test · catalog (operational / structural / cross-cutting) · extracting from domain concerns and requirements · scoping to "top 3" · measuring · governing with fitness functions · Node/NestJS fitness-function examples.

## 1. What qualifies as an architecture characteristic (Fundamentals ch. 4)

A concern becomes an architecture characteristic only when **all three** hold:

| Test | Meaning | Example that passes / fails |
|---|---|---|
| Non-domain | It is about *how* the system succeeds, not *what* it does | "p99 < 300 ms" passes; "customers can refund an order" fails |
| Structural influence | Supporting it forces a structural choice (module, service, topology, infra) | In-house card processing → isolated payment module/service: passes. Third-party PSP with normal hygiene: fails (just design) |
| Critical to success | The system fails its purpose without it | Availability for a trading system passes; "nice to have" i18n for a single-country MVP fails |

Consequences of the definition:

- Every supported characteristic adds structure and complexity; the job is to pick the **fewest**, not the most. Too many → generic architecture that solves no problem well (the "Vasa" failure mode).
- Characteristics interact: security costs performance, elasticity costs simplicity and cost, consistency costs availability. Expect every pair to be a trade-off; if you cannot find the trade-off you have not looked hard enough (First Law, Fundamentals ch. 1).
- Aim for the **least worst** architecture, not the best one. Prefer designs that are cheap to iterate on over designs that try to be right the first time.
- Avoid the terms "non-functional requirements" (self-devaluing) and "quality attributes" (sounds post-hoc). Use "architecture characteristics" and define each one objectively for your org.

## 2. Catalog (Fundamentals ch. 4)

Explicit = written in requirements. Implicit = never written, assumed by the domain (availability, reliability, security, modularity almost always).

### Operational (overlaps with ops / DevOps)

| Characteristic | What to pin down | Typical measure |
|---|---|---|
| Availability | Required uptime window; what "up" means per endpoint | % uptime, error budget |
| Continuity / recoverability | DR capability; RTO / RPO | Time to restore, data loss window |
| Performance | Which flavor: response time, throughput, capacity, peak behavior | p50/p95/p99 latency, RPS, first render budget |
| Reliability / safety | Fail-safe vs fail-fast; cost of a failure (money, lives) | MTBF, incident cost |
| Robustness | Behavior under partial failure (network, dependency, hardware) | Degradation mode per dependency |
| Scalability | Performance stays acceptable as users/requests grow steadily | Latency vs concurrent users curve |
| Elasticity | Survives sudden bursts (scalable ≠ elastic: hotel booking is scalable; concert ticket drop is elastic) | Burst absorption time, autoscale lag |

### Structural (code and change)

| Characteristic | Meaning |
|---|---|
| Configurability | End users can change behavior through supported interfaces |
| Extensibility | Ease of plugging in new functionality |
| Installability / upgradeability | Ease of installing and of moving between versions |
| Leverageability / reuse | Sharing components across products |
| Localization | Multi-language, currency, units, multibyte |
| Maintainability | Ease of applying change and enhancements |
| Portability | Runs on more than one platform / database / cloud |
| Supportability | Logging, tracing and tooling needed to diagnose problems |
| Modularity, testability, deployability | Decomposition of the composite "agility"; the ones most often governed by fitness functions |

### Cross-cutting

Accessibility, archivability (retention/deletion rules), authentication, authorization (down to field level), legal/regulatory (GDPR, SOX, data residency), privacy (hide data even from internal staff/DBAs), security (encryption at rest / in transit), usability.

Nuances worth remembering: *interoperability* (integrates with other systems → implies published APIs) ≠ *compatibility* (adheres to industry/domain standards); *availability* ≠ *reliability* (UDP is available but not reliable); *learnability* means two different things (user learning curve vs self-tuning system). Define each term once in a shared glossary; most cross-team arguments about "performance" are really disagreements about which performance.

## 3. Extracting characteristics (Fundamentals ch. 5)

Three sources: (a) explicit requirements, (b) domain concerns voiced by stakeholders, (c) implicit domain knowledge you bring.

### Domain concern → characteristics translation

| Stakeholder says | Architect hears |
|---|---|
| Mergers & acquisitions, "we might buy/sell a unit" | Interoperability, scalability, adaptability, extensibility |
| Time to market | Agility = testability + deployability + modularity (never performance alone, never "agility" alone) |
| User satisfaction | Performance, availability, fault tolerance, testability, deployability, agility, security |
| Competitive advantage | Agility, testability, deployability, scalability, availability, fault tolerance |
| Tight time and budget | Simplicity, feasibility (sacrificial architecture is a valid answer) |
| "Regulatory deadline: end-of-day batch must finish on time" | Performance **and** availability, scalability, reliability, recoverability (restart from 85 %), auditability (correctness provable) |

Trap: locking onto the single most obvious -ility in a sentence. Decompose the concern into every -ility it needs to actually hold.

### Walk each requirement line and ask four questions

1. Does this line imply a number (users, bursts, data volume, deadline)? → explicit operational characteristic. Decode "thousands, maybe millions of users" into scalability; decode "mealtime rush" or "tickets go on sale at 10:00" into elasticity even if nobody wrote it.
2. Does it introduce an integration point? → reliability/robustness question: *should the system fail or degrade when that dependency is down?* Default to degrade; do not build brittleness by over-specifying.
3. Does it need **structure** to support it, or just a design pattern inside an existing structure? (Customizability can be a microkernel plug-in architecture *or* a Template Method inside a monolith. Both are valid; the choice is a trade-off on performance, coupling and cost.) Only the structural version is an architecture characteristic.
4. Does it only affect UX/design (usability, i18n, mobile layout)? → note it, do not promote it.

Then add the implicit ones the requirements never mention: availability, reliability, security baseline, modularity.

### Keep the list short

- Target ≤ 7 candidates. Then get stakeholders to pick the **top 3 in any order**; do not ask for a full ranking (consensus never arrives and the argument wastes weeks). The top 3 drive style selection and tie-breaking.
- Reverse test: "if we had to drop one, which?" Dropping explicit ones is usually safer than dropping implicit ones; what survives the cut is what you actually design for.
- Decide these with the people who will build and run it. An architect deciding alone is the Ivory Tower anti-pattern and produces characteristics nobody protects.

## 4. Measuring (Fundamentals ch. 6)

Three recurring problems: terms are not physics (no shared meaning), definitions vary even inside one company, and many characteristics are composites. Fix all three by writing an objective definition per characteristic and decomposing composites (agility → modularity + testability + deployability).

| Kind | Measure | Notes |
|---|---|---|
| Operational | Latency percentiles not averages (p99 catches the 1 % that is 10× slower); throughput; per-feature performance budgets (e.g. first render 500 ms, byte weight per page) | Mature teams model expected scale statistically and alert on deviation from the model rather than on a fixed number — a deviation means either the model or the system is wrong, both worth knowing |
| Structural | Cyclomatic complexity (CC = E − N + 2P); afferent/efferent coupling; instability; abstractness; distance from main sequence; LCOM; dependency cycles | Industry threshold CC < 10; authors prefer < 5. CC > 50 is unrecoverable by test coverage. Always ask: essential (domain) or accidental complexity? Metrics need interpretation, never auto-judgement |
| Process | Testability → coverage (necessary, not sufficient: 100 % with weak assertions proves nothing). Deployability → deploy success ratio, deploy duration, defects per deploy, lead time | If testability and deployability are top-3, that *is* an argument for more modularity and isolation at the architecture level |

Define performance **relative to** scalability: baseline latency at nominal load, plus acceptable latency at the target concurrent-user number. A single latency figure without a load figure is meaningless.

## 5. Governance with fitness functions (Fundamentals ch. 1, 6)

Definition: any mechanism giving an objective integrity assessment of one or more architecture characteristics. Not a framework; a perspective on tools you already have: unit tests, metrics, monitors, chaos experiments, CI gates.

Rules of use:

- Put them in CI so they run on every change; code review is too late (a week of reckless imports already damaged modularity).
- They guard the *important-but-not-urgent*: modularity, layer boundaries, cycles, dependency direction. Day-to-day urgency never protects these on its own.
- Developers must understand *why* each function exists before it is imposed. A fitness function nobody understands gets deleted or skipped.
- Think checklist, not bureaucracy: pilots and surgeons use them because details slip under routine, not because they are incompetent.
- Chaos engineering is a fitness function for robustness: the question is *when* something breaks, not *if*. Netflix Simian Army pattern: conformity checks (every service answers health/REST verbs), security checks (open ports, misconfig), janitor checks (orphan instances cost money).

### Fitness-function examples for a Node/NestJS codebase

| Characteristic | Mechanism |
|---|---|
| Modularity: no cycles between modules | `madge --circular src/` (or `dependency-cruiser` with `no-circular`) as a CI step |
| Layering / allowed dependencies | `dependency-cruiser` rules: `modules/billing/**` may not import `modules/orders/**` directly; controllers may not import repositories; nothing outside `modules/x/public/` imports module internals. ESLint `no-restricted-imports` or `boundaries` plugin for the same |
| Component contracts in a modular monolith | Each module exposes an `index.ts`/public facade; rule: cross-module imports only via the facade |
| Performance budget | k6/autocannon in CI against a seeded environment; fail when p95 of the critical endpoint exceeds budget at N virtual users |
| Deployability | Pipeline metric: deploy duration and rollback count; alert on regression |
| Testability | Coverage threshold per module in `jest.config` (`coverageThreshold`), plus mutation score if it matters |
| Availability / robustness | Health endpoints asserted per service (Terminus); synthetic probes; scheduled fault injection (kill a pod, add latency on a dependency) with SLO assertions |
| Service contract stability | Consumer-driven contract tests (Pact) run before the provider deploys |
| Data ownership in services | Query against DB grants: a service's DB role must not have access to another service's schema |
| Cost | Budget alarm per environment; fitness function fails the deploy when projected monthly cost crosses the threshold |

Use these as the "governance" line in each ADR: a decision without a fitness function is a decision that will quietly erode.
