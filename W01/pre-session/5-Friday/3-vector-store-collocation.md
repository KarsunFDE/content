---
week: W01
day: Fri
topic_slug: vector-store-collocation
topic_title: "Vector store collocation — embeddings next to operational data"
parent_overview: W01/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-overview/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.pinecone.io/learn/chunking-strategies/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.anthropic.com/news/contextual-retrieval
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Vector store collocation — embeddings next to operational data

> The overview frames this against the training-project's MongoDB Atlas baseline. This reading is the generic case: when does it make sense to put your vector index *inside* your operational database, and when should it live in a dedicated vector service like Pinecone or Weaviate? The decision drives multi-tenancy correctness, data-residency surface area, and operational complexity.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define vector-store collocation and contrast it with a dedicated/standalone vector database.
- Explain the multi-tenant pre-filter pattern and why post-filtering on `$match` is a correctness bug, not just a performance bug.
- Identify three properties that push a system toward collocation (residency, transactional consistency, single-database simplicity) and three that push it toward a dedicated store (scale beyond ~10M vectors, polyglot persistence already in play, specialized features).
- Reason about the "first-stage rule" and why it matters operationally.

## 2. Introduction

There are two architectural shapes for where your vector embeddings live. The first — the *dedicated* shape — runs a separate service (Pinecone, Weaviate, Qdrant, Milvus) whose only job is to hold vectors and answer nearest-neighbor queries. The second — the *collocated* shape — adds vector-search capability to an existing operational database that is already storing the source documents. MongoDB Atlas, PostgreSQL with pgvector, Elasticsearch with dense vectors, and Redis with vector similarity are all examples of the collocated shape.

The choice used to be obvious — dedicated vector DBs were dramatically better at scale, and operational DBs barely supported vectors at all. In 2026 the gap has narrowed. Atlas Vector Search, pgvector, and OpenSearch's vector engine all support production workloads up to single-millions of vectors per index with acceptable latency. At that scale, the operational simplicity of *one database, two access patterns* starts to outweigh the marginal performance of a dedicated service.

The decision is no longer about raw speed. It is about the *data-residency surface*, the *correctness of multi-tenant queries*, and the *operational cost of keeping two databases in sync*. This reading walks through what those mean concretely, with worked examples outside the federal-acquisitions domain.

## 3. Core Concepts

### 3.1 What "collocation" buys you

Three properties:

- **One residency boundary.** When embeddings live in the same database as the source documents, there is one set of regions to certify, one backup story, one set of access controls, and one audit log. For regulated workloads the calculus is simple — every additional data store is another thing to certify and another thing that can drift out of compliance.
- **Transactional consistency between source and index.** If your application writes a new document and the embedding in the same transaction (or close to it), there is no window where the index lags the source. With a dedicated store you have an embedding pipeline that is *always* asynchronous, and you must reason about the lag explicitly.
- **Pre-filtering inside the vector query.** This is the single biggest correctness property. When the embeddings sit in the same collection as the documents, the vector search can apply MQL-style filters *during* the nearest-neighbor scan rather than as a post-filter. For multi-tenant systems this is the difference between "tenant isolation is correct by construction" and "tenant isolation is a thin wrapper you might forget."

### 3.2 The pre-filter vs. post-filter trap

This is the subtle bug. The natural-feeling but wrong pattern is:

```javascript
// WRONG — post-filter after vector search
db.documents.aggregate([
  { $vectorSearch: {
      index: "embedding_idx",
      path: "embedding",
      queryVector: [/* ... */],
      k: 10,
      numCandidates: 100,
  }},
  { $match: { tenant_id: "tenant-abc" } }   // <-- runs AFTER the ANN search
])
```

The bug is not (only) that it returns fewer than `k` results when most of the top candidates belong to other tenants. The bug is that the ANN search has already scanned other tenants' embeddings to find them — the document IDs leak through the candidate pool. In a strict-isolation environment this is a security incident even if the post-filter drops the rows before they reach the user, because internal logging, query plans, and metrics may have recorded the cross-tenant matches.

The correct pattern uses the `filter` clause *inside* `$vectorSearch`:

```javascript
// CORRECT — pre-filter pushed into the vector search
db.documents.aggregate([
  { $vectorSearch: {
      index: "embedding_idx",
      path: "embedding",
      queryVector: [/* ... */],
      k: 10,
      numCandidates: 100,
      filter: { tenant_id: "tenant-abc" }
  }}
])
```

