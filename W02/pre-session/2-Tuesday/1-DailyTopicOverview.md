---
week: W02
day: Tue
phase: Foundation
topic: "Enterprise Retrieval Engineering — pre-ingestion"
estimated_total_minutes: 60
last_verified: 2026-05-26
fde_situations: [1, 3, 6, 9, 11]
tech: [chunking, embedding-models, aws-bedrock-titan-embeddings-v2, mongodb-atlas-vector-search, bm25, hybrid-retrieval, reranking, citation-grounding, ragas]
sources_research_briefs: []
author: instructor
---

# W2 Tue Pre-Session — Enterprise Retrieval Engineering

> Read **before** W2 Tue morning. ~60 min across 6 topics. Tue is the first incident-mode war-room of the cohort and the first day code lands against Mon's RAG plan. The vendor's Q&A at 09:00 will exercise dual-source FAR vs DFARS precedence head-on; `POST /answer-qa` ships with zero grounding today. Today's pre-read closes the gap between Mon's ADRs (chunking strategy, embedding model, vector store, hybrid retrieval) and Tue afternoon's ingestion pipeline + Atlas Vector Search index + flat-file eval harness scaffold.

## 1. Why naive RAG breaks — failure modes you'll see Tuesday morning (10 min)

Naive RAG is the path of least resistance: chunk the corpus, embed each chunk, run dense top-k against the query, hand the top-k to the LLM, hope for the best. It works in demos. It fails in federal acquisitions.

Tomorrow's 09:00 vendor question — *"Section L.4 says proposals due 30 days; FAR 15.208(a) says 30 calendar days; DFARS 215.371-4 references different timing for streamlined acquisitions over $7.5M. Which applies?"* — is the worked example. Naive dense top-k over FAR alone returns FAR 15.208(a) confidently and misses the DFARS supplement entirely. The model composes a plausible answer, the platform auto-publishes it, and six months later OIG finds the platform contributed to an audit finding. Three failure modes show up tomorrow: **dual-source omission** (corpus missing the supplement), **synonym/paraphrase mismatch** (query uses "due date", clauses use "submission timing"), and **chunk-context loss** (a chunk for "(2) Notwithstanding paragraph (1)..." returned without its parent paragraph reads as nonsense). Mon's ADRs name the dimensions; today's morning is the first lived test of whether the ADRs were any good.

**Key terms:** naive top-k, dual-source omission, paraphrase mismatch, chunk-context loss.

## 2. Retrieval strategies — dense vs sparse vs hybrid (15 min)

Three retrieval modes, three different failure surfaces. You commit on the specific operator today.

**Dense retrieval** uses embedding-vector similarity (Atlas `$vectorSearch`) — strong on paraphrase and semantic queries ("how long do vendors have to respond?" matches "30 calendar days"), weak on rare tokens (clause IDs like `52.212-4(a)(2)` get fuzzed away). **Sparse retrieval** uses lexical match (Atlas `$search`, BM25 under the hood) — strong on exact-token queries (clause IDs, FAR Part numbers), weak on paraphrase. **Hybrid retrieval** merges the two ranked lists via Reciprocal Rank Fusion (RRF): each document's RRF score is the sum over rankers of `1 / (k + rank)` (k=60 is the standard default). MongoDB Atlas ships this as `$rankFusion` — combines `$vectorSearch` + `$search` with default 0.5/0.5 weights. For FAR + DFARS the query language is more semantic than keyword, so you'll typically vector-weight higher — but commit a default Tue, tune Fri with the eval harness.

Composition rule for today: **plain Python, not LCEL pipes**. D-033 LangChain v1.0 posture: hybrid retrieval is `merged = rrf_merge(dense_hits, sparse_hits)` — not `dense | sparse | rerank`. LangChain primitives are fine at each step; the framework abstraction is not.

**Key terms:** dense vs sparse, BM25, Reciprocal Rank Fusion, Atlas `$rankFusion`, `numCandidates`, plain-Python composition.

## 3. Re-ranking fundamentals (8 min)

