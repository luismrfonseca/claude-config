# Modularity, coupling, connascence, quanta, components

Contents: cohesion ladder · coupling metrics · connascence (types, strength order, refactoring rules) · architecture quantum and the monolith-vs-distributed test · top-level partitioning (technical vs domain) · component identification loop and anti-patterns.

## 1. Cohesion (Fundamentals ch. 3)

Best → worst. Use it to judge whether a module (or a NestJS module / a candidate service) should exist as one unit.

| Level | Meaning | Smell / action |
|---|---|---|
| Functional | Everything in the module is needed for one purpose and nothing is missing | Target |
| Sequential | Output of one part is input of the next | Fine; pipeline-shaped |
| Communicational | Parts contribute to the same output / act on the same data (save record + send its email) | Acceptable |
| Procedural | Parts must run in a certain order | Watch for hidden ordering coupling |
| Temporal | Grouped only because they run at the same time (startup init) | Keep tiny and explicit |
| Logical | Related in kind, not function (`StringUtils`, `utils/`) | Classic `shared/` and `common/` dumping grounds; split by consumer |
| Coincidental | Same file, no relation | Split |

Rule from Constantine: splitting a functionally cohesive module only *adds* coupling between the halves. Before extracting `OrderMaintenance` out of `CustomerMaintenance`, ask: are those the only two operations, will the source module keep growing, and would the new module need so much customer knowledge that the split costs more coupling than it removes?

LCOM (lack of cohesion in methods) finds classes whose method sets never share fields: candidates that should have been several classes. Useful when hunting incidental coupling in shared utility classes before a monolith split; blind to logical fit, so interpret, don't obey.

## 2. Coupling metrics (Fundamentals ch. 3)

| Metric | Definition | Use |
|---|---|---|
| Afferent (Ca) | Incoming dependencies | Many → the module is a dependency hub; changing it is expensive |
| Efferent (Ce) | Outgoing dependencies | Many → the module breaks when others change |
| Instability I = Ce / (Ca + Ce) | 0 = stable, 1 = volatile | High I + many callers = danger |
| Abstractness A = abstract / total | Ratio of interfaces/abstract types | Too low → "zone of pain" (concrete, rigid); too high → "zone of uselessness" (`AbstractSingletonProxyFactoryBean`) |
| Distance from main sequence D = \|A + I − 1\| | Ideal is 0 | Fitness-function threshold per package; flag outliers |

Tools for Node: `dependency-cruiser` (rules + cycles + metrics), `madge` (cycles, graph), `ts-prune`, `knip`. Baseline before a migration; track trend, not absolute.

## 3. Connascence (Fundamentals ch. 3, 7)

Two elements are connascent when changing one forces a change in the other to keep the system correct. Use it instead of the 1970s coupling taxonomy because it tells you *how* things are coupled, not just that they are.

### Types, weakest → strongest (refactor downward on this list)

| # | Type | Agreement on | Typical Node/distributed example |
|---|---|---|---|
| 1 | Name (static) | Identifier names | Method names, JSON field names, event names. Most desirable; IDEs refactor it for free |
| 2 | Type (static) | Types | Shared TS types/DTO packages, protobuf schemas |
| 3 | Meaning / convention (static) | Semantics of values | Magic numbers, `status = 3`, `TRUE = 1` → fix with named constants / enums (CoM → CoN) |
| 4 | Position (static) | Order of values | Positional args, CSV columns, tuple-shaped events → fix with named params / objects |
| 5 | Algorithm (static) | Same algorithm on both sides | Both sides compute an HMAC/hash the same way; dual-written validation logic |
| 6 | Execution (dynamic) | Order of calls | `setRecipient()` before `send()`; init order |
| 7 | Timing (dynamic) | When things run | Race conditions; "the cron must finish before the report job" |
| 8 | Value (dynamic) | Several values must change together | Multi-DB atomic update; rectangle corners; **distributed transactions** |
| 9 | Identity (dynamic) | Must reference the same entity instance | Two services sharing one queue or one row as the source of truth |

Static connascence is discoverable by reading code and cheap to fix; dynamic connascence needs runtime analysis and is expensive. Prefer static. Synchronous calls between services add a tenth kind: *synchronous (quantum) connascence*, see §4.

### Three properties that decide how bad it is

- **Strength**: position on the ladder above; the higher the number the harder to refactor.
- **Locality**: distance between the coupled elements. Strong connascence inside one module is tolerable; the same connascence across module or service boundaries is a defect. The rule of locality: as distance increases, only weaker forms are acceptable.
- **Degree**: how many elements are affected. Low-degree dynamic connascence in a small system is fine; code bases grow, so it won't stay small.

### Refactoring rules (Page-Jones, Weirich)

1. Minimize overall connascence by encapsulating into elements with clear boundaries.
2. Minimize connascence that crosses boundaries (module, service, team).
3. Maximize connascence *inside* boundaries (that is what cohesion is).
4. Rule of degree: convert strong forms to weaker ones (CoM → CoN, CoP → CoN, CoA → one owner + published contract, CoV across services → eventual consistency or a single owner).
5. Rule of locality: when you move something further away (module → service), demand a weaker form of connascence at the same time. Extracting a service while keeping CoA/CoV across the wire is the worst of both worlds.

Limit: classic connascence measures code hygiene, not architecture-level choices like sync vs async; that is what the quantum adds.

## 4. Architecture quantum (Fundamentals ch. 7)

