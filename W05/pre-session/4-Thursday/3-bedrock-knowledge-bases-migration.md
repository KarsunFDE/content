---
week: W05
day: Thu
topic_slug: bedrock-knowledge-bases-migration
topic_title: "AWS Managed-Service Migration 1 — Bedrock Knowledge Bases on a hybrid RAG endpoint"
parent_overview: W05/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 13
sources:
  - url: https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://aws.amazon.com/bedrock/knowledge-bases/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking-parsing.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://aws.amazon.com/blogs/machine-learning/build-a-contextual-chatbot-application-using-knowledge-bases-for-amazon-bedrock/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Migrating a Custom RAG Endpoint to Amazon Bedrock Knowledge Bases

## 1. Learning Objectives

By the end of this reading, the learner can:

- Explain what Amazon Bedrock Knowledge Bases manages on the team's behalf and what it does not.
- Compare a custom retriever-plus-vector-store stack against a managed Knowledge Bases stack on at least five operational dimensions.
- Identify the surfaces of a custom RAG implementation that lose value when migrated and the surfaces that gain value.
- Frame an architecture-decision record that commits to full migration, hybrid, or no-migration with a defensible rationale.
- List the multi-tenant filter and citation-surfacing guarantees that must survive any migration.

## 2. Introduction

Retrieval-augmented generation has matured into a deployment pattern with two dominant shapes: **bring-your-own-stack** (custom chunkers, custom embedders, a vector database like Atlas Vector Search, OpenSearch, pgvector, or Pinecone, and a custom retrieval orchestrator) and **managed-pipeline** (a cloud-provider service that bundles chunking, embedding, vector storage, retrieval, and citation surfacing behind one API). Amazon Bedrock Knowledge Bases is AWS's managed-pipeline offering [Amazon Bedrock Knowledge Bases overview, retrieved 2026-05-26].

The migration decision is not "managed is better" or "custom is better." It is a per-endpoint trade-off that pivots on three axes: how much of the retrieval surface the team needs to control, how strong the authorisation-boundary story needs to be, and how fast the corpus changes. A team that owns chunk-size tuning as a competitive differentiator should think hard before giving that control away. A team that just needs reliable retrieval with citations on a stable corpus may save weeks of engineering by handing the pipeline over.

This reading covers the dimensions an honest migration evaluation walks. The Karsun-context placement of which `acquire-gov` endpoint to migrate, what FedRAMP boundary the migration thins, and which feature-inventory items must survive lives in the daily overview — this reading stays generic.

## 3. Core Concepts

### 3.1 What Bedrock Knowledge Bases manages

A Knowledge Base on Bedrock owns the full RAG ingestion-to-retrieval surface [Bedrock Knowledge Bases, retrieved 2026-05-26]:

- **Ingestion source** — typically an S3 bucket, but recent additions include Confluence, SharePoint, Salesforce, and web crawlers. Sources are configured per Knowledge Base.
- **Parsing** — documents are turned into text. For complex documents (tables, images), Bedrock offers a "foundation-model-as-parser" option that uses a multimodal model to extract structure.
- **Chunking** — fixed, hierarchical, semantic, or no-chunking. The team picks a strategy at KB creation; tuning the strategy after the fact requires a re-ingest [Bedrock chunking and parsing, retrieved 2026-05-26].
- **Embedding** — Bedrock invokes a configured embedding model (Titan Embeddings, Cohere Embed, etc.) on each chunk.
- **Vector storage** — Bedrock writes vectors to a managed vector store (OpenSearch Serverless, Aurora PostgreSQL with pgvector, Pinecone, MongoDB Atlas, or Redis).
- **Retrieval** — at query time, the `Retrieve` API (or `RetrieveAndGenerate`) does the similarity search and returns chunks with built-in citations.
- **Citation surfacing** — the response payload carries the source location of each chunk, ready to render in a UI.

### 3.2 What Bedrock Knowledge Bases does *not* manage

Where the lines fall is where the migration debate lives:

- **Re-ranking** — KB returns top-k by similarity. If your custom stack runs a cross-encoder re-ranker or a learn-to-rank model, that logic is not replicated; you may need to layer it back on with a Lambda.
- **Custom retrieval orchestration** — query rewriting, hypothetical-document expansion (HyDE), multi-step retrieval, agent-driven retrieval. KB does some query decomposition internally but does not expose hooks.
- **Fine-grained chunking heuristics** — KB exposes fixed, semantic, hierarchical, or none. If your team wrote a domain-specific chunker (clause-boundary, structured-document-section-aware), that logic does not port — you either accept Bedrock's strategies or you keep the custom chunker upstream.
- **Tenant-aware retrieval logic** that lives outside metadata filters — KB supports metadata filtering [Bedrock Knowledge Bases, retrieved 2026-05-26] but not arbitrary code-driven filtering. If your tenancy boundary requires custom logic, you need to confirm metadata-filter expressiveness covers it.
- **Audit-grade retrieval logging** — Bedrock writes CloudWatch traces, but if your team needs a custom audit row per retrieval (with chunk IDs, scores, and the rendered citations), that wrapper logic stays yours.

