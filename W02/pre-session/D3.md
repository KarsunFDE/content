---
week: W02
day: Wed
phase: Foundation
topic: "Advanced RAG patterns + multi-tenant retrieval — pre-Wed-research-slot"
estimated_total_minutes: 50
last_verified: 2026-05-23
fde_situations: [1, 3, 6, 9, 10, 11]
tech: [parent-child-chunking, multi-query-retrieval, contextual-compression, reranking, multi-tenant-retrieval, atlas-vector-search-filter]
sources_research_briefs: []
author: instructor
---

# Pre-session reading — Advanced RAG patterns + multi-tenant retrieval

Week 2, Day 3 (Wed). Estimated total time on task: ~50 minutes. Last verified: 2026-05-23.

> Read **before** W2 Wed morning. Wed afternoon is the **dedicated `/web-research` slot** (per D-040) — pairs work the 3 scenario-alternatives prompts head-on. This pre-read frames the morning multi-tenant work + the afternoon research dimensions. Topic density risk: Wed is at the 12-topic cap; if morning overruns, **caching patterns for hot queries** moves to Thu pre-session.

## 1. Why this matters (~85 words)

Tue's baseline RAG works on a single agency. The morning of Wed surfaces the multi-tenant problem (Item 10): a CS at agency `DLA` running `GET /api/clauses/search?q=trade-off` saw a draft clause from agency `GSA-FAS`. The clause library itself is cross-tenant (everyone reads the same FAR), but the *retrieval surface for tenant-scoped resources* (drafts, proposals, audit events) leaks across agencies. The afternoon's dedicated research slot is where the cohort defends the Tue baseline against three real alternatives — Atlas vs pgvector vs Pinecone, hybrid vs reranker-only vs full hybrid+rerank, and single-model vs domain-tuned vs embedding-routing.

## 2. Core concept in 5 minutes

Three patterns are worth holding before Wed morning:

**Parent-child indexing.** Index the small chunk (sub-paragraph) for similarity match; return the parent paragraph as the retrieval payload (so the LLM gets context the embedding wasn't keyed on). MongoDB Atlas supports this via a `parent_ref` field on each chunk + a `$lookup` stage after `$vectorSearch`. Cost: ~25% storage increase. Benefit: LLM sees enough context to use the chunk.

**Multi-query retrieval.** When a vendor question is ambiguous ("when's the proposal due?"), run multiple reformulations of the query (cheap Bedrock Claude Haiku call to generate 3-5 paraphrases) and union the retrieved sets. Recall goes up; precision goes down — must be paired with reranking.

**Contextual compression.** After retrieval but before sending to the LLM, run a cheap pass that drops sentences from retrieved chunks that aren't relevant to the query. LangChain v1.0 calls this `EmbeddingsFilter` or `LLMChainExtractor` (the v0.x `LLMChainExtractor.from_llm` API still exists; the wrapper is **NOT** the old `Chain` class — per D-033 verify the call site uses `extractor.invoke(...)` not `extractor.run(...)`).

**Multi-tenant retrieval boundary.** The clause library is *NOT* tenant-scoped — FAR/DFARS is the same everywhere. The tenant-scoped collections are `Solicitation`, `Proposal`, `AuditEvent`, `Cpar`. When these are RAG-retrieved (e.g., "show me past CPARs for vendors with red ratings"), the Atlas Vector Search index *MUST* have `agency_id` as a `filter` field — set at index-definition time, not at query time. Query-level filters can be forgotten; index-level filters can't. **This is the W2 Wed work that closes debt Item 10 for RAG retrieval.**

## 3. What to read or watch tonight

- [MongoDB Atlas — Filter Vector Search results](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/#examples) (~8 min read), retrieved 2026-05-23 via /web-research. Specifically the `filter` field syntax in `$vectorSearch`. Pre-filter (recommended for tenancy) vs post-filter. Pre-filter requires the field to be declared as `filter` type in the index definition. This is the Item 10 closer.
- [LangChain v1.0 — Contextual compression retrievers](https://docs.langchain.com/oss/python/integrations/retrievers/contextual_compression) (~10 min read), retrieved 2026-05-23 via /web-research. Focus on `EmbeddingsFilter` (cheap; dense-vector similarity threshold) vs `LLMChainExtractor` (LLM-based; better quality but expensive). Note the v1.0 API uses `.invoke()` — if the example shows `.run()`, the page is stale.
- [Anthropic — Contextual Retrieval — full results](https://www.anthropic.com/news/contextual-retrieval) (~12 min read — Tue had this as optional; required now), retrieved 2026-05-23 via /web-research. Focus on the ablation table: contextual embeddings alone = 35% failure reduction; contextual embeddings + contextual BM25 + reranker = 67% failure reduction. The "reranker" line is the Wed scenario-alternative #2.
- [Cohere — Rerank 3.5 via AWS Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters-cohere-rerank.html) (~8 min read), retrieved 2026-05-23 via /web-research. The default reranker model on Bedrock. Note: separate model invocation cost on top of retrieval; latency adds ~150-300ms; quality lift on regulatory text is the question Wed afternoon answers.
- *(optional)* [Microsoft Azure — Multi-tenant RAG patterns](https://learn.microsoft.com/en-us/azure/architecture/example-scenario/ai/secure-multitenant-rag) (~10 min skim), retrieved 2026-05-23 via /web-research. Treat as comparative context; Azure-specific implementation, but the *patterns* (index-per-tenant vs filter-per-tenant vs collection-per-tenant) translate.

## 4. Two questions to come in with tomorrow

1. *"If the Atlas Vector Search index for the clause library has NO `agency_id` filter (because clauses are cross-tenant), but the index for `Cpar` retrieval DOES — how does the code prevent a developer from accidentally querying the wrong index? What's the test that catches it?"* (Wed morning exercise.)
2. *"For the 3 Wed scenario-alternatives, what is the *honest* dimension on which Atlas Vector Search loses to its alternatives? (If you can't name one, you haven't researched it.)"* (Wed afternoon research-slot framing.)

## 5. Glossary refresh (terms you'll hear tomorrow)

- **Pre-filter / post-filter** — In Atlas `$vectorSearch`, pre-filter applies the filter *before* kNN search (recommended for tenancy — guarantees the filter holds); post-filter applies after (cheaper but allows tenant leakage if numCandidates too low).
- **Parent-child indexing** — Index the small chunk; retrieve via `$lookup` the larger parent for LLM payload.
- **Multi-query retrieval** — Generate N query paraphrases, union the retrieval sets, rerank.
- **Contextual compression** — Drop irrelevant sentences from retrieved chunks before sending to the LLM.
- **EmbeddingsFilter** — LangChain v1.0 cheap compressor; similarity threshold against the query embedding.
- **LLMChainExtractor** — LangChain v1.0 (note: this is NOT the v0.x `Chain` class — it's a Runnable. Use `.invoke()`).
- **Cohere Rerank 3.5** — Cross-encoder reranker; available on Bedrock; ~150-300ms added latency; 5-15% quality lift typical.
- **`numCandidates`** — Atlas Vector Search kNN candidate pool size before final `limit`. Default 10x limit. Higher = better recall, slower.
- **RRF (Reciprocal Rank Fusion)** — Hybrid merge function. k=60 default.

## 6. Optional deep-dive (not required to participate)

- [AWS — Amazon Bedrock latency optimisation guide](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles.html) (~12 min skim), retrieved 2026-05-23 via /web-research. Cross-region inference profiles for Bedrock; relevant when the `POST /rag/clause-search` p95 budget (under 800ms) gets tight with reranker added.
- [Pinecone — pgvector vs Pinecone vs Atlas comparison](https://www.pinecone.io/learn/series/vector-databases-in-production-for-busy-engineers/) (~15 min read), retrieved 2026-05-23 via /web-research. Pinecone-authored, so treat the framing with skepticism — but the technical dimensions (recall@K, ingestion throughput, filter cost) are real. Useful for Wed scenario-alternative #1. (Reminder: the training-project removes the unused `pinecone-client` dep this week — Item 7 — but understanding the alternative is part of the scenario-alternatives discipline.)

---

## Sources

All citations retrieved 2026-05-23 via /web-research per `pipeline/RESEARCH-PROTOCOL.md`. Recency windows applied: hot-tech 3-month (Cohere Rerank 3.5, Atlas filter syntax, contextual retrieval, LangChain v1.0), foundation-stable 12-month (multi-tenant patterns).

## Known-bad-pattern warnings against the reading

- LangChain v1.0 contextual-compression docs may include legacy v0.x examples in archived sections — check that any example invocation uses `.invoke()` not `.run()`. Flag any example using `RetrievalQA.from_chain_type(...).run(...)` per `langchain-chain-class` blocklist entry.
- The Pinecone comparison post (optional) describes "namespaces for multi-tenancy" — see `pinecone-namespaces-as-multitenancy` blocklist. Namespaces alone are not a tenancy boundary at production scale. If a pair cites this in their Wed scenario-alternative ADR without caveats, flag it.