The vector index must be configured with `tenant_id` declared as a `filter`-type field — otherwise the query errors at runtime, which is the system telling you it cannot enforce isolation [MongoDB, 2026].

### 3.3 The first-stage rule

MongoDB's `$vectorSearch` operator must be the *first* stage of any aggregation pipeline in which it appears. It cannot run inside `$lookup` sub-pipelines, `$facet` branches, or after any other transformation. This sounds like a syntactic quirk but it is a structural property: the operator runs against the on-disk ANN index, not against a transient pipeline document stream, and the index has no way to apply earlier stage transformations.

Concretely, this means:
- Anything you want to use as a pre-filter must already be on the document (tenant ID, document type, status flags). You cannot compute it in an earlier `$addFields` stage.
- Joining across collections happens *after* the vector search via `$lookup` on the matched documents.
- "Search within these particular IDs that I pulled from another query" is a two-trip pattern — query the IDs first, then issue a vector search with the IDs in the `filter` clause [MongoDB, 2026].

Other vector stores have analogous rules. Pinecone requires filters as a metadata predicate on the index; pgvector allows arbitrary SQL but the index plan changes dramatically based on whether the filter is on an indexed column.

### 3.4 ANN vs. ENN — the cost knob

All production vector stores default to approximate-nearest-neighbor (ANN) search using algorithms like HNSW (Hierarchical Navigable Small World). ANN trades a small amount of recall (the fraction of true top-k results returned) for orders-of-magnitude lower latency. The `numCandidates` parameter controls the trade-off: more candidates = better recall = higher latency.

The MongoDB rule of thumb is `numCandidates >= 20 * k`. For applications where you genuinely need exact results (regulatory de-duplication, near-miss search for fraud detection), the `exact: true` flag flips to exhaustive search — orders of magnitude slower but with perfect recall [MongoDB, 2026].

### 3.5 When NOT to collocate

Three pushback cases:

- **Scale beyond what your operational DB supports.** Atlas Vector Search is comfortable to several million vectors per index; past that, dedicated stores with sharding designed for vectors (Pinecone, Milvus) win on cost-per-query.
- **Polyglot persistence already in place.** If your system already uses PostgreSQL for transactional data and you are introducing vectors for a new feature, adding Pinecone may be lower-risk than introducing pgvector and re-tuning your Postgres deployment.
- **Specialized features you actually use.** Hybrid retrieval with learned sparse models, GPU-accelerated reranking, multi-vector ColBERT-style indexes — the dedicated stores ship these earlier and tune them harder. If you need them and your operational DB does not have them, the trade-off shifts.

## 4. Generic Implementation

Worked example outside federal acquisitions — a logistics company indexing shipment-event narratives so dispatchers can ask "find recent shipments with weather-related delay reports near Chicago." Embeddings live in the same Postgres database that already stores the structured shipment records, using `pgvector`.

```sql
-- Schema: shipments table already exists with structured columns.
-- Add an embedding column for narrative text.

ALTER TABLE shipments
ADD COLUMN narrative_embedding vector(1536);

-- Indexed-pre-filter pattern: HNSW index on the embedding,
-- B-tree on the pre-filter column (carrier_id for tenancy).
CREATE INDEX shipments_embedding_hnsw
  ON shipments
  USING hnsw (narrative_embedding vector_cosine_ops);

CREATE INDEX shipments_carrier_idx ON shipments(carrier_id);

-- Query: vector search with carrier-id pre-filter pushed into the WHERE clause.
-- Postgres planner uses the B-tree on carrier_id BEFORE the HNSW scan when
-- the filter is selective enough — this is the equivalent of $vectorSearch's
-- filter clause, just expressed in SQL.
SELECT
  shipment_id,
  narrative,
  narrative_embedding <=> $1 AS distance
FROM shipments
WHERE carrier_id = $2
  AND created_at > NOW() - INTERVAL '90 days'
ORDER BY narrative_embedding <=> $1
LIMIT 10;
```

The pattern is the same as MongoDB's: tenancy is enforced *during* the vector retrieval, not after. The query optimizer is responsible for choosing the right join order — the application developer's job is to make sure the pre-filter column is indexed and the cardinality is high enough to be selective.

## 5. Real-world Patterns

**Notion's AI search.** Notion stores user pages in their primary operational database and added embeddings as a collocated column. The collocation lets them enforce workspace-and-permissions filters in the same query that retrieves semantic matches — a dedicated store would have required mirroring the entire permission graph into the vector service.

