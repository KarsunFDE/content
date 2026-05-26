---
week: W02
day: Wed
topic_slug: rag-caching-and-index-management
topic_title: "Caching patterns for hot queries + vector index management revisited"
parent_overview: W02/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://redis.io/blog/rag-at-scale/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://pyimagesearch.com/2026/04/27/semantic-caching-for-llms-fastapi-redis-and-embeddings/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.getmaxim.ai/articles/semantic-caching-for-llms-cut-cost-and-latency-at-scale/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://oneuptime.com/blog/post/2026-03-31-mongodb-tune-hnsw-vector-search/view
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Caching patterns for hot queries + vector index management revisited

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish the three cache layers in a RAG pipeline (embedding cache, retrieval cache, response cache) by what they store, what they save, and what invalidates them.
- Compare exact-match caching to semantic caching and pick the right layer for a given workload's hit-rate and quality budget.
- Design cache keys that include tenant scope so the cache cannot become a tenancy bug.
- Read the HNSW tuning knobs (`m`, `efConstruction`, `numCandidates`) and understand which require an index rebuild and which are query-time.
- Identify when to raise `numCandidates` based on observed precision plateau or pre-filter convergence behavior.

## 2. Introduction

A RAG pipeline at scale has three obvious cost levers — embedding model calls, vector store reads, and LLM generations — and three obvious cache layers that map onto them. Each layer has different cost savings, different staleness risks, and different correctness traps.

The fundamental result from 2025–2026 production studies is reproducible: well-tuned semantic caching delivers 40–70% cache hit rates on real conversational RAG workloads, and a cache hit returns in single-digit milliseconds versus 1–5 seconds for an LLM call. The savings are both in latency and in token cost. But the gains depend on getting three things right: choosing the correct cache key per layer, setting the right similarity threshold for semantic matching, and never letting the cache become a path that bypasses tenant isolation.

This reading also closes the loop on vector-index management. Tuesday introduced HNSW as the index type behind Atlas Vector Search and named `numCandidates` as the kNN candidate pool size. Wednesday's question is *when* you tune which knob — `m` and `efConstruction` (index-build-time, require rebuild) versus `numCandidates` (query-time, free to change).

## 3. Core Concepts

### Three cache layers in a RAG pipeline

**Embedding cache.** Stores the vector representation of a query so that repeated identical (or semantically near-identical) queries skip the embedding model call. Key: a hash of the query text (exact-match) or the embedding itself indexed for semantic lookup. Savings: one embedding-model call (often the cheapest of the three model calls in the pipeline, but still meaningful at scale). Staleness: effectively zero — embeddings are deterministic for a fixed model version, so the only invalidation is a model-version rollover.

**Retrieval cache.** Stores the top-k document IDs returned by the vector store for a given query+filter. Key: a hash of the query text combined with the filter set (including any tenant scope, date range, content type filter). Savings: one vector-store round-trip per cache hit. Staleness: the cache is invalidated by corpus changes — a new document arriving, an existing document deleted, an embedding re-computed for an updated chunk. Requires a corpus-update hook.

**Response cache (semantic cache).** Stores the LLM's generated answer keyed by the user query (semantically matched) and the retrieval context. Key: typically a hash combining the query embedding, the context hash, and the tenant ID. Savings: the entire LLM call — by far the largest cost saving and the largest latency drop. Staleness: this is the hardest cache to manage correctly. The corpus can change underneath; the LLM model can roll over; the tenant's policy can change.

The total savings stack: at hit rates of 60%+ on the response cache, total API-cost reductions of 60–73% have been documented across multiple production write-ups.

### Exact-match versus semantic caching

**Exact-match caching** keys on `sha256(normalized_query_text)`. Two queries that differ by a single character are different cache keys. Hit rates are typically 10–20% on real workloads because users phrase questions differently even when they mean the same thing.

