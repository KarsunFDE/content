---
week: W02
day: Wed
topic_slug: multi-tenant-retrieval-boundaries
topic_title: "Multi-tenant retrieval boundaries — enforcing tenancy at the index layer"
parent_overview: W02/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://www.mongodb.com/docs/atlas/atlas-vector-search/multi-tenant-architecture/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.maviklabs.com/blog/multi-tenant-rag-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://aws.amazon.com/blogs/machine-learning/multi-tenant-rag-implementation-with-amazon-bedrock-and-amazon-opensearch-service-for-saas-using-jwt/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.pinecone.io/guides/index-data/implement-multitenancy
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.thenile.dev/blog/multi-tenant-rag
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Multi-tenant retrieval boundaries — enforcing tenancy at the index layer

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the three multi-tenant isolation models (silo, pool, bridge) and pick the right one for a given workload's compliance and scale profile.
- Explain why pre-filter and post-filter behave differently under tenant isolation and why only pre-filter is acceptable at audit-grade.
- Identify the failure mode where "namespaces as multi-tenancy" silently leaks data — and the correct framing that fixes it.
- Sketch the index-time configuration that makes a tenant boundary enforceable below the application layer.
- Decide between row-level security, index-per-tenant, and filtered-shared-index for a given tenant count, dataset size, and isolation requirement.

## 2. Introduction

Multi-tenancy is the operating assumption of every modern SaaS application. RAG systems inherit this assumption and add a twist: the retrieval layer is now part of the security perimeter. A vector index that returns "the most semantically similar documents" without regard to *whose* documents they are is a data leak waiting for its first audit.

The industry has converged on three patterns — silo, pool, bridge — and on a handful of implementation primitives (index-level filters at index-creation time, per-tenant indexes, namespaces, row-level security) that combine to enforce the boundary. The patterns are not equivalent: some enforce at the storage layer, others at the application layer, some only after candidate generation has already crossed the boundary internally. For regulated workloads, the difference between "boundary enforced at storage" and "boundary enforced after retrieval" is the difference between a passing audit and a finding. Get the boundary wrong here, and no downstream guardrail rescues you.

## 3. Core Concepts

### Three isolation models

**Silo (index-per-tenant).** Each tenant gets its own physical index or collection. Cross-tenant queries are *impossible* — the index doesn't exist in the same address space. This is the strongest isolation pattern and the natural choice for enterprise customers who pay for it explicitly, for regulated workloads where physical separation is required by the controlling regulation, and for tenants with materially different data shapes. The trade is operational: thousands of small indexes per cluster create management overhead, cold-start latency on infrequently-accessed tenants, and per-index minimum costs that can dominate at small dataset sizes.

**Pool (filtered-shared-index).** All tenants share one index. Every document carries a tenant-identifying field (generically `tenant_id`; in acquire-gov this is `agency_id` on the tenant-scoped collections — `Solicitation`, `Proposal`, `AuditEvent`, `Cpar` — and is *deliberately absent* from the cross-tenant `ClauseLibraryEntry` because every agency reads the same FAR/DFARS). Every query passes the caller's tenant ID as a filter. Cost-efficient (one HNSW graph, one set of replicas, amortized indexing overhead). Operationally simple (one schema, one set of dashboards). The boundary is only as strong as the filter discipline — which is where the implementation primitives below become load-bearing.

**Bridge (hybrid).** A pool model for the long tail of small tenants, with siloed indexes for tenants that exceed a size threshold or have specific compliance requirements (their own KMS key, their own region, a Business-Associate Agreement, etc.). The bridge model is what most production SaaS converges on past about Year Two.

### Why pre-filter and post-filter are not equivalent

The single most important implementation detail in pool isolation: *when does the tenant filter apply relative to the kNN search?*

**Pre-filter** runs the filter *before* the kNN candidate generation. The vector store narrows the searchable set to documents matching the filter, then runs the kNN search inside that subset. In MongoDB Atlas Vector Search the filter must be declared as `filter` type in the index's `fields` array at index-creation time; the same field can then be referenced in the `filter` clause of `$vectorSearch`. The kNN candidate pool is guaranteed to be tenant-scoped. This is the audit-defensible pattern.

