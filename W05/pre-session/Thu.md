---
week: W05
day: Thu
title: "Pre-session — AIOps platform compare-matrix + AWS managed-service migration of acquire-gov"
audience: All cohort members
time_on_task_minutes: 55
last_verified: 2026-05-23
research_recency_windows: [hot-tech-3mo, federal-regulatory-6mo]
---

# W5 Thu Pre-Session — AIOps Platform Compare + AWS Managed-Service Migration

> Read **before** W5 Thu morning. ~55 min. Thu is the migration day — Bedrock Knowledge Base wraps `/rag/clause-search`, Agents-for-Bedrock wraps `POST /agent/intake-triage`. Morning is the AIOps platform compare-matrix workshop (consumes Wed's research).

## 1. The compare-matrix workshop — Thu morning shape (10 min)

Wed afternoon's `/web-research` produced 3 comparative ADRs per pair. Thu morning consolidates **W05-SA-1 (AIOps platform compare-set)** into a cohort-wide compare matrix on the whiteboard. Each pair presents one platform's research:

- **Pair owning Datadog** — presents the hands-on experience from Tue afternoon. Cohort has used this; presenter argues *for* the Datadog choice + the dimensions where it wins.
- **Pair owning Dynatrace Davis AI** — presents the causation-graph approach (Davis builds a topology graph; AI reasons over it for root-cause). Argues where Dynatrace would beat Datadog in `acquire-gov`'s shape.
- **Pair owning New Relic AI Monitoring** — presents the apdex-driven + per-transaction-AI-summary approach. Argues where NR's pricing or simplicity wins.
- **Coralogix AI Observability** is the 4th platform. Allocated to whichever pair has bandwidth (or instructor walks if pair count = 3).

The compare-matrix dimensions (decided Wed pair-research, locked Thu morning):

| Dimension | Datadog | Dynatrace | New Relic | Coralogix |
|-----------|---------|-----------|-----------|-----------|
| AI-native anomaly detection | Watchdog AI | Davis AI | Lookout / Errors Inbox AI | Streama AI |
| LLM observability product? | Yes (LLM Obs) | Limited | Yes (NR AI Monitoring) | Yes (AI Observability) |
| Token/cost-per-request first class? | Yes | Manual | Yes | Yes |
| Causation graph? | No (correlation) | Yes (causation) | No | No |
| FedRAMP authorization | High (GovCloud) | Moderate | Moderate | Moderate |
| Federal-pricing model | Per-host + ingest | Per-host (DPS) | Per-user + ingest | Per-GB ingest |
| Cohort hands-on | **Yes (Tue PM)** | No | No | No |

**The matrix is the deliverable.** Each row in the table has a 1-paragraph justification with `/web-research` citations.

[Datadog FedRAMP, retrieved 2026-05-23, https://www.datadoghq.com/security/]
[Dynatrace Davis AI overview, retrieved 2026-05-23, https://www.dynatrace.com/platform/artificial-intelligence/]
[New Relic AI Monitoring, retrieved 2026-05-23, https://newrelic.com/platform/ai-monitoring]
[Coralogix AI Observability, retrieved 2026-05-23, https://coralogix.com/ai-observability/]

## 2. AWS managed-service migration — what lands Thu afternoon (15 min)

Thu afternoon migrates two `acquire-gov` surfaces from custom implementation toward AWS managed equivalents. Each migration is **NOT a foregone conclusion** — the pair commits a *worth-it ADR* by EOD Thu.

### Migration 1 — `/rag/clause-search` to Bedrock Knowledge Bases

**Today's state.** `ai-orchestrator/app/rag/clause_search.py` runs hybrid RAG over FAR/DFARS clauses, using MongoDB Atlas Vector Search + a custom re-ranker. Item 7 (unused `pinecone-client` in `requirements.txt`) was removed in W2.

**Migrated state (candidate).** Bedrock Knowledge Bases manages: chunking + embedding + retrieval + citation surfacing. The cohort wraps the existing Atlas-backed corpus *or* migrates the corpus to an S3-backed Knowledge Base. Tradeoff:

| Dimension | Custom Atlas Vector Search | Bedrock Knowledge Bases |
|-----------|----------------------------|-------------------------|
| Chunking control | Full | Limited (Bedrock chooses) |
| Citation surface | Custom UX (W2 work) | Built-in citations API |
| Multi-tenant filter | `$vectorSearch` filter on `agency_id` | `filter` on KB metadata |
| FedRAMP boundary | Atlas not AWS-managed (D-050 carve-out N/A) | AWS-managed; clean GovCloud story |
| Vendor lock-in | MongoDB Atlas | AWS Bedrock |
| Debuggability when retrieval fails | High (custom code) | Lower (managed pipeline) |
| Latency | ~100ms (cached embedding) | ~200ms (KB-managed) |
| Cost | Atlas tier + Bedrock embed | Per-query + storage |

The pair's worth-it ADR commits one of: (A) full migration, (B) hybrid (KB for new corpora, Atlas for FAR/DFARS), (C) no migration (stay on Atlas; document why).

[Amazon Bedrock Knowledge Bases overview, retrieved 2026-05-23, https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html]

### Migration 2 — `POST /agent/intake-triage` to Agents-for-Bedrock

**Today's state.** W3 work landed a custom LangGraph multi-agent flow for proposal intake → triage → evaluator routing. Pair owns the state machine; HITL interrupts at handoffs.

**Migrated state (candidate).** Agents-for-Bedrock manages: action groups (tools), session state, HITL approval surfaces. The cohort exposes the existing solicitation-service + evaluation-service endpoints as **Bedrock action groups** (OpenAPI schema), and Bedrock handles agent orchestration.

| Dimension | Custom LangGraph | Agents-for-Bedrock |
|-----------|------------------|---------------------|
| HITL interrupt control | Full (interrupt nodes) | Built-in user-confirmation API |
| Tool authentication | Cohort-owned JWT propagation | Bedrock-managed (IAM-grounded) |
| Debug surface | LangGraph state inspection | CloudWatch + Bedrock traces |
| Vendor lock-in | LangChain v1.0 | AWS Bedrock |
| Re-targetable to non-AWS | Yes (rewrite LangGraph) | No (Bedrock-specific) |
| Cost shape | Per-LLM-call only | Per-LLM-call + agent orchestration markup |

The worth-it ADR considers: does HITL #7's authority decision align with Agents-for-Bedrock's confirmation surface? (Hint: depends on which A/B/C the pair chose Wed.)

[Amazon Bedrock Agents overview, retrieved 2026-05-23, https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html]

## 3. OpenSearch Managed — the third light-up (8 min)

D-050 also opens **OpenSearch Managed** today. The cohort doesn't *migrate* anything to it (Atlas owns vector + Postgres owns audit; no surface is OpenSearch-shaped yet). But the platform is now available for: (a) log aggregation if Datadog ingestion becomes cost-prohibitive, (b) future vector workloads outside FAR/DFARS, (c) the W6 deliverability runbook's "what if we needed to leave Datadog" section.

The Thu afternoon migration work doesn't touch OpenSearch — but the ADR catalog references it as the *third deferred AWS managed service that's now available*. A pair can stretch into OpenSearch for the Fri PR if they finish Resilience4j + audit-race fix + OTel by Thu EOD; not required.

[Amazon OpenSearch Service, retrieved 2026-05-23, https://aws.amazon.com/opensearch-service/]

## 4. Vendor lock-in vs FedRAMP boundary — the federal-context tension (10 min)

Both Thu migrations have the same shape of decision: managed = cleaner FedRAMP boundary (AWS does the SSP work), but tighter vendor lock-in. For a federal client this is **not a one-sided answer**:

- **Pro managed.** GSA / DLA / DoD agencies often *prefer* AWS-managed because the FedRAMP authorization boundary is smaller. The client's SSP gets thinner. The ATO process gets faster.
- **Anti managed.** Karsun's prime-contract value-add is *engineering depth* — knowing the system end-to-end. A pair that wraps everything in Bedrock managed loses some defensibility when the agency asks *why* something didn't work. The fix-PR generation pattern (AI-SRE pattern 4) needs the team to *understand* the failing surface.

The worth-it ADR must take a position on this tension — *for this specific endpoint*. Different endpoints can land differently. `/rag/clause-search` (read-heavy, well-defined) may migrate cleanly. `/agent/intake-triage` (HITL-heavy, evolving) may stay custom.

[GSA Cloud and Software Modernization, retrieved 2026-05-23, https://www.gsa.gov/technology/government-it-initiatives/cloud-computing]
[FedRAMP boundary guidance Rev 5, retrieved 2026-05-23, https://www.fedramp.gov/rev5/]

## 5. The Fri PR setup — what to land Thu EOD (8 min)

By EOD Thu your pair has:

- The compare-matrix (whiteboard photo committed to the pair repo).
- Two worth-it ADRs (Knowledge Bases migration; Agents-for-Bedrock migration).
- Migration code committed *or* a decision-not-to-migrate ADR + the equivalent work hardening the existing surface.
- A draft list of OWASP LLM Top 10 mitigations for Fri's PR (at minimum 3 categories — Wed's pre-reading flagged LLM05, LLM09, LLM10 as natural fits).

Fri morning is the Final Adversarial PR; Thu EOD is the last commit window before instructor + codex review.

## 6. What to skip if pair is behind (2 min)

If your pair is behind: skip the Agents-for-Bedrock migration; keep custom LangGraph; document the decision-not-to-migrate. The Fri PR rubric scores *engineering rigor*, not vendor adoption. A defensible "we kept the custom flow because we own the surface" beats a half-done managed migration.

---

## Sources (all retrieved 2026-05-23 via `/web-research`)

- Amazon Bedrock Knowledge Bases — https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html (hot-tech 3-month)
- Amazon Bedrock Agents — https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html (hot-tech 3-month)
- Amazon OpenSearch Service — https://aws.amazon.com/opensearch-service/ (hot-tech 3-month)
- Datadog FedRAMP — https://www.datadoghq.com/security/ (hot-tech 3-month)
- Dynatrace Davis AI — https://www.dynatrace.com/platform/artificial-intelligence/ (hot-tech 3-month)
- New Relic AI Monitoring — https://newrelic.com/platform/ai-monitoring (hot-tech 3-month)
- Coralogix AI Observability — https://coralogix.com/ai-observability/ (hot-tech 3-month)
- GSA Cloud Computing — https://www.gsa.gov/technology/government-it-initiatives/cloud-computing (federal-regulatory 6-month)
- FedRAMP Rev 5 — https://www.fedramp.gov/rev5/ (federal-regulatory 6-month)
- `pipeline/DECISIONS.md` D-050 + D-060 (in-repo, current)
- `training-project/feature-inventory-target.md` (in-repo)
