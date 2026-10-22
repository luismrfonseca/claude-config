# Inference Optimization: Latency and Cost

Synthesis of Huyen, *AI Engineering*, ch. 9. Most of this applies whether you call an API or host a model: it tells you what to measure, which levers exist, and what a provider's pricing and latency actually reflect.

## Contents

1. [Where time and money go](#1-where-time-and-money-go)
2. [Metrics](#2-metrics)
3. [Online vs batch APIs, streaming](#3-online-vs-batch-apis-streaming)
4. [Levers table](#4-levers-table)
5. [Model-level techniques](#5-model-level-techniques)
6. [Service-level techniques](#6-service-level-techniques)
7. [Hardware in two paragraphs](#7-hardware-in-two-paragraphs)
8. [Diagnosis playbook: "the LLM is too slow / too expensive"](#8-diagnosis-playbook)

---

## 1. Where time and money go

Transformer inference has two phases with different bottlenecks (Huyen ch. 9):

| Phase | What happens | Bound by | Drives |
|---|---|---|---|
| **Prefill** | all input tokens processed in parallel | compute (FLOP/s) | TTFT; cost scales with input length |
| **Decode** | one output token per step; weights re-read from memory each step | memory bandwidth | TPOT; cost scales with output length; output tokens cost ~2–4× input tokens at providers, and one output token ≈ 100 input tokens in latency impact |

Consequences: output length is the dominant latency and cost driver; long contexts push you toward bandwidth limits and KV-cache memory; most LLM serving is bandwidth-bound, which is why quantisation (fewer bytes per weight) helps speed as well as memory. Profilers show this as a roofline chart (arithmetic intensity vs achieved FLOP/s).

## 2. Metrics

| Metric | Definition | Notes |
|---|---|---|
| **TTFT** | time to first token (≈ prefill) | depends on input length; must feel instant for chat; users tolerate more for long-document tasks |
| **TPOT** | time per output token after the first (also TBT, ITL) | streaming only needs to beat reading speed: ~6–8 tokens/s (~120 ms/token) is the *floor* for a readable UI — not a typical serving speed. Hosted frontier APIs usually decode at roughly 30–100+ tokens/s (~10–30 ms/token) (practice, not from book). Sanity-check: output_tokens × TPOT + TTFT must roughly reproduce the observed latency |
| **Total latency** | TTFT + TPOT × output tokens | same total can feel different: fast first token + slower stream vs the reverse; test with users |
| **Time to publish** | TTFT as the *user* sees it | with CoT/agents, internal tokens start early but the user waits for the final answer; measure the user-visible number |
| **Throughput** | output tokens/s (count input and output separately); tokens/s/user; completed requests/min (RPM, since requests take seconds) | maps directly to cost: $2/h at 100 tok/s ≈ $5.56 per 1M output tokens; add prefill cost per request |
| **Goodput** | requests/s that meet the SLO (e.g., TTFT ≤ 200 ms and TPOT ≤ 100 ms) | 100 RPM with 30 meeting SLO = goodput 30; optimise this, not raw throughput |
| **Utilisation** | MFU (achieved vs peak FLOP/s), MBU (achieved vs peak bandwidth; = params × bytes × tokens/s ÷ peak) | `nvidia-smi` "utilisation" only says the GPU is busy, not efficient; MFU > 50% is good for training, lower is normal for decode |

Report latency as percentiles (p50, p90, p95, p99), never averages (one 3 s outlier turns ten ~100 ms requests into a 390 ms "average"). Plot TTFT vs input length. Compare providers on cost per request, not tokens/s, since tokenizers differ.

## 3. Online vs batch APIs, streaming

- **Online API**: latency-optimised; may still micro-batch.
- **Batch API**: throughput/cost-optimised, typically ~50% cheaper with hours-long turnaround at time of writing. Use for synthetic data, periodic reporting (summaries, sentiment over tickets), onboarding backfills, re-processing after a model migration, bulk personalisation, re-indexing a knowledge base. If you run your own service, splitting online and batch lanes lets you protect latency-critical traffic.
- **Streaming**: cuts perceived latency to TTFT but you cannot score a response before the user sees it; keep output guardrails that can retract or amend a streamed answer. Note that foundation-model "batch" is not classic-ML precomputation: open-ended prompts can't be precomputed, only system prompts can be (prompt cache).

## 4. Levers table

| Lever | Level | TTFT | TPOT | Throughput / cost | Quality risk | Trade-off / notes |
|---|---|---|---|---|---|---|
| Shorter prompts, concise output instructions, `max_tokens`, stop sequences | app | ↓ | – | ↓ cost | none | cheapest lever; output tokens dominate |
| Prompt caching (prefix/context cache) | service/API | ↓↓ (up to ~75–80% on long prefixes) | – | ↓ cost (cached input discounted 75–90%) | none | put static content (system prompt, docs, examples) first; some providers bill cache storage per hour; the win scales with prefix length and reuse |
| Smaller model / distilled model | model | ↓ | ↓ | ↓↓ | yes | validate with evals; route easy traffic to it |
| Quantisation (weight-only INT8/INT4) | model | ↓ | ↓ | ↑↑ | small, measurable | the single most effective self-host lever; check eval before/after; load in the trained dtype |
| Speculative decoding | model/service | – | ↓↓ (often >2× on structured text/code) | ↑ | none (verifies against target) | needs a draft model with the same tokenizer; uses idle FLOPs during bandwidth-bound decode; built into major servers |
| Inference with reference (copy spans from input) | model/service | – | ↓ (~2× when output overlaps input) | ↑ | none | RAG answers, code edits, multi-turn |
| Parallel decoding (lookahead, multi-head) | model | – | ↓ | ↑ | low-moderate | harder to implement; extra heads need training |
| Pruning | model | ↓ | ↓ | ↑ | yes | rarely worth it in practice; needs sparsity-aware hardware |
| Attention redesign (local/windowed, grouped-query, cross-layer) | model (train-time) | – | ↓ | ↑↑ (KV cache ÷ 20 reported) | model-dependent | choose models with these when context is long |
| KV-cache management (paged attention, KV quantisation) | service | – | – | ↑↑ batch size | none | framework feature; essential for long contexts |
| Kernels / compilers (fused attention, `torch.compile`, TensorRT-style) | service | ↓ | ↓ | ↑ | none | hardware-specific; one published case: compile → INT8 → INT4 → speculative decoding stacked to several × throughput |
| Static batching | service | ↑ | ↑ | ↑↑ | none | waits for a full batch; batch workloads only |
| Dynamic batching (size or time window) | service | ↑ (bounded) | – | ↑ | none | the usual online default |
| Continuous (in-flight) batching | service | – | – | ↑↑ | none | short responses return without waiting for long ones; standard in modern servers |
| Prefill/decode disaggregation | service | ↓ | ↓ | ↑ | none | separate instances; ratio prefill:decode ≈ 2–4:1 for long inputs / TTFT priority, 1:1–1:2 for short inputs / TPOT priority |
| Replica parallelism | service | ↓ (less contention) | ↓ | ↑ at ↑ cost | none | simplest scale-out; bin-packing models onto GPU sizes |
| Tensor parallelism | service | ↓ | ↓ | enables big models | none | communication overhead; preferred for inference |
| Pipeline parallelism | service | ↑ | ↑ | ↑ | none | adds per-request latency; mostly for training |
| Routing by difficulty (cheap model first, escalate) | app | ↓ | ↓ | ↓ cost | depends on router | see `architecture-and-feedback.md` |
| Response caching (exact / semantic) | app | ↓↓ | – | ↓ cost | semantic cache can return wrong answers | see `architecture-and-feedback.md` |

Throughput and latency trade against each other: batching can double or triple throughput at the price of TTFT/TPOT. Decide the SLO first, then maximise goodput under it.

## 5. Model-level techniques

Model-level changes can alter outputs; the same open model served by different providers scores differently on benchmarks because of their optimisations. Always re-run evals after a model-level change or a provider switch.

- **Compression**: weight-only quantisation (16-bit → 8/4-bit; near the floor already), distillation (small model mimics large; see `finetuning-and-data.md`), pruning (sparsity; uncommon in practice).
- **Autoregressive bottleneck**: speculative decoding (draft K tokens cheaply, target verifies in parallel, accept the longest agreeing prefix plus one; acceptance is higher for structured text; a draft 8× faster halved end-to-end latency in a reported setup), inference with reference, parallel/Jacobi/multi-head decoding.
- **Attention cost**: KV cache size = 2 × batch × seq_len × layers × hidden_dim × bytes (a 13B model at batch 32, 2k context, FP16 ≈ 54 GB unoptimised). Reduce via architecture choices (windowed, multi-/grouped-query, cross-layer sharing), cache management (paging, quantisation, selective retention), and fused kernels.

## 6. Service-level techniques

Service-level changes keep the model intact and should not change quality.

- **Batching**: static → dynamic → continuous (see table). For LLMs continuous batching matters because response lengths vary wildly.
- **Prefill/decode disaggregation** to stop prefill jobs starving decode jobs.
- **Prompt caching**: a 1,000-token system prompt at 1M calls/day is ~1B redundant input tokens/day without it. Cases reported by a provider at time of writing: 100k-token document chat TTFT 11.5 s → 2.4 s and −90% cost; 10k-token many-shot prompt −31% latency, −86% cost; 10-turn conversation with long system prompt −75% latency, −53% cost. Structure prompts so the shared prefix is identical byte-for-byte across requests.
- **Parallelism**: replica (scale-out), tensor (split operators; enables large models, lowers latency), pipeline (stages; throughput at latency cost), context/sequence (long inputs).

## 7. Hardware in two paragraphs

What matters per chip: peak FLOP/s at your precision (halves roughly per doubling of bits), memory size (does the model + KV cache fit?), memory bandwidth (decode speed), and power/cost. Compute-bound workloads want FLOP/s; bandwidth-bound (most LLM decode) want bandwidth. Memory hierarchy: CPU DRAM (tens of GB/s) → GPU HBM (hundreds of GB/s to >1 TB/s; 24–80 GB typical) → on-chip SRAM (>10 TB/s, tens of MB); kernels exist to exploit it.

Inference-specialised chips favour lower precision and faster memory over capacity; inference can be up to ~90% of lifetime ML cost, so serving efficiency outweighs training efficiency for most products. Three questions when choosing: can it run the workload, how fast, at what cost.

## 8. Diagnosis playbook

"Too slow":
1. Measure TTFT, TPOT, output length distribution, p50/p95, per endpoint. Is it prefill (long inputs → TTFT) or decode (long outputs → total)?
2. Cut output: concise instructions, structured output, `max_tokens`, stop sequences, strip CoT from user-visible path or stream it.
3. Cut input: prompt caching for static prefixes, tighter retrieval (fewer/shorter chunks), summarised history instead of full transcripts.
4. Stream, and show progress for multi-step flows (time to publish).
5. Parallelise independent steps (agent fan-out, decomposed prompts).
6. Route simple requests to a smaller/faster model; keep the big model for hard ones.
7. Self-hosting: quantise, enable continuous batching and paged KV cache, add speculative decoding, add replicas; disaggregate prefill/decode at scale.

"Too expensive":
1. Attribute cost to input vs output tokens vs cache storage vs vector DB vs judge calls.
2. Prompt caching; shorter prompts; fewer examples (or finetune them in); batch API for anything not interactive; response cache for repeated queries.
3. Smaller/distilled model behind a router; judge on a sample instead of 100%.
4. Self-host only when steady volume makes GPU utilisation high enough to beat per-token pricing, and you have the team to run it.
