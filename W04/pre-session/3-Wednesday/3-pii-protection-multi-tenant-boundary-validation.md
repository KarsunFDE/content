---
week: W04
day: 3-Wednesday
topic_slug: pii-protection-multi-tenant-boundary-validation
topic_title: "PII Protection + Multi-tenant Boundary Validation"
parent_overview: W04/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://genai.owasp.org/llmrisk/llm082025-vector-and-embedding-weaknesses/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.indusface.com/learning/owasp-llm-vector-and-embedding-weaknesses/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.cobalt.io/blog/vector-and-embedding-weaknesses
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://uplatz.com/blog/multi-tenancy-patterns-in-vector-databases-for-saas-applications-architectural-dynamics-performance-trade-offs-and-future-scaling-strategies/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://redis.io/blog/data-isolation-multi-tenant-saas/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# PII Protection + Multi-tenant Boundary Validation

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the three sub-classes of OWASP LLM08:2025 — cross-tenant leakage, embedding inversion, and data poisoning — and pick the right mitigation per sub-class.
- Compare the canonical multi-tenant vector-store isolation patterns (shared collection with metadata filter, per-tenant namespace, per-tenant collection, per-tenant database) and name the trade-off axis.
- Recognise that the tenant-isolation boundary must hold at **every retrieval seam**, not just at the application's auth layer — application-only enforcement is insufficient in regulated industries.
- Identify what counts as PII in an LLM context (training data, inference inputs, retrieved chunks, generated outputs) and the LLM02:2025 disclosure paths.
- Apply tag-and-classify discipline so retrieved chunks carry their access-level metadata into the LLM prompt.

## 2. Introduction

