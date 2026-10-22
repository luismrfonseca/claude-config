# Finetuning and Dataset Engineering

Synthesis of Huyen, *AI Engineering*, ch. 7 (finetuning) and ch. 8 (dataset engineering).

## Contents

1. [What finetuning is and what it is for](#1-what-finetuning-is-and-what-it-is-for)
2. [When to finetune, when not to](#2-when-to-finetune-when-not-to)
3. [RAG vs finetuning decision](#3-rag-vs-finetuning-decision)
4. [The adaptation ladder](#4-the-adaptation-ladder)
5. [Memory bottleneck, in one page](#5-memory-bottleneck-in-one-page)
6. [PEFT and LoRA](#6-peft-and-lora)
7. [Model merging, briefly](#7-model-merging-briefly)
8. [Practical finetuning tactics](#8-practical-finetuning-tactics)
9. [Data curation: quality, coverage, quantity (ch. 8)](#9-data-curation)
10. [Data acquisition and annotation](#10-data-acquisition-and-annotation)
11. [Augmentation and synthesis](#11-augmentation-and-synthesis)
12. [Data processing pipeline](#12-data-processing-pipeline)

---

## 1. What finetuning is and what it is for

Finetuning = continuing to train a pre-trained model (all or part of its weights) on your data. It is transfer learning: the base model already has most of what you need, so a few hundred to a few thousand examples can refine behaviour that would take millions to learn from scratch. Think of it as unlocking capabilities the model has but that prompting can't reliably reach (Huyen ch. 7).

Kinds you will meet as an application developer:
- **Supervised finetuning (SFT)** on (instruction, response) pairs: the common case; teaches format, style, task behaviour.
- **Preference finetuning** on (instruction, better, worse): aligns to preferences; needs comparative data.
- **Continued pre-training / self-supervised** on raw domain text before SFT: cheap way to soak up domain language when labelled data is scarce.
- Long-context finetuning and infilling exist but are specialist work.

Typical reasons it pays off: reliable structured output in a format the model rarely saw (uncommon SQL dialect, a DSL, your YAML schema); consistent style/persona; domain behaviour; bias mitigation; and above all **making a small model do one task as well as a large model does it** (distillation): e.g., a writing-assistant team reported a finetuned model 60× smaller than the API model beating it on their tasks with ~80k pairs.

## 2. When to finetune, when not to

Finetune only after exhausting prompting; most "prompting doesn't work" stories turn out to be unsystematic prompting (unclear instructions, unrepresentative examples, no metrics). Prompt work also produces the eval pipeline, annotation guideline, and experiment tracking you need before finetuning anyway.

Reasons **not** to (Huyen ch. 7):
- **Regression on other tasks**: tuning for "change order" queries can degrade "product recommendation" and "general feedback". Fix by tuning on the whole query mix, or separate models (merged later if you like).
- **Upfront and ongoing cost**: annotated data; ML skill (learning rate, overfitting, loss curves); serving a custom model (self-host or a provider that supports it); a policy for when a new base model makes your finetune obsolete, which happens often and fast.
- **Domain-specific ≠ must finetune**: a 50B-parameter in-house finance model trained at ~$1–3M of compute was outperformed by a general API model released the same month on the finance benchmarks. General models keep getting better at your domain.
- Prompt caching removed finetuning's old "shorter prompts save tokens" advantage, though finetuning still has no cap on the number of examples.

Finetuning becomes attractive when: open-weight models exist at your required size, you already host models, you have a data flywheel, and the failures are behavioural (next section).

## 3. RAG vs finetuning decision

Diagnose the failure first:

```
Model output is wrong because...
├─ it lacks information (private data, post-cutoff facts, hallucinated specifics)
│     → RAG. Start term-based (BM25); upgrade to hybrid/embeddings if still failing.
│       RAG beats finetuning on knowledge/current-events tasks; finetuned+RAG
│       is often no better than base+RAG for facts.
├─ it has the facts but behaves wrong (ignores format, wrong level of detail,
│  wrong style, doesn't follow a DSL/syntax, unsafe tone)
│     → finetuning (after prompt work). "Finetuning is for form, RAG is for facts."
└─ both
      → RAG first (no training data, no serving changes), then finetune for the
        residual behavioural issues; combine if evals justify it.
```

Finetuning can reduce hallucination with lots of high-quality data and can *increase* it with low-quality data. Embedding-based RAG adds inference-time components; finetuning adds development-time complexity but leaves inference unchanged; choose the complexity you can operate.

## 4. The adaptation ladder

With evals in place before step 1 and after every step:

1. Prompt engineering, versioned, best practices (`prompting.md`).
2. Add examples (1–50 shots).
3. Simple RAG (term-based) if information failures dominate.
4. Then, by failure mode: advanced retrieval (embeddings, hybrid, reranking) for information gaps; finetuning for persistent behavioural gaps (irrelevant, malformed, unsafe output).
5. RAG + finetuning together when each proves its value separately.

## 5. Memory bottleneck, in one page

Why finetuning is hard and PEFT exists (Huyen ch. 7):

- Inference memory ≈ `params × bytes_per_param × 1.2` (the 20% covers activations and KV cache at modest context/batch). 13B at 2 bytes ≈ 31 GB; 70B ≈ 140 GB of weights alone.
- Training adds **gradients + optimizer state per trainable parameter**: with Adam that's 3 extra values per trainable param. Full finetune of 7B in 16-bit = 14 GB weights + 42 GB grads/optimizer ≈ 56 GB before activations, beyond any single consumer GPU. Activations can dwarf weights; gradient checkpointing trades recomputation time for memory.
- Two levers: fewer trainable parameters (PEFT) and fewer bits per value (quantisation). Inference runs in 16, 8, even 4 bits; training is precision-sensitive and uses mixed precision. Load models in the numeric format they were trained for (BF16 vs FP16 confusion has silently degraded quality for many teams).

## 6. PEFT and LoRA

**Parameter-efficient finetuning**: freeze the base, train a small set of added parameters, get close to full-finetune quality with orders of magnitude fewer trainable params and far less data (thousands of examples instead of tens of thousands+).

**LoRA** (dominant method at time of writing): for a weight matrix W, learn low-rank A (n×r) and B (r×m); the effective weight is W + (α/r)·AB; only A and B train. Key facts:
- Rank r of 4–64 is usually enough; raising r rarely helps and can overfit (some tasks want more). α:r between 1:8 and 8:1 in practice; experiment.
- Apply to attention matrices (query and value if you can afford only two; all four if you can) and, empirically, also to feed-forward layers for the biggest gains.
- Adapter weights are tiny (megabytes vs tens of GB), so memory savings come from freezing the base, not shrinking the adapter. **QLoRA** keeps the base in 4-bit and dequantises on the fly: a 65B model finetunes on a single 48 GB GPU, at some training-time cost.
- **Serving**: merge A·B into W for one model (zero extra latency), or keep adapters separate for **multi-LoRA serving**: one base, many adapters (per customer, per task), cheap to store and fast to switch, with a small latency overhead. This is the pattern for per-tenant customisation and for on-device multi-feature setups.
- Trade-off: LoRA < full finetune in peak quality; needs a framework that supports your base model (the popular PEFT libraries do for mainstream models).

Soft-prompt methods (prefix/prompt tuning: trainable embedding vectors prepended to the input) are the other PEFT family; less used, interesting if you want something between prompting and finetuning.

## 7. Model merging, briefly

Combine the weights of several models (often LoRA-tuned siblings of one base) into one, no GPU required for the merge itself. Uses: multi-task models without sequential finetuning's catastrophic forgetting, one deployable model instead of several (on-device), federated-style combination. Approaches: summing (averaging, weighted, spherical interpolation), layer stacking, concatenation. Experimental; finetune after merging if quality matters. Distinct from ensembling, which runs several models per request.

## 8. Practical finetuning tactics

- **Base model**: start with the strongest you can afford to prove feasibility, then work down. Two development paths: *progression* (test code on the cheapest model → test data on a mid model → push quality on the best → map the price/performance frontier across models) or *distillation* (strong model + small dataset → generate more data with it → train a cheaper model).
- **Method**: LoRA first; full finetune later if you have tens of thousands of examples and the quality gap matters. Few hundred examples → PEFT on a stronger base; huge datasets → full finetune on a smaller base. Need many variants sharing a base → LoRA.
- **Framework**: provider finetuning APIs are easiest but restrict base models and knobs; open frameworks (PEFT libraries, Axolotl/unsloth-style at time of writing) give control and need you to provision GPUs; multi-node needs a distributed trainer.
- **Hyperparameters**: learning rate (1e-7 to 1e-3; 0.1–1× the base's final pre-training LR; oscillating loss → too high, slow steady loss → too low); batch size (≥8 for stability; use gradient accumulation when memory forces small batches); epochs (1–2 for millions of examples, 4–10 for thousands; stop when validation loss turns up while training loss keeps falling); prompt-loss weight (~10%: learn mostly from response tokens).
- **Sanity check**: you should see improvement with 50–100 good examples; if not, suspect hyperparameters, data quality, or prompt format before blaming data volume. Plot performance vs 25/50/100% of the data to forecast returns on more labelling.
- Match inference prompts exactly to the training format (a missing `-->` or an extra space breaks a finetuned model).

## 9. Data curation

Training data must demonstrate the behaviours you want and omit the ones you don't (removing examples teaches unlearning: e.g., delete annotations with unsolicited rewrites if the bot is "arrogant"). The three criteria (Huyen ch. 8):

**Quality** — a small clean set beats a large noisy one (one lab: ~10k curated instructions > hundreds of thousands noisy; 1,000 curated examples made a 65B model competitive on many prompts, though less robust than product-grade models). Six properties:

| Property | Meaning |
|---|---|
| Relevant | matches the task and era |
| Aligned with task requirements | annotations reflect what the task wants (factual, creative, concise, scored *with* justification) — "aligned", not merely "correct" |
| Consistent | two annotators agree; same score ⇒ same quality; needs a written guideline |
| Correctly formatted | model's chat template; no stray HTML/Markdown, whitespace, casing or number-format noise |
| Sufficiently unique | duplicates bias and contaminate |
| Compliant | no PII or disallowed sources; licence checked |

**Coverage/diversity** — match the real input distribution: short and long instructions, typos if users make them, every language/programming language you serve, task types, output formats, lengths, single- and multi-turn. High-quality-and-diverse beats either alone on identical sizes. Start with a mix that mirrors production usage; tune the mix by small-scale experiments. Adding heterogeneous data can also hurt; measure.

**Quantity** — depends on method (full finetune: tens of thousands to millions; PEFT: hundreds to thousands), task complexity, and how close the base already is (stronger base → fewer examples; with enough data all bases converge). Diminishing returns are normal; diversity of tasks often matters more than raw count. Reduce the high-quality data needed by staging: self-supervised on raw domain text → SFT; less-relevant labelled data → relevant; synthetic → real (harder to get right). Budget is a constraint: at $2/example, $10k buys 5,000 examples, and data spend competes with compute spend.

## 10. Data acquisition and annotation

- Best source: your own application's data and feedback (the data flywheel); it is exactly on-distribution. Design feedback capture early (`architecture-and-feedback.md`).
- Check public datasets and marketplaces first; verify licence and provenance; never trust them unread.
- Realistic pipeline: find a dataset → drop low-quality instructions → separate low-quality responses → hand-write responses for good instructions → notice topic gaps → write templates → synthesise instructions → annotate → discover guideline problems → re-annotate.
- The annotation guideline is the hard part and is the same artefact as the evaluation rubric. Invest in it once; reuse it for evals, annotation, and judges. Teams abandon careful annotation halfway hoping the model "figures it out"; this is the usual cause of disappointing finetunes.

## 11. Augmentation and synthesis

Augmentation derives new examples from real ones; synthesis fabricates examples with real-like properties. Why: quantity, coverage (targeted rare cases, adversarial examples, class balance), sometimes quality (AI tool-use traces are more AI-appropriate than human ones; AI preference labels are more consistent), privacy (synthetic records), and distillation.

Techniques:
- Rule-based templates (structured documents, transactions, equations), word swaps (incl. swapping gendered terms to counter bias), perturbation/noise for robustness.
- Simulation and self-play (agents negotiating, support agent vs simulated customer, simulated API responses instead of real calls).
- AI paraphrase/translation (query variants for support content; back-translation to verify).
- Instruction synthesis: topic list → subtopics → instructions → responses; seed examples → generate look-alikes; **reverse instruction**: take high-quality existing content and generate the prompt that would elicit it (avoids hallucinated long responses).
- Code pipelines that make synthetic data *verifiable*: generate problem → solution → AI-written unit tests → lint/execute → ask the model to fix failures → keep only passing examples → translate across languages and re-verify → generate explanations and back-translate to check faithfulness.

Verification: functional checks where possible; AI judge/classifier otherwise; factual-consistency filters; "can a detector tell real from synthetic?" as a realism test; heuristics (empty/too short/too long, repeated instructions with different outputs, output copies input); pairwise judges run in both orders, keep only consistent winners.

Limits: quality control is on you; imitation copies style more than capability and can teach a weak student to *look* like it solved things it can't; recursive training on purely synthetic data collapses toward common cases (mix real data; no settled ratio); biases amplify through feedback loops; lineage becomes opaque (licence restrictions on using outputs to train competitors, hidden benchmark contamination). Training on your own model's unverified outputs does not help and can hurt; verified outputs can.

## 12. Data processing pipeline

Order steps to minimise cost (dedupe before expensive cleaning, or filter before expensive dedupe), trial-run scripts on a sample, never modify in place.

1. **Inspect**: provenance, token/length/topic/language distributions, per-source and per-annotator distributions, inter-annotator disagreement, outliers. Spend 15 minutes reading raw examples; annotate a few yourself to test the guideline. Highest value-to-effort step in the pipeline.
2. **Deduplicate**: whole-document, intra-document, cross-document; define the level (document/paragraph/sentence) and threshold (exact vs fuzzy). Methods: pairwise similarity (small sets), hashing (MinHash, Bloom filters), dimensionality reduction + ANN. Repeating 0.1% of data 100× halved an 800M model's effective size in one study.
3. **Clean and filter**: strip HTML/Markdown (one team: +20% accuracy, −60% input tokens), remove PII/sensitive/copyrighted/toxic content and disallowed fields, drop low-quality examples (judge, heuristics, late-session annotations), prune with active learning/importance sampling when you have more than you can afford.
4. **Format**: the target model's tokenizer and chat template exactly; SFT examples usually drop the task description and few-shot examples (the model learns from the examples themselves), so the production prompt becomes minimal and must match the training format character for character.