**Post-filter** runs the kNN search across all documents, then drops candidates that don't match the filter. Two failure modes follow. First, if the post-filter is forgotten — a missed `WHERE`, a wrong middleware, an SDK that didn't propagate context — the index quietly returned other tenants' documents and the candidate pool is contaminated for any subsequent step. Second, even when the post-filter is correctly applied, if `numCandidates` is too low the filter has nothing to keep — the pre-filter would have returned the right answer because it narrowed first; the post-filter returns nothing because the unfiltered candidates were dominated by other tenants' documents that didn't survive the filter. The system silently returns "no results" for legitimate queries while the unfiltered index was visible internally.

**Index-level pre-filter is the only acceptable pattern for tenancy in pool isolation.** Post-filter can be acceptable for non-security filters (date ranges, content types, soft preferences) where leakage is not the failure mode. For tenancy it is not.

### The "namespaces alone" anti-pattern

A common mis-framing in pre-2025 RAG tutorials: *"Use namespaces in your vector store for multi-tenancy."* The pattern is real — Pinecone, OpenSearch, and Weaviate all support namespaces (or per-tenant shards / partitions) and queries are scoped to one namespace at a time, so query-time leakage is constrained.

> [!instructor-review]
> This pattern is flagged in `known-bad-patterns.yml` as `pinecone-namespaces-as-multitenancy`. Namespaces alone are *not* a complete tenancy boundary at production scale. Namespaces enforce query routing — they do not enforce *access control*. Any API client or service principal with credentials to one namespace typically has the same credentials to others, and the only thing preventing cross-tenant access is application code remembering to specify the right namespace. The correct framing is **namespaces + per-tenant index quotas + IAM-level access controls together**, with the IAM layer enforced by the platform rather than by the application. If a learner's `/web-research` reading or ADR draft cites a source that pitches "namespaces for multi-tenancy" without naming IAM and per-tenant credentials in the same breath, flag it at standup — the reading is pre-FedRAMP-grade.

### Implementation primitives across the major stores

- **MongoDB Atlas Vector Search.** Pool with the tenant-identifying field (`tenant_id` generically; **`agency_id` in acquire-gov**) declared as a `filter` field at index-creation time. A flat-index variant exists for many-tenants-small-data workloads (~1M tenants, <10k vectors each). Pre-filter is the recommended pattern in both index types. In the acquire-gov programme this is the load-bearing primitive that closes brownfield debt **Item 10** — `agency_id` is baked into the `Solicitation` / `Proposal` / `AuditEvent` / `Cpar` indexes' `fields` array on Tue afternoon, and every `$vectorSearch` call passes `filter: { agency_id: $caller_agency }`. `ClauseLibraryEntry` is cross-tenant (FAR/DFARS is the same for every agency) and *does not* declare an `agency_id` filter.
- **pgvector on PostgreSQL.** Pool with row-level security (RLS). The engine enforces the tenant filter at the table level — even if a query forgets the `WHERE tenant_id = $1` clause, the policy prevents cross-tenant rows from being returned.
- **Pinecone.** Namespace-per-tenant for query routing, combined with IAM (API key roles, per-namespace access keys, KMS keys). Million-scale namespace support on standard/enterprise plans; vendor capacity-planning for 100k+ namespaces.
- **OpenSearch / Elasticsearch.** Domain-level (silo), index-level (silo within shared cluster), or document-level (pool with `tenant_id` filter). The AWS SaaS RAG reference uses pool with JWT-derived tenant claims passed through as a filter.
- **Weaviate.** Multi-tenancy as a first-class object: per-tenant shards (physical data separation) under a single API surface — silo-by-default.

### Defense in depth

The strongest production architectures stack multiple isolation primitives so that no single failure breaks the boundary:

