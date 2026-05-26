---
week: W05
day: Thu
topic_slug: audit-logging-and-responsible-ai
topic_title: "Audit Logging + Responsible AI Practices — accountability as a lens, logged as a record"
parent_overview: W05/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 18
sources:
  - url: https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final
    retrieved_on: 2026-05-26
    recency_category: federal-regulatory-6mo
  - url: https://opentelemetry.io/docs/specs/semconv/gen-ai/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://www.nist.gov/itl/ai-risk-management-framework
    retrieved_on: 2026-05-26
    recency_category: federal-regulatory-6mo
  - url: https://airc.nist.gov/airmf-resources/airmf/
    retrieved_on: 2026-05-26
    recency_category: federal-regulatory-6mo
  - url: https://owasp.org/www-project-top-10-for-large-language-model-applications/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.iso.org/standard/81230.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://www.fedramp.gov/
    retrieved_on: 2026-05-26
    recency_category: federal-regulatory-6mo
last_verified: 2026-05-26
---

# Audit Logging + Responsible AI Practices — Accountability as a Lens, Logged as a Record

## 1. Learning Objectives

By the end of this reading, the learner can:

- Explain why responsible AI is a **lens** (applied to every engineering decision) rather than a **workstream** (a separate review committee) — and why the lens-driven shape is what makes the audit record a meaningful artifact rather than a ceremonial one.
- Identify the four fields any defensible AuditEvent record must carry (actor, action, target, timestamp + provenance) and the AU-3 extensions on top of them.
- Distinguish between the operational log (debug/observability) and the audit log (compliance/reconstruction), and explain why the two cannot share a store.
- Articulate the "actor is the human who defined the policy" principle for AI-triggered actions, and connect that principle to NIST AI RMF's *accountable and transparent* characteristic.
- Map the four operational properties of trustworthy AI (accountability, fairness, transparency, safety + oversight) onto concrete engineering surfaces — audit logging being the load-bearing one for accountability.
- Recognise that governance documentation is the *output* of lens-driven engineering work, not a substitute for it.

## 2. Introduction

Responsible AI and audit logging are usually filed under different headers. Treating them separately is the standard mistake. The audit log is what makes the trustworthy-AI lens *enforceable*: without an immutable record naming the human responsible, every claim about accountability is decorative. Without a lens that says *what we are willing to be accountable for*, the audit table is a write-only data lake nobody reviews.

This reading covers both as a single discipline because they are inseparable in practice. The lens (responsible AI) decides what gets reviewed at each engineering decision; the record (audit log) is the artifact that proves the review actually happened and lets a future auditor reconstruct what the system did.

"Responsible AI" became a phrase before it became a practice. Vendor websites carry responsible-AI sections; conference talks invoke it; regulators reference it. The risk is that the phrase becomes ceremonial — a sticker on a product, a slide in a deck, a committee that meets quarterly but does not interrupt the engineering work. The framing that makes it operational: **responsible AI is a lens, not a workstream.** Every engineering decision — what to log, what filter to attach, what error to show the user, what threshold to alert on — gets reviewed through the lens. The lens is articulated in standards: NIST's AI Risk Management Framework (AI RMF) lists seven characteristics of trustworthy AI [NIST AI RMF, retrieved 2026-05-26]; ISO/IEC 42001 specifies an AI management system [ISO/IEC 42001:2023, retrieved 2026-05-26]; OWASP's Top 10 for LLM Applications enumerates the threat surface [OWASP LLM Top 10, retrieved 2026-05-26].

Audit logging is where the *accountable and transparent* characteristic of trustworthy AI lands as a concrete engineering surface. When AIOps enters the picture, audit log scope expands. An anomaly detector that fires a runbook is an actor. A drift-policy alert that pages an on-call human is an actor. A managed-service migration that re-routes traffic is an actor. None of these get a silent path — every one of them produces audit-relevant action, and every one of them must produce an AuditEvent that can be reconstructed later. *AIOps doesn't get an exemption. AIOps adds sources of audit-relevant action.* NIST SP 800-53 Rev. 5's AU control family (Audit and Accountability) — particularly AU-2 (Event Logging), AU-3 (Content of Audit Records), AU-9 (Protection of Audit Information), and AU-11 (Audit Record Retention) — applies whether the actor is a human or an automated agent [NIST SP 800-53 Rev. 5, retrieved 2026-05-26].