**Semantic caching** keys on the query embedding. A new query embeds, the cache is searched via vector similarity, and if the nearest neighbor exceeds a similarity threshold (typically 0.85–0.95 cosine) the cached response is returned. Hit rates rise to 40–70% because near-duplicate queries collapse onto the same cache entry. The cost is a vector-search call (single-digit ms) instead of a hash lookup (sub-millisecond), and a calibration knob (the similarity threshold).

The threshold is load-bearing. Set it too low (0.7), and dissimilar queries get the same cached answer — false-positive cache hits that return wrong responses. Set it too high (0.99), and you collapse back to exact-match performance. The right threshold depends on the embedding model's tight-similarity behavior and on the application's tolerance for "approximately the same answer." Calibrate against an eval set: hand-label query pairs as "should share an answer" vs "should not"; tune the threshold until precision is above your tolerance.

A two-layer pattern is common: exact-match first (sub-ms hit for the cheap case), semantic on miss (low-ms hit for the near-duplicate case), full pipeline on second miss.

### Cache keys and the tenancy trap

The retrieval cache and the response cache must include tenant scope in the key. A cache that keys only on the query text will happily return tenant A's previously-cached answer to tenant B's identical query — reproducing the multi-tenancy boundary violation discussed in the previous reading, but inside Redis instead of the vector store.

The correct key shape for the response cache:

```
key = sha256(
    query_embedding_or_text +
    context_hash +
    tenant_id +
    model_version +
    prompt_version
)
```

The `tenant_id` is the line that prevents cache-level leakage. The `model_version` and `prompt_version` are the lines that prevent stale answers surviving a model rollover or a prompt update.

The embedding cache *can* be shared across tenants when the embedded text is corpus-universal (a clause that every tenant queries the same way against a shared regulatory corpus, for instance). For tenant-private text — a user query that contains the tenant's confidential terms — even the embedding cache must be tenant-scoped, because the cache key is the query, and the query itself may be sensitive.

### Cache invalidation hooks

A response cache without an invalidation hook is a stale-answer factory. The invalidation surfaces:

- **Corpus updates** — track a `context_hash` per cached response; flush by `context_hash` when underlying chunks change.
- **Model updates** — include model version in the cache key so a rollover automatically misses all previous entries.
- **Prompt updates** — include a prompt-version hash in the key; template changes invalidate cleanly.
- **Tenant policy updates** — the trickiest; many teams accept a short TTL (5–15 min) as a coarse fallback when fine-grained invalidation is impractical.

A TTL alone is not invalidation — it is a coarse bound. Invalidation hooks keep the cache actually consistent.

### Vector-index management revisited: HNSW tuning

HNSW (Hierarchical Navigable Small Worlds) is the indexing algorithm behind Atlas Vector Search and most production vector stores. It has three tuning knobs that determine the recall/latency/memory trade.

**`m`** — the number of bidirectional connections each node in the HNSW graph maintains. Higher `m` means better recall and more memory. Recall improves substantially up to `m=32`, with diminishing returns past 64. Set at *index build time*; changing it requires an index rebuild.

**`efConstruction`** — the size of the candidate list during index construction. Higher values produce a better-quality graph at higher build cost. Common production starting point: 200. Set at index build time; changes require an index rebuild.

**`numCandidates`** — the size of the candidate pool the query-time search examines before returning the top-k. Set per query. A common starting value is `numCandidates = 10 × limit`; MongoDB's documentation recommends going as high as `20 × limit` for accuracy-sensitive workloads. With a well-tuned graph (`m=32`, `efConstruction=200`), `numCandidates=100` can achieve ~99% recall against the exact nearest-neighbor reference.

When to raise `numCandidates`:

1. **Pre-filter is in use and returning fewer than expected results.** The pre-filter narrows the searchable set; if the filter is highly selective (a tenant with very few documents, a narrow date range), the kNN search may need a larger candidate pool to find enough surviving documents. Raise until the result count stabilizes.
2. **Rerank quality is plateauing.** If you're already reranking and Precision@k_post_rerank has flattened, a larger pre-rerank pool gives the cross-encoder more to work with. Diminishing returns kick in past a point — measure.
3. **Recall against an exact-nearest-neighbor reference is below target.** Re-run a sample of queries with `exact=true` (where the store supports it) and compare to the approximate result; raise `numCandidates` until the gap closes.

