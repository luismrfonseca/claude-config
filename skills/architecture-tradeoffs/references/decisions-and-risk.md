# Architecture decisions, risk analysis, communicating the result

Contents: §1 decision anti-patterns · §2 what is architecturally significant · §3 ADR format and rules · §4 risk matrix and assessment · §5 risk storming · §6 diagramming and presenting (brief).

## 1. Decision anti-patterns (Fundamentals ch. 19)

They appear in sequence; beating one exposes the next.

| Anti-pattern | Symptom | Fix |
|---|---|---|
| Covering Your Assets | Decision deferred out of fear of being wrong | Decide at the **last responsible moment**: enough information to justify, not so late that teams are blocked. Stay in the loop with implementers so a wrong call is corrected in days (e.g. a replicated read-only cache decision reversed when teams find it blows the memory budget) |
| Groundhog Day | The same decision gets re-litigated because nobody knows *why* it was made | Record technical **and** business justification. Business justifications that survive: cost, time to market, user satisfaction, strategic positioning. Match the one the stakeholders actually care about; a cost argument lands badly with a stakeholder who cares about speed. No business value → probably not a decision worth making |
| Email-Driven Architecture | People never saw, lost, or forgot the decision | One system of record (wiki/repo). Email carries only the context and a link ("decision about inter-service communication that directly impacts you: <link>"), and goes only to people it directly impacts |

## 2. What counts as architecturally significant (Fundamentals ch. 19)

A decision earns an ADR when it affects any of: **structure** (style/pattern, e.g. sharing data between two services breaks bounded contexts), **architecture characteristics** (a tech choice made for performance when performance is a driver *is* an architecture decision), **dependencies** (coupling points between components/services), **interfaces** (gateways, contracts, versioning and deprecation strategy), **construction techniques** (platforms, frameworks, processes that constrain the architecture). "It names a specific technology" is not a reason to skip the ADR.

Guide, don't dictate: "use a reactive frontend framework" guides; "use React" decides for the team. Decide the specific technology only when a characteristic depends on it.

## 3. ADR format and rules (Fundamentals ch. 19)

Short (1–2 pages), one file/page per decision, numbered sequentially, written in Markdown/AsciiDoc or a wiki template. Sections:

| Section | Content | Rules |
|---|---|---|
| Title | `NN. <decision in a short phrase>` ("42. Asynchronous messaging between Order and Payment services") | Unambiguous and short |
| Status | Proposed / Accepted / Superseded by NN / RFC with deadline | Superseding ADR says "supersedes 42"; superseded one says "superseded by 68". Only Accepted ADRs get superseded; a Proposed one is edited. RFC always carries a deadline date to kill analysis paralysis |
| Context | The forces: what situation forces this decision; the alternatives in one or two sentences | Doubles as architecture documentation of that area. Add an *Alternatives* section only when each alternative needs real analysis |
| Decision | Imperative voice: "We will use …", plus the full justification | Never "I think X would be best". Spend the words on **why**; how is visible in the code later, why is not (the gRPC-chosen-for-latency replaced by messaging and timing out story) |
| Consequences | Impact, good and bad, and the trade-off analysis that was already settled | Write down the objection you already answered ("what if a bad word is posted async?" → stakeholders accepted responsiveness over sync feedback) so it is not re-raised |
| Compliance | How the decision will be checked: manual review or an automated fitness function, what it checks, where it lives, when it runs, what code changes it needs (annotations, folder rules) | A decision with no compliance line decays |
| Notes | Author, approval date/by, superseded date, last modified by/when | Keep even when the repo has history; the repo is not the only reader |

Self-approval criteria to agree with your lead/ARB up front: cost (licenses, infra, effort × FTE rate above a threshold), cross-team impact, security implications. Any of these → status Proposed and someone else approves.

Storage: wiki or shared directory, structured `application/{common,<app>}`, `integration/`, `enterprise/`. ADRs in the app repo are fine for a single team but exclude non-repo readers and cannot hold integration/enterprise decisions. ADRs are also the right vehicle for **standards**: a standard whose Context and Decision cannot justify it should not exist.

### Example shape (auction system, ADR 76, abbreviated)

Context: bid capture must forward bids to the bid streamer and bid tracker; options are p2p messaging, pub/sub, or REST through the API layer. Decision: pub/sub, because the producer needs nothing back, queues preserve bid order for the stream, and new consumers of bid data can subscribe without changes. Consequences: brokers must be clustered and HA; internal bid events bypass API-layer security checks (ARB accepted this trade-off on review). Compliance: periodic code/design review of the messaging path.

