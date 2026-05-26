---
week: W05
day: Thu
topic_slug: human-oversight-escalation-workflows
topic_title: "Human Oversight — escalation workflows as the operational artifact of authority decisions"
parent_overview: W05/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://sre.google/sre-book/managing-incidents/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://www.atlassian.com/incident-management/kpis/severity-levels
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://airc.nist.gov/airmf-resources/airmf/
    retrieved_on: 2026-05-26
    recency_category: federal-regulatory-6mo
  - url: https://opentelemetry.io/docs/specs/semconv/gen-ai/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
last_verified: 2026-05-26
---

# Escalation Workflows — Translating Authority Decisions Into Operational Tiers

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish an authority decision (policy: who is allowed to act, under what scope) from an escalation workflow (mechanics: how the action gets executed on-call).
- Construct a tiered escalation workflow with named SLA, named owner, and named hand-off audit row at each tier.
- Map severity levels (SEV-1/2/3) onto AI-system incident classes such that auto-remediation tiers align with severity.
- Identify the audit and hand-off records that make an escalation workflow reconstructable months later.
- Recognise the failure mode of an authority decision that has no operational workflow attached.

## 2. Introduction

A human-in-the-loop (HITL) policy in an AI system is, fundamentally, a statement about authority: *who is allowed to do what, in what scope, with what reversibility, supervised by whom?* The policy is the governance artifact. But governance without operations does not survive contact with on-call. A policy that says "the AI may auto-remediate cache failures within bounded scope" does not, on its own, tell the on-call engineer at 02:00 what to do when the AI did remediate and the page fired anyway.

The escalation workflow is the operational artifact that completes the authority decision. It names the tiers of response, the SLA at each tier, the human who answers each tier, and the audit record produced at each tier hand-off. The policy says *what is allowed*; the workflow says *how the allowance is realised in on-call practice*.

Without a workflow, the policy is wishful: "the AI may auto-remediate" with no specified tier, no specified routing, no specified hand-off audit produces nothing useful. With a workflow, the policy becomes operational: the AI auto-remediates within Tier 1; if that fails, page Tier 2; if Tier 2 cannot resolve in SLA, escalate to Tier 3 with a fixed hand-off shape.