When to *rebuild* the index (changing `m` or `efConstruction`): when adding a `filter` field that was not declared at original index creation; when recall ceilings can't be reached by `numCandidates` tuning alone; when the corpus has shifted enough that the existing graph topology is suboptimal. Atlas supports online index rebuilds — no application downtime, but IOPS-consuming during the rebuild. Schedule outside peak load.

## 4. Generic Implementation

A two-layer cache (exact-match then semantic) in plain Python, outside the federal-acquisitions domain. Imagine a customer-support chatbot for a streaming-media service.

```python
import hashlib
import redis
import json

r = redis.Redis(host="cache", port=6379)

def query_with_cache(user_query: str, tenant_id: str, model_version: str) -> dict:
    # Layer 1: exact-match cache
    exact_key = "exact:" + hashlib.sha256(
        (user_query + tenant_id + model_version).encode()
    ).hexdigest()
    cached = r.get(exact_key)
    if cached:
        return {"source": "exact", "answer": json.loads(cached)}

    # Layer 2: semantic cache (vector lookup in Redis Vector or external)
    query_vec = embed(user_query)
    similar = semantic_cache_search(
        query_vec,
        tenant_id=tenant_id,
        model_version=model_version,
        similarity_threshold=0.93,
    )
    if similar is not None:
        return {"source": "semantic", "answer": similar["answer"]}

    # Cache miss: run the pipeline, store under BOTH keys
    retrieved = retrieve(user_query, tenant_id=tenant_id)
    answer = generate(user_query, retrieved, model_version=model_version)

    r.setex(exact_key, ttl_seconds=900, value=json.dumps(answer))
    semantic_cache_store(
        query_vec=query_vec,
        answer=answer,
        tenant_id=tenant_id,
        model_version=model_version,
    )

    return {"source": "fresh", "answer": answer}
```

Notice the keys include `tenant_id` and `model_version` — the cache is scoped per tenant and automatically misses on a model rollover. A corpus-update hook (not shown) flushes entries whose stored `context_hash` references changed documents.

## 5. Real-world Patterns

**Customer-support chatbots at scale.** A B2C ride-hailing platform's 2026 write-up described moving from text-based exact-match to semantic cache. Hit rates rose from 14% to 67%; LLM API cost dropped roughly 73%. The biggest work was tuning the similarity threshold against a labeled query-pair eval set; the wrong threshold produced false-positive hits.

**Healthcare clinical-search.** A clinical-search startup adopted an explicit policy *against* response caching for any query containing patient-identifying tokens; queries with PHI went straight to the full pipeline. Embedding and retrieval caches *were* used, scoped strictly per-tenant. Cached LLM answers for queries containing patient names could outlive policy changes that should have restricted access.

**E-commerce product Q&A.** An electronics retailer cached *retrieval results* aggressively (top-k product IDs, ~80% hit rate) but *not* LLM answers, because the LLM step interpolated real-time inventory and pricing. Latency improved; freshness invariants held.

**Gaming patch-notes search.** A multiplayer-game studio bound a semantic cache to their patch pipeline. On every release, the cache flushed entries whose `context_hash` referenced patch-notes; hit rates were lower (40%) but freshness was stronger than a TTL alone could provide.

## 6. Best Practices