## 4. Risk matrix and risk assessment (Fundamentals ch. 20)

Score = impact (1 low, 2 medium, 3 high) × likelihood (1, 2, 3).

| | Likelihood 1 | 2 | 3 |
|---|---|---|---|
| **Impact 3** | 3 (medium) | 6 (high) | 9 (high) |
| **Impact 2** | 2 (low) | 4 (medium) | 6 (high) |
| **Impact 1** | 1 (low) | 2 (low) | 3 (medium) |

Rules:
- Assess **impact first, then likelihood**. "The DB going down is catastrophic" (impact 3) becomes a 3 (medium) once you note it is clustered (likelihood 1).
- **Unproven or unknown technology is always a 9**; the matrix cannot price what nobody understands. "What is a Redis cache?" from a developer in the room is signal, not noise.
- Build a **risk assessment** grid: rows = risk dimensions (availability, scalability, performance, security, data integrity, ...), columns = domain areas/services, cells = matrix scores; sum per row and per column to find the worst dimension and the riskiest area.
- Present filtered: show only high cells when the meeting is about high risk.
- Show **direction**: `+`/`−` next to the score, or an arrow with the number it is trending to (6 → 4). Arrows alone are read both ways by half the audience. Direction data comes from fitness-function measurements over time.

## 5. Risk storming (Fundamentals ch. 20)

Collaborative because no single person sees all the risk; include senior developers and tech leads, not only architects. Needs an up-to-date architecture diagram (full system for holistic sessions, a contextual slice for focused ones).

1. **Identification (individual, 1–2 days before)**: invitation carries the diagram, the single dimension under review (performance, scalability, availability incl. transitive dependencies, data loss, SPOF, security, unproven tech), date and place. Each participant scores areas alone using the matrix. One dimension per session whenever possible.
2. **Consensus (together)**: everyone sticks scores on the diagram. Discuss only disagreements and lone scores. Both directions matter: the lone "9 on push servers" came from someone who had watched them fall over under load; the lone "6 on the load balancer" came down to 3 once likelihood was argued.
3. **Mitigation (together, then with the business)**: change the architecture (split a DB, add a queue for back-pressure, add a cache, separate gateways). Mitigation costs money, so the stakeholder decides cost vs risk; have a cheaper partial mitigation ready (split without clustering at 40 % of the cost).

Repeat after major features or each iteration; it is a continuous practice. Also works on user stories (impact of not finishing × likelihood of not finishing).

Worked case (nurse triage system) to pattern-match against your own system:

| Dimension | Finding | Mitigation |
|---|---|---|
| Availability | Single central DB: impact 3 × likelihood 2 = 6, because call routing dies without it. External diagnostics engine: 9 (unknown likelihood). Records exchange: 2 (not required to operate) | Split DB into clustered profile DB + single-instance case-notes DB (also improved security); get the SLA/SLO numbers for externals (99.99 % = 52.6 min/yr; 99.9 % = 8.77 h/yr) and write them on the diagram |
| Elasticity | External engine capped at 500 req/s vs hundreds of thousands of self-service users in flu season: 9 | Queues for back-pressure, two channels so nurses outrank self-service (ambulance pattern), outbreak-question cache so epidemic traffic never hits the engine |
| Security | One gateway for admin, self-service and nurses in front of HIPAA data: impact 3 × likelihood 2 = 6 (the architect had it at 2 and was convinced otherwise) | Separate gateway per user class so admin/self-service paths cannot reach the records interface |

## 6. Diagramming and presenting (Fundamentals ch. 21), brief

- **Representational consistency**: show where a part sits in the whole before zooming into it.
- Stay low-fidelity (whiteboard, tablet) until the design has iterated; a two-hour diagram breeds *irrational artifact attachment* proportional to time spent.
- Conventions that readers already expect: solid line = synchronous, dotted = asynchronous; 3-D box = deployable, rectangle = container; arrows for direction; label everything; add a key when any shape is ambiguous; use color to separate different services from instances of one service.
- Standards: UML class and sequence diagrams still earn their keep; C4 (context, container, component, class) suits monoliths better than microservices; ArchiMate is the lightweight enterprise option.
- Presenting: slides are half the story, the speaker is the other half. Use incremental builds instead of bullet walls, hide slide boundaries with uniform transitions and mark topic changes with a distinct one, and drop in a blank slide when you want the room looking at you. An infodeck (read, not presented) is a different artifact and must be self-contained.
