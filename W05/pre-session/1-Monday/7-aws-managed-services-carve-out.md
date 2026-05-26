---
week: W05
day: Mon
topic_slug: aws-managed-services-carve-out
topic_title: "AWS managed services carve-out — Bedrock Knowledge Bases, Agents-for-Bedrock, OpenSearch Managed"
parent_overview: W05/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 13
sources:
  - url: https://aws.amazon.com/bedrock/knowledge-bases/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://aws.amazon.com/blogs/machine-learning/multi-tenancy-in-rag-applications-in-a-single-amazon-bedrock-knowledge-base-with-metadata-filtering/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://aws.amazon.com/bedrock/agentcore/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://aws.amazon.com/blogs/machine-learning/amazon-bedrock-knowledge-bases-now-supports-amazon-opensearch-service-managed-cluster-as-vector-store/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://aws.amazon.com/opensearch-service/features/security/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# AWS managed services carve-out — Bedrock Knowledge Bases, Agents-for-Bedrock, OpenSearch Managed

## 1. Learning Objectives

By the end of this reading, the learner can:

- Explain why staged AWS managed-service onboarding (deferring some services for later) is a defensible engineering posture rather than gold-plating.
- Describe what Bedrock Knowledge Bases, Amazon Bedrock Agents (AgentCore), and Amazon OpenSearch Service (managed) each commit, and the multi-tenant patterns each supports.
- Articulate when a self-managed RAG / agent / vector-store implementation should *migrate* to the managed counterpart, and the boundary calls each migration forces (latency, tenant isolation, FedRAMP, vendor lock-in).
- Recognise the cost-blast-radius shape that makes managed AI services different from "regular" AWS services to onboard.

## 2. Introduction

A common pattern in mature AI-platform engagements: the team uses AWS Bedrock for the *model layer* from week one, but defers Bedrock's adjacent managed services (Knowledge Bases for managed RAG, AgentCore for managed agents, OpenSearch Service managed for vector storage and log analytics) until later in the programme. The deferral is not laziness — it's risk management on three axes:

1. **Cost blast radius.** Managed AI services meter on usage in ways the team must instrument before turning them on. Direct InvokeModel is reasonably predictable; Knowledge Base sync jobs, Agent action-group invocations, and OpenSearch query rate are less predictable until governance and dashboards are in place.
2. **Multi-tenant policy.** Each managed service has a different multi-tenant story. Going live before the tenancy boundary is named guarantees retrofitting later.
3. **FedRAMP / regulatory boundary.** Where each service is authorized matters; the boundary call must precede the data flowing.

This reading walks the three carve-out services generically — what each commits, when migration becomes defensible, what boundary calls each forces. The day's overview ties this to the cohort's specific migration questions for the week; here we look at the patterns across industries.

## 3. Core Concepts

### 3.1 Bedrock Knowledge Bases — managed RAG

