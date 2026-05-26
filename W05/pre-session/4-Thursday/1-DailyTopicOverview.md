---
week: W05
day: Thu
title: "Pre-session — Thursday: compare-matrix + AWS managed-service migration + governance documentation + Fri-PR pack"
audience: All cohort members
time_on_task_minutes: 65
last_verified: 2026-05-26
research_recency_windows: [hot-tech-3mo, federal-regulatory-6mo]
---

# W5 Thu Pre-Session — Pack the PR Backpack

> Read **before** W5 Thu morning. ~65 min. Thu is the **assessment-prep day**: morning consolidates Wed's research into a cohort-wide AIOps compare-matrix; afternoon migrates two `acquire-gov` surfaces toward AWS managed services and locks Fri PR scope. **Cohort #1 fold-in:** because Fri is the Final Adversarial Review PR (no pre-session), the four canonical W5-Fri pre-session topics — Drift Monitoring policy, Escalation Workflows, Deployment Planning, Incident-Response Runbook for AI-on-AI Failures — land here too. Thu = pack-the-backpack day.

## 1. AIOps Platform Compare-Matrix Workshop — Thu morning shape (10 min)

Wed afternoon's `/web-research` produced 3 comparative ADRs per pair. Thu morning consolidates **W05-SA-1 (AIOps platform compare-set)** into a cohort-wide compare-matrix on the whiteboard. Each pair presents one platform's research — including honest losses; loyalty to your assigned platform is not a virtue.

- **Pair owning Datadog** — presents the hands-on experience from Tue afternoon. Argues *for* the Datadog choice + the dimensions where it wins.
- **Pair owning Dynatrace Davis AI** — presents the causation-graph methodology (Davis builds a topology graph; AI reasons over it for root-cause). Argues where Dynatrace would beat Datadog on `acquire-gov`'s shape.
- **Pair owning New Relic AI Monitoring** — presents the apdex-driven + per-transaction-AI-summary approach. Argues where NR's pricing or simplicity wins.
- **Coralogix AI Observability** is the 4th platform. Allocated to whichever pair has bandwidth (or instructor walks if pair count = 3).

The dimensions (locked Thu morning): AI-native anomaly · LLM observability first-class · token/cost-per-request · causation graph · FedRAMP authorization · federal-pricing model · cohort hands-on familiarity · federal market presence · OTel-friendliness.

**The matrix is the deliverable.** Each row carries a 1-paragraph justification with `/web-research` citations.