### 3.3 The five dimensions a migration evaluation pivots on

A short, defensible dimension set for the decision:

1. **Control over chunking and embedding** — full in custom, limited-to-bounded options in managed. If chunking strategy is a differentiator, this dimension dominates.
2. **Citation surface** — custom UX in custom (sometimes already a body of work), built-in citations API in managed. If the team already invested in citation UX, the gain from managed is modest.
3. **Authorisation boundary** — for regulated workloads, "AWS-managed inside an AWS-authorised boundary" is a cleaner story than "self-hosted vector store inside a separate vendor's boundary." This is often decisive.
4. **Debuggability on retrieval failure** — custom stacks let you instrument every step; managed stacks give you traces and rely on the cloud provider's tooling. Teams who debug RAG quality weekly find managed lower-resolution.
5. **Latency and cost shape** — managed services often add a hop and a per-query markup. For low-traffic endpoints, the markup is negligible; for high-traffic, it can dominate.

### 3.4 Migration patterns

Three patterns appear in the wild:

- **Full migration.** All RAG endpoints move behind Knowledge Bases. Custom retriever code retires. The team takes the authorisation-boundary win and accepts the chunking constraints.
- **Hybrid.** New corpora go behind Knowledge Bases; older corpora (where re-ingest cost is high or where custom chunking earns its keep) stay on the custom stack. This is the most common honest outcome.
- **No migration.** The team evaluates and writes a no-migration ADR. The rationale typically cites custom chunking or custom re-ranking that the team owns as a competitive surface.

The honest part of the evaluation is naming the dimensions where the chosen path *loses*. A full-migration ADR that does not name the lost chunking control is missing the trade-off; a no-migration ADR that does not name the missed authorisation-boundary win is missing the trade-off the other direction.

## 4. Generic Implementation

A minimum-viable `RetrieveAndGenerate` call against a Bedrock Knowledge Base, using the AWS SDK (note: AWS SDK for Java v2 — the v1 SDK is in maintenance mode):

```python
import boto3

bedrock_agent_runtime = boto3.client("bedrock-agent-runtime", region_name="us-east-1")

def answer_with_kb(query: str, knowledge_base_id: str, tenant_id: str) -> dict:
    """Run a tenant-scoped retrieve-and-generate against a Bedrock Knowledge Base."""

    response = bedrock_agent_runtime.retrieve_and_generate(
        input={"text": query},
        retrieveAndGenerateConfiguration={
            "type": "KNOWLEDGE_BASE",
            "knowledgeBaseConfiguration": {
                "knowledgeBaseId": knowledge_base_id,
                "modelArn": "arn:aws:bedrock:us-east-1::foundation-model/<current-model-id>",
                # Multi-tenant filter — the load-bearing line.
                # The metadata "tenant_id" was set on each chunk at ingest time.
                "retrievalConfiguration": {
                    "vectorSearchConfiguration": {
                        "numberOfResults": 5,
                        "filter": {
                            "equals": {"key": "tenant_id", "value": tenant_id}
                        },
                    }
                },
            },
        },
    )

    # Citation surface — each citation carries the source URI and the retrieved
    # chunks that contributed to the answer.
    return {
        "answer": response["output"]["text"],
        "citations": [
            {
                "source": ref["location"]["s3Location"]["uri"],
                "text": ref["content"]["text"],
            }
            for citation in response.get("citations", [])
            for ref in citation.get("retrievedReferences", [])
        ],
    }
```

> [!instructor-review]
> The `modelArn` placeholder uses `<current-model-id>` deliberately. Bedrock's catalog of foundation-model IDs (Claude family in particular) rotates faster than the known-bad-patterns blocklist's `anthropic.claude-v2|anthropic.claude-instant` rule can catch every stale ID. Before running this code, verify the current model ID against the Bedrock console or the model-lifecycle docs within the past 3 months. Never paste a model ID from a tutorial without re-verification.

Three things this snippet illustrates and one it deliberately does not:

- The `filter` keyword is the multi-tenant boundary. Metadata is attached at ingest; queries filter on it. If a query forgets the filter, every tenant's chunks become candidates.
- Citations are first-class; the response payload carries them. Custom UX work to render citation badges is reused; the retrieval-step custom code goes away.
- The model ARN is decoupled from the Knowledge Base. The same KB can be queried with different foundation models depending on the workload.
- The snippet does *not* show a re-ranker or query-rewriter step. If the original custom stack had either, that work has to live somewhere — typically a Lambda wrapper that calls `Retrieve` (not `RetrieveAndGenerate`), re-ranks the chunks, then invokes the model directly.