**Shopify product search.** Shopify uses a hybrid approach: dedicated infrastructure for the cross-tenant catalog-wide search, but per-merchant collocated embeddings for in-store recommendations. The split tracks the tenancy boundary — public-facing search lives where scale dominates, merchant-private search lives where consistency dominates.

**Healthcare clinical notes.** Hospitals indexing clinical notes for cohort retrieval typically collocate the embeddings with the EHR's note store. The HIPAA residency surface is large enough that adding a second vector database would require a separate Business Associate Agreement, separate audits, and separate access controls — the operational tax outweighs the marginal performance.

**Gaming — player-generated content moderation.** Game platforms (Roblox-style) often run dedicated vector stores for moderation because the corpus is cross-tenant by design (one ML team, one model, all user content). The collocation argument doesn't apply when there is no per-tenant access boundary to enforce [Anthropic, 2024 — analogous contextual-retrieval patterns].

## 6. Best Practices

- **Default to collocation when the corpus is multi-tenant or regulated** — the pre-filter pattern is correct by construction and the residency surface stays small.
- **Configure your filter fields in the index definition explicitly** — the index must know which fields will be filtered on. Implicit filtering is a future-you trap.
- **Treat `numCandidates` as a tunable SLO knob, not a constant.** Different query patterns (broad questions, narrow lookups) need different recall/latency trade-offs.
- **Pick the similarity metric to match the embedding model's documentation** — not to "cosine because that's what tutorials use." If the embedding model outputs unit-normalized vectors, dot product is faster and produces identical ranking; if not, cosine is safer.
- **Log retrieval scores and no-result rates per tenant.** A tenant whose retrieval returns near-zero scores for everything is a tenant whose embeddings have drifted or whose index is incomplete.
- **Plan the re-embedding migration path on day one.** Embedding models version. When you upgrade, every vector needs re-computing — the database that already holds the source documents makes this a single backfill job; a dedicated store with a separate sync pipeline doubles the work.

## 7. Hands-on Exercise

**Coding exercise (15 minutes).** Sketch the schema and a single vector query for a multi-tenant product-review system. Each review has: `review_id`, `merchant_id` (the tenant boundary), `product_id`, `review_text`, `rating`, `created_at`. You want to support the query "find recent positive reviews semantically similar to this query, scoped to merchant X."

Write:
1. The collection/table definition with the embedding field declared.
2. The index definition (vector field + at least two filter fields).
3. The query body with the correct pre-filter clause.
4. One sentence explaining what would break if you moved any filter outside the vector-search stage.

**What good looks like.** The index declares `merchant_id` and `rating` (or `created_at`) as filter-type fields. The query pushes both into the `filter` clause of the vector search itself, not into a downstream `$match` or `WHERE` after the search. The explanation names the leak: a post-`$match` would let the ANN scan touch other merchants' embeddings before dropping them, which is both a correctness issue at scale (recall degradation) and a residency issue at audit (cross-tenant access in the query plan).

## 8. Key Takeaways

- Can I define vector-store collocation in one sentence and name the alternative?
- Do I know why pre-filtering inside the vector search is a correctness property, not just a performance one?
- Can I name the first-stage rule and one constraint it imposes on my application design?
- Given a corpus's scale, tenancy model, and operational stack, can I make the collocation-vs-dedicated decision and defend it in two sentences?
- Do I know which similarity metric to pick and why "cosine because tutorials" is the wrong reason?

## Sources

1. [Run Vector Search Queries (`$vectorSearch` reference) — MongoDB Atlas](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/) — retrieved 2026-05-26 via /web-research. Canonical syntax, filter operators, first-stage rule, ANN vs. ENN flag.
2. [Vector Search Overview — MongoDB Atlas](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-overview/) — retrieved 2026-05-26 via /web-research. Conceptual framing of collocation and the ANN/HNSW algorithm choice.
3. [Chunking Strategies for LLM Applications — Pinecone](https://www.pinecone.io/learn/chunking-strategies/) — retrieved 2026-05-26 via /web-research. Vendor-neutral discussion of the chunk-size / embedding-context-window trade-off; used here for the metric-vs-model-normalisation note.
4. [Introducing Contextual Retrieval — Anthropic Engineering](https://www.anthropic.com/news/contextual-retrieval) — retrieved 2026-05-26 via /web-research. Used here for the hybrid (dense + BM25) discussion that informs the "what specialized features push to dedicated stores" framing.

Last verified: 2026-05-26
