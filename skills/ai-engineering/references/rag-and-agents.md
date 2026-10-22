# RAG, Agents, and Memory

Synthesis of Huyen, *AI Engineering*, ch. 6.

## Contents

1. [RAG: when and why](#1-rag-when-and-why)
2. [Retrieval algorithms: term-based vs embedding-based](#2-retrieval-algorithms)
3. [Hybrid search and reranking](#3-hybrid-search-and-reranking)
4. [Chunking](#4-chunking)
5. [Query rewriting and contextual retrieval](#5-query-rewriting-and-contextual-retrieval)
6. [Evaluating retrieval and RAG](#6-evaluating-retrieval-and-rag)
7. [Choosing a retrieval/vector solution](#7-choosing-a-retrievalvector-solution)
8. [RAG beyond text: multimodal and tabular (text-to-SQL)](#8-rag-beyond-text)
9. [Agents: definition, tools, and why they need stronger models](#9-agents)
10. [Planning: decouple plan from execution](#10-planning)
11. [Function calling mechanics](#11-function-calling-mechanics)
12. [Reflection and error correction](#12-reflection-and-error-correction)
13. [Tool selection](#13-tool-selection)
14. [Agent failure modes and evaluation](#14-agent-failure-modes-and-evaluation)
15. [Memory](#15-memory)
16. [RAG design checklist](#16-rag-design-checklist)

---

## 1. RAG: when and why

A model needs instructions (shared across queries) and context (specific to each query). RAG constructs per-query context by retrieving from external memory: internal docs, user history, the web. It is the foundation-model analogue of feature engineering (Huyen ch. 6).

Long context does not kill RAG: data always outgrows the window, models use the middle of long contexts poorly, and every context token costs money and latency. RAG keeps input small and relevant. Exception: if the whole corpus is small (one provider's guidance at time of writing: under ~200k tokens, ~500 pages), stuff it in the prompt and use prompt caching instead of building retrieval.

Architecture: **retriever** (index + query) → **generator**. Quality is bounded by the retriever; invest there first. Document = chunk for retrieval purposes.

## 2. Retrieval algorithms

| | Term-based (BM25, Elasticsearch/Lucene, TF-IDF) | Embedding-based (dense vectors + ANN search) |
|---|---|---|
| Relevance | lexical overlap; TF × IDF, BM25 normalises by document length | semantic similarity (cosine) between query and chunk embeddings |
| Indexing | inverted index; fast, cheap | embed every chunk; re-embed on change; vector index build (HNSW etc.) |
| Query speed | much faster | query embedding + ANN search; usually small next to generation latency, but not free |
| Out-of-box quality | strong baseline, hard to beat; "genuinely improving over BM25 is hard" | can exceed term-based, especially after finetuning the embedding model; handles natural-language queries |
| Weaknesses | term ambiguity ("transformer"), no synonyms, tokenisation choices matter (n-grams for "hot dog", lowercasing, stop words) | embeddings blur exact identifiers: error codes, SKUs, product names, ticket IDs; depends on embedding model quality |
| Cost | cheap | embedding generation, vector storage, and query costs; vector DB spend can be 20–50% of model API spend |
| Tuning surface | small | large: embedding model, chunking, index params, finetuning |

Vector search: exact k-NN for small sets; ANN for everything else. Index families (buckets/trees/graphs): LSH, HNSW (graph; high recall, fast queries, expensive to build and memory-hungry), IVF (k-means clusters, ~100–10k vectors each) often paired with product quantization, Annoy (random-projection trees). Evaluate ANN options on recall, QPS, build time, index size; HNSW is the usual default when memory permits. Any database with vector support counts as a vector DB; prefer the one you already run (e.g., Postgres with pgvector, OpenSearch) unless scale forces otherwise.

## 3. Hybrid search and reranking

Production retrieval combines methods:
- **Sequential**: cheap, broad retriever (BM25) fetches candidates → precise, expensive step (vector similarity or a cross-encoder reranker) reorders them. Keyword-anchor then semantic-filter ("fetch everything mentioning customer X, then rank by 'who handles most sales'").
- **Parallel ensemble**: run term and embedding retrievers, merge rankings with **reciprocal rank fusion**: score(D) = Σ over retrievers of 1/(k + rank_i(D)), k ≈ 60. Simple and robust; the standard first implementation of hybrid search.

Reranking is also where you shrink the candidate list to fit the context budget, apply recency weighting (news, email, market data), or apply business rules. In RAG, inclusion matters more than exact rank order, but order still affects attention (start/end of context are favoured), so put the best chunks first and last.

## 4. Chunking

How you index depends on how you retrieve; chunking is the most impactful cheap decision.

| Strategy | Notes |
|---|---|
| Fixed size (chars/words/tokens) | simplest; e.g., 2,048 chars or 512 tokens; token-based aligns with the generator but ties the index to one tokenizer (re-index on model switch) |
| Fixed count of sentences/paragraphs | preserves natural units |
| Recursive (section → paragraph → sentence until it fits) | fewer mid-thought cuts; good default for prose |
| Format-aware | code splitters per language, Q&A pairs as one chunk, per-language rules for non-English text |
| Overlap | small overlap (e.g., ~1% of chunk size) so boundary information lands in at least one chunk ("I left my wife | a note") |

Size trade-off: smaller chunks → more diverse candidates fit in context, but lose document-level context (a fact mentioned only in the first half is unreachable from the second), double the number of embeddings to generate/store, and slow vector search. Chunk ≤ embedding model context and ≤ generator budget. No universal best size; test 256/512/1,024 tokens against your retrieval eval set.

## 5. Query rewriting and contextual retrieval

**Query rewriting**: make the user's turn self-contained before retrieval. "How about Emily Doe?" after a question about John Doe must become "When did Emily Doe last buy from us?". Do it with a small model and a prompt like "Given the conversation, rewrite the last user message to state what the user is actually asking". Handle identity resolution ("his wife" → lookup) explicitly, and make the rewriter say "unresolvable" rather than invent a name. Also use rewriting for expansion (synonyms, spelling) and normalisation.

**Contextual retrieval**: enrich each chunk before indexing so it is findable:
- Metadata: tags, keywords, extracted entities (error codes, product names) → indexed for term search even when the body is embedded.
- Questions the chunk answers (support articles augmented with "I can't log in", "forgot password"); several teams report retrieval works best over Q&A-shaped data.
- Document context: prepend title/summary, or an LLM-generated 50–100-token sentence situating the chunk within its document (prompt: give the whole document and the chunk, ask for a succinct context for retrieval purposes; prepend to the chunk; index the result with both embeddings and BM25).

## 6. Evaluating retrieval and RAG

Evaluate three layers (Huyen ch. 6; methods in `evaluation.md`):

1. **Retriever**: build (query → relevant chunk IDs) sets from real queries, labelled by humans or an AI judge. Metrics: context precision (retrieved that are relevant), context recall (relevant that were retrieved; needs full relevance labels, so often approximated), and rank-aware NDCG / MAP / MRR when order matters. Precision is cheap to compute in production with a judge; recall is not.
2. **Embeddings** (for dense retrieval): public multi-task benchmarks to shortlist (e.g., MTEB at time of writing); your own retrieval set to decide.
3. **End-to-end**: faithfulness of the answer to retrieved context, answer relevance, and task success. A retriever is good only if it makes final answers better.

Retrieval harnesses (BEIR-style) and ANN benchmarks exist; use them for shortlisting, not decisions.

## 7. Choosing a retrieval/vector solution

Ask: supported mechanisms (hybrid? filters on metadata?), supported embedding models and ANN algorithms, scale for storage and query traffic, indexing throughput and bulk add/delete, query latency per algorithm, pricing basis (by vectors stored vs queries), plus enterprise concerns (access control per tenant/document, compliance, data-plane separation). Re-embedding cost on frequent data change is a hidden line item.

## 8. RAG beyond text

- **Multimodal**: retrieve images/audio by metadata (captions, tags) or by content with a joint text–image embedding model (CLIP-style): embed everything into one space, query with the text embedding.
- **Tabular (text-to-SQL)**: workflow = pick relevant tables (an extra step when schemas don't fit the context) → generate SQL from query + schemas → execute → generate the answer from results. Treat the SQL generator as a tool with its own eval (execution accuracy, runtime), and run it read-only with a statement timeout; see injection notes in `prompting.md`.

## 9. Agents

An agent perceives an environment and acts on it; it is defined by its environment and its **tool inventory**. A RAG pipeline is a simple agent (tools: retriever, SQL executor). The model is the planner: reads the task and tool feedback, decides the next action, judges completion (Huyen ch. 6).

Tool categories:
- **Knowledge augmentation** (read-only): retrievers, SQL, internal APIs (inventory, people search), email/Slack readers, web search/APIs. Web access fixes staleness and exposes you to the open internet; pick APIs carefully.
- **Capability extension**: calculator, calendar, unit/timezone converters, translators, code interpreter (sandboxed), image/audio tools that make a text model effectively multimodal. Giving a model a tool beats training it to do the tool's job; tool-augmented agents outperform the bare model on several benchmarks.
- **Write actions**: send email, update DB, initiate transfer. The value is automation of whole workflows; the risk is proportional. Don't give an unreliable agent write access you wouldn't give an intern.

Why agents need stronger models: **compound error** (95% per step → ~60% over 10 steps → ~0.6% over 100) and **higher stakes** (tools make failures consequential). Multi-step tasks also cost real time and money.

## 10. Planning

A task = goal + constraints (two-week trip, $5k budget). Many decompositions solve a task; some are far more efficient (filter by the rarer condition first). Coupling plan and execution in one CoT prompt risks a 1,000-step runaway you only notice on the invoice.

**Decouple**: generate plan → validate plan → execute → evaluate outputs → re-plan if needed. Validation can be heuristic (reject plans using tools not in inventory; cap step count), an AI judge ("is this reasonable?"), or a human. Generate several plans in parallel and pick the best if latency allows. Each component (planner, validator, executor, intent classifier) is effectively its own agent; most real agents are multi-agent systems.

Design decisions:
- **Intent classification first**: routes to the right tools, and classifies IRRELEVANT so the agent refuses instead of burning tokens on impossible tasks.
- **Human in the loop at any stage**: humans can supply the high-level plan, approve it, or execute the risky steps. Define the automation level per action type up front.
- **Granularity**: detailed plans are hard to generate, easy to execute; high-level plans the reverse. Plan hierarchically (coarse → fine). Prefer **natural-language plan steps** ("retrieve best-selling product last week") over literal function names: robust to tool renames, less hallucination, and translation to concrete calls is a simpler task a cheaper model can do.
- **Control flow**: sequential, parallel (fan out over items), conditional, loop. Non-sequential flows are harder to generate and execute; check that your framework supports parallel tool calls, since fan-out is where latency wins come from.
- Planning is search; models plan better when they can predict action outcomes (a world model) and when they can backtrack (revise or restart). Weak planning is often a tooling problem, not only a model problem.

Levers to improve planning: better system prompt and examples; clearer tool descriptions and parameter docs; simpler tools (split complex functions); stronger model; finetune for plan generation.

## 11. Function calling mechanics

1. Declare the tool inventory: name, parameters with types and descriptions, documentation of behaviour.
2. Per request, pass the subset of tools allowed and a tool-choice mode (`auto` / `required` / `none`).
3. The model returns tool calls with arguments; you execute, append results to history, loop until a final answer.

APIs can guarantee a valid function name and argument *shape*, never correct argument *values*. Log every call with arguments and outputs; inspect them during development and sample them in production. Parameter guesses are a top failure source ("best-selling products" over what window? how many?), so design tools with explicit defaults and make the agent ask when ambiguity matters.

## 12. Reflection and error correction

Reflection points: after receiving the task (feasible?), after planning (sensible?), after each step (on track?), after completion (actually done?). Mechanisms: self-critique prompts, a separate evaluator/scorer, or a second agent reviewing the first (actor–critic). The ReAct loop (thought → action → observation, repeated) interleaves planning and reflection; Reflexion-style setups add an evaluator plus a "what went wrong" analysis that feeds the next attempt (e.g., failed 1/3 of tests → "didn't handle all-negative arrays" → regenerate). Reflection is cheap to implement and often a large quality gain, paid for in tokens, latency, and context space consumed by the few-shot examples needed to hold the format.

## 13. Tool selection

More tools = more capability but harder selection and longer tool descriptions eating context. Experiment: compare agent performance across tool sets; ablate (remove a tool; if nothing drops, drop it); find tools the agent misuses and either re-document, re-shape, or replace them; plot tool-call distributions and tool transitions (tools frequently used together can be merged into one). Different models prefer different tools, so re-check when you swap models. Evaluate agent frameworks on the planners, control flows, and tool categories they support, and on how easy it is to add a tool.

## 14. Agent failure modes and evaluation

Evaluate by enumerating failure modes and measuring their frequency.

| Category | Failure | How to measure |
|---|---|---|
| Planning: tool use | invalid tool; valid tool with invalid parameters; valid tool with wrong values | planning dataset of (task, tool inventory); generate K plans per task; % valid plans, plans needed per valid plan, % valid tool calls, breakdown by error type and by tool |
| Planning: goal | plan solves the wrong task or violates constraints (budget, destination) | task-level success with constraint checks; include **time** as a constraint when results expire |
| Reflection error | agent declares success when it hasn't (assigned 40 of 50 people and insists it's done) | independent completion check, not the agent's own claim |
| Tool failure | correct tool, wrong output (bad SQL, wrong caption); translation errors from NL plan to calls; missing tool for the domain | test each tool independently; benchmark the translator; have domain experts identify missing tools; print every call and output |
| Efficiency | valid but wasteful | steps per task, cost per task, latency per action; compare with a baseline agent or human (and remember AI parallelism changes what "efficient" means) |

Public agent benchmarks (function-calling leaderboards, travel-planning style tasks at time of writing) exist for shortlisting models; your own task set decides.

## 15. Memory

Three tiers: **internal knowledge** (weights; change only by training), **short-term** (the context; fast, limited, per-task), **long-term** (external store accessed by retrieval; persistent, deletable without retraining). Place information by frequency of use: needed by every task → finetune in; rarely → long-term; immediate → context.

Benefits: handle information overflow within a session, persist across sessions (personalisation, no re-explaining), consistency (the model can see its own earlier answers), structural integrity (keep tables/queues in a real store instead of flattened text).

Management = what to add/remove from short-term memory when the context budget (e.g., 70% short-term, 30% reserved for retrieved context) is hit:
- FIFO / keep last N messages or tokens: simple, but drops early messages that often hold the purpose of the conversation.
- Summarise older turns (plus entity tracking); optionally a classifier that merges summary with sentences the summary missed.
- Reflective update: after each action, decide whether new information is inserted, merged, or replaces outdated/contradictory memory. Decide a contradiction policy (newest wins, or judge) per use case.

Memory retrieval is RAG over the long-term store.

## 16. RAG design checklist

- [ ] Corpus small enough to stuff into context (+ prompt cache)? If yes, skip retrieval.
- [ ] Start with BM25 (or your existing search engine) as baseline; add embeddings; fuse with RRF.
- [ ] Chunking: recursive by structure, 256–1,024 tokens, small overlap; format-aware for code/Q&A; test sizes against the retrieval eval set.
- [ ] Contextual enrichment: title/summary prefix, extracted identifiers as metadata, candidate questions for support content.
- [ ] Query rewriting for conversational inputs; unresolvable → say so.
- [ ] Reranking/top-k to fit context budget; best chunks first and last.
- [ ] Metadata filters for tenant/ACL before similarity search; never rely on the model for authorisation.
- [ ] Prompt: answer only from context, cite chunk IDs, explicit "not found" behaviour; retrieved text treated as untrusted data.
- [ ] Evals: retrieval precision/recall set, faithfulness judge, end-to-end task success, "answer not in corpus" slice.
- [ ] Ops: index refresh strategy and re-embedding cost; log query, retrieved IDs, scores, prompt version; cache frequent queries.