1. **Application layer:** every retrieval call passes the caller's tenant context, derived from a verified identity (JWT claim, service-mesh-issued mTLS identity, OAuth scope).
2. **Storage layer:** the index requires a tenant filter at query time, declared at index-creation time, and the filter is part of the index plan (not an after-the-fact `$match`).
3. **Identity layer:** the credentials that can read tenant A's index/namespace are not the credentials that can read tenant B's. A compromised application service-principal scoped to one tenant cannot pivot to another.
4. **Audit layer:** every query writes an `AuditEvent` with the caller's tenant, the filter applied, the result count, and a correlation ID. A breach of the boundary leaves a trail.

Any one of these can fail. The boundary holds because no single failure crosses all four.

## 4. Generic Implementation

A pgvector + Postgres row-level-security example, outside the federal-acquisitions domain. Imagine a customer-support tooling SaaS where each customer has private support-ticket history and search must never cross tenant boundaries.

```sql
-- Tenant-aware embedding table
CREATE TABLE support_ticket_chunks (
    chunk_id      bigserial PRIMARY KEY,
    tenant_id     uuid       NOT NULL,
    ticket_id     bigint     NOT NULL,
    chunk_text    text       NOT NULL,
    embedding     vector(1024) NOT NULL
);

-- Standard B-tree on tenant_id for fast pre-filter
CREATE INDEX ix_chunks_tenant ON support_ticket_chunks (tenant_id);

-- HNSW vector index over the embedding
CREATE INDEX ix_chunks_vec
    ON support_ticket_chunks
    USING hnsw (embedding vector_cosine_ops);

-- Row-level security: the tenant filter is enforced by the engine,
-- not by application SQL discipline.
ALTER TABLE support_ticket_chunks ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation
    ON support_ticket_chunks
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- Per-request, the application sets the tenant context once
-- (and the database refuses to return rows from other tenants
-- even if a SQL statement forgets the WHERE clause).
SET LOCAL app.current_tenant = 'a31b4f8c-...';

-- Query: no explicit tenant_id filter needed; RLS adds it for us.
SELECT chunk_id, chunk_text
FROM support_ticket_chunks
ORDER BY embedding <=> :query_vec
LIMIT 20;
```

The defense-in-depth pattern is the `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` line plus the `CREATE POLICY` clause. Even if a downstream query forgets the tenant filter, the engine adds it. The application still sets the tenant context — but a single missed `WHERE tenant_id = ?` no longer creates a leak.

## 5. Real-world Patterns

**B2B SaaS support knowledge bases.** A SaaS team migrated from post-filter-only Pinecone to namespace-per-tenant with per-namespace API keys after an enterprise pre-sales security review. The auditor asked what stopped a compromised service principal from querying another tenant's namespace; the post-filter answer was "application code." The redesign added per-tenant API keys at the IAM layer so a compromised credential scoped to one tenant can no longer pivot.

**Healthcare clinical documentation (HIPAA).** A clinical-documentation startup runs a silo model (one Postgres database per covered entity) for large hospital customers and a pool model with RLS for smaller clinics under a single BAA. The bridge architecture matched the compliance reality: large customers paid for physical isolation; small ones couldn't justify the overhead.

**Financial-services compliance archive.** A regulated archive provider running OpenSearch domain-level isolation per broker-dealer kept silo over pool because FINRA required physical separation to be demonstrable to auditors at the storage layer — not configurable at the application layer.

**E-commerce marketplace.** A large marketplace built per-seller analytics search using pool with `seller_id` as index-level pre-filter on a single OpenSearch domain. Millions of sellers made silo infeasible. The boundary held in a third-party penetration test because JWT-claim-to-filter wiring at the application layer was complemented by service-mesh mTLS identity checks at the OpenSearch ingress.

## 6. Best Practices