[Datadog FedRAMP, retrieved 2026-05-26, https://www.datadoghq.com/security/]
[Dynatrace Davis AI overview, retrieved 2026-05-26, https://www.dynatrace.com/platform/artificial-intelligence/]
[New Relic AI Monitoring, retrieved 2026-05-26, https://newrelic.com/platform/ai-monitoring]
[Coralogix AI Observability, retrieved 2026-05-26, https://coralogix.com/ai-observability/]

## 2. AWS Managed-Service Migration 1 — Bedrock Knowledge Bases on `/rag/clause-search` (10 min)

**Today's state.** `ai-orchestrator/app/rag/clause_search.py` runs hybrid RAG over FAR/DFARS clauses, using MongoDB Atlas Vector Search + a custom re-ranker. Item 7 (unused `pinecone-client`) was removed in W2.

**Migrated state (candidate).** Bedrock Knowledge Bases manages: chunking + embedding + retrieval + citation surfacing. The pair either wraps the existing Atlas-backed corpus or migrates the corpus to an S3-backed KB.

| Dimension | Custom Atlas Vector Search | Bedrock Knowledge Bases |
|-----------|----------------------------|-------------------------|
| Chunking control | Full | Limited (Bedrock chooses) |
| Citation surface | Custom UX (W2 work) | Built-in citations API |
| Multi-tenant filter | `$vectorSearch` filter on `agency_id` | `filter` on KB metadata |
| FedRAMP boundary | Atlas not AWS-managed (D-050 carve-out N/A) | AWS-managed; clean GovCloud story |
| Vendor lock-in | MongoDB Atlas | AWS Bedrock |
| Debuggability on retrieval failure | High (custom code) | Lower (managed pipeline) |
| Latency | ~100ms (cached embedding) | ~200ms (KB-managed) |
| Cost | Atlas tier + Bedrock embed | Per-query + storage |

The worth-it ADR commits one of: (A) full migration, (B) hybrid (KB for new corpora, Atlas for FAR/DFARS), (C) no migration (stay on Atlas; document why). **Item 10 (`agency_id` multi-tenant filter) MUST preserve** in any migration — non-negotiable.

[Amazon Bedrock Knowledge Bases, retrieved 2026-05-26, https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html]

## 3. AWS Managed-Service Migration 2 — Agents-for-Bedrock on `/agent/intake-triage` (8 min)

**Today's state.** W3 work landed a custom LangGraph multi-agent flow for proposal intake → triage → evaluator routing. The pair owns the state machine; HITL interrupts at handoffs (HITL #5).

**Migrated state (candidate).** Agents-for-Bedrock manages: action groups (tools), session state, user-confirmation surfaces. The pair exposes the existing solicitation-service + evaluation-service endpoints as **Bedrock action groups** (OpenAPI schema); Bedrock handles agent orchestration.

| Dimension | Custom LangGraph | Agents-for-Bedrock |
|-----------|------------------|---------------------|
| HITL interrupt control | Full (interrupt nodes) | Built-in user-confirmation API |
| Tool authentication | Cohort-owned JWT propagation | Bedrock-managed (IAM-grounded) |
| Debug surface | LangGraph state inspection | CloudWatch + Bedrock traces |
| Vendor lock-in | LangChain v1.0 | AWS Bedrock |
| Re-targetable to non-AWS | Yes (rewrite LangGraph) | No (Bedrock-specific) |
| Cost shape | Per-LLM-call only | Per-LLM-call + agent-orchestration markup |

The worth-it ADR asks: **does HITL #7's authority decision (Wed) align with Agents-for-Bedrock's user-confirmation surface?** Hint: depends on which A/B/C the pair chose Wed. A "Full Auto" Wed decision plus a "user-confirmation required" Bedrock surface = mismatch worth surfacing in the ADR.

[Amazon Bedrock Agents, retrieved 2026-05-26, https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html]

## 4. Audit Logging + Responsible AI Practices — accountability as a lens, logged as a record (14 min)

PDF-Thu canonical topic (merged). Responsible AI and audit logging are *one* discipline, not two: the lens (responsible AI) decides what gets reviewed at each engineering decision; the record (audit log) is the artifact that proves the review actually happened and lets a future auditor reconstruct what the system did. Treating them separately is the standard mistake — without an immutable record naming the human responsible, every claim about accountability is decorative; without a lens that says *what we are willing to be accountable for*, the audit table is a write-only data lake nobody reviews.

**The lens (Responsible AI).** Not a separate workstream — it's the discipline applied at every engineering decision. For `acquire-gov`'s federal context, the lens has four planes that map directly to Fri PR rubric dimensions:

1. **Accountability (FedRAMP AC-2 + AU-9).** Every model decision is traceable to a tenant + a user + a span. Item 6 (OTel) and the audit-logging mechanics below are the engineering surface. The Wed HITL #7 ADR is the policy.
2. **Fairness + non-discrimination.** Multi-tenant boundary integrity (Item 10) — no tenant's data leaks into another's retrieval or agent context. The Bedrock KB metadata filter (§2) and the Wed `agency_id`-in-prompt audit are the mechanics.
3. **Transparency.** Citation surfacing on `/rag/clause-search`, structured-output schemas on the 4 AI endpoints (LLM05), and the Fri PR's OWASP LLM mitigation set make outputs inspectable.
4. **Safety + human oversight.** §6 escalation workflows + Wed HITL #7 + circuit-breaker on Item 3 ensure failure modes route to humans rather than amplify silently.

**The record (Audit Logging).** The frame: **AIOps doesn't get a silent action.** Every Watchdog-style auto-detection, every drift-policy alert, every managed-service migration, every Wed-authority-bounded auto-remediation — each one writes an `AuditEvent` row. The audit-of-the-audit rule from Wed §5 generalises here. For `acquire-gov`:

- **AIOps detections** route to `AuditEvent.action = "AIOPS_DETECT"` with the raw signal + the trace ID that triggered it. Item 6 (correlation IDs) now lets the audit row pin to the originating user request.
- **Auto-remediation actions** (whichever authority level Wed locked) route to `AuditEvent.action = "AIOPS_REMEDIATE_{A|B|C}"` with the proposed fix payload + the approval state. Even Wed's "Full Auto" decision logs the human who *defined the policy* as `actor` — never the AI as `actor`.
- **Managed-service migration cutovers** (Thu afternoon's KB / Agents migrations) write a one-shot `AuditEvent.action = "MIGRATION_CUTOVER"` with the from/to surface and the ADR ID. The FedRAMP boundary change is itself an audit-relevant event.

FedRAMP AU-9 (audit-integrity) requires every system-of-record action be reconstructable from the audit table. AIOps doesn't get an exemption — it *adds* sources of audit-relevant action.

**Governance Documentation umbrella.** Topics 1, 2, 3, 4, and §5/§6 below all produce documents the cohort delivers to the client: the compare-matrix, the two worth-it ADRs, the Wed HITL #7 ADR, the drift-monitoring policy, the escalation workflow, and the AI-on-AI runbook section. These ARE the governance documentation deliverable — output of the lens-driven work, not substitute for it.

[FedRAMP Rev 5 AU controls, retrieved 2026-05-26, https://www.fedramp.gov/rev5/]
[OpenTelemetry semantic conventions for `gen_ai.*`, retrieved 2026-05-26, https://opentelemetry.io/docs/specs/semconv/gen-ai/]
[NIST AI Risk Management Framework, retrieved 2026-05-26, https://www.nist.gov/itl/ai-risk-management-framework]
[OWASP LLM Top 10 for 2025, retrieved 2026-05-26, https://owasp.org/www-project-top-10-for-large-language-model-applications/]

## 5. Drift Monitoring policy — committed-to-prod thresholds + alert routing (6 min)

**PDF-Fri fold-in.** This closes the Tue→Wed→Thu drift arc:

- **Tue** instrumented drift signals — `gen_ai.usage.*` token deltas, retrieval-k distribution on `/rag/clause-search`, multi-agent fan-out counts.
- **Wed war-room** *responded* to a drift event (Item 2 audit-gap from a Watchdog detection) and locked HITL #7 authority.
- **Thu** writes the **drift-monitoring policy** — the committed-to-prod artifact the Fri PR includes.

The policy commits, per AI endpoint:
- **Threshold.** What deviation triggers an alert? (e.g., `/rag/clause-search` k=0 rate >5% over 1h window; `/agent/intake-triage` token cost p95 >2× baseline; `/answer-qa` citation-mismatch rate >3%.)
- **Alert routing.** Which signal pages on-call? Which writes to a dashboard only? Which auto-tickets in `acquire-gov`'s issue tracker?
- **Authority.** Which threshold breaches trigger Wed-authority-bounded auto-remediation (e.g., circuit-breaker trip on Item 3 retry-storm), versus paging a human?
- **Drift-source tagging.** Is this drift from data (corpus change), model (Bedrock version bump), or traffic (new agency tenant onboarded)? The policy names the diagnostic step before remediation.

For Cohort #1: Thu is the **policy form** of drift. Wed was the **response form** (one incident). The Fri PR contains both — the Wed ADR (one decision, one incident) and the Thu policy (the standing rules that decide all future incidents).

[Datadog Watchdog AI, retrieved 2026-05-26, https://docs.datadoghq.com/watchdog/]
[Amazon Bedrock model versioning, retrieved 2026-05-26, https://docs.aws.amazon.com/bedrock/latest/userguide/model-lifecycle.html]

## 6. Human Oversight — escalation workflows as HITL #7's operational artifact (6 min)

**PDF-Thu + PDF-Fri fold-in.** Wed's HITL #7 ADR locked the **authority decision** — who can act, on what data, with what reversibility. Thu writes the **escalation workflow** — the operational artifact that makes the authority decision real on-call.

The escalation workflow commits:
- **Tier 1 (auto-handled within authority).** Threshold breach → Wed-authority-bounded action (e.g., circuit-breaker trip; audit-replay if pair chose A). Logged to `AuditEvent` per §4. No human paged.
- **Tier 2 (page on-call human within SLA).** Breach exceeds auto-handled scope, OR repeated auto-handled actions within window. Page routes to `acquire-gov` on-call rotation; SLA = 15 min ack.
- **Tier 3 (escalate to sys_admin + agency contracting officer).** FedRAMP-boundary event, multi-tenant data leak signal, or audit-integrity event. SLA = immediate; agency notification per contract.
- **Audit handoff.** Every tier writes the originating trace ID + the AIOps signal source + the chosen action + the human ack (Tier 2/3) to `AuditEvent`. The audit-of-the-audit rule applies to escalations themselves.

For Cohort #1: the Wed ADR alone is not enough; without this workflow the authority decision has no operational substrate. The Fri PR includes both — the ADR (policy) and the workflow (mechanics).

[FedRAMP Rev 5 IR controls (Incident Response), retrieved 2026-05-26, https://www.fedramp.gov/rev5/]
[PagerDuty incident severity guidance, retrieved 2026-05-26, https://www.pagerduty.com/resources/learn/what-is-incident-severity/]

## 7. Deployment Planning + Incident-Response Runbook for AI-on-AI Failures (8 min)

**PDF-Fri fold-in (combined).** Two related Fri-canonical items merged because they answer the same question: *what does shipping this AIOps stack to prod actually take, and what happens when the stack itself fails?*

### 7a. Deployment Planning — what shipping the AIOps stack to prod takes

D-050 also opens **OpenSearch Managed** today. The cohort doesn't migrate to it (Atlas owns vector; Postgres owns audit; no surface is OpenSearch-shaped yet), but it's the **deferred third managed service** — available for log aggregation if the chosen observability vendor's ingestion becomes cost-prohibitive, future vector workloads outside FAR/DFARS, and the W6 deliverability runbook's "what if we needed to leave the current observability platform" section. A pair can stretch into OpenSearch for the Fri PR if they finish Items 2/3/6 + KB/Agents ADRs by Thu EOD; not required.

Beyond OpenSearch, the **deployment-planning artifact** commits:
- **FedRAMP boundary call** — which `acquire-gov` surfaces remain in-boundary; which managed services (KB / Agents / OpenSearch) thin the SSP; what the GovCloud regional split looks like.
- **Log retention policy** — observability-vendor ingest retention + AuditEvent table retention (FedRAMP AU-11 minimum) + Bedrock model-invocation log retention.
- **Alerting-as-code** — drift-monitoring policy (§5) committed as platform-native alerting config (e.g., Datadog monitor JSON, Dynatrace settings, Prometheus AlertManager rules, Grafana alerting YAML, or Terraform wrapping any of those) — not console-clicked. The Fri PR carries the config. **Vendor-neutrality note:** the structural discipline (thresholds, escalation paths, alert grouping, severity routing) is what the rubric scores. If the cohort adopts a platform other than the Tue hands-on one, substitute the platform's native DSL.
- **Cutover sequencing** — Item 3 circuit-breaker before KB cutover (KB latency is higher; circuit-breaker prevents amplification); Item 2 audit-race fix before Agents cutover (Agents tool-calls each write audit rows; race surface widens otherwise).

> [!instructor-review]
> Datadog monitor JSON is one example of alerting-as-code; same structural shape applies to Dynatrace, Prometheus AlertManager, Grafana alerting. If the cohort adopts a different platform (Tue hands-on may rotate between cohorts), substitute the platform's native DSL — the *structural* discipline (thresholds, escalation paths, alert grouping) is what matters for the Fri rubric, not the vendor.

### 7b. Incident-Response Runbook for AI-on-AI Failures

The load-bearing Fri PR deliverable: a runbook section in the pair-project repo covering **what happens when the AI-SRE pattern itself fails**. Distinct from generic incident response (that's W6 territory); AI-on-AI failure is the W5 anchor surface.

Failure modes to runbook:
- **Anomaly-detector false-positive cascade** — anomaly detector (Watchdog, Davis, Streama, or equivalent) flags a benign deploy as drift; auto-remediation (if A/B authority) starts replaying audit events that don't need replay. Runbook: kill-switch on the auto-remediation surface; manual rollback procedure; audit-of-the-audit reconciliation script.
- **Drift-policy misfire** — threshold tuned too aggressively in §5; on-call gets paged every 4 minutes. Runbook: threshold-revision procedure (Tier 2 escalation revises the policy; ADR amendment required); paging-suppression window with audit log.
- **Bedrock managed-service degradation** — KB ingestion stuck; Agents action-group invocations 5xx. Runbook: failover to retained custom implementation (the "hybrid" worth-it ADR choice now earns its keep); incident class = FedRAMP IR-4.
- **Audit-of-the-audit recursion** — the AuditEvent table itself becomes the source of an AIOps signal that itself writes to AuditEvent. Runbook: rate-limit; debounce window; manual audit-chain verification.

The runbook section is the Fri PR's load-bearing deliverable per `assessments/W05-final-adversarial-pr-rubric.md`. Source patterns from `skills/aiops-curriculum/references/ai-sre-patterns.md` §5 (Runbook Augmentation).

[Amazon OpenSearch Service, retrieved 2026-05-26, https://aws.amazon.com/opensearch-service/]
[FedRAMP Rev 5 IR-4 (Incident Handling), retrieved 2026-05-26, https://www.fedramp.gov/rev5/]
[GSA Cloud and Software Modernization, retrieved 2026-05-26, https://www.gsa.gov/technology/government-it-initiatives/cloud-computing]

---

## Closing checklist (not numbered topics)

**Vendor lock-in vs FedRAMP boundary** — the federal-context tension under Topic 4's Responsible AI lens. Managed = cleaner FedRAMP boundary (AWS does SSP work, ATO faster) but tighter vendor lock-in. Karsun's prime-contract value-add is engineering depth — defensibility when the agency asks *why* something didn't work. Different endpoints land differently: `/rag/clause-search` (read-heavy, well-defined) may migrate cleanly; `/agent/intake-triage` (HITL-heavy, evolving) may stay custom. The worth-it ADR takes a position **per endpoint**.

**Fri PR setup — what to land Thu EOD.** Compare-matrix committed (whiteboard photo). Two worth-it ADRs (KB + Agents). Migration code committed OR decision-not-to-migrate ADR + hardening work. Drift-monitoring policy (§5) committed. Escalation workflow (§6) committed. AI-on-AI runbook section drafted. OWASP LLM Top 10 mitigation list (≥3 categories — LLM05/LLM09/LLM10 are natural fits per Wed's continuation).

**What to skip if pair is behind.** Skip the Agents-for-Bedrock migration; keep custom LangGraph; commit decision-not-to-migrate. The Fri rubric scores *engineering rigor*, not vendor adoption — a defensible "we kept the custom flow because we own the surface" beats a half-done managed migration. Don't skip drift-policy, escalation workflow, or AI-on-AI runbook — they're load-bearing Fri PR artifacts regardless of migration choice.

**Tomorrow.** Fri = Final Adversarial Review PR (v2's replacement for v1's Exit Technical Exam). No pre-session. Read the rubric tonight: `assessments/W05-final-adversarial-pr-rubric.md`.

---

## Sources (all retrieved 2026-05-26 via `/web-research`)

- Amazon Bedrock Knowledge Bases — https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html (hot-tech 3-month)
- Amazon Bedrock Agents — https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html (hot-tech 3-month)
- Amazon Bedrock model versioning — https://docs.aws.amazon.com/bedrock/latest/userguide/model-lifecycle.html (hot-tech 3-month)
- Amazon OpenSearch Service — https://aws.amazon.com/opensearch-service/ (hot-tech 3-month)
- Datadog FedRAMP — https://www.datadoghq.com/security/ (hot-tech 3-month)
- Datadog Watchdog AI — https://docs.datadoghq.com/watchdog/ (hot-tech 3-month)
- Dynatrace Davis AI — https://www.dynatrace.com/platform/artificial-intelligence/ (hot-tech 3-month)
- New Relic AI Monitoring — https://newrelic.com/platform/ai-monitoring (hot-tech 3-month)
- Coralogix AI Observability — https://coralogix.com/ai-observability/ (hot-tech 3-month)
- OpenTelemetry semantic conventions for `gen_ai.*` — https://opentelemetry.io/docs/specs/semconv/gen-ai/ (foundation 12-month)
- OWASP LLM Top 10 for 2025 — https://owasp.org/www-project-top-10-for-large-language-model-applications/ (hot-tech 3-month — 2025 list current; OWASP refreshes every 1–2 years)
- NIST AI Risk Management Framework — https://www.nist.gov/itl/ai-risk-management-framework (federal-regulatory 6-month)
- FedRAMP Rev 5 (AU + IR controls) — https://www.fedramp.gov/rev5/ (federal-regulatory 6-month)
- GSA Cloud and Software Modernization — https://www.gsa.gov/technology/government-it-initiatives/cloud-computing (federal-regulatory 6-month)
- PagerDuty incident severity guidance — https://www.pagerduty.com/resources/learn/what-is-incident-severity/ (foundation 12-month)
- `pipeline/DECISIONS.md` D-040, D-043, D-046, D-049, D-050, D-060 (in-repo, current)
- `skills/aiops-curriculum/references/ai-sre-patterns.md` §5 + §6 (in-repo)
- `skills/codex-adversarial-review/references/owasp-llm-top-10.md` (in-repo curated)
- `training-project/feature-inventory-target.md` Items 2, 3, 6, 10 (in-repo)
- `assessments/W05-final-adversarial-pr-rubric.md` (in-repo)

Last verified: 2026-05-26
