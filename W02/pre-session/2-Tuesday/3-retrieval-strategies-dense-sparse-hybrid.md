---
week: W02
day: Tue
topic_slug: retrieval-strategies-dense-sparse-hybrid
topic_title: "Retrieval strategies — dense vs sparse vs hybrid"
parent_overview: W02/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 15
sources:
  - url: https://learn.microsoft.com/en-us/azure/search/hybrid-search-ranking
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://weaviate.io/blog/hybrid-search-explained
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://tianpan.co/blog/2026-04-12-hybrid-search-production-bm25-dense-embeddings
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://medium.com/@kumaran.isk/building-a-production-rag-pipeline-start-with-hybrid-retrieval-dense-bm25-rrf-e901aba17cae
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.mongodb.com/docs/atlas/atlas-vector-search/tutorials/reciprocal-rank-fusion/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Retrieval strategies — dense vs sparse vs hybrid

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define dense, sparse, and hybrid retrieval, and explain the query shapes each one is good and bad at.
- Compute a Reciprocal Rank Fusion (RRF) score by hand for a small example and explain why RRF is rank-based rather than score-based.
- Identify which of dense, sparse, or hybrid retrieval is the right baseline for a given query distribution and corpus shape.
- Compose a hybrid retriever using plain Python — without leaning on framework chain-pipe abstractions.

## 2. Introduction

Retrieval is the part of RAG that decides which slices of the corpus the language model gets to see. Get it wrong and the LLM is working with the wrong material; nothing downstream can fix that. There are three retrieval modes in mainstream use in 2026 — **dense** (embedding-vector similarity), **sparse** (lexical / BM25), and **hybrid** (the two combined and fused) — and they have different strengths, different failure surfaces, and different operational footprints.

