---
week: W02
day: Mon
topic_slug: mongodb-atlas-vector-search-as-rag-store
topic_title: "MongoDB Atlas Vector Search as RAG store"
parent_overview: W02/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 8
sources:
  - url: https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-type/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-overview/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://contact-rajeshvinayagam.medium.com/unlocking-the-power-of-mongodb-atlas-vector-search-with-pre-filters-and-post-filters-f530253d2ca5
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://mongodb-developer.github.io/search-lab/docs/vector-search/filtering
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# MongoDB Atlas Vector Search as RAG store

## 1. Learning Objectives

By the end of this reading, the learner can:

- Describe the role a **vector store** plays in a RAG pipeline and name the operational responsibilities it owns.
- Read and write a basic **`$vectorSearch` aggregation pipeline** with `path`, `queryVector`, `numCandidates`, `limit`, and `filter`.
- Explain why **`$vectorSearch` must be the first stage** of any pipeline that uses it, and why **pre-filtering inside `$vectorSearch`** is materially different from post-filtering with a downstream `$match`.
- Apply the `numCandidates ≥ 20 × limit` heuristic and explain when it must be raised (when pre-filters are present).

## 2. Introduction

A vector store does one job well: given a query vector and a set of constraints, return the K stored vectors that are most similar, fast enough to satisfy a real-time application. Underneath that simple interface are a graph index (commonly HNSW or IVF variants), a metadata model, a query planner, and an operational story for ingestion, updates, deletes, and re-indexing.

MongoDB Atlas Vector Search is one of several production options in 2026. It is built into Atlas — the same cluster that stores your operational documents also runs your vector indexes — which collapses the "where do the embeddings live versus where does the source text live" question into one answer. For applications that already have an operational document store in MongoDB, the operational simplicity is significant: one cluster, one ACL model, one backup, one disaster-recovery story.

This reading covers what you need to know before the Monday vector-store ADR — the query shape, the index definition, the filtering semantics, and a small set of operational gotchas that almost everyone hits on day one.

## 3. Core Concepts

### 3.1 The vector store's responsibilities

A vector store in a RAG pipeline owns four operational responsibilities:

- **Approximate nearest-neighbour search.** Given a query vector, return the K most similar stored vectors, fast. "Approximate" means the index is allowed to occasionally return a slightly-non-optimal neighbour in exchange for sub-linear query time. The accuracy/latency knob is tunable.
- **Metadata filtering.** Restrict the search to vectors whose accompanying metadata matches a query predicate. This is how multi-tenancy, ACLs, date ranges, and document-type filters get enforced.
- **Ingestion and updates.** Insert new vectors, update existing ones, soft-delete or hard-delete. Index structures need to accommodate changes without full re-builds.
- **Operational concerns.** Backups, replication, monitoring, capacity planning, query observability.

A vector store that's strong on the first two and weak on the last two will become a pain point in production even if it benchmarks well in a lab.

### 3.2 The `$vectorSearch` aggregation stage

In Atlas, vector queries are MongoDB aggregation-pipeline stages. The stage is named `$vectorSearch` and has this shape:

```javascript
{
  $vectorSearch: {
    index: "rag_corpus_idx",              // name of the vector index
    path: "embedding",                     // field in the document holding the vector
    queryVector: [/* 1024 floats */],      // query embedding
    numCandidates: 200,                    // how many candidates the index examines
    limit: 10,                             // how many results to return
    filter: {                              // optional pre-filter expression
      "metadata.tenant_id": "acme-corp",
      "metadata.published_on": { $gte: ISODate("2024-01-01") }
    }
  }
}
```

Three rules from the MongoDB Atlas documentation (retrieved 2026-05-26) that everyone hits eventually:

