# Evaluation: Methods, Criteria, and Building the Pipeline

Synthesis of Huyen, *AI Engineering*, ch. 3 (evaluation methodology) and ch. 4 (criteria, evaluation pipeline).

## Contents

1. [Why evaluation is the bottleneck](#1-why-evaluation-is-the-bottleneck)
2. [Exact evaluation](#2-exact-evaluation)
3. [AI as a judge](#3-ai-as-a-judge)
4. [Judge biases and mitigations](#4-judge-biases-and-mitigations)
5. [Which model judges, and specialised judges](#5-which-model-judges)
6. [Comparative evaluation](#6-comparative-evaluation)
7. [Evaluation criteria for an application (ch. 4)](#7-evaluation-criteria-for-an-application)
8. [Factual consistency / hallucination detection](#8-factual-consistency)
9. [Instruction following and role adherence](#9-instruction-following-and-role-adherence)
10. [Designing the evaluation pipeline](#10-designing-the-evaluation-pipeline)
11. [Eval set size and statistical sanity](#11-eval-set-size-and-statistical-sanity)
12. [Evaluating the evaluator](#12-evaluating-the-evaluator)
13. [Judge prompt template](#13-judge-prompt-template)

---

## 1. Why evaluation is the bottleneck

A deployed feature nobody can evaluate is worse than one never shipped: it costs to run and costs more to remove. Foundation models make evaluation harder than classic ML (Huyen ch. 3):

- Outputs are open-ended, so there is no exhaustive set of correct answers to diff against.
- Stronger models produce plausible-sounding wrong answers; grading needs domain expertise and fact-checking, not a glance.
- Models are black boxes; you can evaluate only by observing outputs.
- Public benchmarks saturate within a year or two of release, and are contaminated.
- The same model behaves differently under different prompts, so you are evaluating a *system* (model + prompt + context + sampling), never a model in isolation.

"Eyeballing a few favourite prompts" is tolerable for a spike and inadequate for iteration. **Evaluation-driven development**: define criteria and the eval harness before building, the way you would write tests first. The features that reach production in enterprises are the ones with clear evaluation signals (classification, extraction, code with tests, recommendations with CTR).

## 2. Exact evaluation

Deterministic judgments; prefer these wherever the task allows.

| Method | Use when | Notes |
|---|---|---|
| **Functional correctness** | output can be executed or checked against a measurable objective: code (unit tests, `pass@k`), text-to-SQL (execution match, plus runtime/efficiency), tool calls (did the booking happen?), schedules (energy saved) | The ultimate metric; automate whenever the domain permits. Lexical similarity to a reference does *not* predict correctness for code |
| **Exact match** | short canonical answers (numbers, entity names, labels, fill-in) | Variants: "contains reference" (risky: "September 12, 1929" contains "1929" and is wrong) |
| **Lexical similarity** (edit distance, n-gram overlap: BLEU/ROUGE family) | translation, captioning, when you have many reference outputs | Needs a comprehensive reference set; penalises correct answers phrased differently; references themselves are often wrong |
| **Semantic similarity** (cosine over embeddings; BERTScore-style) | paraphrase-tolerant comparison against references; relevance of answer to question | Score is exact given the embeddings, but depends on embedding model quality; adds compute |
| **Classification metrics** | intent detection, routing, sentiment, moderation | accuracy, precision/recall/F1; compare against the random baseline; use logprobs for confidence |

Perplexity/logprobs (ch. 3): a model's perplexity on text measures how predictable the text is to it. Uses beyond training: detect benchmark contamination (unusually low perplexity on an eval set), detect gibberish/outlier inputs, cheap fluency proxy, and training-data dedup. Post-trained (chat) models and quantised models have distorted perplexity, so do not compare across them.

Reference data is the bottleneck for similarity methods: humans are slow and expensive; AI-generated references need human review but far less of it.

## 3. AI as a judge

Using a model (with a prompt) to score outputs. Fast, cheap relative to humans, works without references, can explain its verdict, and can be asked about any criterion. Agreement with humans can match or exceed inter-human agreement for general chat quality (e.g., ~85% on MT-Bench at time of writing). It is now the default production evaluation method, and the right default for open-ended criteria, provided you treat it as the subjective instrument it is (Huyen ch. 3).

Three shapes:

1. **Pointwise, reference-free**: score a (question, answer) on a criterion.
2. **Reference-based**: is this answer equivalent to / as good as the reference? A drop-in for similarity metrics.
3. **Pairwise**: which of A/B is better? Used for comparative eval, preference data, and best-of-N selection.

A judge prompt must state: the task, the criteria in detail, the scoring system, and examples per score level with reasons. Practical findings:
- Models judge better with **classification labels** than numbers; discrete scales beat continuous; wider discrete ranges degrade (1–5 is the usual ceiling).
- Few-shot examples raise consistency (e.g., 65% → 77.5% for one GPT-4 setup) but lengthen prompts and multiply cost; consistency ≠ accuracy (a judge can be consistently wrong).
- Set judge temperature to 0. Pin the judge model version. **A judge is model + prompt + sampling config; change any of them and you have a different judge** and your trend lines break.

Limitations to design around:
- **Inconsistency**: probabilistic scores; mitigate with temperature 0, examples, repeated runs.
- **Criteria ambiguity**: "faithfulness" in one library is 1–5, in another 0/1, in another YES/NO with different instructions. Scores across tools are not comparable. Never adopt a judge whose model and prompt you cannot see.
- **Cost and latency**: a judge call per output per criterion doubles or quadruples model calls. Use weaker/smaller judges, spot-check a sample, run judges async off the request path. In-line judges as guardrails trade latency for risk.
- **Privacy/IP**: a commercial judge sees your data.

Always pair judges with some exact evaluation and periodic human review.

## 4. Judge biases and mitigations

| Bias | Symptom | Mitigation |
|---|---|---|
| Self-bias | judge prefers outputs from its own model family (10–25% inflated win rate observed) | use a judge from a different family than the generator, or multiple judges |
| Position bias | prefers the first (sometimes last) option in pairwise prompts | run both orderings and keep only consistent verdicts (or average); randomise order |
| Verbosity bias | prefers longer answers even with factual errors | enforce length-insensitive rubrics; compare at similar lengths; add explicit "penalise padding" criteria; check correlation between judge score and length |
| Rubric drift | scores shift when someone "fixes a typo" in the judge prompt | version the judge prompt; log judge config with every eval run; re-baseline when it changes |
| Sycophancy to the question framing | judge trusts the answer's confident tone | require the judge to cite the evidence (context span) supporting its verdict |

## 5. Which model judges

- **Stronger judge than generator**: usual choice; lets a cheap generator serve traffic while an expensive model grades a sample (e.g., 1%) in the background. Reverse is also common: strong generator, cheap judge on 100% of outputs.
- **Self-evaluation**: self-bias makes it a weak score, but useful as a sanity check and as a self-correction step ("is this correct?" → revised answer).
- **Weaker/specialised judges**: judging is often easier than generating. Small specialised scorers are cheaper, faster, and can beat general judges for a fixed criterion:
  - *Reward models*: (prompt, response) → quality score.
  - *Reference-based judges*: (response, reference[, rubric]) → similarity or quality score.
  - *Preference models*: (prompt, A, B) → which is preferred.
  - *Entailment (NLI) classifiers*: (premise, hypothesis) → entailment / contradiction / neutral; a few-hundred-million-parameter model does this well and is the cheapest faithfulness check for RAG.
  - *Toxicity/hate classifiers*: small, fast, multilingual options exist.

## 6. Comparative evaluation

Rank systems by pairwise matches instead of absolute scores (Huyen ch. 3). Easier for humans when quality is subjective; does not saturate; hard to game by training on references; captures preference directly. Rating algorithms: Elo, Bradley–Terry (preferred by the main public arena at time of writing because Elo is order-sensitive), TrueSkill.

Constraints:
- Pairs grow quadratically; rankings lean on transitivity that may not hold for preferences.
- Preference is the wrong signal for questions with a correct answer ("is X linked to Y?"); users can only vote usefully on tasks they could do themselves. Asking a user who doesn't know the answer to pick one is also a bad UX.
- Win rate does not tell you *how good* either side is, nor map to business metrics (a 51% win rate may or may not move ticket resolution). You still need absolute evaluation for go/no-go decisions and cost–benefit.
- Not the same as A/B testing (one output per user) — in comparative eval the user sees both.

In-product use: show two candidates in the user's workflow (two code suggestions, two drafts) and record the pick; expect noise from users who click randomly, but a minority of careful voters can still carry the signal.

## 7. Evaluation criteria for an application

Four buckets (Huyen ch. 4). For a "summarise this contract" feature: domain capability = understands legal text; generation = summary is coherent and faithful; instruction following = respects length/format; cost/latency = acceptable per summary.

| Bucket | What it measures | How to measure |
|---|---|---|
| **Domain-specific capability** | does the model know/reason about the domain (code, SQL, legal, your product) | public domain benchmarks to shortlist; private close-ended sets (MCQ, classification) are cheap and reproducible but measure recognition, not generation; functional correctness where possible, plus efficiency (query runtime) and readability (judge) |
| **Generation capability** | fluency/coherence (now mostly solved for strong models; still matters for small models, low-resource languages), **factual consistency**, safety (toxicity, harmful instructions, hate, stereotypes, political bias), plus app-specific traits (conciseness, controversiality, friendliness) | AI judge, specialised classifiers, perplexity for fluency |
| **Instruction-following** | does it obey format and content constraints you give it | automatically verifiable constraints (IFEval-style: keywords, length, JSON, bullet counts) + yes/no checklists per instruction judged by AI (INFOBench-style). Build your own set with *your* instructions (YAML output, "never say 'as an AI'") |
| **Cost and latency** | TTFT, TPOT, total latency, cost per request, throughput/rate limits | measure on your prompts, not vendor claims; separate must-have from nice-to-have; see `inference-optimization.md` |

Write criteria as a table of criterion → metric → benchmark/dataset → hard requirement → ideal (e.g., cost per 1M output tokens < $30 hard / < $15 ideal; p90 TTFT < 200 ms; factual consistency on internal hallucination set > 0.8 hard / > 0.9 ideal; required TPM capacity).

Poor instruction following is easy to confuse with missing capability (model outputs HAPPY when asked for POSITIVE/NEGATIVE/NEUTRAL: it understood sentiment but ignored the label set). When a model fails, suspect the instruction before the model.

## 8. Factual consistency

Two settings:
- **Local**: is the output supported by the provided context (summary vs document, answer vs retrieved chunks, reply vs policy)? This is the RAG metric, and the easier one.
- **Global**: is it true in the world? Requires first deciding what the facts are; much harder.

Detection approaches, cheapest to dearest:

| Approach | Mechanism | Trade-off |
|---|---|---|
| NLI classifier | (context, claim) → entailment / contradiction / neutral | cheap, fast, local only; claims must be split out first |
| AI judge with context | "does the summary contain claims unsupported by the source?" | general; good correlation with humans; costs a model call |
| Self-consistency (SelfCheckGPT) | sample N answers; disagreement implies hallucination | reference-free; N× cost |
| Search-augmented verification (SAFE-style) | decompose into atomic claims → make each self-contained → query a search/retrieval API → judge each claim against results | the way to do global checks; many calls per output |

Design your hallucination eval set around where the model actually hallucinates. Common hot spots: niche entities (less-referenced things), and questions presupposing something that does not exist ("what did X say about Y" when X never did). Include "no answer in context" cases and verify the system declines.

Safety: use provider moderation endpoints or small toxicity classifiers on inputs and outputs; general judges can do it but cost more. Benchmarks exist for adversarial prompts (e.g., RealToxicityPrompts at time of writing); your own red-team set matters more.

## 9. Instruction following and role adherence

- Build a private instruction set mirroring production instructions; score = fraction of constraints satisfied. Automate what is verifiable (regex, schema validation, length, forbidden phrases); use yes/no judge checklists for the rest.
- Watch for prompt-format sensitivity: trivial changes (an extra space, "Choices:" header) can flip answers on close-ended tasks; include format variants in the eval set.
- Role/persona features: evaluate both style ("sounds like the persona") and knowledge boundaries ("does not know things the persona wouldn't", no spoilers). Heuristics help (average reply length for a terse persona); otherwise AI judge with a role-specific rubric.

## 10. Designing the evaluation pipeline

Step 1. **Evaluate every component, and the whole.** A two-step pipeline (PDF → text → employer extraction) fails for two different reasons; score each step with its own method (text similarity for extraction; accuracy given correct text for the second). Evaluate per **turn** (each output's quality) and per **task** (did the user get the job done, in how many turns). Task-level is what users care about; turn-level is what you can debug. Define task boundaries explicitly for multi-turn features.

Step 2. **Write the evaluation guideline.** The hardest part is defining "good", not scoring. Correct ≠ good ("you are a terrible fit" may be correct and still a bad response). Define what the app must *not* do (out-of-scope topics and the expected refusal). Pick 2–3 criteria per feature (support bot: relevance, factual consistency, safety). Choose a scoring system per criterion (binary, 3-way, 1–5), write a rubric with an example and justification for every score level, and validate it with humans: if two colleagues can't apply it consistently, refine it. The rubric doubles as the annotation guideline for future finetuning data.

**Tie metrics to business metrics**: e.g., 80% factual consistency → can automate 30% of tickets; 90% → 50%; 98% → 90%. Define the usefulness threshold below which the feature is not shippable. Beware optimising engagement/stickiness metrics that reward addictive or extreme behaviour.

Step 3. **Choose methods and data per criterion.** Mix: a cheap classifier on 100% of traffic plus an expensive judge on 1%; semantic similarity for relevance; NLI for faithfulness; logprobs for classification confidence (three labels at 30–40% each = unconfident; 95% = confident). Keep human evaluation as the North Star: a daily human-reviewed sample of production conversations (hundreds/day at larger orgs) catches drift that metrics miss. Plan how production feedback (see `architecture-and-feedback.md`) maps onto offline metrics.

**Annotate and slice.** Use real production inputs. Maintain several eval sets:
- a production-distribution sample (overall health),
- slices by tier, channel, language, input length, topic,
- known-failure cases,
- user-error cases (typos, bad formatting) if common,
- out-of-scope inputs (verify refusal/handoff),
- adversarial/injection cases (see `prompting.md`).

Slicing avoids Simpson's paradox (A beats B overall but loses on every slice) and localises regressions. Rule: if you care about something, put a test set on it.

Step 4. **Iterate, but version everything.** Criteria evolve; the pipeline must still be stable enough that score changes mean application changes. Log eval data version, rubric version, judge model + prompt + sampling config, and the application config under test, per run.

## 11. Eval set size and statistical sanity

- Bootstrap check: resample your N examples with replacement, re-score several times; if scores swing (90% vs 70%), N is too small.
- Rough sample sizes to be ~95% confident one system beats another (OpenAI's heuristic, via Huyen ch. 4): detecting a 30% difference ≈ 10 examples; 10% ≈ 100; 3% ≈ 1,000; 1% ≈ 10,000. Every 3× smaller effect needs ~10× more data.
- Public benchmarks typically have ~1,000 examples; ~300 is a floor, more if examples are synthetic. For a feature-level eval, a few hundred well-chosen, sliced examples beat thousands of random ones.

## 12. Evaluating the evaluator

Ask, periodically:
- Does a higher score actually correspond to better outputs and better business outcomes? (Spot-check judge verdicts against human labels; compute agreement.)
- Is it reproducible? Same pipeline twice → same result? Variance across eval sets?
- Are metrics redundant (perfectly correlated → drop one) or suspiciously uncorrelated (either an insight or a broken metric)?
- What latency and cost does evaluation add in production? Skipping it to save latency is a risky bet; make it async instead.

## 13. Judge prompt template

```
You are grading an answer produced by a support assistant for <product>.

TASK: Decide whether the ANSWER is factually consistent with the CONTEXT.
CRITERIA:
- Every factual claim in the ANSWER must be supported by the CONTEXT.
- Claims not covered by the CONTEXT count as unsupported, even if true in the world.
- Ignore style, tone, and length. Do not reward longer answers.
SCORING (choose exactly one label):
- SUPPORTED: all claims are supported.
- PARTIAL: at least one claim unsupported, none contradicted.
- CONTRADICTED: at least one claim contradicts the CONTEXT.
EXAMPLES:
<one example per label, with the quoted span that justifies it>

CONTEXT: {{context}}
QUESTION: {{question}}
ANSWER: {{answer}}

Return JSON: {"label": "...", "evidence": "<quoted span or 'none'>", "reason": "<one sentence>"}
```

Run at temperature 0, pinned model version, both A/B orderings for pairwise variants, and log the full config with each run.
