---
week: W02
day: Tue
phase: Foundation
topic: "Chunking strategies + embedding model selection — pre-ingestion"
estimated_total_minutes: 50
last_verified: 2026-05-23
fde_situations: [1, 3, 6, 9, 11]
tech: [chunking, embedding-models, aws-bedrock-titan-embeddings-v2, mongodb-atlas-vector-search, bm25, hybrid-retrieval]
sources_research_briefs: []
author: instructor
---

# Pre-session reading — Chunking strategies + embedding model selection

Week 2, Day 2 (Tue). Estimated total time on task: ~50 minutes. Last verified: 2026-05-23.

> Read **before** W2 Tue morning. Tue is implementation day — the pair's Mon ADRs become a working ingestion pipeline + Atlas Vector Search index by EOD. This pre-read closes the gap between the Mon ADR ("we'll chunk at the sub-paragraph boundary") and the Tue afternoon decision ("here's exactly how").

## 1. Why this matters (~85 words)

Chunking is the most-overlooked RAG decision and the most common cause of bad retrieval. Chunk too small and the model loses context; chunk too large and the retriever surfaces irrelevant adjacent material. For FAR/DFARS the natural boundary is the **sub-paragraph** (e.g., FAR 52.212-4(a)(1) is one chunk, 52.212-4(a)(2) is another) — but this isn't free: it forces the embedding model to capture meaning of fragments like *"(2) Notwithstanding paragraph (1)..."* without their parent context. Tue's work is making that trade-off real.

## 2. Core concept in 5 minutes

