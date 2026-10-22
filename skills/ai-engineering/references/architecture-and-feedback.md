# Reference Architecture, Observability, and User Feedback

Synthesis of Huyen, *AI Engineering*, ch. 10.

## Contents

1. [Grow the architecture step by step](#1-grow-the-architecture-step-by-step)
2. [Step 1: context construction](#2-step-1-context-construction)
3. [Step 2: guardrails](#3-step-2-guardrails)
4. [Step 3: router and gateway](#4-step-3-router-and-gateway)
5. [Step 4: caches](#5-step-4-caches)
6. [Step 5: agent patterns and write actions](#6-step-5-agent-patterns-and-write-actions)
7. [Observability](#7-observability)
8. [Orchestration](#8-orchestration)
9. [User feedback: signals](#9-user-feedback-signals)
10. [Feedback design](#10-feedback-design)
11. [Feedback limitations](#11-feedback-limitations)
12. [NestJS mapping](#12-nestjs-mapping)

---

## 1. Grow the architecture step by step

Start with `query → model API → response`. Add components only when a need appears, in roughly this order (Huyen ch. 10): context construction → guardrails → router + gateway → caches → agentic loops and write actions, with observability designed in from day one rather than bolted on. Every component adds capability and a new failure mode; keep the system as small as the requirements allow.

Final shape:

```
                 ┌──────────────── input guardrails (PII redaction, injection/scope checks) ──────────┐
 query ─► router (intent / scope / next-action) ─► context construction (RAG, tools, query rewrite) ─► model gateway ─► output guardrails ─► response
                      │                                   ▲                                         (catalog, auth,      (format, safety,
                      ▼                                   │ read-only actions                         fallback, logging)   verification)   │
                 cache (exact / semantic) ◄───────────────┘                                                                                │
                                                                                                  loop back for agents ◄───────────────────┘
                                                                                                  write actions (gated, human-approved)
```

## 2. Step 1: context construction

Give the model what it needs per query: retrieval (text, images, tables/SQL), tools (search, APIs), query rewriting, memory. Design detail is in `rag-and-agents.md`. Providers bundle some of this (file upload, tool use) with limits on document count, chunking control, parallel tool execution, and long-running jobs; a dedicated retrieval layer removes those limits at the cost of owning it.

## 3. Step 2: guardrails

**Input guardrails** protect against (a) leaking private data to external APIs and (b) prompts that compromise the system (see `prompting.md` for attack taxonomy).
- Leak paths: users paste secrets; developers embed internal policy/data in system prompts; tools retrieve private records into context.
- Detect sensitive classes you define (IDs, phone numbers, account numbers, faces, IP keywords). Then block, or **mask with placeholders and unmask the response** through a reversible map (`[ACCESS_TOKEN]` → original) so the model still helps without seeing the secret.

**Output guardrails** catch failures and encode the policy for each:

| Failure class | Examples | Typical handling |
|---|---|---|
| Quality | empty response; malformed JSON; unsupported by context; just bad | retry (bounded), repair, or regenerate; parallel duplicate calls to pick the better one when latency matters more than cost |
| Security | toxicity; PII in output; content that triggers tool/code execution; brand-damaging claims | block, redact, or fall back to a stock response |
| Escalation | sentiment turns angry; conversation exceeds N turns; specific trigger phrases | hand off to a human |

Track the **false refusal rate** alongside violations; an over-cautious system is also broken.

Trade-offs: guardrails add latency (some teams skip them for that reason, which is a risky bet; run them async or on partial outputs where possible), and they fit streaming poorly (tokens reach the user before a full-response check can run; design for retraction). Provider APIs ship moderation and some guardrails; self-hosting removes the external-leak class of input risk but leaves everything else to you. Off-the-shelf guardrail toolkits and moderation endpoints exist; most cover both inputs and outputs.

## 4. Step 3: router and gateway

**Router** = usually an intent classifier (small model or trained classifier; must be fast and cheap because you may run several). Uses:
- Send each intent to the right solution: FAQ page, specialised prompt/model, human operator.
- Reject out-of-scope requests with a stock reply before spending a model call.
- Detect ambiguity and ask for clarification ("Freezing": account or weather?).
- Next-action prediction in agents (code interpreter vs search; which memory tier to consult).
- Cost: cheap model for easy queries, expensive model for hard ones.
Routing typically happens before retrieval (is it in scope? does it need retrieval?) and sometimes after (should this go to a human?). When routing across models with different context limits, truncate or re-route when retrieved context overflows.

**Model gateway** = one internal interface in front of all models (self-hosted and commercial). Benefits: one place to change when a provider API changes; central credential custody (apps never hold provider keys); per-app/per-user access control and quotas; usage metering and cost caps; fallback and retry policies across providers for rate limits and outages; load balancing; logging; sometimes caching and guardrails. Simple enough to build in-house, and many open-source/commercial gateways exist at time of writing. The same abstraction is useful for tools (a tool gateway), though less common.

## 5. Step 4: caches

(Inference-level KV and prompt caches live inside the model service; see `inference-optimization.md`.)

**Exact cache**: key = exact request (or exact retrieval query for vector-search results). Worth most for multi-step or slow pipelines (CoT, retrieval, SQL, web search). Store in memory, Redis, or Postgres with tiered storage; evict with LRU/LFU/FIFO. Decide cacheability per query: user-specific ("my order status") and time-sensitive ("weather") queries must not be cached; some teams train a classifier for this. **Leak hazard**: a generic-looking question answered with user-specific context ("return policy" that depends on membership tier) cached and served to another user. Key caches by tenant/user when context is personalised.

**Semantic cache**: embed the query, nearest-neighbour search over cached queries, return the cached answer above a similarity threshold. Raises hit rate, but every component can fail (embedding quality, vector search, threshold tuning), wrong hits return wrong answers, and the vector search itself costs latency and compute. Adopt only with evidence of a high hit rate on near-duplicate queries, and evaluate the error rate it introduces.

## 6. Step 5: agent patterns and write actions

Loop the output back: the system evaluates its own response, decides more retrieval or another step is needed, and re-enters the pipeline (same or different model). Then add **write actions** (send email, update order, transfer money), which multiply both capability and risk. Gate them: least privilege, explicit human approval for impactful actions, per-action automation levels, audit logs. Details in `rag-and-agents.md`.

## 7. Observability

Monitoring's goals mirror evaluation's: mitigate risk (failures, attacks, drift) and find improvement and cost opportunities. Evaluation and monitoring must agree (good offline ⇒ good online), and production issues feed back into the eval sets. Health of the practice: MTTD, MTTR, change failure rate (if you can't state your CFR, instrument first).

**Metrics**: design them around the failure modes you fear, not the other way round.
- Format: invalid JSON rate, and how many are trivially repairable.
- Quality: factual consistency vs context, relevance, conciseness, other app-specific judge scores (spot-check or exhaustive, by budget).
- Safety: toxicity, PII in inputs/outputs, guardrail trigger rate, refusal rate, anomalous queries.
- Conversational proxies: early-stop rate, turns per conversation, input/output token lengths and their drift, output token distribution (diversity).
- Component metrics: retrieval precision/relevance, vector DB storage and query time, tool success rates.
- Latency: TTFT, TPOT, total, per user and per endpoint; cost: requests, input/output tokens, TPS, cache hit rate and cache cost, rate-limit headroom.
- Correlate metrics with each other and with the business north star (DAU, session length, subscriptions) to learn what to optimise and what to ignore.
- Slice everything by user, release, prompt/chain version and type, model version, time.

**Logs**: log everything, because you don't know what you'll need: full config (endpoint, model name, sampling settings, prompt template version), user query, final rendered prompt, intermediate outputs, tool calls and tool outputs, component start/end/crash events, with IDs and tags for correlation. Logs must be available within minutes to be useful. Read a sample of production conversations by hand daily; your definition of "good" sharpens with exposure and so do prompts and evals.

**Traces**: reconstruct each request's path (routing → retrieval → prompt → model → guardrails → tools) with per-step latency and cost, so a bad answer can be attributed to the step that caused it.

**Drift**: alert on system prompt changes (template edits, "typo fixes"); watch for user behaviour changes (users learn to prompt more tersely, which shifts length metrics); detect silent provider model updates by re-running a fixed canary eval set on a schedule and comparing to the pinned baseline.

## 8. Orchestration

An orchestrator defines components (models via the gateway, data sources, tools, evaluators) and chains them: process query → retrieve → build prompt → generate → evaluate → return or escalate. It should validate data shapes between steps and surface broken flows. Run independent steps in parallel (routing and PII scrubbing can run concurrently). Start without a framework; add one later if it earns its keep, judged on integration/extensibility, support for branching/parallel/error handling, and absence of hidden API calls or added latency. A good `NestJS` module structure with typed DTOs between steps is already an orchestrator for most products.

## 9. User feedback: signals

Feedback is proprietary data and the engine of the data flywheel; it is also user data, so privacy and disclosure apply.

**Explicit**: thumbs, stars, "did this solve it?", pairwise "better/worse/same" after regeneration. Easy to interpret, sparse, response-biased.

**Implicit conversational** (abundant, noisy; validate interpretations with user studies):

| Signal | Reading |
|---|---|
| Early termination (stop generation, leave, ignore the agent's question) | conversation going badly |
| Corrections ("No, …", "I meant …"), rephrasing | model missed the intent |
| Pointed corrections ("Bill is the suspect, not the victim"), action nudges ("also check their GitHub") | actionable error data, especially for agents |
| "Are you sure?", "show sources" | missing detail or distrust |
| User edits the output (code, draft) | strong negative on the original; (original, edited) = a preference pair |
| Complaints (wrong, irrelevant, too long, not grounded, not specific, hedging) | cluster them; each cluster maps to a prompt or retrieval fix |
| Sentiment trajectory | anger rising = escalate; anger → relief = resolved |
| Model refusal / "I don't know" rate | user probably unhappy |
| Regeneration | dissatisfaction or curiosity; stronger under usage-based billing; follow with a comparative prompt |
| Delete / rename / share / bookmark | delete ≈ bad; rename ≈ good content, bad title; share is ambiguous (study your users) |
| Turns per conversation, dialogue diversity | long + repetitive = stuck in a loop; long can be good for companions, bad for support |

Combine signals to disambiguate (share followed by rephrase ≠ share followed by bookmark).

## 10. Feedback design

**When**: throughout the journey, never intrusive. Onboarding calibration (optional unless necessary); when something goes wrong (downvote, regenerate, switch model, free-text correction, let the user fix the output and still finish the task, hand off to a human); when the model is unsure (show two candidates, side by side or truncated with expand; ask "same or different?" on uncertain matches). Positive feedback is debated: Apple's guidance says good results should be the default so don't ask; product teams find it reveals which features users love. Compromise: ask a small random slice of users (e.g., 1%) and watch for sampling bias.

**How**: in the workflow, zero extra work, easy to ignore. Good patterns: a grid of candidates where upscale/variations/regenerate each carry a different signal; inline suggestions accepted with Tab or ignored by typing; tracking whether a drafted email was sent. Standalone chat apps lack this integration and get poorer feedback than embedded features. Capture context (previous 5–10 turns) with consent, via terms of service or an explicit "share recent interaction with this report" flow. Tell users how feedback is used (personalisation, statistics, training) and be truthful about what leaves the device. Don't ask users to judge what they can't (two answers to a stats question with no "I don't know"). Label options unambiguously (an emoji scale rendered in reverse order produced 1-star ratings with glowing comments). Decide whether feedback is private or public: private is more candid; public aids discovery.

## 11. Feedback limitations

- **Leniency bias**: people rate high to avoid follow-up questions; five stars becomes the default and four means "something was wrong". Inspect rating distributions; replace numbers with descriptive options.
- **Randomness**: unread side-by-side comparisons, random variation picks.
- **Position bias**: first option gets more clicks; randomise order or model the position effect.
- **Preference biases**: longer answers, most recent answer.
- **Degenerate feedback loops**: you only get feedback on what you show; what ranks higher gets more clicks, gets ranked higher (popularity bias, filter bubbles). A small vocal group can steer the product (the "cat photo" spiral), and training on feedback indiscriminately breeds sycophancy: the model learns to tell users what they want to hear. Understand the biases before acting on the data.

## 12. NestJS mapping

| Component | Where it lives |
|---|---|
| Gateway | `LlmGatewayModule`: provider adapters behind one interface, credentials in config, per-caller quotas, retry/fallback, structured logging interceptor |
| Router | `IntentService` (small model or classifier) invoked by a guard/interceptor before the main pipeline |
| Context construction | `RetrievalService` (search engine / pgvector), `QueryRewriteService`, tool providers with read-only credentials |
| Guardrails | request pipe for PII masking + injection heuristics; response interceptor for schema validation, PII, toxicity; escalation events to the support queue |
| Caches | Redis exact cache keyed by (tenant, prompt version, normalised input); semantic cache only after measurement |
| Observability | per-request trace ID through all steps; structured logs of config/prompt/outputs/tool calls; metrics exported per slice; nightly canary eval job against pinned prompts |
| Feedback | feedback entity linked to trace ID, with signal type, explicit/implicit, context snapshot under consent; export to the eval dataset builder |