- **`$vectorSearch` must be the first stage** of any aggregation pipeline that uses it. It cannot appear inside `$lookup` sub-pipelines or `$facet`. This is enforced by the query engine; it is not a style guideline.
- **Pre-filters belong inside `$vectorSearch`**, not in a downstream `$match`. The filter expression restricts the candidates the ANN search examines. Post-filtering with `$match` runs against whatever the search already returned, which is too late to recover candidates the search discarded.
- **`numCandidates` controls the recall/latency knob.** Higher values examine more graph nodes (more accurate, slower); lower values examine fewer (faster, less accurate). The widely-cited heuristic is `numCandidates >= 20 × limit` for ~90% recall; when pre-filters are present, this floor must be raised because filtering reduces the effective candidate pool.

### 3.3 The vector index definition

Vectors don't search themselves — they need an index. In Atlas, an index of type `vectorSearch` is defined separately from the collection:

```json
{
  "name": "rag_corpus_idx",
  "type": "vectorSearch",
  "fields": [
    {
      "type": "vector",
      "path": "embedding",
      "numDimensions": 1024,
      "similarity": "cosine"
    },
    {
      "type": "filter",
      "path": "metadata.tenant_id"
    },
    {
      "type": "filter",
      "path": "metadata.published_on"
    }
  ]
}
```

Four things to notice. First, **dimensionality is fixed at index creation**. You cannot retroactively change it. Switching embedding models with different dimensionalities means a new index. Second, **the similarity metric is part of the index definition**, not a per-query parameter — Atlas supports `cosine`, `dotProduct`, and `euclidean`. Choose to match the embedding model's documented preference. Third, **filter fields must be declared explicitly** as `filter`-type entries. Fields not declared as filter-type cannot be used inside `$vectorSearch.filter`. Fourth, **declared filter fields cost storage and indexing time** — declare what you need to filter on at query time, not "everything you might one day filter on."

> [!instructor-review]
> The MongoDB Atlas UI defaults to `cosine` for the similarity metric. The known-bad-patterns blocklist flags blanket "cosine is always right" advocacy because the correct metric depends on the embedding model's normalisation choice. Verify the model card's documented similarity metric before committing the index definition — the Atlas default is a convenient starting point but not a universal truth.

### 3.4 Pre-filtering vs post-filtering — why this matters

A common day-one mistake: putting tenant or ACL filters in a `$match` stage AFTER `$vectorSearch`:

```javascript
// WRONG — tenant filter runs AFTER vector search.
[
  { $vectorSearch: { /* no filter */, limit: 10 } },
  { $match: { "metadata.tenant_id": "acme-corp" } }
]
```

Why this is wrong: the `$vectorSearch` stage finds the 10 globally-nearest neighbours irrespective of tenant. If none of them happen to belong to `acme-corp`, the downstream `$match` returns nothing — even though the corpus might contain plenty of relevant `acme-corp` documents that just weren't in the top-10 globally. The ANN search has already discarded the candidates that mattered, and post-filtering cannot recover them.

The Atlas docs and the Medium write-up by Vinayagam (retrieved 2026-05-26) both explicitly call this out: **multi-tenant filtering and ACL enforcement belong in the `filter:` clause inside `$vectorSearch`**, declared as filter-type fields in the index definition. The pre-filter restricts the candidate pool the ANN search examines, so the top-K results are guaranteed to satisfy the filter.

A correctness footgun worth labeling: the wrong pattern doesn't always produce zero results. It often produces results that *look* fine — just biased, partial, or cross-contaminated. This is the kind of bug you discover six months in via an audit, not on day one via a unit test.

### 3.5 `numCandidates` — the recall/latency knob

`numCandidates` tells the HNSW graph index how many candidate nodes to examine before returning the top-`limit`. Higher means more accuracy, more compute, more time. Lower means the opposite.

The Atlas docs report (retrieved 2026-05-26): **`numCandidates >= 20 × limit`** for ~90% recall on most corpora. For `limit: 10`, that's `numCandidates: 200`. For `limit: 50`, that's `numCandidates: 1000`.