Retrieval gives you candidates; reranking gives you confidence in the top-k that reaches the LLM. The standard pattern is "retrieve 50, rerank to 5" — bi-encoder retriever pulls a wide net cheaply, cross-encoder reranker scores each candidate against the query expensively but accurately.

The trade-off is latency. A cross-encoder rerank over 50 candidates adds 200–500ms to the request; the cohort will feel it Thu when the latency budget gets argued at the war-room. The candidates worth knowing tonight: **Cohere Rerank-3** (managed API, strong on English regulatory text, $-per-1k-rerank pricing), **BGE-reranker-v2-m3** (open-weight, self-host on Bedrock or Sagemaker, no per-call billing but you carry the GPU), and **in-prompt LLM rerank** (send candidates to Bedrock Claude as a "score these 1-5" prompt — slowest, most flexible, easiest to abuse). Mon's ADR may have committed to one; Wed's scenario-alternative will force the defense.

**Key terms:** bi-encoder vs cross-encoder, cascade rerank, Cohere Rerank-3, BGE-reranker-v2, in-prompt rerank, latency cost of precision.

## 4. Citation grounding — every quote ties to a chunk-ID + clause-ID + last_revised (10 min)

Citation grounding is the federal-acquisitions discipline that separates a real RAG platform from a demo. Every clause-quote the drafter emits must trace to: a **chunk-ID** (the retrieved span), a **clause-ID** (`FAR 15.208(a)`, `DFARS 215.371-4`), a **`corpus_source`** (`far` or `dfars`), and a **`last_revised`** date — all visible in the Drafting Wizard UI, all auditable.

Tomorrow afternoon's `POST /rag/clause-search` response envelope makes this concrete. The shape committed at the morning war-room:

```
chunks: [{text, clause_id, corpus_source, last_revised, score}, ...]
citations: [{clause_id, corpus_source, last_revised}, ...]
precedence_note: "DFARS supplements FAR per 48 CFR §201.104"  // when dual-corpus + DoD context
```

The discipline matters because clause precedence (48 CFR §201.104 — DFARS supplements FAR for DoD acquisitions) can't be encoded if the chunks don't carry their corpus source. This is also the **first thread of HITL #2** — the W2 Thu war-room will hang the `needs_human_review` envelope off this exact field set (when faithfulness < threshold OR when corpus precedence is ambiguous, escalate to a contracting officer). Don't memorize HITL #2 details tonight; just know that Tue's citation-grounding shape is what Thu's HITL gate reads from.

**Key terms:** chunk-ID, clause-ID, `corpus_source`, `last_revised`, dual-source citation envelope, clause precedence (48 CFR §201.104), HITL #2 (preview).

## 5. Retrieval evaluation as a flat-file pattern + RAGAS metrics (10 min)

You can't fix what you can't measure. Tuesday scaffolds the eval harness; Friday builds it out and turns it into a CI gate.

The scaffold is deliberately flat-file: **`tests/rag-eval/qa.jsonl`** with hand-curated `{question, expected_chunks, expected_answer_substrings}` rows. ~10 rows tomorrow; ~50 by Friday over FAR Part 15 + DFARS 215.3xx. LangSmith is deferred to W5 per D-031 — not because it's wrong, but because the cohort needs to feel what an eval harness actually does before reaching for managed tracing.

The four **RAGAS-style** dimensions you'll hear tomorrow (operational definitions, not marketing terms):
- **Faithfulness** — does the answer make claims only supported by the retrieved chunks?
- **Context recall** — did retrieval return all the chunks needed to answer?
- **Context precision** — of the chunks returned, how many were actually used?
- **Answer relevance** — does the answer address the question that was asked?

> [!instructor-review]
> Anti-pattern guarded against: "RAGAS faithfulness is sufficient" (known-bad-pattern `ragas-faithfulness-only`). All four dimensions are taught as a set per programme spec — faithfulness alone misses context-recall (you cited correctly from the wrong corpus) and context-precision (you cited 30 chunks when 3 were needed) failure modes.