- Declare the tenant filter field as a `filter`-type field at index-creation time. Adding it later requires an online index rebuild on most stores and an outage on others.
- Default to pool with index-level pre-filter for new SaaS workloads; revisit once you have a tenant that materially requires silo (enterprise customer, regulatory body, region pinning).
- Stack at least two independent isolation layers (e.g., application-layer filter + database RLS, or namespace + per-namespace credentials). A single layer is one bug away from a leak.
- Never use post-filter to enforce tenancy. Post-filter is for non-security filters where leakage is not the threat model.
- Issue per-tenant credentials at the identity layer when the platform supports it (KMS keys, per-namespace API keys, scoped service-principal roles). A compromised credential should map to a single tenant, not a fleet.
- Audit-log every retrieval call with tenant, filter applied, and result count. The audit trail is what turns a "we don't think it leaked" into a "we can prove what was returned."
- Test the boundary directly: run a chaos-style test that fires queries with deliberately wrong tenant context and verifies the response is zero-results or an error, not a leak.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** You are designing the retrieval layer for a multi-tenant SaaS that helps law firms search their own internal case histories. Each law firm is a tenant. Within each firm, individual attorneys also need scoped views (an attorney sees only the matters they're staffed on). The corpus is ~50M documents across ~3,000 firms. Queries must complete in under 800ms p95. The product team requires that a malicious attorney with valid credentials at firm A cannot, by any combination of API calls, retrieve a document from firm B.

Design the isolation architecture. Name:

1. The isolation model (silo, pool, or bridge) and why.
2. The vector store and the specific tenancy primitive you'd use (Atlas filter field, pgvector RLS, Pinecone namespace + key roles, OpenSearch tenant filter).
3. The within-tenant scoping for attorney-level visibility — is it a second filter on the same index, or a separate mechanism?
4. The defense-in-depth layers you'd add so no single failure crosses the boundary.

**What good looks like.** A pool or bridge model justified by the tenant count (silo at 3,000 indexes is operationally painful but defensible if the data shapes differ; pool is the default unless a specific firm pays for silo). The vector store choice cites the specific primitive (e.g., "Atlas with `firm_id` as index-time filter field, secondary `matter_id_set` filter for attorney visibility"). Defense-in-depth includes at least two layers: identity-derived tenant context propagated as a filter *and* a database-level enforcement (RLS or per-tenant credentials). The chaos-test is named explicitly.

## 8. Key Takeaways

- *What are the three isolation models and when do you pick each?* — Silo for strict regulatory isolation or enterprise customers paying for it; pool for cost-efficient SMB workloads; bridge for mixed customer bases (the most common production endpoint).
- *Why is pre-filter the only acceptable pattern for tenancy?* — Because post-filter narrows after the kNN candidates have already been generated across tenants; a missed filter silently returns other tenants' documents, and even a present filter can return zero hits for legitimate queries.
- *What is the correct framing for namespaces in multi-tenancy?* — Namespaces enforce query routing, not access control. The correct combination is namespaces + per-tenant credentials/IAM + per-namespace keys, not namespaces alone.
- *What does defense-in-depth look like in practice?* — Application-layer filter from verified identity, storage-layer enforcement (RLS, index-time filter declaration, per-tenant credentials), identity-layer credential scoping, and an audit trail. Two independent layers must fail for the boundary to break.
- *How do you prove the boundary holds?* — Chaos tests that fire queries with wrong tenant context and verify zero-results-or-error. An audit log every retrieval call writes that includes tenant, filter applied, and result count.

## Sources

1. [Build a Multi-Tenant Architecture for MongoDB Vector Search (Atlas docs)](https://www.mongodb.com/docs/atlas/atlas-vector-search/multi-tenant-architecture/) — retrieved 2026-05-26
2. [Multi-Tenant RAG in 2026: Building Secure Retrieval-Augmented Generation for SaaS (Mavik Labs)](https://www.maviklabs.com/blog/multi-tenant-rag-2026) — retrieved 2026-05-26
3. [Multi-tenant RAG implementation with Amazon Bedrock and Amazon OpenSearch Service for SaaS using JWT (AWS ML Blog)](https://aws.amazon.com/blogs/machine-learning/multi-tenant-rag-implementation-with-amazon-bedrock-and-amazon-opensearch-service-for-saas-using-jwt/) — retrieved 2026-05-26
4. [Implement multitenancy using namespaces (Pinecone docs)](https://docs.pinecone.io/guides/index-data/implement-multitenancy) — retrieved 2026-05-26
5. [Building successful multi-tenant RAG applications (Nile blog)](https://www.thenile.dev/blog/multi-tenant-rag) — retrieved 2026-05-26

Last verified: 2026-05-26