Definition: an **independently deployable** artifact with **high functional cohesion** and **synchronous connascence** (everything it must call synchronously to work, including its database).

- Independently deployable means it carries everything it needs to function. A shared database pulls everything that uses it into **one** quantum. Most legacy systems and most "microservices on one Postgres" are a quantum of one.
- Synchronous calls bind the operational characteristics of caller and callee for the duration of the call: if A (elastic, p99 50 ms) calls B (one worker, 500 ms/payment) synchronously, A has B's characteristics during peak. Buffer with a queue/async to let them differ (`Auction` → queue → `Payment`), at the cost of eventual consistency and more moving parts.
- Scope architecture characteristics **per quantum**, not per system. "The system must be highly available" is meaningless once different parts need different things.

### The test that decides monolith vs distributed

1. List candidate components (see §6).
2. For each, list its top characteristics (availability, elasticity, reliability, security, performance, ...).
3. Group components that can share one set of characteristics and one deployment cadence → one quantum each.
4. **One quantum → build a monolith** (modular, domain-partitioned). The advantages of distribution buy you nothing and the fallacies (see distributed.md) cost you plenty.
5. **Several quanta with genuinely different operational characteristics → distributed**, with the quanta as the first cut of service boundaries.

Worked example (auction kata): bidder read-only streams (availability, scalability, performance), auctioneer input (availability, reliability, elasticity, security, far more critical than any single bidder), online bidders (reliability, availability, scalability, elasticity). Three quanta → distributed, and the `BidCapture` component was split into `BidCapture` and `AuctioneerCapture` purely because their characteristics differ, not for functional reasons.

Apply this to a NestJS monolith: if `reports`, `webhooks-ingest` and `checkout` need different scaling and availability, those are quanta candidates. If they all run at the same scale and ship together, the pain you feel is modularity, not topology; fix the module boundaries first.

## 5. Top-level partitioning (Fundamentals ch. 8)

The first structural decision: what is the top level of the tree?

| | Technical partitioning (layers: controllers/, services/, repositories/) | Domain partitioning (modules: orders/, billing/, catalog/) |
|---|---|---|
| Organizing principle | Separation of technical concerns | Workflows / bounded contexts |
| Where a feature lives | Smeared across every layer | Inside one component (which may have layers internally) |
| Change impact | A persistence change touches one layer; a *feature* change touches all | A feature change is local; cross-cutting technical change touches all modules |
| Coupling | High global coupling, usually one shared schema | Lower; data boundaries visible early |
| Team shape (Conway) | Backend team / DBA team / frontend team | Cross-functional team per domain (Inverse Conway Maneuver) |
| Migration to services | Hard: data and behavior must be untangled first | Easy: each module is a service candidate |
| Downsides | Duplicated domain concepts across layers; feature work = coordination across teams | Customization / shared code appears in several places; needs a discipline for shared kernels |
| Matching styles | Layered, microkernel core, orchestration-driven SOA | Modular monolith, service-based, microservices, event-driven |

Industry trend and the book's recommendation: domain partitioning for both monoliths and distributed systems. A domain-partitioned modular monolith (one deployable, one DB, domain modules with enforced boundaries) is the cheapest architecture that stays migratable.

NestJS mapping: one `@Module` per domain, each exporting a small facade; a `shared/` module limited to truly generic infrastructure; fitness functions forbidding `modules/a/**` from importing `modules/b/internal/**` (see characteristics.md §5).

## 6. Component identification (Fundamentals ch. 8)

Components are the physical packaging of modules (package, library, layer, event processor, service). The architect owns the component level; class design belongs to tech leads and developers. The first component set is never right; iterate:

1. **Identify initial components** from the chosen partitioning.
2. **Assign requirements / stories** to them; merge, split, create as needed. Coarse is fine.
3. **Analyze roles and responsibilities** so granularity matches the actors and their actions.
4. **Analyze architecture characteristics** per component: two components with identical function but different characteristics (hundreds of concurrent users vs five) must be split. This is where quanta appear.
5. **Restructure** with developer feedback; repeat.

Granularity: too fine → chatty inter-component communication to do anything; too coarse → internal coupling that hurts deployability, testability and modularity.

### Discovery techniques

| Technique | When |
|---|---|
| Actor/actions | Generic default; requirements have distinct roles and actions; works for monoliths and distributed |
| Event storming (DDD) | You will communicate with events/messages; outputs the message catalog for free |
| Workflow approach | Like event storming without committing to messaging; components around activities of each role |

### Anti-pattern: the entity trap

`AuctionManager`, `ItemManager`, `BidManager` — one component per noun with CRUD methods. That is an ORM of the schema, not an architecture: it ignores workflows, produces components too coarse to guide packaging, and signals nobody modeled what the system *does*. If the system really is CRUD, use a scaffolding framework and skip architecture. Otherwise model workflows (`BidCapture`, `BidStreamer`, `AuctionSession`, `Payment`) and let entities live inside them.

## 7. Quick checklist before proposing a split

- Is the pain coupling inside one deployable (fix: domain partitioning + fitness functions) or genuinely different operational characteristics (fix: separate quantum)?
- What connascence would cross the new boundary? Anything ≥ Algorithm, Value or Identity across a network → redesign the contract first.
- Will the two halves share a database? If yes you have not created a new quantum, only a network hop.
- Can the new boundary be owned by one team (Conway)?
- Which fitness function proves the boundary holds a year from now?