## 3. Core Concepts

### 3.1 The seven characteristics of trustworthy AI (NIST AI RMF), compressed to four operational properties

NIST AI RMF 1.0 names seven characteristics that a trustworthy AI system should exhibit [NIST AI RMF Knowledge Base, retrieved 2026-05-26]:

1. **Valid and reliable** — the AI does what it is supposed to, robustly.
2. **Safe** — operation does not lead to physical harm, system harm, or other unacceptable risks.
3. **Secure and resilient** — the AI withstands attacks and recovers from disruption.
4. **Accountable and transparent** — there is a clear chain of responsibility and a visible decision surface.
5. **Explainable and interpretable** — outputs can be understood by appropriate stakeholders.
6. **Privacy-enhanced** — personal and sensitive data are protected.
7. **Fair, with managed harmful bias** — bias is identified, measured, and mitigated.

These are not model properties. They are *application* properties. A model that is "fair" in isolation can still produce an unfair application if the surrounding code routes its outputs differently for different user groups. A model that is "explainable" in a research notebook can produce an opaque application if the UI does not surface explanations to users. Responsible AI applies at the application boundary, not the model boundary.

A pragmatic shortlist compresses NIST's seven into four engineering-actionable buckets that map directly onto concrete surfaces:

- **Accountability.** Every model decision is traceable to a tenant, a user, and a span. Audit logging (§3.2–§3.5 below) is the mechanism. Policies that authorise model actions name the human responsible.
- **Fairness and non-discrimination.** Multi-tenant boundary integrity (no tenant's data leaks into another's retrieval); per-cohort behaviour evaluated rather than aggregate behaviour. Metadata filters in retrievers, group-disaggregated evaluation in eval pipelines.
- **Transparency.** Citation surfacing on retrieval-augmented endpoints; structured-output schemas where claim-level inspection matters; user-facing surface that distinguishes AI-generated content from non-AI content.
- **Safety and human oversight.** Escalation workflows; HITL touchpoints at decision points where reversibility is bounded; circuit-breakers on auto-remediation paths.

Accountability is the load-bearing one for this reading — the next four subsections drill into how the AuditEvent record makes it real.

### 3.2 What an AuditEvent record carries

The minimal field set for an AuditEvent that survives forensic review:

- **Actor.** Who or what triggered the action. For human-triggered actions, a user identifier. For scheduled actions, the system identifier and the schedule that fired it. For AI-triggered actions, see §3.5 — the actor is the human who defined the policy, never the AI.
- **Action.** A typed identifier of what was attempted. `LOGIN`, `RECORD_UPDATE`, `MIGRATION_CUTOVER`, `AIOPS_REMEDIATE_CIRCUIT_BREAKER`. The vocabulary is finite and reviewed.
- **Target.** What the action was performed on. Record IDs, resource ARNs, configuration items.
- **Timestamp + provenance.** The action time (server clock, monotonic if possible), plus a correlation identifier — a trace ID, a request ID — that lets the audit row pin back to the originating signal.

NIST AU-3 lists eight required content items: type of event, when, where, source, outcome, identity of subjects, and (where applicable) identity of objects affected [NIST SP 800-53 Rev. 5, retrieved 2026-05-26]. The four-field model above is the operational shorthand; the full AU-3 list is the regulatory minimum.

### 3.3 Why operational log and audit log are not the same store

Operational logs:
- Are written by engineers for engineers.
- Sample, rotate, and may be truncated.
- Live in a system optimised for query speed (observability vendor — Datadog, Splunk, Elasticsearch, Honeycomb).
- Have access controls keyed to who can debug the system.

Audit logs:
- Are written by the application for auditors.
- Are append-only; rotation is allowed but mutation is not.
- Live in a system optimised for integrity and retention (a relational database with no UPDATE/DELETE permission, or a write-once object store).
- Have access controls keyed to who can read audit records (typically a small set, separate from engineering).

Putting them in the same store collapses the access-control story: anyone who can debug now also has access to the regulator-facing record, which violates separation-of-duties principles and AU-9 (Protection of Audit Information) [NIST SP 800-53 Rev. 5, retrieved 2026-05-26].

### 3.4 Correlation IDs let the audit row pin to the trace

