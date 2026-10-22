---
name: ai-engineering
description: Practical playbook for building LLM features into products, distilled from Chip Huyen's "AI Engineering" for senior backend engineers (Node.js/NestJS/AWS). Use it whenever a task involves planning or scoping an AI/LLM feature, choosing a model or provider (API vs self-hosted, open vs commercial), building an evaluation pipeline or answering "how do I know if this prompt/model is better", writing or hardening prompts, defending against prompt injection or jailbreaks, designing RAG (chunking, retrieval, hybrid search, reranking, query rewriting), building agents and tool use, deciding RAG vs finetuning, optimising inference cost or latency (TTFT, TPOT, caching, batching, quantisation), adding routing, gateways, guardrails or caches, or collecting user feedback. Trigger it even on vague asks like "the chatbot hallucinates", "the LLM is too slow/expensive", "add AI to feature X", "our support bot gives wrong answers", or "should we finetune".
---

# AI Engineering

## Core stance

1. **Eval first.** Define criteria, rubric and a sliced eval set before changing anything; otherwise you cannot tell improvement from noise. An AI feature you cannot evaluate is worse than one you never shipped.
2. **Climb the ladder in order: prompting → few-shot → simple RAG (BM25) → better retrieval → finetuning.** Each step only when the eval shows the previous one is exhausted. Most "prompting doesn't work" stories are unsystematic prompting.
3. **The model is a volatile dependency with a contract.** Pin versions, put it behind a gateway, keep a regression eval set, and expect prices, quality and behaviour to shift under you; providers also change models silently.
4. **Cost and latency are first-class requirements**, not afterthoughts: measure TTFT, TPOT, cost per request and goodput from the first prototype. Output tokens dominate both.
5. **Treat every piece of retrieved or tool-returned text as untrusted input.** Instruction hierarchy, least privilege and human approval for write actions are architecture, not prompt tricks.
6. **Ship the product, then collect data, then invest in models**: the data flywheel from real usage is the moat; technology is shared.

Read the matching reference file before giving a non-trivial answer; the workflows below are the index, the references hold the detail and the numbers.

The `(Huyen ch. N)` markers in the references are for your navigation and for when the user asks where a rule comes from. Keep them, and phrasing like "the skill's template" or "per the playbook", out of plans and design docs — the user sees a document, not a skill.

## Quick triage

| Symptom | Start with |
|---|---|
| "It hallucinates / gives wrong answers" | faithfulness eval + hallucination slice (`evaluation.md` §8); is it facts (→ RAG) or form (→ prompt/finetune)? (`finetuning-and-data.md` §3); shorter answers, cite sources (`prompting.md` §4) |
| "Too slow / too expensive" | diagnosis playbook (`inference-optimization.md` §8): measure, cut output, cache prefixes, route to a smaller model |
| "Add AI to feature X" | scope + usefulness threshold (`planning-and-model-selection.md` §1–3), then eval pipeline before prompting |
| "Is the new prompt/model better?" | eval set size + judge setup (`evaluation.md` §3, §11) |
| "Users are injecting instructions / bot goes off-script" | defences (`prompting.md` §9), router scope control and guardrails (`architecture-and-feedback.md` §3–4) |
| "Should we finetune?" | decision tree below and `finetuning-and-data.md` §2–4 |
| "Agent does the wrong thing / loops" | plan–validate–execute split and failure-mode eval (`rag-and-agents.md` §10, §14) |

## Workflows

### Scope an AI feature (`references/planning-and-model-selection.md`)
- Why build it: existential / profit / exploration? Buy unless it's core.
- Role of AI: critical vs complementary; reactive vs proactive; dynamic vs static. Role of humans: crawl (AI drafts, human acts) → walk (internal users) → run (external), promoted on measured evidence.
- Business metric and usefulness threshold (quality, latency, cost) written down before model choice. Map eval metric → business metric ("80% faithfulness ⇒ automate 30% of tickets").
- Evaluate off-the-shelf first; budget for the last mile (80% in a month, 95% takes months); plan maintenance (model swaps, regulation, IP).