A chunking strategy answers three questions: **what is a chunk** (clause? sub-paragraph? sentence? sliding window of N tokens?), **what context does it carry** (does the chunk include the parent clause's heading? the FAR Part number?), and **how do chunks overlap** (overlap helps recall but inflates index size and cost). For federal regulatory text, three patterns are worth knowing: (1) **structure-aware chunking** — let the document's existing hierarchy (FAR Parts → Subparts → Sections → Paragraphs) define boundaries; (2) **parent-child indexing** — index sub-paragraphs but retrieve the parent paragraph too, so the LLM sees context; (3) **contextual retrieval** (Anthropic, Sep 2024) — prepend a one-sentence context summary to each chunk **before embedding**, so chunks aren't isolated from their parent meaning. Pick one for Tue; the others become Wed scenario-alternatives.

The embedding model decision interacts with chunking. Bedrock Titan Embeddings v2 supports three output dimensions (1024 default, 512, 256). Smaller dims = lower cost + lower latency, but lossy. Domain-tuned models (e.g., a legal-domain finetune) sometimes beat general-purpose ones on regulatory text — but cost goes up and you lose Bedrock's FedRAMP-managed boundary. Wed's scenario-alternative will force you to defend the choice.

## 3. What to read or watch tonight

- [Anthropic — Introducing Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval) (~12 min read), retrieved 2026-05-23 via /web-research. Focus on the "Contextual Embeddings" pattern (prepend a context summary to each chunk before embedding) and the reported 35% retrieval-failure-reduction. This becomes the Wed scenario-alternative #2's lead candidate.
- [MongoDB Atlas — Define Vector Search index syntax](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-type/) (~10 min read), retrieved 2026-05-23 via /web-research. Focus on the `fields` array — `vector` field + `filter` field. The `filter: agency_id` field is how we close debt **Item 10** at the index level. Note: ANN (approximate) is the default; ENN (exact) is for small corpora — FAR/DFARS at 3,500 pages is borderline for both, default to ANN with `numCandidates: 100`.
- [AWS — Amazon Titan Embeddings v2 model card](https://docs.aws.amazon.com/bedrock/latest/userguide/titan-embedding-models.html) (~8 min read), retrieved 2026-05-23 via /web-research. Focus on: dimension options (1024/512/256), normalization (Titan v2 returns unit-length vectors so cosine and dot-product agree — relevant to the `vector-cosine-default` known-bad-pattern warning), token limits (8,192 input tokens per call), pricing per 1M input tokens.
- [MongoDB Atlas — Hybrid Search with $rankFusion](https://www.mongodb.com/docs/atlas/atlas-vector-search/tutorials/reciprocal-rank-fusion/) (~10 min read), retrieved 2026-05-23 via /web-research. Atlas's built-in Reciprocal Rank Fusion (RRF) for combining `$vectorSearch` + `$search` (BM25 sparse). This is the hybrid-retrieval baseline Tue ships.
- *(optional)* [LangChain v1.0 — `RecursiveCharacterTextSplitter` reference](https://docs.langchain.com/oss/python/integrations/splitters/character_text_splitter) (~6 min skim), retrieved 2026-05-23 via /web-research. The standard text-splitter. Useful as a fallback for sub-paragraphs that exceed Titan's 8,192-token limit (rare but possible for long FAR clauses with embedded tables).

## 4. Two questions to come in with tomorrow

1. *"FAR clauses sometimes contain tables (e.g., FAR 52.232-7 has a payment schedule table). What does the chunker do — preserve the table as one chunk, flatten it to text, or skip it? What does the retriever do when the vendor question references a number in that table?"* (Tue morning war-room exercises this when the vendor's Q&A drops in.)
2. *"DFARS 215.371-4 references 'FAR 15.371' explicitly in its text. When the cohort ingests both corpora, how does the chunk for DFARS 215.371-4 know to also retrieve FAR 15.371 alongside? Or does it?"* (The cross-clause-link / precedence problem from Mon.)

## 5. Glossary refresh (terms you'll hear tomorrow)

- **Chunk boundary** — Where one chunk ends and the next begins (structural: clause, paragraph, sub-paragraph; or character-based: N-token window).
- **Chunk overlap** — Tokens shared between adjacent chunks. Common defaults: 10–20% of chunk size.
- **Parent-child indexing** — Index the smaller chunk for retrieval; return the larger parent chunk to the LLM.
- **Contextual embedding** — Prepend a one-sentence summary of the parent context to each chunk *before* embedding (Anthropic, Sep 2024).
- **ANN / ENN** — Approximate / Exact Nearest Neighbor. Atlas Vector Search supports both; ANN is the production default.
- **`$vectorSearch`** — Atlas MongoDB aggregation stage for dense vector retrieval.
- **`$search`** — Atlas MongoDB aggregation stage for full-text search (BM25 under the hood).
- **`$rankFusion`** — Atlas's built-in Reciprocal Rank Fusion aggregation operator (combines `$vectorSearch` + `$search` rankings).
- **Reciprocal Rank Fusion (RRF)** — Merge function for hybrid retrieval. Each document's RRF score = sum over rankers of `1 / (k + rank)`. k=60 is the common default.
- **Titan Embeddings v2** — AWS Bedrock embedding model. 1024/512/256-dim. Unit-normalized output.
- **Clause precedence** — DFARS supplements FAR for DoD acquisitions (48 CFR §201.104). Cross-references must be encoded as chunk metadata.

## 6. Optional deep-dive (not required to participate)

- [Pinecone — Chunking strategies for RAG](https://www.pinecone.io/learn/chunking-strategies/) (~12 min read), retrieved 2026-05-23 via /web-research. Worth knowing for the Wed scenario-alternative #1 (vector store choice). Note: ignore Pinecone-specific recommendations — the training-project removes the unused `pinecone-client` dep this week (debt Item 7).
- [GovInfo — eCFR Title 48 (FAR + DFARS bulk download)](https://www.ecfr.gov/current/title-48) (~10 min skim), retrieved 2026-05-23 via /web-research. The actual source the cohort downloads on Tue afternoon for the ingestion pipeline. Public-domain text. Note structure: Title 48 → Chapter 1 (FAR) + Chapter 2 (DFARS) + agency-specific supplements.

---

## Sources

All citations retrieved 2026-05-23 via /web-research per `pipeline/RESEARCH-PROTOCOL.md`. Recency windows applied: hot-tech 3-month (contextual retrieval, Titan v2, Atlas hybrid search, LangChain v1.0), foundation-stable 12-month (chunking concepts), federal-regulatory 6-month (Title 48 eCFR).

## Known-bad-pattern warnings against the reading

- The MongoDB hybrid-search docs use the verb "combining" not "chaining" — consistent with `langchain-chaining-verb` blocklist entry. If a tutorial linked from Atlas docs uses the word "chain" in a LangChain v0.x sense, flag it.
- Some Pinecone tutorials at the optional link teach `cosine similarity is the standard` — see `vector-cosine-default` blocklist entry. Titan v2 returns unit-length vectors so cosine and dot-product agree; for non-normalized embeddings you'd want dot-product or L2. Don't pattern-match.