When an AuditEvent is written, it should carry a correlation identifier — the trace ID of the originating request — so the audit row links back to the operational trace that produced it. OpenTelemetry's general semantic conventions already encode trace ID and span ID; for GenAI workloads, the dedicated `gen_ai.*` conventions add `gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.usage.*` fields that let an audit row reconstruct *which model invocation* led to the action [OpenTelemetry GenAI semantic conventions, retrieved 2026-05-26].

The shape that works:

- Operational trace lives in the observability backend (vendor-neutral: Datadog, Honeycomb, Grafana Tempo, etc.).
- Audit row lives in the audit datastore.
- The two are joined by trace ID at incident-response time, not pre-joined at write time.

This keeps the audit datastore from depending on the observability vendor while preserving forensic reconstructability.

### 3.5 The "actor is the human who defined the policy" principle

When an automated action fires — an anomaly detector that flips a circuit breaker, an agent that schedules a payment, a drift policy that paged on-call — the AuditEvent's `actor` field is *not* the model or the rule. It is the human (or, more precisely, the human-identifiable role) that authorised the action.

Concretely:

- **Scheduled job.** Actor = the service account or the human who configured the schedule.
- **AI-detected anomaly with auto-remediation.** Actor = the human who defined the policy that grants the AI permission to remediate within scope. The model is captured separately as `actor.delegated_to = "<model id>"` or in an equivalent metadata field, but it is not the responsible party.
- **AI-detected anomaly with human approval before remediation.** Actor = the approving human. The detection event is its own AuditEvent with `actor = <policy-owner>`; the remediation event has `actor = <approver>`.

This principle survives audit because the chain of accountability points back to people: the policy author, the approver, the policy owner. An audit-row that says `actor = "the model"` cannot be subpoenaed. An audit-row that says `actor = "<policy-owner>" delegated_to = "<model id>"` can. This is the *accountable and transparent* characteristic from NIST AI RMF rendered as a single field on a single row.

### 3.6 Audit-of-the-audit, plus the OWASP LLM Top 10 as a threat lens

The recursive case: when an AIOps action *writes* to the audit table, that write is itself an audit-relevant action. Most teams handle this by:

- Treating audit-table writes as observable but not themselves auditable (the write goes into the audit table; it does not produce a second audit row about producing the first).
- Putting integrity controls (cryptographic hash chain, write-once storage, separate-account ownership) on the audit table such that tampering after the write is detectable. NIST AU-9 explicitly calls for protecting audit information against unauthorised access, modification, and deletion [NIST SP 800-53 Rev. 5, retrieved 2026-05-26].

The trap to avoid: an AIOps signal that detects an "anomaly" in the audit table itself and triggers another audit-table write, producing a feedback loop. The runbook for that case is rate-limiting and a manual chain-verification step (see Thursday topic 8 §3.3.4 for the runbook entry).

OWASP's Top 10 for LLM Applications enumerates the threat surface [OWASP LLM Top 10, retrieved 2026-05-26]. The current list includes prompt injection (LLM01), sensitive information disclosure (LLM02), supply-chain risks (LLM03), data and model poisoning (LLM04), improper output handling (LLM05), excessive agency (LLM06), and others. The threat lens overlays the trustworthy-AI lens: the trustworthy-AI list says "what good looks like"; the OWASP list says "what attackers will do." A responsible-AI practice runs both at the same review. An accountability requirement (trustworthy-AI) combined with a supply-chain risk (OWASP LLM03) produces a concrete engineering decision: pin model versions, verify checksums, audit-log model updates. Neither lens alone would have produced that decision; both together did.

### 3.7 Governance versus engineering, and why documentation is output not substitute

A governance decision is one that sets policy: who is allowed to do what, under which conditions, with what authority. An engineering decision is one that implements that policy: where the code-level check lives, what threshold the alert fires at, how the audit row is written.

The lens-driven workflow treats them as adjacent, not separate:

- The governance decision is captured in an ADR or policy document with named owners.
- The engineering decision implements the policy; the implementation references the policy by ID.
- When the implementation is questioned, the chain runs implementation → policy → governance owner. The chain is short.

The deliverable that responsible-AI practice produces is documentation: ADRs that link to policies, eval reports that link to ADRs, runbooks that reference both. Documentation is the *output* of the lens-driven work; it is what the work produces. A team that writes documentation without doing the engineering work has done nothing; a team that does the engineering work without documenting it has done the work but cannot prove it.