The pattern this reading covers is generic across SRE practice (Google's SRE book, PagerDuty's incident-response documentation, Atlassian's severity-level framework) and applies cleanly to AI-system oversight when each tier is annotated with the AI-action authority it is allowed to take.

## 3. Core Concepts

### 3.1 Authority decision versus escalation workflow

An **authority decision** answers four questions:

- *Who* may take an action? (A human role; an automated process under a named policy; both.)
- *On what scope*? (Specific resources, specific endpoints, specific tenancies.)
- *With what reversibility*? (Self-reversing actions versus actions that require a separate restoration step.)
- *Supervised by whom*? (Which human role is responsible for the policy's continued correctness.)

An **escalation workflow** answers five questions:

- *Which tier* handles which class of incident?
- *What is the SLA* for each tier?
- *Who is the named owner* of each tier (a role, on-call rotation, or human queue)?
- *What action authority* does each tier have, including auto-remediation?
- *What audit record* is produced at each tier hand-off?

The policy is upstream; the workflow is downstream. The workflow must reference the policy by ID so the audit trail at tier hand-off can pin back to the authorisation.

### 3.2 The three-tier shape

The simplest workflow that scales has three tiers:

- **Tier 1 — auto-handled within authority.** A signal fires and a defined automated remediation runs within the scope the authority decision allows. The action writes an AuditEvent. No human is paged unless escalation criteria are met (e.g., repeated Tier 1 actions inside a window).
- **Tier 2 — on-call human, paged within SLA.** The signal exceeded Tier 1 scope, or repeated Tier 1 actions tripped a paging rule, or the signal is one that Tier 1 cannot handle. An on-call engineer is paged with a named SLA for acknowledgement (commonly 15 minutes for SEV-1/2, longer for SEV-3) [Google SRE book — managing incidents, retrieved 2026-05-26].
- **Tier 3 — escalate to broader stakeholders.** The incident is severity-1 or has implications beyond on-call's scope: a regulatory boundary event, multi-tenant data leak signal, audit-integrity event, or repeated Tier 2 escalations. Tier 3 often involves legal, security, executive leadership, or external parties (e.g., regulators, partners) [Atlassian severity-levels guidance, retrieved 2026-05-26].

The names vary by organisation — some teams use SEV-1/2/3 numbering directly, some use named tiers, some use auto/human/exec. The shape matters more than the labels.

### 3.3 Severity mapping

Atlassian's published guidance gives a typical severity definition [Atlassian severity-levels, retrieved 2026-05-26]:

- **SEV-1** — critical, very-high impact. Customer data loss, security breach, client-facing service down for all customers.
- **SEV-2** — major, significant impact. Client-facing service down for a subset of customers, or critical function not working.
- **SEV-3** — minor, low impact. System glitch causing slight inconvenience.

Auto-remediation authority should track severity. A reasonable mapping:

- **SEV-3** → Tier 1 auto-remediation acceptable for bounded scope (the AI may take action without paging anyone).
- **SEV-2** → Tier 1 may be permitted for highly-bounded scope (e.g., circuit-breaker trip), but Tier 2 paged in parallel.
- **SEV-1** → Tier 2 (page on-call immediately); auto-remediation discouraged because reversibility may be limited. Tier 3 stakeholders notified.

This mapping is not universal — some organisations are more aggressive with auto-remediation, some less — but the *principle* (severity scales with paging immediacy, not with the model's confidence in its remediation) is robust.

### 3.4 The hand-off audit row

Every tier transition produces a hand-off audit record. Without these, post-incident reconstruction cannot tell when a Tier 1 action escalated to Tier 2, or when Tier 2's investigation handed control to Tier 3. The hand-off record carries at minimum:

- `from_tier` and `to_tier`.
- `trigger` — the rule, SLA breach, or human escalation that caused the transition.
- `originating_trace_id` — the trace from which the original signal came.
- `outcome_at_previous_tier` — what was attempted, with what result.
- `handed_to` — the role or human now responsible.

OpenTelemetry's general trace conventions give the team a trace-ID surface to use; for GenAI workloads, the `gen_ai.*` conventions add model invocation context that lets a hand-off row carry which model decisions led to the escalation [OpenTelemetry GenAI semantic conventions, retrieved 2026-05-26].

### 3.5 SLA and SLA breach

Each tier carries a named SLA — typically an acknowledgement time and a resolution time. Acknowledgement SLA is hard (it is the page-response time); resolution SLA is soft (it depends on the incident's complexity).

The workflow specifies what happens at SLA breach: re-page (escalate within tier), auto-escalate to next tier (escalate between tiers), or trigger a third condition (e.g., notify a stakeholder list). Choosing between these is workload-specific; the workflow must choose one explicitly. "We'll figure it out" is not a workflow.

### 3.6 The "policy without workflow" failure mode

A common antipattern: a team writes an authority policy for an AI auto-remediation, gets it reviewed, commits it to the repo, and never writes the workflow. The first time the policy's auto-remediation fires in anger, the on-call engineer does not know whether the AI's action succeeded, failed silently, or is in flight. The page that fires has no tier, no SLA, and no documented next step. The team produces an ad-hoc response that does not match the policy, and the post-incident review struggles to reconcile "what the policy says" with "what we did."

The fix is to require the workflow as a deliverable at the same review pass as the policy. A policy without a workflow is not finished.

## 4. Generic Implementation

A workflow as a YAML file that the on-call tooling and the audit pipeline can both consume:

```yaml
# escalation-workflow.yaml
# Owned by: <workflow-owner-human>
# References policy: authority-policy-v3

workflow_id: escalation-workflow-v1
policy_reference: authority-policy-v3

tiers:
  - id: tier-1
    name: auto-handled
    severity_classes: [SEV-3]
    owner: automated-system
    sla:
      ack_minutes: 0      # automated; no acknowledgement step
      resolve_minutes: 5  # if not resolved in 5 minutes, escalate
    permitted_actions:
      - circuit-breaker-trip
      - retry-with-fallback
      - traffic-shed
    escalation_triggers:
      - sla_breach
      - repeated_in_window:
          window: 10m
          count: 3

  - id: tier-2
    name: on-call-human
    severity_classes: [SEV-2, SEV-1]
    owner: oncall-llm-rotation
    sla:
      ack_minutes: 15
      resolve_minutes: 120
    permitted_actions:
      - rollback-deploy
      - disable-endpoint
      - revise-threshold
    escalation_triggers:
      - sla_breach
      - regulatory_event_detected
      - cross_tenant_leak_signal

  - id: tier-3
    name: stakeholder-escalation
    severity_classes: [SEV-1]
    owner: incident-commander
    sla:
      ack_minutes: 30
      resolve_minutes: post-incident-review-driven
    notifies:
      - legal
      - security
      - executive-on-call
    permitted_actions:
      - external-disclosure
      - regulatory-notification
      - architectural-change-authorisation

handoff_record_fields:
  - from_tier
  - to_tier
  - trigger
  - originating_trace_id
  - outcome_at_previous_tier
  - handed_to
  - occurred_at
```

A code reader can produce an immediate operational test:

```python
def trigger_escalation(
    current_tier: dict,
    next_tier: dict,
    trigger_reason: str,
    incident_state: dict,
) -> None:
    """Move an incident from current_tier to next_tier with a hand-off audit row."""
    write_audit_event(
        action="ESCALATION_HANDOFF",
        actor=incident_state["incident_commander"] or "automated-system",
        target=f"incident:{incident_state['id']}",
        outcome="HANDOFF",
        trace_id=incident_state["originating_trace_id"],
        extra={
            "from_tier": current_tier["id"],
            "to_tier": next_tier["id"],
            "trigger": trigger_reason,
            "outcome_at_previous_tier": incident_state["last_outcome"],
            "handed_to": next_tier["owner"],
        },
    )
    notify(next_tier["owner"], incident_state, sla=next_tier["sla"])
```

Three things to notice:

- The handoff record is itself an audit event. The "actor" for an automated hand-off is the automated-system identifier; for a human-initiated hand-off it is the human escalating. Either way, accountability is preserved.
- Permitted-action lists are tier-scoped. Tier 1 actions are conservative; Tier 3 actions include external-disclosure-class items. The tier scope makes the authority surface inspectable.
- SLA breach is a first-class escalation trigger. Workflows that omit SLA breach as a trigger produce silent failures (the incident gets stuck at a tier with nobody acting).

## 5. Real-world Patterns

**Healthcare — escalation for an AI-driven clinical alert.** A hospital network's vital-signs anomaly detector had a tiered workflow: Tier 1 routed bedside-equipment alerts to the floor nurse, Tier 2 escalated unacknowledged alerts to the charge nurse within 5 minutes, Tier 3 paged the on-call physician for SEV-1 conditions. Each tier transition wrote a record to the patient's care log; post-incident review of any adverse outcome could reconstruct the alert chain [Google SRE book — managing incidents, retrieved 2026-05-26].

**Fintech — workflow for an automated-trading risk-limit breach.** A trading desk's AI risk-monitor had a workflow: Tier 1 auto-tightened position limits within a small scope; Tier 2 paged a risk-desk human if limits were tightened repeatedly within a 10-minute window; Tier 3 paged the chief risk officer for any breach that crossed a regulatory-reporting threshold. The Tier 3 escalation also triggered an external regulator notification within a 24-hour SLA. The workflow's hand-off records were retained for seven years per the firm's compliance policy.

**E-commerce — escalation for a search-relevance drift.** A retailer's LLM-driven product search had a workflow tied to a drift-monitoring policy. Tier 1 auto-rolled-back the active model to the previous version if relevance metrics dropped by a defined margin; Tier 2 paged a search-team engineer if Tier 1 had rolled back more than once in 24 hours; Tier 3 paged product leadership if the rollback affected revenue-critical search categories. The audit chain let the team reconstruct months later why a particular Friday's search-quality dip turned into a Saturday rollback [Atlassian severity-levels guidance, retrieved 2026-05-26].

**Logistics — escalation for an auto-routing anomaly.** A parcel-delivery network's auto-routing AI had a workflow tied to package-level severity: a small batch of mis-routed packages was Tier 1 (re-route automatically); a regional disruption was Tier 2 (page the regional ops lead); a multi-region routing failure was Tier 3 (page the ops vice-president and notify the affected carrier partners). Tier 3 included a customer-communication SLA that ran on a 4-hour clock.

## 6. Best Practices

- Write the workflow at the same review pass as the policy; a policy without a workflow is not finished.
- Map severity to tier explicitly; do not let the operational team guess which tier handles which severity.
- Specify what happens at SLA breach for each tier; "we'll figure it out" is not a workflow.
- Treat every tier transition as an audit event with named fields; without hand-off records, post-incident review cannot reconstruct the chain.
- Keep the permitted-actions list per tier closed and reviewable; ad-hoc actions during an incident are sometimes necessary, but they must be retrospectively reviewed and either added to the list or refused as policy violations.
- Run incident-response drills against the workflow; a workflow that has never been exercised will fail in production.
- Re-tune SLA values based on incident-response retros; SLAs that always breach are too tight; SLAs that never bind are too loose.

## 7. Hands-on Exercise

Pick an automated system you know (a deploy pipeline, a fraud-detection model, a content-moderation system, a chatbot). In 15 minutes:

1. Write a three-tier escalation workflow with tier name, owner, SLA (ack and resolve), and permitted actions.
2. Identify the escalation triggers between tiers (SLA breach, repeated-action threshold, severity escalation).
3. Write the hand-off record fields and one example hand-off in narrative form ("At 02:14 UTC, Tier 1 auto-remediation tripped circuit-breaker for cache-east-1; after 5 minutes without recovery, Tier 2 was paged at 02:19 UTC with originating_trace_id=abc, handed_to=oncall-search-rotation.")

**What good looks like.** Each tier has a specific owner (named rotation, named role, named human queue), not "engineering." Each tier's permitted-action list is closed and includes the actions you actually expect to take. The hand-off example reads like a real audit row — it would still make sense to a stranger reading it six months later.

## 8. Key Takeaways

- What is the distinction between an authority decision (policy) and an escalation workflow (mechanics), and why must both exist?
- What three tiers commonly appear in escalation workflows, and how does auto-remediation authority typically scope per tier?
- How do severity levels (SEV-1/2/3) map onto escalation tiers, and why is the mapping severity-driven rather than confidence-driven?
- What does a hand-off audit record contain, and why is it the load-bearing artifact for post-incident reconstruction?
- What is the failure mode of an authority policy without an attached workflow, and how is the failure mode usually surfaced?

## Sources

1. [Google SRE Book — Managing Incidents](https://sre.google/sre-book/managing-incidents/) — retrieved 2026-05-26
2. [Atlassian — Understanding incident severity levels](https://www.atlassian.com/incident-management/kpis/severity-levels) — retrieved 2026-05-26
3. [NIST AI RMF Knowledge Base — trustworthy AI characteristics](https://airc.nist.gov/airmf-resources/airmf/) — retrieved 2026-05-26
4. [OpenTelemetry — Semantic conventions for generative AI systems](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — retrieved 2026-05-26

Last verified: 2026-05-26