OWASP added LLM08:2025 Vector and Embedding Weaknesses as a *new* item in the 2025 revision because the RAG pattern moved from experimental to default and the security community caught up to its failure modes. The canonical entry is blunt: "weaknesses in how vectors and embeddings are generated, stored, or retrieved" let attackers inject harmful content or **access sensitive information across tenant boundaries** ([OWASP GenAI, 2025](https://genai.owasp.org/llmrisk/llm082025-vector-and-embedding-weaknesses/)).

The multi-tenant boundary is where LLM08 and LLM02 (Sensitive Information Disclosure) overlap. In a single-tenant LLM application, "PII protection" is about not echoing back what the user shared. In a multi-tenant LLM application, PII protection is *also* about not returning Tenant A's data when Tenant B asks — even when the underlying vector index is shared. The literature on production failures is unambiguous: "in healthcare, a patient query can pull back a chunk from another hospital's internal protocol, and in B2B support, a bot can answer with competitor pricing, all stemming from a shared vector index with inadequate tenant isolation" ([Uplatz, 2025](https://uplatz.com/blog/multi-tenancy-patterns-in-vector-databases-for-saas-applications-architectural-dynamics-performance-trade-offs-and-future-scaling-strategies/)).

The discipline question for the cohort: *can the boundary be proved at every retrieval seam, or does it rely on application-layer hope?* Tomorrow's exercises ask you to put a test on every retrieval call. Tonight's reading gives you the vocabulary, the isolation-pattern menu, and the validation shape.

## 3. Core Concepts

### 3.1 LLM08 sub-classes

OWASP names three ([OWASP GenAI, 2025](https://genai.owasp.org/llmrisk/llm082025-vector-and-embedding-weaknesses/); [Cobalt, 2025](https://www.cobalt.io/blog/vector-and-embedding-weaknesses)):

1. **Cross-tenant information leakage.** Different user classes share infrastructure; embeddings intended for one tenant get retrieved for another. Most common in shared-collection patterns with weak metadata filtering.
2. **Embedding inversion.** Attackers reverse-engineer embeddings to recover significant portions of the source text. The defense is to treat the vector store itself as PII-bearing storage — encrypt at rest, access-control reads, and audit-log all queries.
3. **Data poisoning.** Adversaries (or careless insiders) inject corrupt or adversarial content into the indexed corpus. Affects every downstream tenant that queries the poisoned region.

### 3.2 The four canonical multi-tenant vector-store isolation patterns

The 2025 SaaS literature converges on four patterns ([Uplatz, 2025](https://uplatz.com/blog/multi-tenancy-patterns-in-vector-databases-for-saas-applications-architectural-dynamics-performance-trade-offs-and-future-scaling-strategies/); [Redis, 2024](https://redis.io/blog/data-isolation-multi-tenant-saas/)):

| Pattern | How isolation works | Cost / Compliance trade-off |
|---|---|---|
| **Shared collection + metadata filter** | One global index; every vector tagged with `tenant_id`; queries must include `WHERE tenant_id = ?` filter | Cheapest, hardest to audit, most likely to fail when the filter is forgotten |
| **Per-tenant namespace** | Logical partitions inside one physical index (Pinecone-style namespaces) | Moderate cost; namespaces alone are NOT a tenancy boundary at production scale per known-bad-patterns guidance below |
| **Per-tenant collection** | Each tenant gets a dedicated index/collection; cross-tenant query is physically impossible | Higher cost; recommended for under-100-tenant healthcare / fintech deployments |
| **Per-tenant database** | Dedicated database per tenant | Highest cost; clearest compliance story; standard for regulated enterprise SaaS |

> [!instructor-review]
> **Known-bad-pattern flag (pinecone-namespaces-as-multitenancy from `known-bad-patterns.yml`).** Sources sometimes advocate Pinecone namespaces as *the* multi-tenancy boundary. They are not — at production scale they must be combined with per-tenant indexes for sensitive segments + application-layer access controls. This reading lists namespaces as one option in the menu, but the instructor should reinforce in war-room that namespaces alone do not pass a regulated-industry audit.

### 3.3 The boundary must hold at every retrieval seam

The most common production failure is not *missing* the isolation pattern — it's applying it at the front door and forgetting it at a back door. Common back doors:

- **Re-ranker calls.** The initial vector search filters by `tenant_id`, but a secondary re-ranker fetches additional context without re-applying the filter.
- **Reference-resolution calls.** The LLM produces a citation like `[doc-7]`, and the application fetches `doc-7` by ID without re-checking that it belongs to the calling tenant.
- **Cache layers.** A retrieval cache keyed only on the query string returns cross-tenant hits.
- **Embedding-store admin tools.** Internal dashboards or ETL jobs query the index with elevated permissions and accidentally surface results into a tenant-scoped UI.

Every retrieval seam = every place code reaches the vector store. The validation discipline is to write a test per seam, not per top-level endpoint.

### 3.4 What is PII in an LLM context (LLM02:2025)

Four leakage paths matter ([Indusface, 2025](https://www.indusface.com/learning/owasp-llm-vector-and-embedding-weaknesses/)):

1. **Training data.** Fine-tuned models can memorise verbatim training examples, including PII.
2. **Inference inputs.** Sensitive data passed in prompts can leak via logging, side channels, or echoing back.
3. **Retrieved chunks.** RAG retrieval pulls PII-bearing chunks into the prompt; the LLM may surface them in the response even when the retrieved chunk wasn't meant to be quoted.
4. **Generated outputs.** Confident-sounding hallucinations that look like PII (fabricated names, addresses, account numbers).

The mitigations stack: redact PII at indexing time, redact again at retrieval time, redact again at output time. Layered redaction is not paranoia — each layer catches a different failure mode.

### 3.5 Tag-and-classify discipline

OWASP's recommendation is to "tag and classify data to control access levels and prevent mismatches." In practice this means: every chunk in the vector store carries metadata for tenant ID, classification level (public / internal / restricted), retention class, and origin. The LLM prompt instruction explicitly references these tags ("only quote chunks tagged `classification=public`; treat `restricted` chunks as background context only"). The retrieval filter enforces them. The output filter audits them.

## 4. Generic Implementation

A generic Python sketch of a tenant-filtered vector-search wrapper that closes the most common boundary holes:

```python
# app/rag/tenant_filtered_search.py
# Generic multi-tenant vector retrieval wrapper.

from dataclasses import dataclass
from typing import Iterable

@dataclass(frozen=True)
class TenantContext:
    tenant_id: str
    classification_ceiling: str   # e.g., "internal" — user cannot see "restricted"

@dataclass(frozen=True)
class RetrievedChunk:
    chunk_id: str
    text: str
    tenant_id: str
    classification: str           # "public" / "internal" / "restricted"
    source: str

class TenantBoundaryError(Exception):
    """Raised when a chunk fails the boundary check — a finding, not a warning."""

def tenant_filtered_search(
    query_embedding: list[float],
    tenant: TenantContext,
    top_k: int = 10,
) -> Iterable[RetrievedChunk]:
    # 1. Push the tenant filter into the vector store itself.
    raw_hits = vector_store.search(
        embedding=query_embedding,
        filter={"tenant_id": tenant.tenant_id},   # NOT a post-filter
        limit=top_k,
    )

    # 2. Belt-and-braces: re-check tenant + classification on every returned chunk.
    #    A mismatch here is a security incident — log + raise, do NOT silently drop.
    for hit in raw_hits:
        if hit.tenant_id != tenant.tenant_id:
            audit_log.security_event(
                kind="cross_tenant_retrieval",
                expected=tenant.tenant_id, actual=hit.tenant_id,
                chunk_id=hit.chunk_id,
            )
            raise TenantBoundaryError(f"chunk {hit.chunk_id} crossed tenant boundary")
        if not classification_allows(hit.classification, tenant.classification_ceiling):
            audit_log.security_event(
                kind="classification_ceiling_violation",
                ceiling=tenant.classification_ceiling, chunk_class=hit.classification,
                chunk_id=hit.chunk_id,
            )
            continue   # drop, but log
        yield hit
```

What each section does:

- **Push the filter into the store.** The `filter={"tenant_id": …}` argument enforces isolation *inside* the vector index, not after the fact. Post-filtering in application code is the most common leak path because a developer can forget the filter.
- **Belt-and-braces re-check.** Even with the store-level filter, every returned chunk is re-validated. A mismatch *raises* — it is a security incident, not a normal flow exception.
- **Classification ceiling.** A second axis of authorisation: the calling user may belong to the right tenant but lack clearance for `restricted` chunks. Drop with audit.
- **Audit log on every violation.** The audit trail is what lets you prove the boundary holds during an audit.

## 5. Real-world Patterns

**Healthcare — clinical RAG with PHI boundaries.** A 2025 hospital-network deployment of a clinical-question RAG system uses per-tenant *databases* (each hospital is a tenant) backed by an additional in-database row-level security policy keyed on requesting clinician's department. The compliance argument named in their HIMSS write-up: "Clinical products require PHI boundaries to be enforceable at the infrastructure layer, because application code alone isn't enough when auditors come asking" ([Uplatz, 2025](https://uplatz.com/blog/multi-tenancy-patterns-in-vector-databases-for-saas-applications-architectural-dynamics-performance-trade-offs-and-future-scaling-strategies/)).

**Fintech — embedding inversion as a leakage vector.** A 2025 documented red-team finding at a wealth-management SaaS: an attacker with API access could submit crafted nearest-neighbour queries and reconstruct portions of indexed advisor notes (PII). The fix combined three measures — encrypt embeddings at rest, rate-limit nearest-neighbour queries per API key, and add differential-privacy noise to query results above a per-day query-volume threshold. The lesson: the vector store is itself PII storage, not "just an index" ([Cobalt, 2025](https://www.cobalt.io/blog/vector-and-embedding-weaknesses)).

**B2B SaaS — cross-tenant pricing leak.** A widely-cited 2025 incident: a B2B customer-support bot served a chunk from a competitor's onboarding documents (also a tenant on the same platform) because the shared-collection metadata filter was bypassed when an internal "search across all tenants" admin tool's results pipeline accidentally fed a tenant-scoped UI. The fix moved high-sensitivity tenants to per-tenant collections and forbade any admin tool from emitting results into tenant-scoped UIs ([Indusface, 2025](https://www.indusface.com/learning/owasp-llm-vector-and-embedding-weaknesses/)).

**E-commerce — per-merchant catalog isolation.** A multi-merchant marketplace platform uses per-tenant namespaces (acceptable here because the data is not regulated PHI/PCI/PII) but layers application-side authorization on top: every retrieval result is re-checked against an authorization service before being returned to the merchant's UI. The platform's engineering blog notes that namespaces alone "would not survive a SOC-2 audit if we were storing customer payment data — for catalog we accept the trade-off" ([Redis, 2024](https://redis.io/blog/data-isolation-multi-tenant-saas/)).

## 6. Best Practices

- **Match the isolation pattern to the regulation.** Healthcare / financial / federal-sensitive data → per-tenant database or per-tenant collection. Non-regulated B2B → shared collection with metadata filter is acceptable IF you can prove it at every seam.
- **Push the tenant filter into the vector store, not application code.** Application-layer filters are routinely forgotten when a developer adds a new retrieval seam.
- **Belt-and-braces re-check every returned chunk.** A boundary that holds 99% of the time is broken — a single cross-tenant retrieval is a finding.
- **Audit-log every boundary event.** Both successful retrievals (for forensics) and rejected retrievals (for incident detection).
- **Treat embedding inversion as a real threat.** Encrypt embeddings at rest, rate-limit nearest-neighbour APIs, and consider DP noise above query-volume thresholds for sensitive corpora.
- **Tag-and-classify every chunk.** Tenant ID + classification + retention class + origin, set at indexing time, enforced at every retrieval seam.
- **Re-validate after every code change that touches retrieval.** A test per seam, not per top-level endpoint.

## 7. Hands-on Exercise

**Scenario (10–15 min):** You are given a multi-tenant SaaS RAG application with three retrieval seams in the codebase:

1. The primary `/search` endpoint that filters by `tenant_id` correctly.
2. A re-ranker call that fetches additional context for the top-3 hits.
3. A citation-resolution endpoint that fetches the full text of a chunk by ID when the user clicks a citation.

**Whiteboarding prompt:** Draw the three retrieval seams. For each, name the tenant-filter enforcement point (where in the call stack is the boundary checked?). Identify which seams are vulnerable to cross-tenant leakage and why. Propose a test (one per seam) that would prove the boundary holds.

**What good looks like:** the diagram has all three seams. The candidate identifies that seams 2 and 3 are most vulnerable (the re-ranker may skip the filter; citation resolution by ID skips the filter entirely). The proposed tests are *adversarial* — they actively attempt cross-tenant retrieval as Tenant B and assert it fails, rather than just asserting Tenant A's retrieval works. The candidate also names that audit-log entries should be present on the failed adversarial attempts as evidence the boundary check fired.

## 8. Key Takeaways

- LLM08 has three sub-classes (cross-tenant leakage, embedding inversion, data poisoning) — can I pick the right mitigation per sub-class?
- The isolation-pattern menu has four options (shared+filter, namespace, per-tenant collection, per-tenant database) — can I match the pattern to the regulatory context?
- The boundary must hold at every retrieval seam, not just at the application's front door — have I enumerated my seams and put a test on each?
- PII in an LLM context flows through four paths (training, inference inputs, retrieved chunks, generated outputs) — does my layered redaction cover all four?
- The vector store is itself PII-bearing storage — am I encrypting embeddings at rest and rate-limiting nearest-neighbour APIs to defend against inversion?

## Sources

1. [OWASP GenAI Security Project — LLM08:2025 Vector and Embedding Weaknesses (canonical entry)](https://genai.owasp.org/llmrisk/llm082025-vector-and-embedding-weaknesses/) — retrieved 2026-05-26
2. [OWASP LLM08: Vector and Embedding Security Risks 2025 (Indusface)](https://www.indusface.com/learning/owasp-llm-vector-and-embedding-weaknesses/) — retrieved 2026-05-26
3. [Vector and Embedding Weaknesses: Vulnerabilities and Mitigations (Cobalt)](https://www.cobalt.io/blog/vector-and-embedding-weaknesses) — retrieved 2026-05-26
4. [Multi-Tenancy Patterns in Vector Databases for SaaS Applications (Uplatz, 2025)](https://uplatz.com/blog/multi-tenancy-patterns-in-vector-databases-for-saas-applications-architectural-dynamics-performance-trade-offs-and-future-scaling-strategies/) — retrieved 2026-05-26
5. [Data Isolation in Multi-Tenant SaaS (Redis, 2024)](https://redis.io/blog/data-isolation-multi-tenant-saas/) — retrieved 2026-05-26

Last verified: 2026-05-26
