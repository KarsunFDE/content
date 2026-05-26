---
week: W02
day: Tue
topic_slug: atlas-index-management-and-design-artifacts
topic_title: "Atlas index management + design-planning artifacts"
parent_overview: W02/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 7
sources:
  - url: https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-type/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://mongodb-developer.github.io/search-lab/docs/vector-search/filtering
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-overview/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://oneuptime.com/blog/post/2026-03-31-mongodb-how-to-set-up-atlas-vector-search-indexes-in-mongodb/view
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Atlas index management + design-planning artifacts

## 1. Learning Objectives

By the end of this reading, the learner can:

- Declare a MongoDB Atlas Vector Search index with both `vector` and `filter` field types, and explain why the `filter` field has to be declared at index-creation rather than added at query time.
- Choose between ANN (Approximate Nearest Neighbour) and ENN (Exact Nearest Neighbour) for a given corpus size and explain the trade-off.
- Set `numCandidates` and `limit` to balance recall and latency for ANN searches.
- Treat ADRs (architectural decision records) as living documents that are annotated as implementation lands, rather than written once and abandoned.

## 2. Introduction

A vector index is not a free abstraction. It costs storage, build-time compute, and query-time latency — and if you do not declare its filter fields up front, it can quietly fail to enforce tenancy boundaries you assumed were enforced. The discipline of vector-index management is what separates a RAG system that works on a test corpus from one that holds up at production scale.

MongoDB Atlas Vector Search exposes a few load-bearing knobs: `vector` vs `filter` field types, ANN vs ENN search modes, and the `numCandidates` parameter that trades recall for latency. None are exotic; all are commit-time decisions painful to change after indexing ([MongoDB Atlas — Define Vector Search index syntax](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-type/), retrieved 2026-05-26).

The second half of this reading is about ADRs (architectural decision records) as *living* operator-choice records — annotated as implementation reveals what the choices actually cost. ADRs that are written once and abandoned are process tax; ADRs that are annotated as you implement are how future-you remembers why a choice was made.

## 3. Core Concepts

### Index field types — `vector` and `filter`

An Atlas Vector Search index declaration is a JSON document with a `fields` array. Each entry has a `type` — `vector` for the embedding column, `filter` for any column you want to pre-filter on. A minimal example:

```json
{
  "fields": [
    {
      "type": "vector",
      "path": "embedding",
      "numDimensions": 1024,
      "similarity": "cosine"
    },
    { "type": "filter", "path": "tenant_id" },
    { "type": "filter", "path": "doc_type" },
    { "type": "filter", "path": "last_revised" }
  ]
}
```