The reason this topic is its own reading is that the choice between them is not "pick the best one and move on." Production retrievers in 2026 almost universally ship hybrid as the default because the two retrievers fail on different query shapes — meaning together they cover most of the failure surface that either one alone misses ([TianPan — Hybrid Search in Production](https://tianpan.co/blog/2026-04-12-hybrid-search-production-bm25-dense-embeddings), retrieved 2026-05-26). Knowing *which retriever does what* and *how to compose them* is the difference between a retriever that works on demos and one that survives a heterogeneous query distribution.

This reading is the generic technique. The day's overview connects it to a specific federal-acquisitions worked example; this reading reaches into other industries on purpose so you can see the same pattern in different terminology.

## 3. Core Concepts

### Dense retrieval

A dense retriever embeds the query and the documents into the same vector space and returns documents whose vectors are closest by some metric (cosine, dot-product, or Euclidean depending on how the embedding model is normalised). Dense retrieval excels at **semantic similarity**: a query phrased as "how long until I get my refund?" can match a document phrased as "return processing typically takes 7–10 business days" even though no significant words overlap.

The standard implementation uses an approximate-nearest-neighbour (ANN) algorithm — most production systems use HNSW (Hierarchical Navigable Small World) graphs — so the lookup is sub-linear in corpus size at the cost of approximate (not exact) recall. The candidate pool size (`numCandidates` in Atlas, `efSearch` in Lucene) trades recall for latency.

**Strengths:** paraphrase robustness, synonym handling, multilingual matching with a multilingual embedding model, intent-level matching.

**Weaknesses:** rare tokens (product codes, error numbers, statute IDs) get fuzzed away because the embedding model has rarely seen them; exact phrases lose to "close enough" semantic alternatives; long documents lose discriminative power as chunk size grows.

### Sparse retrieval (BM25)

Sparse retrieval is the modern descendant of classical information retrieval. **BM25** (Best Match 25) is the standard scoring function — it ranks documents by term-frequency weighted by inverse-document-frequency, with saturation and length-normalisation parameters that make it more robust than raw TF-IDF.

The vocabulary in BM25 is the literal tokens in the corpus. No embeddings; no neural network at query time; just an inverted index and a scoring formula. This is why BM25 is fast (lookups are O(query terms × posting list length)) and why it has been the dominant lexical search algorithm for over twenty years.

**Strengths:** exact-token queries (clause IDs, SKUs, function names, IP addresses, version strings), rare tokens (which dense retrievers miss precisely because they are rare), interpretability (you can explain why a document ranked where it did).

**Weaknesses:** zero tolerance for paraphrase or synonym; vocabulary mismatch produces zero recall, not "close enough" recall.

### Hybrid retrieval

Hybrid retrieval runs both retrievers in parallel against the same corpus and merges the two ranked lists into one output. The two retrievers have **complementary failure surfaces**: dense misses what sparse catches, and vice versa. In production benchmarks, hybrid retrieval improves over either alone across precision and recall metrics on heterogeneous query distributions ([Weaviate — Hybrid Search Explained](https://weaviate.io/blog/hybrid-search-explained), retrieved 2026-05-26).

The composition is the interesting part. There are two common merge strategies:

**Weighted score fusion.** Normalise both retrievers' scores into the same range and compute a weighted sum. Sounds simple; in practice the normalisation is fragile because BM25 scores and cosine similarities live on incompatible scales and have different empirical distributions per query.

**Reciprocal Rank Fusion (RRF).** Ignore the scores entirely. Each document gets a score that depends only on its **rank** in each list:

```
RRF_score(doc) = sum over retrievers of  1 / (k + rank_in_that_retriever)
```

`k` is a constant (60 is the field-standard default — large enough that small rank changes do not dominate, small enough that the top of each list still counts). RRF is **score-scale agnostic** — it works whether BM25 scores are in `[0, 25]` or cosines are in `[-1, 1]` — and that property is why every major vector database now ships an RRF implementation as the recommended hybrid fusion strategy ([MongoDB Atlas — Hybrid Search with $rankFusion](https://www.mongodb.com/docs/atlas/atlas-vector-search/tutorials/reciprocal-rank-fusion/), retrieved 2026-05-26; [Azure AI Search — Hybrid Search Scoring](https://learn.microsoft.com/en-us/azure/search/hybrid-search-ranking), retrieved 2026-05-26).

#### Worked RRF example

Suppose the dense retriever and the BM25 retriever each return their top-3 for a query:

| Rank | Dense | BM25 |
|------|-------|------|
| 1 | Doc-A | Doc-C |
| 2 | Doc-B | Doc-A |
| 3 | Doc-C | Doc-D |

With `k = 60`:

- Doc-A: `1/(60+1) + 1/(60+2) = 0.01639 + 0.01613 = 0.03252`
- Doc-B: `1/(60+2) + 0          = 0.01613`
- Doc-C: `1/(60+3) + 1/(60+1) = 0.01587 + 0.01639 = 0.03226`
- Doc-D: `0          + 1/(60+3) = 0.01587`

Final ranking: Doc-A, Doc-C, Doc-B, Doc-D. Notice how Doc-A and Doc-C — appearing in both lists — beat Doc-B and Doc-D, which appeared in only one. That is RRF's structural bias: documents that both retrievers agree on rank highest.

### Hybrid composition — plain Python, not framework pipes

> [!instructor-review]
> Anti-pattern guarded against: LangChain v1.0 LCEL `|` pipe syntax. The blocklist (`langchain-lcel-pipe`, `langchain-chain-class`) flags any code that frames hybrid composition as `dense | sparse | fuse`. The v1.0 posture is plain Python composition — call the steps as functions, hold the result in variables, and use LangChain primitives only at the leaves (the actual retriever calls). Do not present LCEL pipes as foundational to learners.

The composition pattern is just two function calls and a merge:

```python
dense_hits  = dense_retriever.search(query, k=50)
sparse_hits = sparse_retriever.search(query, k=50)
merged      = rrf_merge(dense_hits, sparse_hits, k=60)
top_5       = merged[:5]
```

That is the architecture. No framework abstraction earns its place here.

### Choosing the right strategy

The mistake is treating this as a permanent commitment. Most production teams in 2026 run hybrid as the baseline, then *tune the weights or skip a retriever* per query class based on instrumentation ([Kumaran — Production RAG Pipeline](https://medium.com/@kumaran.isk/building-a-production-rag-pipeline-start-with-hybrid-retrieval-dense-bm25-rrf-e901aba17cae), retrieved 2026-05-26). For example: queries that match the regex for "SKU-XXXX" can be routed dense-skip (BM25 alone); queries that look conversational can be routed sparse-skip (dense alone) if instrumentation shows BM25 adds nothing. This kind of query-shape routing is a small bit of plumbing with a large quality payoff.

## 4. Generic Implementation

A minimal hybrid retriever in plain Python — domain-neutral, no framework abstractions:

```python
from dataclasses import dataclass

@dataclass
class Hit:
    doc_id: str
    rank: int
    score: float       # native score of the retriever — kept for diagnostics
    source: str        # 'dense' or 'sparse'

def rrf_merge(*ranked_lists: list[Hit], k: int = 60) -> list[Hit]:
    """Merge N ranked lists by Reciprocal Rank Fusion."""
    scores: dict[str, float] = {}
    seen_hits: dict[str, Hit] = {}
    for hits in ranked_lists:
        for hit in hits:
            scores[hit.doc_id] = scores.get(hit.doc_id, 0.0) + 1.0 / (k + hit.rank)
            seen_hits.setdefault(hit.doc_id, hit)
    fused = [
        Hit(doc_id=did, rank=i + 1, score=score, source='hybrid')
        for i, (did, score) in enumerate(
            sorted(scores.items(), key=lambda kv: -kv[1])
        )
    ]
    return fused

def hybrid_search(query: str, k_each: int = 50, k_final: int = 5) -> list[Hit]:
    """Run dense + sparse in parallel and fuse — plain function composition."""
    dense  = dense_retriever_search(query, k=k_each)   # leaf call: vector store
    sparse = sparse_retriever_search(query, k=k_each)  # leaf call: BM25 / OpenSearch
    fused  = rrf_merge(dense, sparse, k=60)
    return fused[:k_final]
```

Notes on what this is doing:

- `rrf_merge` operates purely on ranks — the `hit.score` is preserved only for diagnostics, not used in the merge.
- The two retriever calls are independent and could be parallelised with `asyncio.gather` in production.
- `k=60` is the conventional RRF constant; do not tune it as the first lever.
- The actual retriever implementations (`dense_retriever_search`, `sparse_retriever_search`) wrap whichever vector store and inverted index you use. That is the only place a vendor SDK lives.

## 5. Real-world Patterns

**E-commerce product search.** Major online retailers route hybrid retrieval as the default for site-search over their product catalog. The query distribution is genuinely mixed: "running shoes for flat feet" is a semantic-paraphrase query (dense wins) while "Brooks Ghost 16 size 11 men's" is an exact-token query (BM25 wins). Switching from dense-only to hybrid lifted exact-match recall measurably on the SKU-shaped query subset without harming the semantic-shaped subset ([TianPan — Hybrid Search in Production](https://tianpan.co/blog/2026-04-12-hybrid-search-production-bm25-dense-embeddings), retrieved 2026-05-26).

**Developer-tool documentation search.** A widely-used IDE's docs search runs hybrid retrieval because their query distribution has the same shape: exact-token searches for symbol names ("readFileSync") need lexical match while conceptual searches ("how do I read a file") need semantic. The hybrid retriever made it possible to serve both query classes from one index without separate retrieval paths.

**Healthcare clinical knowledge base.** A clinical-decision-support system serving primary-care physicians runs hybrid retrieval because clinical queries split between exact-term ("ICD-10 code J45.901") and semantic ("patient with persistent dry cough and no fever"). Dense alone surfaced semantically-adjacent but coding-incorrect entries; BM25 alone missed the long-form descriptive queries. Hybrid + RRF closed both gaps with one architecture.

**Customer-support knowledge base, SaaS company.** A SaaS support team replaced their BM25-only knowledge base search with hybrid retrieval and reported that the new top-3 results were correct for queries phrased as natural language complaints (where BM25 was previously failing because customers do not use product vocabulary) while still surfacing exact error-code matches as before. The team reported that RRF was the lower-touch choice because they did not have to maintain score normalisation as the corpus and embedding model evolved.

## 6. Best Practices

- Default to hybrid retrieval as the baseline rather than as an optimisation; the cost is one extra retriever call and a tiny merge function.
- Use RRF as your fusion strategy unless you have a measured reason to use weighted score fusion — RRF avoids the brittleness of score normalisation.
- Keep `k=60` in RRF as the starting constant; reach for a different value only after instrumentation says it matters.
- Pull a wide candidate pool from each retriever (50 each is a reasonable default) before fusing — RRF works better with depth than with narrow top-5 inputs.
- Instrument per-retriever recall on a held-out QA set so you can tell which retriever is doing the work for which query class.
- Compose retrieval steps as plain Python function calls — avoid LangChain `|` pipe syntax and `Chain` abstractions per the v1.0 posture in the blocklist.
- Match your dense-retriever distance metric to the embedding model's normalisation (cosine for unit-normalised vectors; dot-product for non-normalised) — Atlas defaults to cosine but that is a UI default, not a guarantee.

## 7. Hands-on Exercise

**Compute RRF by hand (10 min).** Given the following two ranked lists from a query "wireless headphones with active noise cancelling":

| Rank | Dense retriever | Sparse / BM25 retriever |
|------|-----------------|--------------------------|
| 1 | sony-wh1000xm5 | sony-wh1000xm5 |
| 2 | bose-qc-ultra | jbl-tune-770nc |
| 3 | jbl-tune-770nc | sennheiser-momentum-4 |
| 4 | sennheiser-momentum-4 | bose-qc-ultra |
| 5 | apple-airpods-max | apple-airpods-max |

Using `k = 60`:

1. Compute the RRF score for each of the five products.
2. Produce the final hybrid ranking.
3. Explain which product moved up the ranking under RRF compared to where it ranked in dense alone, and why.
4. Then re-run the calculation with `k = 5` instead of `60`. Does the order change? Explain in one sentence why `k` choice matters or doesn't here.

**What good looks like.** You should be able to compute the RRF scores by hand (no calculator beyond basic arithmetic). You should notice that products appearing in both lists at high ranks rise to the top under RRF — that is the rank-agreement bias that makes RRF stable across noisy score distributions. With `k = 5` the top of the list dominates more aggressively, which is why the field-standard default `k = 60` smooths the contribution from across the list rather than letting the top-3 win every time.

## 8. Key Takeaways

- Can I define dense, sparse, and hybrid retrieval and explain the query shapes that each one wins and loses on?
- Can I compute an RRF score by hand and explain why RRF is rank-based rather than score-based?
- For a given query distribution, can I choose the right retrieval baseline and identify which queries the baseline will struggle on?
- Can I compose a hybrid retriever in plain Python without reaching for framework chain-pipe abstractions?

## Sources

1. [Microsoft Learn — Hybrid Search Scoring (RRF), Azure AI Search](https://learn.microsoft.com/en-us/azure/search/hybrid-search-ranking) — retrieved 2026-05-26
2. [Weaviate — Hybrid Search Explained](https://weaviate.io/blog/hybrid-search-explained) — retrieved 2026-05-26
3. [Hybrid Search in Production: Why BM25 Still Wins on the Queries That Matter](https://tianpan.co/blog/2026-04-12-hybrid-search-production-bm25-dense-embeddings) — retrieved 2026-05-26
4. [Building a Production RAG Pipeline? Start With Hybrid Retrieval (Dense + BM25 + RRF)](https://medium.com/@kumaran.isk/building-a-production-rag-pipeline-start-with-hybrid-retrieval-dense-bm25-rrf-e901aba17cae) — retrieved 2026-05-26
5. [MongoDB Atlas — Hybrid Search with $rankFusion (Reciprocal Rank Fusion)](https://www.mongodb.com/docs/atlas/atlas-vector-search/tutorials/reciprocal-rank-fusion/) — retrieved 2026-05-26

Last verified: 2026-05-26