### Select a model (`references/planning-and-model-selection.md`)
Funnel: **hard attributes** (licence, privacy/residency, modality, context, latency and cost ceilings, self-host requirement) → **public benchmarks/leaderboards** to shortlist 3–5 (contaminated, aggregated, setup-sensitive: prune only) → **your own evals** on your prompts (this decides) → **online eval** in production. Start from the strongest model to prove feasibility, then work backwards to the cheapest that clears the bar. Keep a candidates matrix and re-run it when the landscape moves. Build vs buy hinges on privacy, lineage, functionality (structured output, tools, logprobs, finetuning access), cost at your volume, control/transparency, and edge needs.

### Build an eval pipeline (`references/evaluation.md`)
1. Criteria per feature (2–3): domain capability, generation quality (faithfulness, safety, app-specific traits), instruction following, cost/latency. Define what the system must *not* do.
2. Method per criterion: exact where possible (functional tests, execution, schema checks, classification metrics, NLI for faithfulness) → AI judge for the rest (classification labels over numbers, 1–5 max, examples per level, temperature 0, pinned model, both orderings for pairwise) → human spot-checks as North Star.
3. Judge is model + prompt + sampling; version it; never trust a judge whose prompt you can't see; watch self-, position- and verbosity bias.
4. Data: real production inputs, sliced (distribution, known failures, typos, out-of-scope, adversarial). ~100 examples detects a 10-point gap; ~1,000 for 3 points. Bootstrap to check stability.
5. Evaluate each component and end-to-end, per turn and per task. Iterate on the pipeline, but log every config so score changes mean app changes.

### Prompt engineering checklist (`references/prompting.md`)
- Explicit task, scoring/format rules, tie-breaks, "unsure" behaviour; persona when perspective matters; compact examples; end-of-input marker; forbid preambles.
- Context: give facts rather than relying on recall; instructions at start and end (middle gets lost); test long contexts with private needle-in-haystack.
- Decompose (classify → per-intent prompt): better accuracy, debuggability, parallelism, cheaper models per step. CoT/self-critique for reasoning; bound tokens.
- Version prompts outside code with metadata (model, sampling, schemas); any prompt/model/sampling change re-runs the eval suite. Inspect any framework's templates and hidden calls.

### Defensive prompting checklist (`references/prompting.md`)
- Assume the system prompt leaks; no secrets in it.
- Prefer models trained on an instruction hierarchy (system > user > model > tool output); label retrieved/tool content as data; sandwich key instructions; pre-empt known framings.
- System level: sandbox code execution; human approval + read-only defaults for impactful tools; input filters (patterns, odd characters, intent classifier over the whole conversation); output filters (PII, toxicity, policy); rate-based abuse detection; continuous red teaming into an adversarial eval slice.
- Track violation rate **and** false refusal rate.

### RAG design checklist (`references/rag-and-agents.md`)
- Small corpus (~<200k tokens)? Stuff it and prompt-cache; skip retrieval.
- Baseline BM25 → add embeddings → fuse with reciprocal rank fusion; rerank to fit the context budget; recency weighting when it matters.
- Chunk recursively by structure (256–1,024 tokens, small overlap, format-aware); enrich chunks (title/summary prefix, extracted identifiers as metadata, candidate questions).
- Rewrite conversational queries to be self-contained; make unresolvable references fail loudly.
- Tenant/ACL filters before similarity search; answers cite chunk IDs; explicit "not in corpus" behaviour.
- Evaluate retriever (precision/recall/NDCG), embeddings, and end-to-end faithfulness separately. Vector DB cost can rival model cost; budget re-embedding.

### Agent design checklist (`references/rag-and-agents.md`)
- Intent classifier first, including IRRELEVANT. Tools: knowledge, capability extension, write actions; least privilege and approval gates for writes.
- Decouple plan → validate → execute → reflect; cap steps; natural-language plan steps translated to calls by a cheaper model; parallel fan-out where independent.
- Function calling guarantees argument *shape*, not *values*: log every call, design explicit defaults, make the agent ask when ambiguous.
- Evaluate failure modes: invalid tool / invalid params / wrong values, goal and constraint violations (incl. time), false "done" claims, tool errors, efficiency (steps, cost, latency). Ablate tools; merge frequently co-used ones.
- Compound error: 95% per step ≈ 60% over 10 steps; keep chains short and models strong.

