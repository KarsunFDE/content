---
week: W05
day: Wed
title: "HITL #7 — Auto-Remediation Authority ADR Template"
audience: pair (one ADR per pair)
template_for: pair-authored ADR committed EOD Wed; load-bearing for Fri PR rubric
input_artifact: weeks/W05/scenarios/W05-SA-2.md
output_consumed_by: weeks/W05/war-room/Fri-final-adversarial-pr.md (Fri PR includes the ADR + matching enforcement)
last_verified: 2026-05-23
hitl_touchpoint: 7 (closing)
---

# HITL #7 — Auto-Remediation Authority ADR Template

> Per `scenarios/W05-SA-2.md`, each pair commits one HITL #7 authority ADR by EOD Wed. This file is the *template* for the per-pair ADR. The ADR is **load-bearing** for Fri's Final Adversarial PR — the chosen authority option must be enforced in code or config, not just documented.

## ADR filename convention

`pair-projects/pair-N/docs/adrs/HITL-7-auto-remediation-authority.md`

(Mirrors W3 multi-agent ADRs; lives in pair-project repo.)

## ADR frontmatter

```yaml
---
adr_id: HITL-7-pair-N
title: "Auto-remediation authority for the audit-gap detector"
date: 2026-06-24  # W5 Wed
status: accepted
pair: pair-N
hitl_touchpoint: 7
load_bearing_for: weeks/W05/war-room/Fri-final-adversarial-pr.md
references:
  - skills/aiops-curriculum/references/ai-sre-patterns.md
  - training-project/feature-inventory-target.md (Item 2 — audit-log race)
  - pipeline/DECISIONS.md D-043 (7 HITL touchpoints)
  - FedRAMP Rev 5 AC-5, AC-3, AU-2, AU-9
---
```

## Required sections

### 1. Context

The pair narrates the case in their own words (3–5 sentences). Reference points:

- The Item 2 audit-log race condition produces gaps when a process crash interleaves between the primary table commit and the AuditEvent insert.
- W5 Tue's OTel instrumentation made the gap *detectable* — the platform has the trace data for the missing AuditEvent.
- W5 Mon's AWS managed-service gate opened — the platform now has options (Datadog automation, Bedrock Agents) for auto-remediation that weren't available W1–W4.
- The cohort's pair is operating against a federal client; FedRAMP AC-5 (separation of duties) + AU-9 (audit log integrity) shape the decision.

### 2. Decision

**One of three options:**

