---
week: W06
day: Tue
topic_slug: audit-trail-walkthrough
topic_title: "Audit-trail walkthrough"
parent_overview: W06/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://cpahalltalk.com/audit-walkthroughs/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.hubifi.com/blog/immutable-audit-log-basics
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.cybersecurityintelligence.com/blog/audit-trails-as-evidence-from-logs-to-proof-8989.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://devsecopsschool.com/blog/audit-trails/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://oneuptime.com/blog/post/2026-02-06-sox-compliant-audit-trails-opentelemetry/view
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Audit-trail walkthrough

## 1. Learning Objectives

By the end of this reading, the learner can:

- Conduct a cradle-to-grave walkthrough of a transaction or workflow from the originating event through every state transition to the final outcome.
- Identify the five fields that every defensible audit log row must capture (actor, timestamp, action, target, before/after state).
- Recognise the additional fields an AI-assisted system audit trail must carry (model, prompt, temperature, HITL confirmation event, assistance annotation).
- Build a Findings-tracker view of the audit trail that lets a reviewer move from a finding back to the underlying audit rows in two clicks.
- Spot the three most common audit-trail gaps (UI-only state changes, late writes after the fact-of-record, and silent automation).

## 2. Introduction

An audit trail is not the same as an application log. Application logs help engineers debug; audit trails help auditors *prove*. An audit trail must be tamper-evident, attributable to a specific actor, complete across every state-changing event, and survivable under retention policies ([Audit Trails as Evidence: From Logs to Proof, retrieved 2026-05-26](https://www.cybersecurityintelligence.com/blog/audit-trails-as-evidence-from-logs-to-proof-8989.html); [DevSecOps School 2026 Audit Trails Guide, retrieved 2026-05-26](https://devsecopsschool.com/blog/audit-trails/)).

A *walkthrough* is the auditor's primary verification technique. The accounting-audit literature defines it as a "cradle-to-grave review of a transaction cycle" — start at the originating event, follow it step by step, end at the final disposition ([CPA Hall Talk, retrieved 2026-05-26](https://cpahalltalk.com/audit-walkthroughs/)). The auditor is not looking at sample summaries; they are looking at *one* transaction in detail.

The discipline transfers directly to AI-assisted systems. The walkthrough is the same — pick a recent transaction, walk every state transition, show the anchoring row. The difference is the additional fields AI-assisted systems must surface at each step: model, prompt, temperature, HITL confirmation, assistance annotation. Without those, the walkthrough is incomplete by audit standards.

## 3. Core Concepts

### 3.1 What a walkthrough actually is

A walkthrough is *not* a code review and *not* a tour of the user interface. It is a focused, evidence-driven traversal of a single transaction. The audit-process literature describes it as:

- **Cradle-to-grave** — start at the originating event, end at the final disposition. No skipping middle steps.
- **Single-transaction** — one real transaction, not a "typical" composite. Auditors specifically resist composite walkthroughs because composites can hide whichever step is actually broken.
- **Observable** — every state change is anchored by a row, a record, or a piece of evidence the auditor can point to and reproduce.

The audit literature emphasises *picking a transaction the team did not pre-select*. The auditor often picks; if the team picks, the auditor will pick a second one to verify the team did not curate the example. The walkthrough is a randomised sample of one, by design.

### 3.2 The five mandatory fields

Every defensible audit-log row carries at minimum:

1. **Actor** — who or what initiated the change. For human actors: a user ID or service account. For automated actions: the service/workflow identity. "System" is not an acceptable actor; if a change is automatic, the *automation that triggered it* is the actor.
2. **Timestamp** — UTC, ISO 8601 to at least millisecond precision, server-clock not client-clock.
3. **Action** — a controlled-vocabulary verb (e.g., `CREATE`, `APPROVE`, `PUBLISH`, `REVOKE`). Free-text actions defeat aggregation and look unprofessional in audit settings.
4. **Target** — the entity changed. Typed reference (collection + ID, or table + primary key) so the reviewer can pull the entity's full history.
5. **Before/after state** — the values of any fields that changed. Either a diff, a full snapshot, or pointers to versioned records. Without this, the row records that *something* happened but not *what* changed.

Several industry guides converge on this minimum set ([Hubifi on immutable audit trails, retrieved 2026-05-26](https://www.hubifi.com/blog/immutable-audit-log-basics); [DevSecOps School audit-trails guide, retrieved 2026-05-26](https://devsecopsschool.com/blog/audit-trails/); [OneUptime on SOX-compliant audit trails with OpenTelemetry, retrieved 2026-05-26](https://oneuptime.com/blog/post/2026-02-06-sox-compliant-audit-trails-opentelemetry/view)).

### 3.3 The additional fields for AI-assisted systems

A system that uses generative models must capture additional fields per audit row:

- **Model identifier** — the exact name and version (not "GPT" — the specific `claude-3.5-sonnet-20241022`-style ID).
- **Prompt fingerprint** — full prompt or hash + pointer to prompt-version store. Makes drift and injection detectable.
- **Temperature / sampling parameters** — outputs are not reproducible without these.
- **Human-confirmation event** — if HITL-gated, the row references the approval event (a separate audit row) with approver identity and timestamp.
- **Assistance annotation** — downstream code/content notes the assistance mode (`AI-drafted`, `AI-suggested`, `hand-authored`, `AI-reviewed`).

Without these, the audit cannot answer "did a human review the model's output?", "would the output reproduce?", or "which prompt version was in effect last quarter?" With them, those questions have row-level answers.

### 3.4 Walkthrough structure

A workable structure for a live walkthrough, transferable across domains:

1. **Pick the most material recent transaction.** Recent is important — stale transactions invite questions about whether the current system would still behave this way.
2. **Open the Findings tracker.** Show the entity-level view: status, evidence requests, remediation state, owner, ETA. The Findings tracker is the auditor's primary entry point ([Hyperproof on findings remediation, retrieved 2026-05-26](https://hyperproof.io/resource/audit-findings-remediation-efforts/)).
3. **Walk the state transitions in order.** Each transition is one or two minutes — name the actor, show the audit row, point to the before/after state, show the AI-assistance fields if applicable.
4. **Drop into the database row.** Pre-empt the question "is there a row, or just a UI?" by showing the underlying data.
5. **Show the human-confirmation event** if HITL-gated. Bidirectional linkage between action row and approval row is the structural signal of HITL completeness.
6. **End at the final disposition** — the published document, the posted ledger entry, the closed Finding.

### 3.5 The three most common gaps

- **UI-only state changes.** A state change is visible in the UI but produces no row-level audit entry. Fix: every state change writes a before-and-after row in the audit collection.
- **Late writes after the fact-of-record.** An action takes effect before its audit row commits; a crash between leaves the action without trail. Fix: write the audit row in the same transaction, or use an outbox pattern with at-least-once delivery.
- **Silent automation.** A scheduled job changes state but the actor is "system" or empty. Fix: every automated actor has a stable identity (service account or workflow-version ID), and the trigger condition appears in row metadata.

Heuristic: "could I, working from the audit table alone, reconstruct this entity's state at any moment in the last 90 days?" If the answer involves application logs, one of the three gaps is present.

### 3.6 The Findings tracker as walkthrough surface

The Findings tracker is itself the walkthrough surface. The reviewer moves from a finding to the originating evidence row in two clicks. When the Findings UI exists, the walkthrough becomes "open Findings, click in, click in." Without it, the walkthrough is a tour of the database client — much harder under pressure. The remediation-tracking literature describes the binding: "every corrective step should be logged with timestamps, responsible owners, and evidence of change" ([Hyperproof, retrieved 2026-05-26](https://hyperproof.io/resource/audit-findings-remediation-efforts/)).

## 4. Generic Implementation

A short audit-row schema and walkthrough script for a generic SaaS billing system.

**Audit-row schema (Postgres example):**

```sql
CREATE TABLE audit_logs (
  id              BIGSERIAL PRIMARY KEY,
  occurred_at     TIMESTAMPTZ NOT NULL,            -- ISO 8601, UTC
  actor_type      TEXT NOT NULL,                   -- 'user' | 'service' | 'workflow'
  actor_id        TEXT NOT NULL,                   -- user ID, service account, workflow-version
  action          TEXT NOT NULL,                   -- controlled vocabulary
  target_type     TEXT NOT NULL,                   -- 'invoice' | 'customer' | 'refund'
  target_id       TEXT NOT NULL,                   -- entity ID
  before_state    JSONB,                           -- snapshot of changed fields, prior values
  after_state     JSONB,                           -- snapshot of changed fields, new values
  reason_code     TEXT,                            -- optional structured reason
  request_id      UUID NOT NULL,                   -- joins to request log for full context
  -- AI-assistance fields (NULL when not AI-touched)
  ai_model        TEXT,                            -- e.g. 'claude-3.5-sonnet-20241022'
  ai_prompt_hash  TEXT,                            -- pointer to prompt-version store
  ai_temperature  NUMERIC(3,2),
  hitl_approval_id BIGINT REFERENCES audit_logs(id), -- backreference if HITL-gated
  CONSTRAINT actor_is_named CHECK (actor_id IS NOT NULL AND actor_id <> '')
);
-- Append-only enforcement (no UPDATE, no DELETE permissions)
REVOKE UPDATE, DELETE ON audit_logs FROM PUBLIC;
```

Every row is attributable; action is controlled-vocabulary; before/after state is explicit; AI-assistance fields are nullable but populated whenever a model touched the path. The `CHECK` constraint plus revoked `UPDATE`/`DELETE` permissions enforce append-only at the database layer ([Hubifi on immutable audit trails, retrieved 2026-05-26](https://www.hubifi.com/blog/immutable-audit-log-basics)).

**Walkthrough script (5 minutes, for an invoice-refund transaction):**

1. **Open the Findings page.** The refund-related Finding F-2026-Q2-014 is open.
2. **Click the Finding.** Opened by `risk-reviewer`, evidence requests for refund rows R-77321 + R-77322, remediation status `evidence-pending`, owner finance-lead, due in two weeks.
3. **Click into evidence for R-77321.** The audit-row sequence: created by `customer-support-agent-04`, approved by `refund-admin-09`, posted by the `refund-poster` workflow, reconciled by `finance-reconciler`.
4. **Click any row** for the schema view — actor, timestamp, action, target, before/after, request ID. The approval row's `hitl_approval_id` points to the confirmation event with approver identity and timestamp.
5. **End at the ledger entry.** Click the `posted` row's target; the ledger entry shows amount, customer ID, reconciliation status, and a `provenance` field linking back to the audit-row chain.

Total: 5 clicks, 5 minutes. The reviewer has seen the actor, timestamp, before/after state, and HITL confirmation at every step.

## 5. Real-world Patterns

**Healthcare — EHR access audit.** Electronic Health Record systems under HIPAA capture every patient-record access — who, when, what was viewed, whether they were authorised. EHR audit logs are subject to walkthrough during HIPAA audits and break-glass investigations. The pattern is identical to financial-system walkthroughs: pick a record, walk the access history, verify each access had a legitimate basis. Hospitals with dedicated "audit-log viewer" interfaces run these walkthroughs in minutes; hospitals relying on raw queries run them in hours.

**Aviation — flight data recorders and maintenance logs.** Commercial aviation maintains two parallel audit trails: the Flight Data Recorder captures every instrument reading; the maintenance log captures every part replacement, inspection, and signoff. Both are subject to walkthrough during incident investigations and routine FAA audits. The aviation industry's audit-trail discipline is mature precisely because incomplete trails create unresolvable incidents.

**Financial services — SOX walkthrough requirements.** Sarbanes-Oxley requires auditors to perform walkthroughs of every significant financial-reporting process. The audit-trail technology stack has matured around this requirement; tools like Trullion and Hyperproof exist precisely to make Findings tracking and walkthrough delivery efficient under audit scrutiny ([SOX-compliant audit-trail patterns, OneUptime, retrieved 2026-05-26](https://oneuptime.com/blog/post/2026-02-06-sox-compliant-audit-trails-opentelemetry/view)). The patterns now appear in adjacent contexts — model-risk reviews, vendor-management audits, ISO certification audits — because the structural problem is the same.

**Manufacturing — track-and-trace under FDA / DSCSA.** The Drug Supply Chain Security Act requires pharmaceutical supply chains to maintain an audit trail of every unit of medicine. Walkthroughs are performed at the unit level: pick a serial number, trace it backward through every handoff. Audit-trail walkthrough is not specifically a software discipline — it is a records discipline software systems must serve correctly.

## 6. Best Practices

- **Every state-changing event writes an audit row in the same transaction as the change.** Late writes are gaps; out-of-band writes are gaps; conditional writes are gaps.
- **Enforce append-only at the database layer.** Revoke `UPDATE` and `DELETE` on the audit table; if amendments are required, write them as additional rows referencing the original.
- **Use a controlled vocabulary for actions.** Free-text actions defeat aggregation and look unprofessional under audit.
- **Always capture before-and-after state.** Diff form is fine if storage matters, but prior state must be reconstructible from the audit table alone.
- **AI-touched paths carry model, prompt, temperature, HITL-approval link, and assistance annotation.** Mandatory, not optional.
- **Maintain a Findings tracker UI** that surfaces named gaps and links to evidence rows in two clicks.
- **Rehearse the walkthrough.** Time yourself; tighten any step that exceeds a minute. Fluent delivery signals operational maturity.

## 7. Hands-on Exercise

**Time:** 15 minutes. **Format:** schema design + walkthrough script.

You are designing the audit trail for a generic peer-to-peer payments app. A "send money" transaction goes: user A initiates → fraud-check service evaluates → user A confirms with biometric → user B's account credits → reconciliation runs → ledger posts.

**Steps:**

1. **Draft the audit-row schema** for this app. Use the five mandatory fields plus any AI-assistance fields you'd add for the fraud-check service (which uses a model). Keep it to one table; no need for full SQL — pseudo-schema is fine.
2. **Write the walkthrough script** for a real $42.50 payment from `user-A` to `user-B` on 2026-05-22 at 14:33 UTC. List the audit rows the reviewer would see, in order, with the actor and action for each.
3. **Identify one place where a *late write* could create a gap** and propose the mitigation.

**Self-check:**

- Does every state-changing event in the workflow have a corresponding row in your schema?
- Is the fraud-check row attributable to a specific model version and reproducible from its prompt fingerprint?
- Does your walkthrough script include the biometric-confirmation event as a row referenced by the next action's `hitl_approval_id`?
- Could a reviewer working only from your audit table reconstruct user B's balance history?

**What good looks like:** seven or eight rows — initiate, fraud-evaluate (with model fields), confirm-biometric, debit-A, credit-B, reconcile, post — each with a named actor, controlled-vocabulary action, and before/after state. The fraud-check row carries model name, prompt hash, temperature, and confidence score. The biometric row is referenced by the debit-A action as `hitl_approval_id`. You've identified a late-write risk (likely credit-B crossing a service boundary) and proposed an outbox-pattern mitigation. If your schema has a `description TEXT` column where the action lives, that's a free-text action problem — fix to a controlled vocabulary.

## 8. Key Takeaways

- Could I walk an auditor through a single real recent transaction in five minutes, end-to-end, with a row to anchor every state transition?
- Does every audit row carry actor, timestamp, action, target, and before/after state?
- For every AI-touched path, do my rows carry model, prompt fingerprint, temperature, HITL-approval link, and assistance annotation?
- Is my audit table append-only at the *database* layer, not just at the application layer?
- Does my Findings tracker surface named gaps and link to evidence rows in two clicks?

## Sources

1. [How to Use Audit Walkthroughs to Find Control Weaknesses (CPA Hall Talk)](https://cpahalltalk.com/audit-walkthroughs/) — retrieved 2026-05-26
2. [Immutable Audit Trails: A Complete Guide (Hubifi)](https://www.hubifi.com/blog/immutable-audit-log-basics) — retrieved 2026-05-26
3. [Audit Trails as Evidence: From Logs to Proof (Cyber Security Intelligence)](https://www.cybersecurityintelligence.com/blog/audit-trails-as-evidence-from-logs-to-proof-8989.html) — retrieved 2026-05-26
4. [What is Audit Trails? 2026 Guide (DevSecOps School)](https://devsecopsschool.com/blog/audit-trails/) — retrieved 2026-05-26
5. [SOX-compliant audit trails with OpenTelemetry (OneUptime)](https://oneuptime.com/blog/post/2026-02-06-sox-compliant-audit-trails-opentelemetry/view) — retrieved 2026-05-26

Last verified: 2026-05-26