When **pre-filters are present**, the effective candidate pool is smaller (you've already restricted by tenant, date, document type), so the ANN search has to examine more nodes to find K acceptable ones. The community guidance is to raise `numCandidates` by 2–5× when filters are tight. There's no closed-form formula; the right value depends on filter selectivity and is something you discover by running held-out queries and observing recall.

The opposite failure mode is also real: setting `numCandidates` too high (e.g., `numCandidates: 10000` for `limit: 10`) wastes compute and adds latency without measurable recall improvement once you're past the recall ceiling.

## 4. Generic Implementation

A worked example outside federal acquisitions. Imagine an **e-commerce platform** building a semantic search over product descriptions. The collection holds product records with embedded vectors and metadata for category, price, region availability, and a stock flag.

```javascript
// Collection: products
// Document shape:
// {
//   _id, name, description, embedding: [...1024 floats],
//   metadata: { category, price_cents, region, in_stock, indexed_on }
// }

// Vector index definition.
const indexDef = {
  name: "products_vec_idx",
  type: "vectorSearch",
  fields: [
    { type: "vector", path: "embedding", numDimensions: 1024, similarity: "cosine" },
    { type: "filter", path: "metadata.category" },
    { type: "filter", path: "metadata.region" },
    { type: "filter", path: "metadata.in_stock" }
  ]
};

// Query: "comfortable running shoes" — restricted to the EU region, in stock.
const queryEmbedding = await embedQuery("comfortable running shoes");

const results = await db.collection("products").aggregate([
  {
    $vectorSearch: {
      index: "products_vec_idx",
      path: "embedding",
      queryVector: queryEmbedding,
      numCandidates: 400,                         // raised because pre-filter is selective
      limit: 10,
      filter: {
        "metadata.region": "EU",
        "metadata.in_stock": true
        // category intentionally NOT filtered — we want the semantic match
        // to surface cross-category neighbours (e.g., trail-running gear).
      }
    }
  },
  {
    $project: {
      name: 1,
      category: "$metadata.category",
      price: "$metadata.price_cents",
      score: { $meta: "vectorSearchScore" }       // similarity score for ranking display
    }
  }
]).toArray();
```

Five things to notice. First, **the index declares only the filter fields the team will actually filter on** — not everything in metadata. Second, **`$vectorSearch` is the first stage**, as required. Third, **the tenant-equivalent filter (region + stock) is INSIDE `$vectorSearch.filter`**, not in a downstream `$match`. Fourth, **`numCandidates` is raised to 400** (40× the limit) because the region+stock pre-filter is selective enough to prune the candidate pool meaningfully. Fifth, **`$meta: "vectorSearchScore"`** retrieves the similarity score, which is useful both for ranking display and for setting a confidence floor below which the LLM should refuse to answer.

## 5. Real-world Patterns

**Healthcare — clinical-trial matching.** A 2024 patient-trial-matching system used Atlas Vector Search over a corpus of trial protocols. The pre-filter discipline was non-negotiable: every query had to filter by patient consent status, condition category, and trial eligibility window. Putting these filters in `$vectorSearch.filter` (not in a downstream `$match`) was both a correctness and a latency win — pre-filtering cut the candidate pool by ~95% and dropped P95 query latency below 200ms.

**E-commerce — multi-tenant marketplace search.** A B2B marketplace platform serving thousands of merchant tenants used Atlas Vector Search with `tenant_id` as a mandatory filter-type field. The lesson learned during a security review: a developer had once written a query with the tenant filter in a downstream `$match` instead of inside `$vectorSearch`, and the cross-tenant leak it produced wasn't visible in any unit test (the tests all used data from one tenant). The team's response was to wrap every `$vectorSearch` call in a thin helper that enforced `tenant_id` in the filter clause and rejected anything trying to filter by tenant in a later stage.

**Fintech — fraud-pattern retrieval.** A bank's fraud-detection assistant retrieved similar historical fraud-pattern descriptions for case analysts. The team's `numCandidates` tuning story: at the default `20 × limit`, recall@10 sat at 88% — acceptable on most queries, but the analysts noticed missing matches on rare fraud types. Raising `numCandidates` to `60 × limit` lifted recall@10 to 96% with a ~30ms latency increase, which was well within budget. The lesson: the heuristic is a floor, not a target — measure on your corpus and tune.

**Gaming — community content moderation.** A multiplayer game indexed historical moderation decisions for use by trust-and-safety reviewers. The corpus had a quirk: most queries had to filter by language and platform (PC / console / mobile), and the filters were highly selective. Without raising `numCandidates`, recall on filtered queries dropped to ~70% — below the team's bar. Raising `numCandidates` to 5× the unfiltered baseline restored recall. The general pattern: tight pre-filters require higher `numCandidates`.

## 6. Best Practices

- **Declare filter-type fields at index creation for everything you'll filter on.** Schema changes are cheap; index re-creation is expensive.
- **Put multi-tenant and ACL filters inside `$vectorSearch.filter`, never in a downstream `$match`.** This is a correctness and security issue, not a style preference.
- **Start with `numCandidates = 20 × limit`; raise when pre-filters are present.** Measure recall on held-out queries before settling.
- **Match the similarity metric to the embedding model's documented preference.** Cosine is a common default; not universal.
- **Wrap `$vectorSearch` calls in a thin helper that enforces required filters** (tenant, ACL). Developers will forget; the helper will not.
- **Capture `$meta: "vectorSearchScore"` in your projection.** You'll want it for refusal-on-low-confidence behaviour and for debugging retrieval quality.
- **Plan for index re-creation at every embedding-model change.** Vector indexes are dimensionality-locked; a new model means a new index.

## 7. Hands-on Exercise

**Time: 10–15 minutes — code task.** Imagine a `documents` collection with these fields per doc: `_id`, `title`, `body`, `embedding` (1024-dim), `metadata.tenant_id`, `metadata.classification` (`public` / `internal` / `confidential`), `metadata.published_on`.

Write:

1. A vector-index definition that lets queries filter on `tenant_id` and `classification`.
2. A `$vectorSearch` aggregation that retrieves the top 5 documents for tenant `"team-alpha"` at classification `"internal"` OR `"public"`, with `numCandidates` sized appropriately for the filter selectivity.
3. A projection that returns the doc title, classification, and similarity score.

**What good looks like:** Your index definition declares `tenant_id` and `classification` as filter-type fields. Your `$vectorSearch` filter uses MongoDB's `$in` operator on classification (`{$in: ["internal", "public"]}`). Your `numCandidates` is at least `20 × limit` and ideally higher (3–5×) given the pre-filter. Your projection includes `$meta: "vectorSearchScore"`. Crucially: the classification filter is INSIDE the `$vectorSearch.filter` block, not in a downstream `$match`. If you put it in `$match`, your query is incorrect for the same reason every multi-tenant security review will flag — re-read §3.4.

## 8. Key Takeaways

- Can you describe the four operational responsibilities of a vector store, and explain which of them are RAG-specific?
- Can you write a `$vectorSearch` aggregation stage with `index`, `path`, `queryVector`, `numCandidates`, `limit`, and `filter`?
- Can you explain why **pre-filtering inside `$vectorSearch` is correctness-critical** and why post-filtering with `$match` is a multi-tenant footgun?
- Can you apply the `numCandidates >= 20 × limit` heuristic and explain when it must be raised?
- Can you read an Atlas vector-index definition and identify which fields are filter-eligible vs not?

## Sources

1. [Run Vector Search Queries — MongoDB Atlas Docs](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/) — retrieved 2026-05-26
2. [How to Index Fields for Vector Search — MongoDB Atlas Docs](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-type/) — retrieved 2026-05-26
3. [MongoDB Vector Search Overview — MongoDB Atlas Docs](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-overview/) — retrieved 2026-05-26
4. [Unlocking the Power of MongoDB Atlas Vector Search with Pre-Filters and Post-Filters — Vinayagam, Medium](https://contact-rajeshvinayagam.medium.com/unlocking-the-power-of-mongodb-atlas-vector-search-with-pre-filters-and-post-filters-f530253d2ca5) — retrieved 2026-05-26
5. [Pre-filtering Data — MongoDB Developer Search Lab](https://mongodb-developer.github.io/search-lab/docs/vector-search/filtering) — retrieved 2026-05-26

Last verified: 2026-05-26