## 5. Real-world Patterns

**Healthcare — managed retrieval for a clinical-guidelines assistant.** A mid-sized hospital network had a custom RAG stack indexing clinical guidelines and protocols. They migrated to a managed retrieval service because the HIPAA boundary story was simpler when the vector store sat inside the cloud provider's HIPAA-authorised perimeter rather than in a third-party SaaS. Chunking control was a concern initially; they accepted the trade-off after determining that semantic chunking on guideline documents (which have natural section boundaries) produced comparable retrieval precision [Bedrock Knowledge Bases overview, retrieved 2026-05-26].

**E-commerce — hybrid for a product-search assistant.** A retailer ran custom RAG over product reviews (high-cardinality, slangy text) and managed RAG over product specifications (structured, stable). The custom chunker on reviews used a sliding-window plus sentiment-band heuristic that did not port to a managed service's chunking strategies; they kept it. Specifications, which changed only at SKU onboarding time and had clean structure, moved to a managed Knowledge Base. The hybrid ADR explicitly named "we lose unified observability" as the trade-off.

**Gaming — full migration for an in-game NPC assistant.** A studio building an NPC dialogue assistant migrated their entire RAG stack to a managed Knowledge Base because their team did not have the operational appetite to run a vector store at game-launch traffic spikes. The chunking limitation was less painful than expected because the lore documents had natural chapter boundaries. They accepted higher per-query cost in exchange for not running a database during launch week [AWS Knowledge Bases blog, retrieved 2026-05-26].

**Fintech — no-migration with documented trade-offs.** A fintech evaluated migrating their compliance-document retriever to a managed Knowledge Base. The chunker they had written knew about regulation section numbering (the chunk boundaries are critical — splitting a regulation across chunks creates retrieval failures the regulators care about). They wrote a no-migration ADR; the trade-off documented was "we accept the larger authorisation-boundary surface in exchange for chunk-quality control that we believe affects audit-defensibility."

## 6. Best Practices

- Evaluate per endpoint, not per platform — different RAG surfaces have different chunking and authorisation needs.
- Pin the foundation-model ID at the call site; never let a tutorial-pasted model ID into production unverified.
- Carry the multi-tenant filter as a non-negotiable test gate; write an integration test that asserts cross-tenant retrieval fails closed.
- Keep citation-rendering UX even after migration; do not throw away citation work just because the API surface changed.
- Re-ingest cost is real — KB chunking strategies cannot be changed without re-processing the corpus. Budget the re-ingest in the migration plan.
- Write the trade-off explicitly in the ADR. A no-trade-off ADR is hiding something.
- Verify the managed vector-store backend matches your regulatory profile (OpenSearch Serverless, Aurora pgvector, etc. each have different deployment-region constraints).

## 7. Hands-on Exercise

Pick a custom RAG endpoint you know (your own, a sample from an open-source project, or a hypothetical built from a public corpus like arXiv abstracts). In 15 minutes, draft a one-page migration ADR with three sections:

1. **Endpoint description** — corpus shape, current chunker, current retriever, traffic shape, latency budget.
2. **Five-dimension comparison** — control / citation / boundary / debuggability / cost — one paragraph per dimension comparing custom against managed.
3. **Decision** — full migration, hybrid, or no migration, with the trade-off you accept.

**What good looks like.** The ADR names a real trade-off the chosen direction accepts. The dimensions are operational, not marketing. The decision is defensible to someone who would have chosen differently. The multi-tenant filter (or equivalent isolation control) is identified as a non-negotiable that survives the chosen direction.

## 8. Key Takeaways

- What does Bedrock Knowledge Bases manage on your behalf, and what does it leave to you?
- Which five dimensions does a per-endpoint migration evaluation pivot on?
- When is a hybrid (some endpoints managed, others custom) the honest answer?
- How does the multi-tenant filter survive a migration, and why is it a non-negotiable?
- What makes a no-migration ADR defensible versus what makes it a stall?

## Sources

1. [Amazon Bedrock Knowledge Bases — User Guide](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html) — retrieved 2026-05-26
2. [Amazon Bedrock Knowledge Bases — product page](https://aws.amazon.com/bedrock/knowledge-bases/) — retrieved 2026-05-26
3. [Knowledge Bases chunking and parsing](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking-parsing.html) — retrieved 2026-05-26
4. [Build a contextual chatbot using Knowledge Bases for Amazon Bedrock — AWS ML blog](https://aws.amazon.com/blogs/machine-learning/build-a-contextual-chatbot-application-using-knowledge-bases-for-amazon-bedrock/) — retrieved 2026-05-26

Last verified: 2026-05-26