**Key terms:** flat-file `qa.jsonl`, faithfulness, context recall, context precision, answer relevance, LLM-as-judge (Friday), held-out QA set.

## 6. Atlas index management + design-planning artifacts (7 min)

Atlas Vector Search indexes are not free, and they're not silent. Building the FAR+DFARS index tomorrow costs Bedrock embedding tokens (real `InvokeModel` per D-060 — token/cost accounting from request 1) plus Atlas compute. Re-indexing a 3,500-page corpus end-to-end takes ~20 minutes; you'll do it twice tomorrow and that's already a meaningful budget line.

Two things to get right at index-build time:

1. **Filter fields baked into the schema** — the `agency_id` filter field is declared at index creation, not bolted on at query time. This is how debt **Item 10** gets closed at the index level (cross-tenant retrieval boundary). Atlas's `fields` array supports `vector` + `filter` types; both committed Tue. Note: `agency_id` lives on tenant-scoped collections (`Solicitation`, `Proposal`, `AuditEvent`) — *not* on `ClauseLibraryEntry`, which is cross-tenant (every agency reads the same FAR). Wed war-room makes this concrete with a cross-tenant leak scare.
2. **ANN vs ENN choice** — Approximate Nearest Neighbor is the production default (`numCandidates: 100`, `limit: 10`); ENN (exact) is for small corpora. FAR+DFARS at 3,500 pages → ANN.

The design-planning artifacts (chunking-strategy ADR, embedding-model ADR, vector-store ADR, hybrid-retrieval ADR) committed Mon become living docs today — annotated with the operator choices (`$rankFusion` weights, `numCandidates`, rerank cascade depth) as the implementation lands. ADRs aren't write-and-forget; the discipline is annotate-as-you-implement so Wed's scenario-alternatives have honest baselines to compare against.

**Key terms:** Atlas filter field declaration, ANN vs ENN, `numCandidates`, `agency_id` (tenant-scoped only), cross-tenant vs tenant-scoped collections, ADR-as-living-doc.

## What to read or watch tonight