The documents most teams produce, in rough order of frequency:

- **Per-endpoint mitigation table.** For each AI endpoint, which OWASP LLM categories are mitigated, by which controls.
- **Authority policy.** For each automated action, who authorised it and with what scope.
- **Drift-monitoring policy** and **escalation workflow** (covered in adjacent topic readings).
- **Eval report.** Group-disaggregated metrics, citation-correctness rates, hallucination rates over time.
- **Model-version audit trail.** What model version was in production when, who approved the change, what eval gated the change.

## 4. Generic Implementation

A minimum-viable AuditEvent writer, with the four-field schema plus AU-3 extensions, in vendor-neutral pseudocode:

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from enum import Enum
from typing import Optional

class AuditAction(str, Enum):
    """Closed vocabulary of audit actions. Add new actions here only via review."""
    LOGIN = "LOGIN"
    RECORD_UPDATE = "RECORD_UPDATE"
    MIGRATION_CUTOVER = "MIGRATION_CUTOVER"
    AIOPS_DETECT = "AIOPS_DETECT"
    AIOPS_REMEDIATE_BOUNDED = "AIOPS_REMEDIATE_BOUNDED"
    AIOPS_PAGE = "AIOPS_PAGE"
    LLM_CALL_FAILED = "LLM_CALL_FAILED"

@dataclass(frozen=True)
class AuditEvent:
    """Immutable audit-record DTO. Audit table has no UPDATE/DELETE permission."""

    # Required fields — the four-field minimum.
    actor: str                       # Human or service-account identifier; never a model name.
    action: AuditAction              # Typed action from the closed vocabulary.
    target: str                      # Resource being acted on (URN, ARN, record ID).
    occurred_at: datetime            # UTC timestamp at the action site.

    # Provenance — lets the audit row join back to operational trace.
    trace_id: str                    # OpenTelemetry trace ID.
    span_id: Optional[str] = None

    # AU-3 extensions.
    outcome: str = "SUCCESS"         # SUCCESS | FAILURE | DENIED.
    source_ip: Optional[str] = None
    request_id: Optional[str] = None

    # AI delegation, if applicable. The actor is still the human policy-owner.
    delegated_to: Optional[str] = None    # e.g., "model:claude@2026-05-15".
    policy_id: Optional[str] = None       # ID of the policy that authorised the delegation.

def write_audit_event(event: AuditEvent, repo) -> None:
    """Write an AuditEvent to the audit repository.

    The repository's contract is: INSERT only. UPDATE and DELETE are not permitted by the
    database role bound to the writer. Reads are mediated by an audit-access role that is
    separate from the writer role.
    """
    # The repo's `insert` is the only mutation. There is no `update` or `delete` method.
    repo.insert(event)

# Example: an AI-detected anomaly leading to an auto-remediation.
write_audit_event(
    AuditEvent(
        actor="user:policy-owner@example.com",
        action=AuditAction.AIOPS_REMEDIATE_BOUNDED,
        target="resource:cache-cluster-prod-east-1",
        occurred_at=datetime.now(timezone.utc),
        trace_id="0af7651916cd43dd8448eb211c80319c",
        outcome="SUCCESS",
        delegated_to="model:anomaly-detector@2026-05-12",
        policy_id="policy:drift-circuit-breaker-v3",
    ),
    repo,
)
```

Three things to notice:

- The `actor` is a human identifier even though the trigger was an AI. The `delegated_to` field captures the AI's involvement; the `policy_id` ties back to the policy that grants the AI permission to act.
- The `AuditAction` is a closed enum, not free-form. New action types go through review; this is what keeps the audit vocabulary tractable for compliance reviewers.
- The repository contract enforces append-only at the database role level, not in application code. Application bugs cannot bypass a database role that lacks UPDATE permission.

### 4.1 Applying the responsible-AI lens to one engineering decision

The audit-event writer above is the accountability surface. To see the *lens* applied end-to-end, walk through one engineering decision — *what error message to return when an LLM call fails?*

The naive engineering answer: return a 500 with a generic error message.

The lens-driven answer walks through the four operational properties:

```text
Decision: What does the API return when the LLM call fails?

