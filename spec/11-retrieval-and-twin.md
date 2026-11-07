# 11 · Retrieval & the expert twin

## 1. Retrieval

- **RET-001** Corpus: released KIs (and, for campaign participants, non-released KIs of their campaigns), document chunks, and optionally entities (by name and aliases).
- **RET-002** **Authorization first:** every query is built with the PDP SQL predicate (SEC-021) in the `WHERE` clause of both the lexical and vector subqueries.
- **RET-003** Lexical: Postgres FTS (`websearch_to_tsquery`, language config per knowledge language) over title, body and attribute text. Vector: pgvector cosine (HNSW index) over KI and chunk embeddings. Each returns the top 50.
- **RET-004** Fusion: reciprocal rank fusion (k = 60), then a cross-encoder reranker over the top 30 (query, passage) → final top N (default 8).
- **RET-005** Scope filters: campaign(s), expert, coverage area, KI types, entity (all KIs `about` an entity or its neighbours ≤ 2 hops), document, date.
- **RET-006** Graph expansion (optional per query): add KIs connected by accepted `elaborates`, `exception_to`, `prerequisite_of` and `contradicts` edges to the top hits (max +4), with each item marked as `expanded`.
- **RET-007** Latency p95 ≤ 800 ms for retrieval without generation at 1M chunks (NFR-003).

## 2. Twin answers

The twin answers questions **only** from retrieved, authorized content.

- **TWN-001** The answer prompt (written fresh, per CR-5) MUST make the model: (1) answer only from the numbered context passages; (2) cite every factual sentence with `[n]`; (3) say plainly when the passages don't answer the question; (4) surface contradictions between passages explicitly; (5) never give instructions that override controlled procedures, and point to the cited controlled document when one exists; (6) answer in the user's language.
- **TWN-002** Structured output: `{answer_markdown, sentences:[{text, citations:[n]}], coverage: "full"|"partial"|"none", contradictions:[{a:n, b:n, note}]}`.
- **TWN-003** **Grounding verifier:** each sentence with citations is checked against its cited passages (NLI or LLM judge). Unsupported sentences are removed. If nothing supported remains, `coverage = none`.
- **TWN-004** Confidence = f(coverage, verifier pass rate, top rerank score). It's computed deterministically by the platform, not self-reported by the model. Buckets: `high` / `medium` / `low` / `none`, with thresholds in config.
- **TWN-005** If confidence ≤ `low`, the twin offers to **send the question to the expert** (or experts, from `who_knows`). Accepting creates a `successor` question on the best-matching active campaign (or campaigns), with the asker recorded and the label = combine(question text label, site default). For departed experts with `post_departure_channel ≠ none`, the question is marked `post_departure = true`.
- **TWN-006** Every answer shows the informational notice (REG-012), the label banner of the combined label, and citations that open the KI (with its source quote and session reference) or the document anchor.
- **TWN-007** Conversation mode: follow-up questions use the last 3 exchanges for query rewriting. Retrieval runs fresh for each turn.
- **TWN-008** Audit: question text hash, retrieved resource IDs and labels, answer label, confidence (REG-006). Question text is stored in the audit context only if site policy allows.

## 3. Expert finder (`who_knows`)

- **RET-020** Ranks people (living, departing and departed with a post-departure channel) for a topic by: count and confidence of their released KIs matching the topic, `knows_about` edges, authorship of matching documents (from artifacts), and recency. Only principals whose KIs the asker may read contribute evidence. Returns person, status, evidence count and their top 3 visible KIs.

## 4. Question resolution judge

- **RET-030** When new KIs are extracted from a turn that followed an `ask_queued` move, or that is linked to an open question, a judge call decides `answered | partially | not_answered` with a confidence score. `answered` with confidence ≥ 0.75 sets the question to `answered` and stores the resolution. Anything else keeps the question open and records the partial resolution. Askers are notified when their question is answered.