- [Anthropic — Introducing Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval) (~12 min read), retrieved 2026-05-23 via /web-research. Focus on the "Contextual Embeddings" pattern (prepend a one-sentence parent-context summary to each chunk before embedding) and the reported retrieval-failure reduction. This becomes Wed's scenario-alternative #2 lead candidate.
- [MongoDB Atlas — Define Vector Search index syntax](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-type/) (~10 min read), retrieved 2026-05-23 via /web-research. Focus on the `fields` array — `vector` field + `filter` field. The `filter: agency_id` field is how Item 10 closes at the index level. ANN default with `numCandidates: 100`.
- [MongoDB Atlas — Hybrid Search with $rankFusion](https://www.mongodb.com/docs/atlas/atlas-vector-search/tutorials/reciprocal-rank-fusion/) (~10 min read), retrieved 2026-05-23 via /web-research. Atlas's built-in RRF aggregation operator for combining `$vectorSearch` + `$search`. The hybrid baseline Tue ships.
- [AWS — Amazon Titan Embeddings v2 model card](https://docs.aws.amazon.com/bedrock/latest/userguide/titan-embedding-models.html) (~8 min read), retrieved 2026-05-23 via /web-research. Dimension options (1024/512/256), unit-normalized output (cosine = dot-product), 8,192-token input limit, pricing per 1M input tokens.
- [Cohere — Rerank-3 model overview](https://docs.cohere.com/docs/rerank-overview) (~6 min read), retrieved 2026-05-26 via /web-research. Cross-encoder rerank API; latency cost; the "retrieve 50, rerank to 5" pattern named explicitly.
- [Ragas — Core Concepts: Metrics](https://docs.ragas.io/en/latest/concepts/metrics/index.html) (~8 min skim), retrieved 2026-05-26 via /web-research. Operational definitions of faithfulness, context recall, context precision, answer relevance. Read the four metric pages, skip the LLM-judge wiring (Friday).
- *(optional)* [GovInfo — eCFR Title 48 (FAR + DFARS bulk download)](https://www.ecfr.gov/current/title-48) (~10 min skim), retrieved 2026-05-23 via /web-research. The actual source the cohort downloads tomorrow afternoon. Public-domain text. Title 48 → Chapter 1 (FAR) + Chapter 2 (DFARS).

## Two questions to come in with tomorrow

1. *"FAR clauses sometimes contain tables (e.g., FAR 52.232-7 has a payment schedule table). What does the chunker do — preserve the table as one chunk, flatten it to text, or skip it? What does the retriever do when the vendor question references a number in that table?"* (The morning war-room exercises this when the vendor's Q&A drops in.)
2. *"DFARS 215.371-4 references 'FAR 15.371' explicitly in its text. When the cohort ingests both corpora, how does the chunk for DFARS 215.371-4 know to also retrieve FAR 15.371 alongside — or does it? And where does the `corpus_source` metadata get set: at chunk creation, at embedding time, or at index time?"* (The dual-source citation envelope shape from the morning whiteboard.)

## What you'll do W2 Tue

- **Morning war-room (3hr, incident-mode):** vendor Q&A surfaces FAR vs DFARS timing conflict; cohort traces the failure path through `POST /answer-qa`; whiteboards the dual-source citation envelope; commits 4 tickets across pairs (ingestion, schema, retrieval, Item 7 cleanup); writes the ADR draft on dual-source citation envelope.
- **Afternoon practical (3hr):** ingestion pipeline for FAR Part 15 + DFARS Part 215 into Atlas Vector Search with `corpus_source` + `supplements_far_clause_id` metadata; Bedrock Titan Embeddings v2 real `InvokeModel` calls (token/cost accounting from request 1); Atlas index built with `agency_id` filter field on tenant-scoped collections; dense + sparse + hybrid (`$rankFusion`) baselines; temporary HITL-escalate envelope on `POST /answer-qa`; flat-file `qa.jsonl` scaffold with ~10 rows.
- **EOD (16:30):** 4 PRs against `acquire-gov` merged or in review; temporary HITL envelope on `POST /answer-qa`; ADR on dual-source citation envelope committed to each pair-project repo; Atlas Vector Search index populated.

---

## Sources

All citations retrieved 2026-05-23 or 2026-05-26 via /web-research per `pipeline/RESEARCH-PROTOCOL.md`. Recency windows applied: hot-tech 3-month (Anthropic Contextual Retrieval, Bedrock Titan v2, Atlas hybrid `$rankFusion`, Cohere Rerank-3, Ragas metrics, LangChain v1.0 composition), foundation-stable 12-month (chunking/retrieval/reranking concepts), federal-regulatory 6-month (Title 48 eCFR, 48 CFR §201.104 precedence).

## Known-bad-pattern warnings against the reading

- **LangChain composition verbs** — when reading Atlas or Cohere tutorials, watch for the verb "chaining" used in a LangChain v0.x sense (`langchain-chaining-verb` blocklist entry) or LCEL `|` pipe syntax presented as foundational (`langchain-lcel-pipe`). v1.0 posture per D-033: plain Python composition, no `Chain` class, no `|` pipes as architecture. If a linked tutorial advocates LCEL pipes as fundamental, flag it — don't pattern-match.
- **Cosine similarity defaults** — some retrieval tutorials assert "cosine similarity is the standard" (`vector-cosine-default` blocklist entry). Titan v2 returns unit-length vectors so cosine and dot-product agree; for non-normalized embeddings you'd want dot-product or L2. Match the metric to the embedding model.
- **RAGAS faithfulness as sufficient** — flagged in Topic 5 above as an `[!instructor-review]` callout (`ragas-faithfulness-only`). All four dimensions are programme-required.
- **Pinecone namespaces as multi-tenancy** — if any Pinecone-adjacent tutorial linked from MongoDB/RAG materials advocates "namespaces for multi-tenancy" (`pinecone-namespaces-as-multitenancy`), ignore the framing. Tomorrow's Item 7 cleanup removes the unused `pinecone-client` dep entirely; the cohort doesn't carry that posture into W2.

Last verified: 2026-05-26