Accountability:
  - The failure must produce an AuditEvent with actor = caller, action = LLM_CALL_FAILED,
    target = endpoint, trace_id = current trace.
  - The audit row captures the failure mode (timeout, content-filter, rate-limit,
    upstream-error) for later analysis.

Fairness / non-discrimination:
  - The error surface must be the same shape for every tenant. Different tenants do not
    get different error messages (which would leak whether they are "premium" tenants).
  - Per-tenant rate-limit failures must not surface tenant-identifying information to
    other tenants' callers (via shared error pages, etc.).

Transparency:
  - The error returned to the user distinguishes "the AI feature is unavailable" from
    "your input was rejected" from "the system is overloaded." Users can act on the
    difference. The error does not pretend the AI succeeded.

Safety / human oversight:
  - If failure rate exceeds threshold, the endpoint enters circuit-breaker mode. The
    circuit-breaker trip writes an AuditEvent and pages on-call.
  - Repeated failures for one tenant route to a tenant-success channel, not a generic
    on-call queue.
```

The change from naive to lens-driven added: an audit write, a circuit-breaker integration, a four-mode user-message surface, and an identical-shape error envelope. None of these are gold-plating — each is required by one of the four operational properties. The lens determined *what* to do; the audit record proves it was done.

## 5. Real-world Patterns

**Healthcare — HIPAA-compliant audit for an AI clinical-decision assistant.** A hospital network's AI assistant suggested medication-dosing options to clinicians. Every suggestion produced an AuditEvent with `actor = <clinician-id>` (the human who consulted the AI), `action = "AI_DECISION_SURFACED"`, `delegated_to = <model-id>`, and the full input context hash. When a clinician adopted the suggestion, a second AuditEvent fired with `actor = <clinician-id>` and `action = "PRESCRIPTION_WRITTEN"`. The audit table linked the two via the trace ID. The governance document was a single 12-page PDF that referenced ADRs implementing each control; eval reports were disaggregated by patient demographic, and bias-detection thresholds gated model updates [NIST SP 800-53 Rev. 5; NIST AI RMF, retrieved 2026-05-26].

**Fintech — automated-trading audit + fairness-aware credit decisions.** An algorithmic trading desk treated every model-driven trade as an AuditEvent. The `actor` was the desk head who authorised the model's trading mandate; `delegated_to` was the model version; `policy_id` was the risk-limit policy that scoped the model's authority. The audit table sat in a separate AWS account with cross-account write-only access from the trading service. Auditors had read access via a third role. Three roles, three accounts; AU-9 separation by design. The same firm's consumer-lending platform used disaggregated evaluation as the lens: every model version was evaluated separately on protected-class proxies before deployment. The deployment gate was a quantitative fairness test, not a committee approval. The governance documentation was generated from the eval pipeline output rather than written by hand, which kept the documentation honest.

**E-commerce — transparency in recommendation systems.** A retailer deploying an LLM-driven product-recommendation feature surfaced *why* the recommendation appeared ("Because you viewed X and reviewers like you bought Y"). Transparency was enforced as a UX requirement, not a back-office one. The OWASP threat lens added: protection against prompt-injection in user-generated review text used by the recommender. Every recommendation surfaced was logged with the input context and the model version so that disputed recommendations could be reconstructed [OWASP LLM Top 10, retrieved 2026-05-26].

**Manufacturing — ISO/IEC 42001 adoption for predictive-maintenance AI.** A manufacturer's predictive-maintenance system used ISO/IEC 42001 as the governance framework [ISO/IEC 42001:2023, retrieved 2026-05-26]. The standard defines a management-system shape with roles, responsibilities, risk treatment, and continual improvement. The team mapped each clause to existing engineering artifacts (audit log, eval pipeline, runbook) so the certification audit had a paper trail without inventing new documentation just for the certification. Each auto-route decision produced an AuditEvent with `actor = <ops-team-lead>`, `delegated_to = <model-id>`, and `target = <component-id>`.

## 6. Best Practices

- Treat trustworthy-AI characteristics as application properties, not model properties; the application is where they are or are not realised.
- Run the trustworthy-AI lens and the threat lens (OWASP LLM Top 10) on the same engineering decision; they often produce different controls that combine into a single implementation.
- Keep the audit-action vocabulary closed and reviewed; new action types go through a small-change review.
- Use a separate datastore role (or separate account, for higher-stakes workloads) for the audit table writer; the role's permissions exclude UPDATE and DELETE at the database level.
- Carry the trace ID in every AuditEvent so the audit row joins back to operational trace at incident-response time.
- Set `actor` to the human-identifiable party who authorised the action; capture AI involvement in `delegated_to` and `policy_id`.
- Sample operational logs aggressively, but never sample audit logs — every audit-relevant action gets a row.
- Generate governance documentation from engineering artifacts where possible; hand-authored governance documents drift from implementation.
- Disaggregate evaluation by relevant cohorts (demographic, tenant, geography) rather than reporting aggregate metrics only.
- Tie OWASP LLM Top 10 mitigations to per-endpoint controls; a generic "we follow OWASP" claim is not a mitigation.
- Run a chain-of-custody verification script on the audit table on a schedule (daily or weekly), and produce a verification AuditEvent itself.
- Re-run the lens whenever the model version, the corpus, or the user surface changes — these are the points where assumptions can quietly break.
- Retain audit logs per the applicable regulatory minimum (FedRAMP AU-11, HIPAA, PCI all specify minimums) and add a margin for incident-response reach-back.

## 7. Hands-on Exercise

Pick an LLM-driven feature you know (or design one — e.g., an internal-search assistant, a customer-service summariser, a code-review helper). In 20 minutes (split: 12 min lens, 8 min audit):

**Part A — Lens walk-through (12 min).**

1. List the four operational properties (accountability, fairness, transparency, safety + oversight). For each, write one concrete engineering control you would implement on the feature.
2. Pick one decision in the feature (an error path, a retrieval boundary, an output rendering choice) and walk through how each of the four properties affects the implementation.
3. Identify which of the OWASP LLM Top 10 categories your feature must mitigate; pick the top three and name one control each.

**Part B — Audit-event design (8 min).**

4. List the top five actions in the feature that should produce an AuditEvent.
5. For each action, write down the four required fields (actor, action, target, occurred_at) and what populates them.
6. Identify which one of the five would be triggered by an automated process (AI or otherwise) and write the `delegated_to` and `policy_id` fields for it.

**What good looks like.** Each operational property maps to a specific control, not a generic claim. The walked-through decision changes shape between the naive and lens-driven versions (audit row added, error shape constrained, escalation surface added). The five actions are specific (`PASSWORD_RESET`, `INVOICE_PAID`, `MODEL_RETRAINED`), not generic (`UPDATE`, `ACTION`). The `actor` for the automated action is a human role, not the system or the model. The `policy_id` points to a real policy document that exists or that you would create. The OWASP mitigations are control-level (e.g., "structured-output schema validation on every LLM response to mitigate LLM05 improper output handling"), not category-level.

## 8. Key Takeaways

- Why is responsible AI a *lens* applied at every engineering decision, rather than a *workstream* run by a separate committee — and why does the lens-driven shape make the audit record meaningful?
- Why must operational logs and audit logs live in different stores with different access controls?
- What are the four required AuditEvent fields, and which AU-3 extensions go on top of them?
- For an AI-triggered action, who is the `actor`? Who is captured in `delegated_to` and `policy_id`? How does this render NIST AI RMF's "accountable and transparent" characteristic as a single concrete field?
- How does the OWASP LLM Top 10 threat lens combine with the trustworthy-AI lens to produce concrete engineering controls?
- Why is governance documentation an output of the work, not a substitute for it?

## Sources

1. [NIST SP 800-53 Revision 5 — Security and Privacy Controls for Information Systems and Organizations](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final) — retrieved 2026-05-26
2. [OpenTelemetry — Semantic conventions for generative AI systems](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — retrieved 2026-05-26
3. [NIST AI Risk Management Framework — overview](https://www.nist.gov/itl/ai-risk-management-framework) — retrieved 2026-05-26
4. [NIST AI RMF Knowledge Base — characteristics of trustworthy AI](https://airc.nist.gov/airmf-resources/airmf/) — retrieved 2026-05-26
5. [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/) — retrieved 2026-05-26
6. [ISO/IEC 42001:2023 — AI Management System standard](https://www.iso.org/standard/81230.html) — retrieved 2026-05-26
7. [FedRAMP Marketplace — homepage and program overview](https://www.fedramp.gov/) — retrieved 2026-05-26

Last verified: 2026-05-26
