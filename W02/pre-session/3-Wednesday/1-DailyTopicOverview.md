---
week: W02
day: Wed
phase: Foundation
topic: "Advanced RAG patterns + multi-tenant retrieval + dedicated /web-research slot"
estimated_total_minutes: 75
last_verified: 2026-05-26
fde_situations: [1, 3, 6, 9, 10, 11]
tech: [parent-child-chunking, multi-query-retrieval, contextual-compression, reranking, multi-tenant-retrieval, atlas-vector-search-filter, redis-cache, deterministic-fallback, ragas-faithfulness, observability]
sources_research_briefs: []
author: instructor
---

# Pre-session reading — Advanced RAG patterns + multi-tenant retrieval boundaries

Week 2, Day 3 (Wed). Estimated total time on task: ~75 minutes. Last verified: 2026-05-26.

> Read **before** W2 Wed morning. Wed afternoon is the **dedicated `/web-research` slot per D-040** — pairs work the three scenario-alternatives prompts (`W02-SA-1` vector store, `W02-SA-2` hybrid-retrieval, `W02-SA-3` embedding model) with 2–3 candidate techs each, ADR drafts due Thu EOD. **No implementation in the PM.** This pre-read frames the morning multi-tenant + advanced-pattern work and the afternoon research dimensions. **Density risk: Wed is at the 12-topic cap in PLAN.md** — if the morning runs over, topic 4 (caching) defers to Thu pre-session per PLAN.md §"Density risk".

## 1. Parent-child chunking + multi-query + contextual compression — three patterns for when top-k fails (15 min)

Tuesday's baseline retrieves the right chunk most of the time. Wednesday names three patterns for when it doesn't, and each one maps to a different failure mode.

**Parent-child indexing** is the context-recovery pattern. Index a sub-paragraph (FAR 52.212-4(a)(2)) for similarity match, but return its parent paragraph (the full 52.212-4(a)) as the retrieval payload. The LLM gets context the embedding wasn't keyed on. In Atlas this is a `parent_ref` field on each child document + a `$lookup` stage after `$vectorSearch`. Storage cost ~25% over baseline. Use it when the embedding hits but the LLM answer is fragmentary because the sub-paragraph reads as "(2) Notwithstanding paragraph (1)..." with no anchor.

**Multi-query retrieval** is the synonym/paraphrase pattern. A vendor asks *"when's the proposal due?"* — the chunk says *"offers shall be received no later than..."*. Word overlap is zero. Run a cheap Bedrock Claude Haiku call to generate 3–5 paraphrases of the query, union the retrieval sets, then rerank. Recall up, precision down — must pair with reranking (topic 2). Cost: one extra Haiku call per user query (~$0.0003 at Haiku pricing).

**Contextual compression** is the token-budget pattern. After retrieval but before the LLM call, run a cheap pass that drops sentences from retrieved chunks that aren't relevant to the query. LangChain v1.0 calls this `EmbeddingsFilter` (cheap; dense-vector similarity threshold against the query) or `LLMChainExtractor` (LLM-based; better quality, expensive). Both are Runnables — invoke with `.invoke()`, not `.run()`. The `Chain` class is removed in v1.0 (D-033 / Item 5).

For Karsun acquire-gov: parent-child handles the FAR sub-paragraph context-loss problem from Tuesday; multi-query handles the vendor-vocabulary mismatch ("trade-off" vs "best-value continuum"); contextual compression handles the DFARS-supplement-bloat problem where retrieving a 215.371-4 chunk drags in 800 tokens of cross-reference table.

## 2. Reranking strategy depth — cascade designs and the cost-of-precision curve (10 min)

Tuesday introduced rerankers as the second half of hybrid retrieval. Wednesday goes deeper: rerankers come in cascade designs, and each layer adds latency.

The **standard cascade** is *retrieve 50 → rerank 20 → top-5 to LLM*. The bi-encoder retriever (Titan v2 → Atlas $vectorSearch) is cheap and approximate; it casts a wide net. The cross-encoder reranker (Cohere Rerank-3.5 on Bedrock, or BGE-reranker-v2 self-hosted) scores each query-document pair jointly — slow but precise. The LLM is the final reranker by virtue of which chunks it actually cites.

Three reranker options sit on the spectrum:

- **Cohere Rerank-3.5 on Bedrock** — managed, ~150–300ms added latency for 20 documents, 5–15% quality lift typical on regulatory text. Cost: separate model invocation on top of the embedding call. This is the W02 default and the lead candidate for `W02-SA-2`.
- **BGE-reranker-v2** — open-weight, self-hosted (GPU or CPU with quantization). Cheaper per call at scale, but you own the deployment. Quality close to Cohere on English text.
- **In-prompt LLM rerank** — pass top-20 chunks to a Haiku call with a "rank these 1–20 by relevance to *X*" prompt. Highest quality, highest cost, slowest. Useful as a tier-3 cascade for high-value queries (the CO's morning brief) but not for `GET /api/clauses/search`.

The cost-of-precision curve flattens fast: going retrieve-100 → retrieve-200 buys almost nothing once recall@50 is already > 95%, but going rerank-20 → rerank-50 measurably improves precision@5 because the cross-encoder is comparing pairs the bi-encoder ranked poorly. Tune the cascade by measuring, not by intuition.

For Karsun acquire-gov: the `POST /rag/clause-search` p95 budget is 800ms (per Tuesday's ADR). A Cohere Rerank-3.5 call at 250ms eats a third of that. Topic 4's caching is part of how the budget closes.

## 3. Multi-tenant retrieval boundaries — Item 10, load-bearing for FAR/FedRAMP (12 min)

This is the load-bearing topic of the day. It frames the morning war-room directly (the DLA sys_admin's cross-tenant clause leak) and is the W2 closure of debt **Item 10** at the index level.

The acquire-gov clause library is *cross-tenant* — every agency reads the same FAR/DFARS corpus, so the FAR collection's Atlas Vector Search index has no `agency_id` filter. But the **tenant-scoped collections** — `Solicitation` (in-progress drafts), `Proposal` (vendor submissions), `AuditEvent` (action logs), `Cpar` (past-performance ratings) — must enforce tenancy at retrieval. When a CS at DLA runs `GET /api/clauses/search?q=trade-off` against a *search-everything* surface that joins clause library + drafts, the search-everything index is the tenancy boundary, not the application layer.

Three patterns exist; only one is acceptable at FAR/FedRAMP scale:

- **Index-level pre-filter (Atlas `filter` field declared at index-definition time)** — pre-filter applies *before* kNN search. The `agency_id` field is declared as a `filter` type in the index `fields` array, and every `$vectorSearch` call passes `filter: { agency_id: $caller_agency }`. The filter holds at the index level, not at the query level. **This is the production pattern and the Item 10 closer.** Requires online index re-creation (no downtime, ~minutes on Atlas) to land.
- **Query-level post-filter** — apply `$match` after `$vectorSearch`. Cheaper to add, but the kNN candidate pool was already polluted with other tenants' documents — if `numCandidates` is too low, the post-filter returns zero results for legitimate queries while the index quietly served leaks for the candidate generation. **Not acceptable for tenancy.**
- **"Namespaces as multi-tenancy"** — the Pinecone-vintage pattern where each tenant gets a namespace. **Known-bad-pattern** (`pinecone-namespaces-as-multitenancy` in `skills/tech-research/references/known-bad-patterns.yml`): namespaces alone are not a tenancy boundary at production scale because they don't enforce access controls — they're a routing convenience. The correct framing is **namespaces + per-tenant indexes + IAM-level access controls together**, not namespaces alone. Any reading that pitches namespaces as the tenancy story is pre-FedRAMP-grade. [!instructor-review]: if a pair cites a "namespaces for multi-tenancy" source in their `W02-SA-1` ADR without this caveat, flag it at Thu standup.

For Karsun acquire-gov: FAR/FedRAMP audit requires that retrieval boundaries are **enforceable at the storage layer, not at the application layer**. Index-level pre-filter is the only one of the three that satisfies the audit. The morning war-room ships the Atlas index re-creation; the closure ticket lands on the W4 modernization tracker as Item 10 closed at the W2 boundary.

## 4. Caching patterns for hot queries + Atlas index management revisited (10 min)

*Defer-candidate per PLAN.md density risk — if Wed morning overruns, this whole topic moves to Thu pre-session. Caching is foundation-stable; the boundary work in topic 3 is not.*

Three cache layers exist in a RAG flow, and they cost and break differently:

- **Embedding cache (Redis, keyed by `sha256(query_text)`)** — store the query embedding so repeat queries skip the Titan call. Cheap to add, big latency win for hot queries. Multi-tenancy concern: the embedding is the same across agencies (FAR text is universal), so this cache is **safe to share across tenants**. TTL 24h is a reasonable default.
- **Retrieval cache (Redis, keyed by `sha256(query_text + filter_hash)`)** — store the top-k document IDs returned by `$vectorSearch`. Saves the Atlas round-trip on repeat queries. Multi-tenancy concern: the filter hash must include `agency_id`, or you leak the same way as topic 3. Get this wrong and the cache *becomes* the tenancy bug.
- **Response cache (Redis, keyed by `sha256(query_text + context_hash + agency_id)`)** — store the LLM's final answer. Highest savings (skip Bedrock entirely), highest staleness risk (corpus updates invalidate). Tenant-scoped by construction because the answer can include tenant-scoped chunks. Don't deploy this until you have a corpus-update invalidation hook.

**What's safe to cache in a multi-tenant corpus:** FAR/DFARS clause-library queries (cross-tenant, immutable per regulation cycle) are aggressively cacheable. `Solicitation`/`Proposal`/`Cpar` retrievals are tenant-scoped — the cache key must include `agency_id` or you reproduce Item 10 inside Redis.

**Atlas index management revisited under the hot-query lens.** Tuesday named the `numCandidates` setting (kNN candidate pool size; default 10× limit). Wednesday's question is: when do you raise it? Two signals — (1) post-filter is in use and returning fewer results than `limit` (raise `numCandidates` until the post-filter floor stabilizes — but better, switch to pre-filter); (2) hot-query rerank quality is plateauing (raise `numCandidates`, retrieve wider, give the reranker more to work with). Online index rebuild on Atlas is free of downtime but consumes IOPS — schedule it outside the CO's demo window.

## 5. Deterministic fallback patterns — what runs when retrieval returns nothing (10 min)

Empty results are not exceptional — they are a designed-for state. Three fallback paths exist, in escalating cost order:

- **Empty-result fallback → keyword search (`$search` BM25 sparse).** When `$vectorSearch` returns zero hits, fall back to BM25 on the same corpus. Often catches the long-tail clauses whose embedding-space neighbors are dominated by their abbreviations rather than their text. Cheap. Always on.
- **Low-confidence fallback → RAGAS-faithfulness gate.** When retrieval returns results but the faithfulness score (does the answer's claims appear in the retrieved chunks?) is below threshold (e.g., 0.85), don't ship the LLM answer. Return *"I'm not confident enough to answer this — escalating to a contracting officer."* Tees up Thursday's HITL #2 topic directly: **the human is one of the deterministic fallback paths**, not an exception.
- **Corpus-gap fallback → explicit escalation.** When the query references a clause that the corpus doesn't contain ("Section L Factor 3 in this solicitation references FAR 9.999-99, which doesn't exist") → escalate to the CO with the gap named, do NOT guess. This is the W1 Thu "48 CFR 47.305-2 hallucination" failure mode reframed for retrieval-grounded systems: now the model can retrieve, but the corpus has a hole.

For Karsun acquire-gov, the fallback ladder is the federal-acquisitions audit story: every escalation point becomes an `AuditEvent` with `correlation_id`, every refusal-to-guess is logged with the gap reason, and the CO review queue is the named operational target. The full HITL #2 architecture lands Thursday in the morning war-room — Wednesday's pre-read names the *patterns* that make HITL a defensible default, not a surprise.

## 6. RAG pipeline composition + observability surface (10 min)

**Composition in plain Python, not LCEL `|` pipes (D-033).** LangChain v1.0 removed the `Chain` class and demoted the `|` pipe operator from "central composition mechanism" to "historical convenience". Wednesday's pipeline composition for the dense → sparse → hybrid → rerank → fallback stack is **plain Python function calls**: `dense_hits = vector_search(query); sparse_hits = bm25_search(query); fused = rrf_fuse(dense_hits, sparse_hits); reranked = cohere_rerank(query, fused); final = deterministic_fallback_chain(reranked, query)`. Each step is a normal function. Assign to variables; log between steps; debug with `breakpoint()`. Anything that uses LangChain primitives at each step is fine — the framework is the per-step toolkit, not the orchestration shape.

[!instructor-review]: any reading or example that teaches LCEL `|` pipes as the composition pattern is from the v0.x era. Per `langchain-lcel-pipe` blocklist entry: substitute with plain Python composition and call it out at standup.

**The observability surface** you'll need by Friday's eval-driven war-room ("PR #47 latency +30%, faithfulness -12% — do we ship?"):

- **Per-stage latency** — embedding call (Titan), vector search (Atlas), BM25 search (Atlas), rerank (Cohere), LLM call (Bedrock Claude). Log each as a separate span with `correlation_id`.
- **Per-stage hit-rate** — what % of queries had cache hits at each cache layer; what % triggered each fallback path; what % were caught by the RAGAS-faithfulness gate.
- **Per-stage cost** — Titan tokens, Cohere rerank invocations, Bedrock Claude input/output tokens. Per query and per agency.

Production tracing (LangSmith) arrives W5 per D-031. W2 builds the surface in structured logs (`logging` module, JSON-formatted, one event per stage) so the same fields lift into LangSmith later. The observability surface is the source of truth for Friday's "do we ship?" question — without it you're guessing.

## 7. RAG anti-patterns + framing for the afternoon `/web-research` slot (8 min)

**Top 5 RAG anti-patterns** to avoid this week:

1. **Silent context truncation.** Concatenating retrieved chunks until the prompt fits, dropping the tail without logging. Test: log `n_chunks_retrieved` vs `n_chunks_sent_to_llm` per call; alert when they diverge.
2. **Retrieval-confidence-as-a-string.** Logging "high confidence" / "low confidence" instead of the numeric similarity score. Makes regression analysis impossible. Always log the float.
3. **Citation without chunk ID.** Returning *"according to the FAR..."* without a `chunk_id` the audit log can resolve to a corpus location. Citation must be machine-traceable, not human-readable prose.
4. **Schema drift between embedding model and stored vectors.** Switching from Titan v2 1024-dim to 512-dim without re-embedding the corpus; cosine similarity stops being meaningful. Embed-model version must be a corpus-level invariant, enforced at write time.
5. **Async ingestion without dead-letter.** Ingestion-queue failures swallowed. The DFARS-215.3xx-supplement-loss scenario in tomorrow's war-room is exactly this — `max_depth=2` walker bug silently dropped a corpus segment. Dead-letter queue + ingestion-completeness assertion ("Title 48 should have N parts; we ingested M; M < N → fail") catches it.

**The afternoon `/web-research` slot per D-040.** Three scenario-alternatives prompts are loaded:

- **`W02-SA-1` — Vector store choice.** Atlas Vector Search vs pgvector vs Pinecone. 2–3 candidate techs grounded in `/web-research` citations. Evaluate against the FAR/FedRAMP boundary, the multi-tenant constraints from topic 3, ingestion throughput, recall@K, filter cost, and operational maturity at federal scale.
- **`W02-SA-2` — Hybrid-retrieval strategy.** Reranker-only vs full hybrid (dense + BM25 + rerank) vs contextual-embedding-only. 2–3 candidate techs. Evaluate against the 800ms p95 budget, the quality dimensions from topic 2, and the eval harness arriving Friday.
- **`W02-SA-3` — Embedding-model selection.** Bedrock Titan v2 vs domain-tuned alternative vs query/document routing (different embeddings for clauses vs vendor-submitted text). 2–3 candidate techs. Evaluate against FedRAMP boundary (Titan is in-boundary; some alternatives aren't), token cost, and dimension trade-off.

**The discipline:** each ADR draft cites at least one `/web-research`-retrieved source per candidate tech, with retrieval date, recency category, and an honest dimension on which the *chosen* tech loses to the alternatives. If you can't name the dimension on which Atlas loses, you haven't researched the alternatives. ADR drafts due Thu EOD; Live Defense Fri 14:00.

---

## Sources

All citations retrieved 2026-05-26 via `/web-research` per `pipeline/RESEARCH-PROTOCOL.md`. No sub-30-day sources used (none required HITL halt).

**Recency-category breakdown (10 sources):**

- **Hot-tech 3-month** (6): Cohere Rerank-3.5 on Bedrock; LangChain v1.0 contextual compression; LangChain v1.0 composition patterns (Chain class removal, LCEL demotion); Anthropic Contextual Retrieval ablation; MongoDB Atlas `$vectorSearch` filter (pre-filter mechanics); MongoDB Atlas RAGAS-style faithfulness scoring guidance.
- **Foundation-stable 12-month** (3): parent-child indexing pattern; Reciprocal Rank Fusion (RRF); multi-tenant retrieval index-level pre-filter pattern.
- **Federal-regulatory 6-month** (1): FedRAMP boundary implications for vector-store choice (cross-checks `W02-SA-1` candidate-tech FedRAMP authorization status).

Inline citations:

- [MongoDB Atlas — Filter Vector Search results (pre-filter at index-definition time)](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/#examples) — retrieved 2026-05-26 via `/web-research`. Hot-tech. Topics 3, 4. Closes Item 10 at the index layer.
- [LangChain v1.0 — Contextual compression retrievers (`EmbeddingsFilter`, `LLMChainExtractor`, Runnable `.invoke()` API)](https://docs.langchain.com/oss/python/integrations/retrievers/contextual_compression) — retrieved 2026-05-26 via `/web-research`. Hot-tech. Topic 1.
- [LangChain v1.0 — Composition patterns (no `Chain` class, plain-Python sequential composition)](https://docs.langchain.com/oss/python/concepts) — retrieved 2026-05-26 via `/web-research`. Hot-tech. Topic 6. D-033 enforcement.
- [Anthropic — Contextual Retrieval (full ablation table: contextual-embeddings + contextual-BM25 + reranker = 67% failure reduction)](https://www.anthropic.com/news/contextual-retrieval) — retrieved 2026-05-26 via `/web-research`. Hot-tech. Topics 1, 2. Tee-up for `W02-SA-2`.
- [AWS — Cohere Rerank 3.5 on Bedrock (model parameters, latency, pricing)](https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters-cohere-rerank.html) — retrieved 2026-05-26 via `/web-research`. Hot-tech. Topic 2.
- [AWS — Bedrock latency-optimised inference profiles (cross-region for p95 tuning)](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles.html) — retrieved 2026-05-26 via `/web-research`. Hot-tech. Topics 2, 4.
- [MongoDB Atlas — RAGAS-style retrieval eval (faithfulness, context recall, context precision)](https://www.mongodb.com/docs/atlas/atlas-vector-search/ai-integrations/ragas/) — retrieved 2026-05-26 via `/web-research`. Hot-tech. Topic 5.
- [Microsoft Azure — Secure multi-tenant RAG architecture patterns (index-per-tenant vs filter-per-tenant comparison)](https://learn.microsoft.com/en-us/azure/architecture/example-scenario/ai/secure-multitenant-rag) — retrieved 2026-05-26 via `/web-research`. Foundation-stable. Topic 3. Comparative context, not the Atlas implementation.
- [Atlas Vector Search — Parent-child retrieval pattern with `$lookup`](https://www.mongodb.com/docs/atlas/atlas-vector-search/tutorials/) — retrieved 2026-05-26 via `/web-research`. Foundation-stable. Topic 1.
- [FedRAMP Marketplace — vector-store authorization status (cross-check for `W02-SA-1` candidate-tech selection)](https://marketplace.fedramp.gov/) — retrieved 2026-05-26 via `/web-research`. Federal-regulatory 6-month. Topic 7.

## Known-bad-pattern callouts against the reading

- [!instructor-review] **`pinecone-namespaces-as-multitenancy`** — if any reading or ADR-draft source pitches "namespaces alone for multi-tenancy" without naming per-tenant indexes + IAM-level access controls in the same breath, flag it at Thu standup. Correct framing per blocklist: **namespaces + per-tenant indexes + access controls together**, never namespaces alone. Especially watch `W02-SA-1` Pinecone-candidate drafts.
- [!instructor-review] **`langchain-chain-class`** — any v0.x example showing `LLMChain(...).run(query)` or `RetrievalQA.from_chain_type(...).run(query)` is pre-v1.0 and blocking per D-034 Codex Adversarial Review (Ramping strictness from today). Substitute: Runnable `.invoke()`. This is also debt **Item 5** — the 3 `LLMChain.run()` entry points in acquire-gov migrate Thursday.
- [!instructor-review] **`langchain-lcel-pipe`** — any reading that teaches LCEL `|` pipes as the *fundamental* composition pattern is stale. Substitute with plain Python sequential composition (topic 6). Reference D-033.
- [!instructor-review] **`langchain-chaining-verb`** — if a tutorial linked from any RAG source uses "chaining retrievers" / "chaining prompts" in the v0.x `Chain`-class sense, flag at standup. Substitute: "composing" or "sequential composition".

Last verified: 2026-05-26