- Layer exact-match and semantic caches; the exact-match hit is essentially free and catches the trivial duplicates before any embedding work.
- Calibrate the semantic similarity threshold against a labeled eval set; never pick a threshold from a blog post.
- Include `tenant_id`, `model_version`, and `prompt_version` in every cache key for the retrieval and response layers. The cache must not become a way to bypass tenancy or to ship stale answers across version boundaries.
- Build an invalidation hook before you build the cache. A response cache without a corpus-update hook is a stale-answer factory.
- Start HNSW tuning at `m=16`, `efConstruction=100`, and `numCandidates = 10 × limit`. Adjust `numCandidates` first (query-time, free). Only rebuild for `m` and `efConstruction` changes if recall is below target after `numCandidates` is maxed.
- Treat `numCandidates` as a per-query knob in production. High-stakes queries can pay for a larger candidate pool; bulk-ingest sweeps can save IOPS with a smaller one.
- Schedule index rebuilds outside peak load; even online rebuilds consume IOPS.

## 7. Hands-on Exercise

**Code task (15 min).** You inherit a RAG service with no caching. It serves a B2B inventory-management SaaS with 200 tenants and ~50 active users per tenant. Most queries follow a repeating weekly pattern (warehouse staff ask similar questions every Monday). The current p95 latency is 2.3s; the target is 800ms.

Write the cache wrapper as a Python function that takes a `user_query`, a `tenant_id`, and a `model_version`. Return a dict `{source: exact|semantic|fresh, answer: ...}`. Your implementation should:

1. Try an exact-match Redis lookup first.
2. On miss, do a semantic vector lookup in a separate cache index (you can mock the vector store as a function `semantic_lookup(query_vec, tenant_id, threshold)`).
3. On both misses, call `run_full_pipeline(user_query, tenant_id)` and store the result in both caches.
4. Ensure all cache keys include `tenant_id` and `model_version`.

**What good looks like.** A working two-layer cache that correctly scopes by tenant. Keys are deterministic and include the model version. On a model rollover, the cache automatically misses. No code path lets one tenant read another tenant's cached answer. The function returns the source so callers can log hit rates. Bonus: include an explicit invalidation function that takes a `context_hash` and flushes matching entries.

## 8. Key Takeaways

- *What are the three cache layers in a RAG pipeline?* — Embedding cache (saves embedding-model calls; zero staleness), retrieval cache (saves vector-store reads; needs corpus-update invalidation), response cache (saves LLM calls; needs corpus + model + prompt invalidation).
- *When do you use semantic caching instead of exact-match?* — Always layer both. Exact-match is essentially free and catches identical-text repeats; semantic catches paraphrase repeats at much higher hit rates but requires threshold calibration.
- *Why must cache keys include tenant scope?* — Because a cache that keys only on query text reproduces the multi-tenancy boundary violation from the previous reading inside the cache layer.
- *Which HNSW knobs are build-time and which are query-time?* — `m` and `efConstruction` are build-time (require rebuild to change). `numCandidates` is query-time (free to change per query).
- *When do you raise `numCandidates`?* — When pre-filter selectivity narrows the pool, when rerank quality is plateauing, or when recall against an exact-NN reference falls below target.

## Sources

1. [RAG at Scale: How to Build Production AI Systems in 2026 (Redis Blog)](https://redis.io/blog/rag-at-scale/) — retrieved 2026-05-26
2. [Semantic Caching for LLMs: FastAPI, Redis, and Embeddings (PyImageSearch, Apr 2026)](https://pyimagesearch.com/2026/04/27/semantic-caching-for-llms-fastapi-redis-and-embeddings/) — retrieved 2026-05-26
3. [Semantic Caching for LLMs: Cut Cost and Latency at Scale (Maxim AI)](https://www.getmaxim.ai/articles/semantic-caching-for-llms-cut-cost-and-latency-at-scale/) — retrieved 2026-05-26
4. [How to Tune HNSW Parameters for Vector Search in MongoDB (OneUptime, Mar 2026)](https://oneuptime.com/blog/post/2026-03-31-mongodb-tune-hnsw-vector-search/view) — retrieved 2026-05-26
5. [Run Vector Search Queries (MongoDB Atlas docs)](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/) — retrieved 2026-05-26

Last verified: 2026-05-26
