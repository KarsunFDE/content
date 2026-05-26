---
template: research-brief
tech: MongoDB Atlas Vector Search
version_pinned: "$vectorSearch aggregation stage (Atlas only; not in community server)"
last_verified: 2026-05-22
last_verified_via: WebSearch+WebFetch (fallback; web-research unavailable)
recency_window: hot-tech 3mo
sources_count: 3
target_weeks: [W02]
candidates_deferred: []
known_bad_patterns_flagged:
  - id: vector-cosine-default
    note: "Some Atlas tutorials still default to cosine. Brief notes metric choice depends on embedding model normalisation; cohort should match metric to embedding-model docs."
known_bad_patterns_checked: true
author: research-subagent
---

# Tech research brief — MongoDB Atlas Vector Search

Last verified: 2026-05-22 · Recency window applied: hot-tech 3mo · Pinned version: $vectorSearch aggregation stage (Atlas-only)

## 1. What it is (3–5 sentences, no jargon)

MongoDB Atlas Vector Search lets you store vector embeddings alongside operational documents in MongoDB and query them with approximate-nearest-neighbour (ANN) or exact-nearest-neighbour (ENN) search via the `$vectorSearch` aggregation stage. For the FDE cohort, this is the RAG store of record in W2 — embeddings live next to tenant data, so multi-tenant pre-filtering happens in the same query as the vector search, not as a post-filter. The required-first-stage rule (see §2) is the operational gotcha most teams hit, and the `filter` clause is the load-bearing primitive for our multi-tenant story.

## 2. Current stable state (as of `last_verified`)

- **`$vectorSearch` is the canonical operator.** It MUST be the **first stage** of any aggregation pipeline where it appears. Cannot be used inside `$lookup` sub-pipelines or `$facet` stages.
- **Signature:**
  ```javascript
  { $vectorSearch: {
      index: "<index_name>",
      path: "<embedding_field>",
      queryVector: [<floats>],
      k: <int>,                 // results to return
      numCandidates: <int>,     // ANN candidate pool; recommend >= 20*k
      filter: <optional MQL document>,
      exact: <bool>             // true => ENN; omit or false => ANN
  }}
  ```
- **Filter clause (multi-tenant pre-filter):** uses MongoDB Query API operators on indexed fields. Supports `$eq` (short form: bare value), `$in`, `$gt/$gte/$lt/$lte`, `$ne`, `$and`, `$or`, `$nin`. Fields used in `filter` MUST be declared as `filter`-type fields in the vector index definition — otherwise the query errors.
- **Index creation:** done via Atlas UI or `db.collection.createSearchIndex(...)` with `type: "vectorSearch"`. Index definition declares fields: one `vector` field (path, numDimensions, similarity = cosine | euclidean | dotProduct) plus zero or more `filter` fields by path.
- **Score:** `$vectorSearch` adds a `vectorSearchScore` 0–1 (higher = closer). Project it with `$meta: "vectorSearchScore"` in a downstream `$project`.
- Breaking changes in the last 3 months: none of consequence — the `$vectorSearch` API has been stable. Filter operator list and the first-stage rule have been in place since GA.
- Known-bad-pattern check: **flagged** (vector-cosine-default) — Atlas defaults the index UI to cosine but the cohort should pick the metric to match the embedding model's normalisation.

## 3. What we teach (and what we deliberately don't)

- **In scope (W2):** `$vectorSearch` aggregation; the first-stage rule (and why post-`$match` filtering is a multi-tenant correctness footgun); the `filter` clause for pre-filtering on `tenant_id`; index creation with one `vector` field + one `filter` field for `tenant_id`; `numCandidates >= 20*k` recall heuristic; choosing similarity metric to match the embedding model.
- **Out of scope:** Atlas Search (`$search`, text/lexical) beyond a one-slide comparison; hybrid search via `$rankFusion`/`$unionWith` (mentioned only as the v3-syllabus upgrade path); the on-prem Enterprise Advanced vector flavour.
- **Misconceptions to pre-empt:** (a) "I'll add `$match` after `$vectorSearch` for tenant isolation." — Wrong: the vector search runs against the full ANN graph first; you'll waste candidates and may leak cross-tenant near-neighbours before the post-filter. Use `filter:` inside `$vectorSearch`. (b) "`$vectorSearch` can go anywhere in the pipeline." — false; first stage only.

## 4. Recommended primary sources

- [`$vectorSearch` aggregation stage (Atlas docs)](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/) — accessed 2026-05-22. Canonical reference for syntax, filter operators, first-stage rule.
- [MongoDB Vector Search Overview](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-overview/) — accessed 2026-05-22. Conceptual overview including ANN vs ENN trade-offs.
- [MongoDB Vector Search Quick Start](https://www.mongodb.com/docs/atlas/atlas-vector-search/tutorials/vector-search-quick-start/) — accessed 2026-05-22. End-to-end index creation + sample query.

## 5. Code reference snippets (idiomatic, current API)

```javascript
// Multi-tenant pre-filtered $vectorSearch — W2 canonical example
db.documents.aggregate([
  {
    $vectorSearch: {
      index: "embedding_idx",
      path: "embedding",
      queryVector: [/* 1536 floats */],
      k: 8,
      numCandidates: 200,                // ~25 * k
      filter: {
        "tenant_id": "tenant-abc",       // short-form $eq
        "doc_type": { "$in": ["policy", "memo"] }
      }
    }
  },
  { $project: {
      _id: 1, title: 1, tenant_id: 1,
      score: { $meta: "vectorSearchScore" }
  }}
])
```

```javascript
// Vector index definition — filter field must be declared
db.documents.createSearchIndex(
  "embedding_idx",
  "vectorSearch",
  {
    fields: [
      { type: "vector", path: "embedding",
        numDimensions: 1536, similarity: "cosine" },
      { type: "filter", path: "tenant_id" },
      { type: "filter", path: "doc_type" }
    ]
  }
)
```

## 6. Risks and watch-items

- Atlas tier matters — vector indexes require M10+ cluster. Cohort sandbox must be provisioned on M10 or shared-tier vector preview.
- Embedding dimensionality is fixed at index creation. Changing embedding model = rebuild index.
- `numCandidates` is the hidden knob. Too low → poor recall; too high → latency. The 20× heuristic is a starting point, not a law.

## 7. Alternatives the cohort should be aware of

- pgvector (Postgres) — surfaced as comparison in W2 scenario-alternatives; different filter semantics (SQL WHERE).
- Qdrant / Pinecone — purpose-built vector DBs; mention only briefly.

## 8. Brief sign-off

- Drafted by: research-subagent (2026-05-22)
- Reviewed against `known-bad-patterns.yml`: **flagged** (vector-cosine-default); brief counter-teaches.
- 1-month-release check: not triggered.
- Approved for downstream artifact authoring: 2026-05-22

## Sources

- `$vectorSearch` Atlas docs, mongodb.com, retrieved 2026-05-22 via WebFetch. <https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/>
- Atlas Vector Search overview, mongodb.com, retrieved 2026-05-22 via WebSearch. <https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-overview/>
- Atlas Vector Search quick start, mongodb.com, retrieved 2026-05-22 via WebSearch. <https://www.mongodb.com/docs/atlas/atlas-vector-search/tutorials/vector-search-quick-start/>