Bedrock Knowledge Bases is a fully managed Retrieval-Augmented Generation service: AWS handles document ingestion, chunking, embedding, vector storage, retrieval, and the augmentation step that hands the retrieved context to a model ([Bedrock Knowledge Bases product page, 2026-05-26](https://aws.amazon.com/bedrock/knowledge-bases/)).

What it commits when you adopt it:

- **Ingestion pipeline ownership.** AWS owns the source-to-chunk-to-embedding pipeline. The team owns the source location (typically S3) and the schedule of ingestion runs.
- **Vector store choice.** Multiple backends supported — including OpenSearch Service managed clusters (cohort-relevant) and OpenSearch Serverless ([Knowledge Bases + OpenSearch Managed, 2026-05-26](https://aws.amazon.com/blogs/machine-learning/amazon-bedrock-knowledge-bases-now-supports-amazon-opensearch-service-managed-cluster-as-vector-store/)).
- **Multi-tenant pattern choice.** Two canonical approaches: silo (separate knowledge base per tenant — strongest isolation, scales linearly with cost) and pool (single knowledge base with metadata filtering on tenant_id — efficient at scale, requires filter discipline) ([Multi-tenancy with metadata filtering, 2026-05-26](https://aws.amazon.com/blogs/machine-learning/multi-tenancy-in-rag-applications-in-a-single-amazon-bedrock-knowledge-base-with-metadata-filtering/)).

When migration from self-managed RAG becomes defensible:

- Self-managed pipeline ops cost is dominating the team's capacity.
- The team's chunking strategy is generic enough that AWS's defaults are acceptable (or near-acceptable with metadata tweaks).
- The latency budget allows the managed retrieval path.
- The multi-tenant filter on a stable per-tenant identifier (e.g., `tenant_id`, `agency_id`) is enforceable.

When *not* yet:

- The team needs custom chunking the managed service doesn't expose hooks for.
- The tenancy boundary is not yet stable enough to commit a filter contract.
- Cost-per-tenant attribution isn't instrumented yet — without it, the migration is a finance black box.

### 3.2 Amazon Bedrock Agents (AgentCore) — managed agents

Amazon Bedrock AgentCore is the platform for building, connecting, and optimising AI agents at production scale: you can deploy agents using any framework and model with production-grade security from day one ([Bedrock AgentCore product page, 2026-05-26](https://aws.amazon.com/bedrock/agentcore/)).

What it commits:

- **Agent loop ownership.** AgentCore manages the reasoning loop, tool selection, action execution, and response streaming for agents you describe declaratively.
- **Framework neutrality.** Supports LangChain, OpenAI Agents SDK, Claude Agent SDK, Strands SDK, and custom frameworks — the platform's value is the production harness, not a framework lock-in.
- **Identity + connection layer.** AgentCore Identity handles how agents access external services across compute platforms (ECS, EKS, Lambda, on-premises).
- **Multi-agent coordination.** Supervisor-and-specialist patterns supported natively.
- **Memory retention.** Persistent agent memory for task continuity.

When migration from a self-built agent loop becomes defensible:

- The team's agent loop has stabilised — the surface that AgentCore manages is no longer the team's competitive edge.
- HITL authority is well-defined and aligns with AgentCore's user-confirmation surface (this is the load-bearing alignment question).
- Identity and connection security overhead is high enough that AgentCore's harness is net-positive.

When *not* yet:

- HITL semantics from your existing agent diverge from what AgentCore's confirmation surface offers. A mismatch here is a *correctness* risk, not just a UX one.
- The agent's tool surface includes custom invariants the managed harness doesn't enforce.
- The team needs to retain auditability of every reasoning step — verify AgentCore's tracing meets the bar before committing.

### 3.3 Amazon OpenSearch Service (managed) — vector + log analytics

Amazon OpenSearch Service managed clusters offer a scalable, secure, high-performance vector database for generative AI applications, supporting both k-NN and ANN algorithms at billion-scale ([OpenSearch vector capabilities, 2026-05-26](https://aws.amazon.com/opensearch-service/serverless-vector-database/)).

What it commits:

- **Cluster ops ownership.** AWS handles the cluster (provisioning, patching, snapshot, version upgrades) under a managed-service contract.
- **Dual workload.** Same service supports vector search (for RAG) and log analytics (for AIOps) — a teams can converge two workloads on one platform.
- **Regulatory posture.** HIPAA eligible; compliant with PCI DSS, SOC, ISO, FedRAMP ([OpenSearch security features, 2026-05-26](https://aws.amazon.com/opensearch-service/features/security/)).
- **Integration with Bedrock Knowledge Bases** as a vector backend, making it the canonical pairing when both are adopted together.

When migration from a self-managed alternative becomes defensible:

- The team is already onboarding Bedrock Knowledge Bases and benefits from the canonical pairing.
- Regulatory posture (FedRAMP, HIPAA) is a hard requirement and the existing alternative's authorization is uncertain.
- Cluster ops effort on the existing alternative is dominating capacity.

When *not* yet:

- Existing platform (e.g., MongoDB Atlas Vector Search, Pinecone, Qdrant) meets the latency, isolation, and cost profile and the team has operational fluency.
- A managed-service migration would require a rebuild of the embedding pipeline.

### 3.4 The cost-blast-radius shape

Why managed AI services need governance to land *before* they go live, not after ([Multi-tenant RAG patterns, 2026-05-26](https://aws.amazon.com/blogs/machine-learning/multi-tenant-rag-with-amazon-bedrock-knowledge-bases/)):

- Knowledge Base ingestion runs meter on document count + sync frequency. A naïve daily-full-reindex pattern can be 30× more expensive than a delta-sync, and the gap is invisible until the bill arrives.
- AgentCore agents meter on action-group invocations. A multi-agent flow that fans out to five specialists per request multiplies the meter by five — predictable if instrumented, surprising if not.
- OpenSearch cluster cost is a fixed-floor plus query-driven scaling; the floor is non-trivial.

The carve-out is a discipline. The team turns on each service after it has named: cost attribution per tenant, alert threshold for unexpected cost spikes, audit trail of which actions the managed service is taking on the team's behalf.

## 4. Generic Implementation

A generic decision matrix for whether to migrate an existing self-managed RAG workload onto Bedrock Knowledge Bases. The structure is portable to any managed-AI-service migration question.

```markdown
# Migration evaluation: self-managed RAG → managed RAG (generic worked example)

## Workload profile
- Document count: ~50k chunks, growing 1k/week.
- Query volume: ~200 QPS peak, ~50 QPS average.
- Latency budget: p95 < 800ms from query to grounded response.
- Multi-tenant: yes, ~40 tenants, isolation enforced via metadata filter.
- Regulatory: PCI-DSS in some tenants (cardholder context in chunks).

## Required-to-migrate signals
- [ ] Managed-service latency observed under load < 800ms p95.
- [ ] Per-tenant cost attribution instrumented before live cutover.
- [ ] Multi-tenant filter contract committed in scope survey + ADR.
- [ ] Regulatory authorization of managed service ≥ self-managed status.
- [ ] Vendor lock-in cost (re-platforming back) within tolerance.

## Stop-the-migration signals
- [ ] Latency p95 regression on parallel measurement run.
- [ ] Managed service does not expose chunking hook the workload depends on.
- [ ] Tenant isolation cannot be enforced by a stable per-tenant identifier.
- [ ] Cost spike on initial sync exceeds budget by > 2×.

## Migration mode (pick one)
- Full cutover: shadow + parallel + cut + retire. Highest confidence required.
- Hybrid: managed for tenant-class A (regulatory-pressed), self-managed for
  tenant-class B (latency-critical). Compromise; durable when classes diverge.
- Port-and-adapter: abstract the RAG interface so swap is configuration.
  Reversible; pays back if migration confidence is borderline.

## Rollback plan
- Trigger: latency p95 > budget for 3 consecutive days, OR
  cost projection > 1.5× planned, OR isolation incident.
- Action: re-route via port-and-adapter back to self-managed; retain managed
  resources for 30 days for forensic comparison; tear down on day 31.
```

The decision matrix is the durable shape. The team writes one per migration candidate, files it alongside the scope survey, and references it from the relevant ADR.

> [!instructor-review]
> Bedrock managed-services pricing and feature availability shift quickly. Direct learners to re-verify cost-per-document, cost-per-action-group-invocation, and FedRAMP authorization status via the AWS Bedrock console within the last 3 months before any migration commitment.

## 5. Real-world Patterns

**Healthcare — Knowledge Bases adoption with metadata-filter tenancy**. A health-tech SaaS serving multiple provider organizations migrated from a self-managed RAG pipeline to Bedrock Knowledge Bases using the pool pattern with metadata filtering on `provider_org_id`. The migration was gated on three signals: tenant-isolation testing across 50 synthetic queries with leak detection, latency parity under load, and a cost-per-provider-org dashboard live before cutover. Pool over silo because at 200+ provider orgs, silo-per-tenant would have multiplied infrastructure cost 200× ([Multi-tenancy with metadata filtering, 2026-05-26](https://aws.amazon.com/blogs/machine-learning/multi-tenancy-in-rag-applications-in-a-single-amazon-bedrock-knowledge-base-with-metadata-filtering/)).

**Fintech — Agents migration only after HITL stabilised**. A fintech that had been running a self-built LangChain agent loop for six months evaluated AgentCore migration and deferred for two more quarters — not because AgentCore was lacking, but because the team's HITL semantics for high-value transactions diverged from the AgentCore confirmation surface. Migrating before stabilising HITL would have forced a behaviour change on the user-facing confirmation flow. They wrote the deferral as an ADR with explicit re-evaluation triggers (HITL final ADR accepted, AgentCore custom-confirmation hook released) ([AgentCore product page, 2026-05-26](https://aws.amazon.com/bedrock/agentcore/)).

**E-commerce — OpenSearch managed for dual workload**. A retail SaaS that ran product-search on a self-managed cluster and AI-feature logs on a different platform consolidated onto OpenSearch Service managed once both workloads' cost was high enough to justify the convergence. The architectural win was the dual-workload fit — vector search for the recommender plus log analytics for the AIOps stack on one platform. The team's migration ADR named the convergence explicitly and committed to *not* using a separate platform for either workload going forward ([OpenSearch managed features, 2026-05-26](https://aws.amazon.com/opensearch-service/features/managed/)).

**Public-sector — staged carve-out for regulatory boundary**. A defence-modernisation engagement adopted Bedrock for InvokeModel from project start (GovCloud authorized) but deferred Knowledge Bases until the FedRAMP boundary documentation referenced the specific managed-service authorization. The carve-out was not technical — it was procedural. Once the boundary doc landed, Knowledge Bases turned on; until it did, self-managed RAG carried the load. The pattern keeps regulatory artifacts and engineering artifacts aligned ([AWS GovCloud RAG sample, 2026-05-26](https://github.com/aws-samples/sample-aws-bedrock-rag-govcloud-cdk-python)).

## 6. Best Practices

- **Treat managed-AI-service onboarding as gated, not opportunistic.** Each service turns on after the team has named cost attribution, alert thresholds, and audit trail for the actions it will take.
- **Choose silo vs pool tenancy explicitly.** Silo for strongest isolation in regulated workloads; pool with metadata filtering for cost efficiency at scale. The choice is an ADR, not a default.
- **Verify per-tenant cost attribution before cutover.** Migration without it is a finance black box; the lift to add tagging later is non-trivial.
- **Run parallel for at least one workload cycle before retiring the self-managed predecessor.** Shadow traffic is cheaper than rollback regret.
- **Treat agent-loop HITL fidelity as load-bearing.** Migrating to a managed agent harness whose confirmation surface diverges from your HITL semantics is a correctness risk, not a cosmetic one.
- **Re-verify cost and regulatory posture quarterly.** Both move quickly; an ADR that cites a quarter-old price or authorization is stale.
- **Use port-and-adapter when migration confidence is borderline.** Abstract the AI interface so the swap is configuration; reversibility pays back the abstraction cost.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** Pick a self-managed AI workload (real or invented: a self-managed RAG pipeline, a self-built agent loop, or a self-hosted vector store). Walk the §4 decision matrix:

1. Document the workload profile (volume, latency budget, multi-tenant, regulatory).
2. List the required-to-migrate signals.
3. List the stop-the-migration signals.
4. Pick a migration mode (full cutover / hybrid / port-and-adapter) and justify.
5. Write the rollback plan.

**What good looks like.** A finished page reads like a go/no-go gate, not a vendor pitch. Required signals are testable (a number, a measurement); stop signals are events not feelings. The migration mode is justified by the workload profile, not by enthusiasm. The rollback plan names the trigger and the action concretely. If you can't write a rollback plan, the migration isn't ready.

## 8. Key Takeaways

- *Why stage AWS managed-AI services rather than onboard them all at once?* Cost blast radius, multi-tenant policy, and regulatory boundary each force decisions that calcify after the service is live.
- *What does Bedrock Knowledge Bases commit when adopted?* The ingestion pipeline, the vector store choice, and a multi-tenant pattern (silo or pool with metadata filtering).
- *When is a managed-agent migration unsafe to make yet?* When your HITL semantics diverge from the managed harness's confirmation surface — that's a correctness risk, not a UX one.
- *What turns OpenSearch managed into a converging platform?* The dual workload — vector search for RAG plus log analytics for AIOps — on a FedRAMP/HIPAA-authorized backbone.

## Sources

1. [Foundation Models for RAG — Amazon Bedrock Knowledge Bases](https://aws.amazon.com/bedrock/knowledge-bases/) — retrieved 2026-05-26
2. [Multi-tenancy in RAG applications with metadata filtering (AWS ML Blog)](https://aws.amazon.com/blogs/machine-learning/multi-tenancy-in-rag-applications-in-a-single-amazon-bedrock-knowledge-base-with-metadata-filtering/) — retrieved 2026-05-26
3. [Amazon Bedrock AgentCore — product page](https://aws.amazon.com/bedrock/agentcore/) — retrieved 2026-05-26
4. [Amazon Bedrock Knowledge Bases now supports Amazon OpenSearch Service Managed Cluster as vector store](https://aws.amazon.com/blogs/machine-learning/amazon-bedrock-knowledge-bases-now-supports-amazon-opensearch-service-managed-cluster-as-vector-store/) — retrieved 2026-05-26
5. [Amazon OpenSearch Service — security and compliance features](https://aws.amazon.com/opensearch-service/features/security/) — retrieved 2026-05-26

Last verified: 2026-05-26