- **A — Full Auto:** The platform writes the missing AuditEvent atomically, using the OTel trace data + the original request context. The platform's own action is itself an AuditEvent (`replayed_by_ai_sre: true`).
- **B — Propose-and-Await-Approval:** The platform generates the AuditEvent payload + opens a ticket to sys_admin + pages on-call. Human reviews + merges.
- **C — Escalate-only:** The platform pages on-call human + does *not* generate the payload. Human investigates + writes (or doesn't) manually.

Pair commits **one** option. *"It depends"* is not an ADR; the pair commits a position and names where it would change.

### 3. Rationale

3–7 bullets. Required content:

- FedRAMP control reference — which control supports the choice (AC-5, AC-3, AU-2, AU-9)? Cite specifically.
- Reversibility analysis — if the auto-remediation fires incorrectly, what's the undo path?
- Audit-of-the-audit shape — what AuditEvent does the platform write about itself? Who reviews it?
- Failure mode the chosen option handles — what bad outcome does this prevent?
- Failure mode the chosen option *creates* — name the worst case under this choice (every option creates *some* failure mode; the ADR is honest about it).

### 4. Alternatives considered

Both alternatives evaluated (the two options not chosen). For each:

- Strongest argument FOR the alternative.
- Strongest argument AGAINST the alternative.
- The specific condition under which the pair would pivot to it (e.g., "If Karsun engagement scope grew to 24/7 on-call, we'd consider Option A").

### 5. Consequences

What happens downstream:

- W6 implications — how does this ADR shape the W6 deliverability runbook?
- The cohort-wide HITL authority-boundary table (W6 Wed deliverable) gains a row from this ADR.
- The pair's Fri PR includes code/config enforcing the chosen option (see §6).

### 6. Enforcement (load-bearing for Fri PR)

**The ADR is rubric-failed if §6 is doc-only.** Pair specifies:

- **What code change enforces the choice.** (E.g., for Option B: a new endpoint `POST /admin/audit/replay-proposal` that requires sys_admin auth + accepts the platform's proposed payload + writes the AuditEvent. For Option A: same endpoint but invocable by the platform's IAM role with `replayed_by_ai_sre: true` flag enforced in code. For Option C: no endpoint at all; instead, a PagerDuty integration spec + a runbook entry.)
- **What test demonstrates enforcement.** (E.g., for Option B: a test where the platform tries to write the AuditEvent without sys_admin approval + fails. For Option A: a test where the platform writes + the `replayed_by_ai_sre` audit row is created.)
- **What Datadog automation policy enforces the choice.** (E.g., for Option B: a Datadog Workflow that proposes the fix + pages but does NOT write. For Option A: a Workflow that calls the endpoint directly.)

### 7. Rollback story

If Cohort #2 or a future engagement needs a different option:

- What code changes to undo?
- What automation policies to swap?
- What runbook entries to revise?

3–5 bullets minimum.

### 8. Sources

Per `pipeline/RESEARCH-PROTOCOL.md` D-046 — all citations via `/web-research`. Required citations:

- At minimum 1 FedRAMP / NIST citation grounding the AC-5 / AU-9 reasoning.
- At minimum 1 vendor citation grounding the enforcement choice (Datadog automation policy docs, Bedrock Agents user-confirmation API docs, or equivalent).
- The `skills/aiops-curriculum/references/ai-sre-patterns.md` §6 reference.

## What makes a strong HITL #7 ADR (rubric weighting)

Per `assessments/W05-final-adversarial-pr-rubric.md`, the HITL #7 ADR contributes to:

- **Brownfield Awareness** — names Item 2 by number; explains the AU-table criticality.
- **Architectural Fit** — the enforcement choice respects multi-tenant boundaries (Item 10) and consistent OTel propagation.
- **Security Pass** — addresses LLM06 (Excessive Agency) by explicitly bounding the AI-SRE's authority.
- **Code Quality** — enforcement code is idiomatic with the rest of the codebase.
- **PR Clarity** — Fri PR description links this ADR; ADR is internally consistent + readable by a new team member.

## Common anti-patterns to avoid

- **Picking Option C because it feels "safest" without engaging the audit-binder gap consequence.** Escalate-only leaves the gap longer; the OIG audit binder still misses the row. Defensible only if on-call response SLA is fast enough.
- **Picking Option A without naming the audit-of-the-audit.** Full Auto without the platform's own action being auditable is a separation-of-duties violation.
- **Picking Option B without designing the human-approval UX.** "Propose-and-await" requires a UX surface for sys_admin to review the payload. Skipping that = the PR doesn't actually enforce B.
- **Citing FedRAMP generically.** Cite the specific control (AC-5, AU-9) by number + clause.
- **Not naming the failure mode the choice creates.** Every option has one. An ADR that only names the wins is incomplete.

## Sample skeleton (instructor reference; not for cohort consumption)

```markdown
# HITL-7-pair-1 — Auto-remediation authority for audit-gap detector

## Context
Item 2 of acquire-gov inventory is a race condition: ... [3-5 sentences]

## Decision
We choose **Option B — Propose-and-Await-Approval**.

## Rationale
- AC-5 (separation of duties) requires write authority on AU surface be human.
- AU-9 (audit log integrity) is preserved by AI-proposed payload + human approval — gap fills *with* attestation.
- Reversibility: if the proposed payload is wrong, sys_admin rejects; no data is written.
- Failure mode handled: gap-creation under crash.
- Failure mode created: latency between detection + write; mitigated by PagerDuty escalation policy.

## Alternatives considered
**Option A — Full Auto.** Pivots to A only if 24/7 on-call coverage and audit-of-the-audit infrastructure are confirmed.
**Option C — Escalate-only.** Pivots to C only if engagement explicitly disallows AI-proposed audit content.

## Consequences
W6 runbook gains a section on the propose-and-approve flow. The cohort-wide HITL authority-boundary table gains pair-1's row.

## Enforcement
- Code: `POST /admin/audit/replay-proposal` endpoint with sys_admin-only auth.
- Test: `AuditReplayProposalTest.testRejectsUnauthorizedActor()`.
- Datadog automation: Workflow `acquire-gov-item-2-race-fix` proposes payload + pages on-call.

## Rollback
[3-5 bullets]

## Sources
- [FedRAMP Rev 5 AC-5 + AU-9 citation]
- [Datadog Workflows docs citation]
- skills/aiops-curriculum/references/ai-sre-patterns.md §6
```
