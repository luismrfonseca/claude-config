# Planning an AI Feature and Selecting a Model

Synthesis of Huyen, *AI Engineering*, ch. 1 (planning, the stack), ch. 2 (sampling), ch. 4 (model selection).

## Contents

1. [Should this feature exist? (ch. 1)](#1-should-this-feature-exist)
2. [Define the role of AI and humans](#2-define-the-role-of-ai-and-humans)
3. [Set expectations: business metrics and usefulness thresholds](#3-set-expectations)
4. [Milestones, the last mile, and maintenance](#4-milestones-the-last-mile-and-maintenance)
5. [The AI engineering stack: what changed](#5-the-ai-engineering-stack)
6. [Sampling: the knobs you control at inference (ch. 2)](#6-sampling-knobs)
7. [Structured outputs](#7-structured-outputs)
8. [Inconsistency and hallucination: causes and first-line mitigations](#8-inconsistency-and-hallucination)
9. [Model selection workflow (ch. 4)](#9-model-selection-workflow)
10. [Build vs buy, open vs commercial](#10-build-vs-buy-open-vs-commercial)
11. [Public benchmarks and leaderboards: use and limits](#11-public-benchmarks-and-leaderboards)

---

## 1. Should this feature exist?

Demos are cheap; products are expensive. Before building, answer *why* (Huyen ch. 1):

| Motivation | Implication |
|---|---|
| Existential: competitors with AI make you obsolete | Highest priority; prefer in-house capability over outsourcing the core |
| Profit/productivity boost | Plenty of "buy" options; buy unless the feature is your differentiator |
| Exploration / don't want to be left behind | Fine if budgeted as R&D; keep scope and expectations small |

Defensibility question for standalone AI products: if the underlying model gains the capability your layer provides (e.g., "parse PDFs well"), does your product survive? The three moats are technology, data, and distribution. With shared foundation models, technology is mostly commoditised and distribution favours incumbents; the realistic moat for a smaller team is **usage data** gathered by shipping first and feeding it back into evaluation and improvement.

Enterprises deploy internal-facing features (knowledge management, summarisation) faster than external-facing ones (customer-facing chat) because risk is lower. Close-ended tasks (classification, extraction) are easier to evaluate than open-ended generation, which makes their risk easier to bound. Start there when you can.

## 2. Define the role of AI and humans

Three axes (adapted from Apple's HI guidance, via Huyen ch. 1):

| Axis | Options | Consequence |
|---|---|---|
| Critical vs complementary | App breaks without AI vs app still works | Critical → much higher accuracy/reliability bar; users forgive complementary features |
| Reactive vs proactive | Responds to user action vs surfaces on its own | Reactive usually needs low latency; proactive can be precomputed but needs a higher quality bar because nobody asked for it |
| Dynamic vs static | Per-user adaptation (memory, personalisation, per-user finetune) vs one shared model updated periodically | Dynamic adds data/privacy and infra complexity |

Human-in-the-loop ladder (Microsoft's crawl-walk-run):

1. **Crawl** – AI drafts, a human must act (e.g., suggested replies for support agents).
2. **Walk** – AI interacts directly with internal staff.
3. **Run** – AI interacts directly with external users, typically for a scoped subset of requests.

Promote a feature up the ladder based on measured evidence (e.g., agents accept ≥95% of suggestions verbatim for simple ticket categories → let AI answer those categories directly).

## 3. Set expectations

Define success in business terms first, then map to system metrics.

**Business metrics** (support-bot example): % of messages automated, extra throughput, response-time reduction, labour saved — plus customer satisfaction, because automating more messages badly is a loss.

**Usefulness threshold** – the bar the system must clear before it reaches users:

| Group | Examples |
|---|---|
| Quality | task-specific correctness, faithfulness, instruction following (see `evaluation.md`) |
| Latency | TTFT, TPOT, total latency; "acceptable" is relative to the status quo (if humans take an hour, 10 s is fine) |
| Cost | cost per request, cost per resolved task |
| Other | interpretability, fairness, compliance constraints |

Write these numbers down before you pick a model; they become the hard requirements in the selection funnel (section 9).

## 4. Milestones, the last mile, and maintenance

- **Evaluate off-the-shelf models first.** Where the baseline already lands determines the plan (30% automated out of the box vs 0% is a different project). Goals often change after this evaluation; sometimes the honest answer is "not worth it".
- **Budget for the last mile.** 0→60 is fast; 60→100 is slow. A frequently cited pattern: ~1 month to reach 80% of the target experience, then several more months to pass 95%, mostly spent on hallucinations and edge cases. Do not extrapolate the timeline from the demo.
- **Treat the model as a volatile dependency.** Prices, capabilities, and APIs shift quickly; providers converge on similar APIs, which makes swapping easier, but every model has quirks, so prompts and data need re-validation. Without versioning and a regression eval suite, a model swap is a gamble.
- **Watch regulatory and IP exposure** (data residency, compute access, training-data provenance) as part of the maintenance plan, not as an afterthought.

## 5. The AI engineering stack

Three layers; you start at the top and descend only as needed (Huyen ch. 1):

| Layer | Responsibilities | Change vs classic ML |
|---|---|---|
| Application development | prompt engineering, context construction (RAG, memory), evaluation, AI interface | Evaluation matters more (open-ended outputs); interface and prompting are new first-class work |
| Model development | modelling/training, finetuning, dataset engineering, inference optimisation | ML knowledge is nice-to-have, not required; data work is about dedup, quality control, context retrieval rather than feature engineering |
| Infrastructure | serving, compute/data management, monitoring | Largely unchanged |

Key shifts for a backend engineer:
- Workflow inverts: **product → data → model**, not data → model → product. Ship with prompting, collect usage data, invest in finetuning only when the product proves out.
- Adaptation techniques split into prompt-based (no weight changes; cheap, lets you try many models) and finetuning (weight changes; more data and ops, but can improve quality, cost, and latency substantially). Default to prompt-based.
- What you experiment with changes: models, prompts, retrieval settings, sampling variables — not hyperparameters. The discipline (systematic experiments, feedback loop from production) is the same.

## 6. Sampling knobs

Everything a model returns is sampled from a next-token distribution, so outputs are probabilistic by construction (Huyen ch. 2). The knobs:

| Knob | What it does | Practical guidance |
|---|---|---|
| `temperature` | Divides logits before softmax. Lower → sharper distribution, more deterministic; higher → flatter, more diverse | 0 (effectively greedy) for extraction, classification, code, judges; ~0.7 as a starting point for creative/open-ended text; providers usually cap at 2 |
| `top_k` | Softmax over only the k highest logits | 50–500 typical; smaller = more predictable. Mostly a compute optimisation; rarely the knob you tune |
| `top_p` (nucleus) | Sample from the smallest set of tokens whose cumulative probability ≥ p | 0.9–0.95 typical; adapts the candidate set to the context (2 candidates for yes/no, many for open questions). Tune temperature *or* top_p, not both aggressively |
| `min_p` | Discard tokens below an absolute probability floor | Alternative to top_p on some open-weight runtimes |
| `seed` | Fix the RNG | Improves reproducibility; not a guarantee (hardware/batching differences still leak in) |
| `max_tokens` | Hard stop | Bounds cost and latency; set it per task, and remember truncation breaks JSON |
| stop sequences | Stop on a literal | Cheaper than max_tokens for delimited outputs |
| `logprobs` | Return token log-probabilities | Useful for classification confidence, cheap eval signals, and debugging ("does the model actually prefer this label?"). Many providers restrict it |

**Test-time compute** (sample N, pick one): raises quality at ~N× cost. Selection strategies: highest average logprob (average, not sum, or you bias toward short outputs), a reward/verifier model, majority vote for exact-answer tasks, application heuristics (shortest, first valid SQL), or race N parallel generations and return the first valid one to cut tail latency. Diversify samples (vary temperature) for better coverage. Most useful when the model is not robust to small input changes; e.g., retrying an OCR-style extraction three times rescued most previously failed images. Nobody samples hundreds of candidates in production; 2–5 is the realistic range.

## 7. Structured outputs

Needed when the task is inherently structured (text-to-SQL, classification into a fixed label set, regex generation) or when a downstream component parses the output (almost always true in a backend; mandatory for tool calls in agents).

Layers, from bandage to cure (Huyen ch. 2):

| Technique | Cost | Reliability | Notes |
|---|---|---|---|
| Prompting (schema + examples in prompt) | free | depends on instruction-following strength; a few % invalid is common | always do this; never rely on it alone for machine-consumed output |
| Provider JSON mode / schema-constrained APIs | free–cheap | guarantees syntactically valid JSON, **not** correct content; truncation at max_tokens still yields unparsable output | prefer schema-enforced modes where offered |
| Post-processing (defensive parser) | cheap | surprisingly high; models repeat the *same* mistakes, so targeted fixes compound | e.g., a lenient YAML parser took validity from 90% to 99.99% at LinkedIn; YAML is less token-heavy than JSON |
| Validate-and-retry (second model call as judge/corrector) | 2× calls | high | adds latency; fine for async pipelines |
| Constrained sampling (grammar-filtered logits) | needs logit access (self-hosted runtimes, `outlines`, `guidance`, llama.cpp) | guaranteed format | per-format grammars; some latency overhead |
| Finetuning on formatted examples | highest upfront | most general and reliable, still not 100% | classifier head for closed label sets gives a hard guarantee |

Backend pattern: Zod/class-validator schema → provider structured-output mode → lenient parse + repair → validate → bounded retry with the validation error fed back → fail closed.

## 8. Inconsistency and hallucination

**Inconsistency** has two shapes: same input → different outputs (mitigate with caching, fixed sampling settings, seed; still not a guarantee), and slightly different input → very different outputs (harder; mitigated by better prompts and memory/context design, see `prompting.md`, `rag-and-agents.md`).

**Hallucination** is not merely sampling noise. Two complementary explanations (Huyen ch. 2):

1. *Self-delusion / snowballing*: the model cannot tell its own generated tokens from given context, so one wrong early token becomes a "fact" it then elaborates on — it will even get questions wrong that it would answer correctly in isolation.
2. *Knowledge mismatch from supervised training*: labelers wrote answers using knowledge the model lacks, teaching it that confident answers without grounding are acceptable.

What follows for application design:
- Shorter answers hallucinate less (fewer tokens, fewer chances). Ask for concise outputs where possible.
- Give the model the facts (RAG) rather than asking it to recall them; ask it to cite the source chunk so you can verify.
- Prompt-level "say I don't know if unsure" helps somewhat; it is not a control.
- Detection is its own problem: see hallucination/faithfulness evaluation in `evaluation.md`.

## 9. Model selection workflow

Model choice is never final; you will redo it as the landscape moves, so make the process cheap and repeatable (Huyen ch. 4). The funnel:

1. **Hard attributes filter** (non-negotiable): licence terms, data privacy / residency, required modalities, max context length, latency ceiling, cost ceiling, self-hosting requirement, availability in your region/provider (e.g., Bedrock vs direct API). Eliminate before testing anything.
2. **Public information shortlist**: benchmarks, leaderboards, release notes, community reports. Use these to pick ~3–5 candidates, not to decide.
3. **Your own evals**: run the shortlist against your task-specific eval set and your prompts (see `evaluation.md`). This is the step that decides.
4. **Online evaluation**: A/B or shadow traffic with production feedback, because offline evals never fully match real usage.

**Hard vs soft attributes.** Hard = what you cannot or will not change (provider decisions: licence, training data, size; your policies: privacy, control). Soft = what you can improve with work (accuracy, toxicity, factual consistency). Which bucket an attribute falls in depends on the setup: latency is soft if you host the model and can optimise it, hard if a provider hosts it. Be honest about soft attributes: the same model can jump from 20% to 70% accuracy after decomposing the task into two steps, or stay unusable after weeks of prompt work.

The two underlying questions for any adaptation technique: (1) what is the best achievable performance, (2) where do candidates sit on the cost–performance curve. With prompting, start from the strongest model to establish feasibility, then work *backwards* to the cheapest model that still clears the bar. With finetuning, start small to validate the pipeline, then scale up to the largest model your hardware budget allows.

Keep a candidates matrix: model × {hard attributes pass?, eval score per criterion, p50/p95 latency, cost per 1k requests}. Re-run it when a provider ships something new; the suite is the asset, not the decision. The steps are iterative: a private eval may send you back to step 1 (e.g., open models can't reach the bar → reopen the API option).

## 10. Build vs buy, open vs commercial

Decision factors (Huyen ch. 4):

| Factor | Commercial API | Open-weight, self-hosted |
|---|---|---|
| Data privacy | data leaves your boundary; depends on DPA/retention terms (zero-retention options exist) | data stays in-house |
| Data lineage / copyright | opaque training data | sometimes documented; still opaque for most |
| Performance | strongest models usually arrive here first | gap closes over time; the best open models are competitive for many tasks |
| Functionality | structured outputs, tool calling, caching, batch APIs, guardrails bundled | you build or integrate each |
| Cost | pay per token; expensive at high, steady volume | engineering + GPU cost; cheaper only at scale and with a team to run it |
| Control & transparency | provider can change/deprecate models, moderate your inputs, change prices | you pin the version; you own the quirks |
| Lock-in | moderate (APIs converge, prompts don't) | lower, but you depend on an inference stack |
| Latency | shared infra, rate limits | tuneable; your SLO, your problem |
| Edge / on-device | no | yes |

"Open" has gradations: *open weight* (weights only, the vast majority) vs *open model* (weights + training data). Licence questions to ask of any open model: commercial use allowed? restrictions (e.g., MAU caps requiring a special licence)? may its outputs be used to train/distil other models? Read the licence as a hard attribute. Also note that the same open model served by different API providers can behave slightly differently (different optimisations), so re-run evals when switching providers.

Other factors from Huyen ch. 4: commercial providers bundle scalability, function calling, structured outputs, moderation; they rarely expose logprobs and only sometimes allow finetuning (and only certain kinds). Self-hosting gives logprobs, intermediate outputs, quantisation and any finetuning the licence permits, at the price of building and running the inference service. API cost is linear in tokens; self-hosted cost per token falls with utilisation, so the answer flips with scale. Commercial models are also more likely to over-refuse (a 3D-character product kept getting "as an AI I have no physical abilities" and moved to finetuned open models). Silent model updates can change your results either way: the same upgrade dropped one team's intent classification by 10% and improved another's support bot.

Practical default for a product team without an ML platform: start with a commercial API behind an abstraction (your own gateway interface), build the eval suite, and revisit self-hosting when volume, privacy, or cost makes it obviously worthwhile. Keep inference-provider choice (who serves an open model) separate from model choice.

## 11. Public benchmarks and leaderboards

Use them to prune, not to decide:

- **Contamination**: benchmark data leaks into training sets (web scraping, shared sources, or deliberately because benchmark data is high quality); scores inflate without capability gains. A 1M-parameter model trained on test sets can top large models. Detect with n-gram overlap against training data (accurate, needs the data) or unusually low perplexity on the benchmark (cheap, approximate). Treat suspiciously high public scores as a reason to test privately; prefer benchmarks with private held-out splits.
- **Aggregation hides relevance**: a leaderboard averages benchmarks that may have nothing to do with your task, weights them equally regardless of difficulty, and different leaderboards pick different sets (two major ones overlapped on only 2 of ~10 at time of writing). Check benchmark correlation: strongly correlated benchmarks (several reasoning sets correlate > 0.85) add bias, not information; truthfulness correlates only moderately with reasoning, so it needs its own measurement. Pick the handful closest to your use case and read those columns only. Running full suites yourself is expensive (tens of thousands of dollars for 30 models on a broad suite), so pick narrowly.
- **Setup sensitivity**: few-shot count, prompt format, CoT, and sampling settings change results materially (the same model reported with different prompting setups can differ by several points). Compare apples to apples or not at all.
- **Saturation**: benchmarks stop discriminating once top models cluster near the ceiling.
- Human-preference arenas (pairwise votes, e.g., Chatbot Arena at time of writing) capture general chat quality, not your domain, and are subject to voter bias and gaming.

Rule: public benchmarks answer "is this model in the right league?"; only your evals answer "is it right for this feature?".