The `vector` field declares the embedding column and pins three things at index-creation time: the vector dimension (must match the embedding model's output), the similarity metric (cosine, dotProduct, or euclidean), and the path under the document where the vector lives.

The `filter` field declares which columns can appear inside the `$vectorSearch` operator's `filter` clause at query time. **A filter field that is not declared at index-creation time cannot be filtered on at query time** — you get a runtime error. This is not a usability bug; it is the property that lets Atlas push the filter down into the ANN graph traversal so the filter does not become a post-filter ([Atlas Search Lab — Pre-filtering Data](https://mongodb-developer.github.io/search-lab/docs/vector-search/filtering), retrieved 2026-05-26).

### Pre-filter vs post-filter — why this is load-bearing

If you forget to declare a filter field at index time, you can still filter — by adding a `$match` stage *after* `$vectorSearch`. This is a **post-filter** and it is dangerous for two reasons:

1. **Tenant correctness.** The vector search runs the ANN graph traversal over the *full* index, returning the global top-k. The `$match` post-filter then strips out the wrong-tenant results. The remaining results are correct for the tenant — but they are not the *top-k for that tenant*, they are the surviving subset of the global top-k. You can return fewer (or zero) results than the user expected because the right-tenant chunks were ranked below the filter cutoff in the global graph.
2. **Performance.** You waste candidate slots on documents you will discard. To get `k` results for a small tenant you may need `numCandidates` much larger than the heuristic default — and you may still not get enough.

The fix is to declare every filter field you intend to query on. Tenancy, document type, source freshness, language — anything you will filter at retrieval time has to be a declared `filter` field.

### ANN vs ENN

Atlas Vector Search supports two search modes:

**ANN (Approximate Nearest Neighbour)** — uses an HNSW graph to find approximately-closest vectors in sub-linear time. The approximation knob is `numCandidates`: the number of candidate vectors the graph examines during traversal. Higher `numCandidates` → higher recall → higher latency. MongoDB's recommended heuristic: `numCandidates ≥ 20 × limit` ([MongoDB Atlas — $vectorSearch aggregation stage](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/), retrieved 2026-05-26). This is the production default.

**ENN (Exact Nearest Neighbour)** — brute-force linear scan that returns the true nearest neighbours. Enabled by setting `exact: true` in the `$vectorSearch` stage. Latency is linear in corpus size; only viable for small corpora (typically thousands of vectors, not millions). Useful for small reference collections, evaluation harnesses, or specific debug scenarios.

The choice is corpus-size-driven, not preference-driven. A 3,500-page regulatory corpus, chunked at a few hundred tokens each, produces tens of thousands of vectors — ANN territory ([MongoDB Vector Search Overview](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-overview/), retrieved 2026-05-26).

### `numCandidates` — the recall-latency dial

The 20× heuristic (`numCandidates ≥ 20 × limit`) is a starting point, not a law. The right number depends on your filter selectivity:

- **Unfiltered or low-selectivity filters:** 20× is usually fine. Most of the candidate pool is plausible top-k material.
- **High-selectivity filters** (e.g., a tenant with a small share of the corpus, or a strict `last_revised >` cutoff): you may need 50× or 100× because most candidates will be filtered out before they can rank.
- **Tiny corpus:** consider ENN — it skips this question entirely.

The right move is to **measure recall on your held-out QA set** at different `numCandidates` values and pick the lowest value that hits your recall floor. Production teams in 2026 typically settle in the 100–500 range for `limit = 5–10`, with the exact number tuned by domain ([OneUptime — How to Set Up Atlas Vector Search Indexes](https://oneuptime.com/blog/post/2026-03-31-mongodb-how-to-set-up-atlas-vector-search-indexes-in-mongodb/view), retrieved 2026-05-26).

### Index build, rebuild, and cost

Index builds run in the background and are observable through the Atlas API — the index is unusable for `$vectorSearch` until it reports `READY`. Re-indexing a multi-million-vector corpus takes minutes to hours. Embedding-model swaps **require** a full re-index because the vector space changes — plan as a parallel-index swap (build new, switch queries when ready, drop old). Storage cost scales with vector count × dimension × 4 bytes (2 bytes with quantisation) and is measurable on the cluster bill for tens of millions of 1024-dim vectors.

### ADRs as living documents

The four ADRs the team writes on Plan Day capture commit-time decisions — what chunking strategy, what embedding model, what vector store, what hybrid composition. The temptation is to write them, accept them at the design review, and never look at them again. That is how ADRs become process tax.

The discipline that makes ADRs useful is **annotate-as-you-implement**:

- When you choose `numCandidates: 100` on Tue afternoon, that choice annotates the hybrid-retrieval ADR ("default 100; reach for higher when filter selectivity is below 10%").
- When you set `numDimensions: 1024` on the embedding-model ADR, that pin is what next quarter's "should we swap to model X?" question consults.
- When the eval harness shows context precision drops at `k > 8`, that empirical finding annotates the hybrid-retrieval ADR's "Why we chose 5" section.

The ADR becomes a *record of why current production looks the way it does*, with the dated annotations that show how the choices evolved. Future-you, reading the ADR six months in, can reconstruct the reasoning. Future-you reading a stale write-once ADR cannot.

ADR templates vary; the canonical Michael Nygard format (Context / Decision / Status / Consequences) is the field-standard starting point. Whichever template you pick, add a **Change log** section and treat it as the most-edited part of the document.

## 4. Generic Implementation

A minimal index-creation snippet and a query pattern — domain-neutral, illustrating the load-bearing operators:

```javascript
// 1. Declare the index — both vector AND filter fields named here
db.documents.createSearchIndex(
  "documents_vector_idx",
  "vectorSearch",
  {
    fields: [
      // The embedding column — dimensions must match the embedding model
      { type: "vector", path: "embedding", numDimensions: 1024, similarity: "cosine" },

      // Every column you'll filter on at query time must be declared here
      { type: "filter", path: "tenant_id" },
      { type: "filter", path: "doc_type" },
      { type: "filter", path: "last_revised" }
    ]
  }
)

// 2. Query with pre-filter pushed into the vector search (the correct pattern)
db.documents.aggregate([
  {
    $vectorSearch: {
      index: "documents_vector_idx",
      path: "embedding",
      queryVector: [/* 1024 floats from the embedding model */],
      limit: 5,                       // top-k returned
      numCandidates: 100,             // ANN pool — 20x limit is the starting heuristic
      filter: {                       // pushed down into the ANN graph traversal
        "tenant_id": "tenant-abc",    // tenancy enforced inside the index
        "doc_type": { "$in": ["policy", "support-article"] },
        "last_revised": { "$gte": new ISODate("2025-01-01") }
      }
    }
  },
  {
    $project: {
      _id: 1, title: 1, tenant_id: 1, last_revised: 1,
      vector_score: { $meta: "vectorSearchScore" }
    }
  }
])
```

Notes on what this is doing:

- The index declares `tenant_id` as a filter field. The query's `filter` clause uses it — Atlas pushes that filter down into the ANN graph traversal, so the top-5 returned are the top-5 *within the tenant*, not the global top-5 with the wrong tenants stripped.
- `numCandidates: 100` is the 20× heuristic for `limit: 5`. Tune up if the tenant filter is highly selective (a small tenant in a large index) and the recall on the QA set drops.
- `$vectorSearch` is the first stage of the aggregation — it cannot appear elsewhere; this is a runtime-enforced constraint, not a style suggestion.
- The `$project` stage exposes the vector score (`$meta: "vectorSearchScore"`) for diagnostics. The score is between 0 and 1 for cosine similarity, higher = closer.

This is the canonical shape. Any deviation (post-`$match` for filtering, undeclared filter fields, default `numCandidates`) is worth a code review.

## 5. Real-world Patterns

**SaaS knowledge-base over multiple workspaces.** A B2B SaaS product reported that early versions of their workspace-isolated knowledge base used a `$match` post-filter after `$vectorSearch`. Under load, certain small workspaces started returning zero results even when the documents existed — the global top-k was dominated by larger workspaces. Migrating the workspace ID into a declared `filter` field and pushing the filter down into `$vectorSearch` restored correct behaviour, and the team also reduced `numCandidates` once the filter was index-pushed because each candidate slot now mattered ([Atlas Search Lab — Pre-filtering Data](https://mongodb-developer.github.io/search-lab/docs/vector-search/filtering), retrieved 2026-05-26).

**E-commerce product-search ANN tuning.** A retailer running Atlas Vector Search over their product catalog reported tuning `numCandidates` from the 20× default to ~80× for their category-page search after observing that the category filter (often selecting <5% of products) was starving the ANN pool. The latency cost (~30ms extra) was acceptable for the recall improvement, and they captured the choice as an ADR annotation rather than a buried config constant.

**Healthcare clinical-knowledge index, mixed exact-vs-approximate.** A clinical-knowledge platform runs ENN (`exact: true`) over a small high-precision reference index (a few thousand authoritative protocol documents) and ANN over a larger general-medical-literature corpus, with the application layer choosing per query type. ENN gives hard recall guarantees for high-stakes protocol queries; ANN gives acceptable latency for broader literature search. The ADR captures the split and the per-index `numCandidates` choices.

**Embedding-model migration via parallel indexes.** An e-commerce search team migrated from a 768-dim to a 1024-dim embedding model by building the new index alongside the old, mirroring writes for a week, A/B-testing, then cutting over. The ADR's date-stamped annotations recorded both the original choice and the migration — six months later an external auditor had the full provenance trail in one document ([OneUptime — How to Set Up Atlas Vector Search Indexes](https://oneuptime.com/blog/post/2026-03-31-mongodb-how-to-set-up-atlas-vector-search-indexes-in-mongodb/view), retrieved 2026-05-26).

## 6. Best Practices

- Declare every filter field at index-creation time — do not rely on post-`$match` filtering for correctness, especially across tenancy or workspace boundaries.
- Use ANN as the production default; reserve ENN for small reference collections or evaluation harnesses where exact recall guarantees matter.
- Start `numCandidates` at 20 × `limit`, then tune up based on measured recall on a held-out QA set — particularly when filter selectivity is high.
- Match `numDimensions` to the embedding model exactly — a mismatch is a runtime error at query time, not at index-creation time.
- Match the `similarity` metric to the embedding model's normalisation — `cosine` for unit-normalised vectors, `dotProduct` for non-normalised, `euclidean` rarely outside specific domains.
- Plan embedding-model swaps as parallel-index migrations — build the new index, mirror writes, validate, cut over, drop the old. Never rebuild in place.
- Treat ADRs as living documents — every implementation discovery that changes an operator value or assumption gets a dated annotation in the relevant ADR.
- Re-verify index health after every corpus refresh — `READY` status, document count, sample query latency.

## 7. Hands-on Exercise

**Index-declaration drill (10 min).** Pick any domain you can mentally model — an e-commerce catalog, a job-board listing site, a tech-support knowledge base, an e-learning content library. Imagine you are building the RAG index for it.

1. Write the JSON `createSearchIndex` declaration for your domain. Decide:
   - What embedding column path (`embedding`, `description_vector`, whichever fits)?
   - What `numDimensions` (pin to a specific embedding model — Titan v2 1024, OpenAI text-embedding-3-large 3072, etc.)?
   - What `similarity` metric (cosine for unit-normalised; dotProduct for non-normalised — check the model's docs)?
   - What `filter` fields will queries actually need (tenancy / workspace / category / freshness / language)?

2. Write the `$vectorSearch` query stage that pre-filters on at least two of your declared filter fields, with a `limit` and a `numCandidates` you can defend.

3. Identify **two** queries against this index that would fail if you forgot to declare one of the filter fields — describe the failure mode (zero results, wrong tenant leak, latency spike).

**What good looks like.** Your index declaration includes every filter field that your queries will use, named at index-creation time. Your `$vectorSearch` filter pushes those fields into the ANN graph traversal rather than into a downstream `$match`. The two failure-mode queries should each demonstrate a different failure (small-tenant starvation, cross-workspace leak, freshness-cutoff bypass). If you cannot describe two genuinely different failure modes, the index probably does not need that many filter fields — but more often you will find you have under-specified the filter set.

## 8. Key Takeaways

- Can I declare an Atlas Vector Search index with the right `vector` and `filter` fields for a given query pattern, and explain why filter fields must be declared at index-creation?
- Do I know when to choose ANN vs ENN, and how `numCandidates` trades recall for latency?
- Can I argue why pre-filtering inside `$vectorSearch` is correctness-load-bearing, not just performance-load-bearing?
- Do I treat ADRs as living, dated, annotation-rich documents rather than write-once design records?

## Sources

1. [MongoDB Atlas — Define Vector Search index syntax (filter field declaration)](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-type/) — retrieved 2026-05-26
2. [MongoDB Atlas — `$vectorSearch` aggregation stage (filter, numCandidates, ANN/ENN)](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/) — retrieved 2026-05-26
3. [MongoDB Atlas Search Lab — Pre-filtering Data in Vector Search](https://mongodb-developer.github.io/search-lab/docs/vector-search/filtering) — retrieved 2026-05-26
4. [MongoDB Atlas — Vector Search Overview (ANN vs ENN)](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-overview/) — retrieved 2026-05-26
5. [OneUptime — How to Set Up Atlas Vector Search Indexes in MongoDB (2026)](https://oneuptime.com/blog/post/2026-03-31-mongodb-how-to-set-up-atlas-vector-search-indexes-in-mongodb/view) — retrieved 2026-05-26

Last verified: 2026-05-26