### RAG vs finetune (`references/finetuning-and-data.md`)
```
Wrong because the model lacks facts (private, stale, hallucinated specifics)  → RAG (facts)
Wrong because it behaves wrong (format, DSL/syntax, style, detail level, tone) → finetune (form), after prompting is exhausted
Both                                                                             → RAG first, finetune the residual, combine if evals justify
```
Finetune with LoRA/QLoRA on the strongest affordable base for small data; expect gains from 50–100 good examples or suspect the setup; quality and diversity beat volume; the annotation guideline is the eval rubric. Budget for serving and for the day a new base model obsoletes your finetune.

### Inference optimisation levers (`references/inference-optimization.md`)

| Lever | TTFT | TPOT | Cost/throughput | Trade-off |
|---|---|---|---|---|
| Shorter prompts, concise output, `max_tokens`, stop sequences | ↓ | – | ↓ | none; do first |
| Prompt caching of static prefixes | ↓↓ | – | ↓ (75–90% on cached input) | prefix must be byte-identical; storage fees |
| Smaller/distilled model + router | ↓ | ↓ | ↓↓ | quality; needs evals |
| Quantisation (self-host) | ↓ | ↓ | ↑↑ | small quality drift; re-eval |
| Speculative decoding | – | ↓↓ | ↑ | draft model, same tokenizer |
| Continuous batching, paged KV cache | – | – | ↑↑ | server feature |
| Batch API for non-interactive work | n/a | n/a | ~50% cheaper | hours of latency |
| Streaming | perceived ↓↓ | – | – | guardrails see partial output |
| Replicas / tensor parallelism | ↓ | ↓ | ↑ at ↑ cost | infra |
| Response cache (exact; semantic with care) | ↓↓ | – | ↓ | leaks across users; wrong semantic hits |

Measure p50/p95 TTFT, TPOT, output length and goodput against an SLO before touching anything.

### Reference architecture, in order (`references/architecture-and-feedback.md`)
query → router (intent/scope) → context construction → **model gateway** (one interface, credentials, quotas, fallback, logging) → output guardrails → response; add input guardrails (PII masking with reversible map), exact cache keyed by tenant + prompt version, agent loop-back, gated write actions. Observability from day one: metrics designed around feared failures, log everything (config, rendered prompt, intermediate outputs, tool I/O), per-request traces, drift detection (prompt changes, user behaviour, silent model updates via nightly canary evals). Start without an orchestration framework.

### Feedback collection (`references/architecture-and-feedback.md`)
Explicit (thumbs, better/worse after regenerate) is sparse and biased; implicit conversational signals (early stop, "No, I meant", rephrase, edits, complaints, sentiment, regeneration, delete/rename, turns, refusal rate) are abundant and noisy. Collect in-workflow, zero-effort, ignorable; capture context with consent; link every signal to a trace ID and feed it into eval sets and preference data. Watch leniency, randomness, position and length biases, and degenerate loops (popularity spirals, sycophancy) before training on feedback.

## Reference files

- `references/planning-and-model-selection.md` — scoping questions, human/AI role, usefulness thresholds, the stack, sampling knobs and structured outputs, hallucination causes, model selection funnel, build vs buy, benchmark limits.
- `references/evaluation.md` — exact vs judge methods, judge biases and template, comparative eval, criteria per feature, factual-consistency detection, eval pipeline design, sample sizes.
- `references/prompting.md` — anatomy, chat templates, context position, best-practice checklist, decomposition, CoT, prompt management, attack taxonomy and layered defences, templates.
- `references/rag-and-agents.md` — retrieval algorithms, hybrid/RRF, chunking, query rewriting, contextual retrieval, retrieval evals, text-to-SQL, agent planning, function calling, reflection, failure modes, memory, RAG checklist.
- `references/finetuning-and-data.md` — when (not) to finetune, RAG vs finetune, memory math, LoRA/QLoRA, merging, tactics and hyperparameters, data quality/coverage/quantity, synthesis and verification, processing pipeline.
- `references/inference-optimization.md` — prefill/decode, metrics, online vs batch, full levers table, model- and service-level techniques, diagnosis playbook.
- `references/architecture-and-feedback.md` — step-by-step architecture, guardrails, router/gateway, caches, observability, orchestration, feedback signals and design, limitations, NestJS mapping.
