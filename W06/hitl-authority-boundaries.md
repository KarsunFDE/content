---
template: w6-capstone-synthesis
week: W06
artifact_class: standalone-capstone
synthesizes:
  - D-043 (6 HITL touchpoints, original)
  - D-044 (HITL touchpoint #1 added at W1 Fri — total 7)
  - acquire-gov ai-orchestrator (8 AI endpoints)
  - pair-project AI endpoints (varies — 1-3 per pair)
audience:
  - next FDE picking up the pair-project repo
  - Karsun manager (5-min scan)
  - OIG-style auditor (30-min depth)
  - Cohort #2 (inheriting the synthesis as baseline)
ship_target: Wed W6 (capstone deliverable)
last_verified: 2026-05-23
---

> **What this artifact is.** The W6 **capstone synthesis** — the one document that answers "for every AI endpoint in our system, what is the AI allowed to do alone, what must propose-and-await human approval, what must escalate, and what is forbidden?" — across all 7 HITL programme touchpoints (D-043 + D-044). Each pair instantiates this template as `docs/hitl-authority-boundaries.md` in their pair-project repo Wed W6 afternoon. The synthesis IS the W6 deliverable per Phase 2 agent brief.

# HITL Authority Boundaries — `<pair-project-name>`

Authored: <YYYY-MM-DD> by <Pair-N>. Last verified: <YYYY-MM-DD>.

## §0 — How to read this doc (Karsun-manager / OIG-auditor / next-FDE)

**5-min scan:** read §1 (the authority levels) + §2 (the matrix). That tells you what AI does alone.

**30-min depth:** read §3 (per-endpoint detail) + §4 (per-touchpoint narrative). That tells you why.

**Reference scan:** §5 cross-references (runbook, security-attestation, eval-report, ADR catalog).

## §1 — Authority levels (the columns of the matrix)

| Level | Definition | Audit-log requirement | Reversibility expectation |
|-------|-----------|------------------------|---------------------------|
| **full-auto** | AI executes without human approval; result logged | Result logged with `decision_authority: ai` | Reversible OR low blast-radius |
| **propose-and-await-approval** | AI proposes; human approves before persistence | Both proposal + approval logged with `decision_authority: human, ai_proposed: true` | Any (human gates persistence) |
| **escalate-only** | AI flags + suspends action; routes to human queue | Flag event logged; human-action event logged separately | Any |
| **never-AI** | Regulation or pair-decision forbids AI execution | AI may DRAFT (logged with `is_draft: true`); cannot DECIDE | N/A — humans decide |

## §2 — The matrix — acquire-gov ai-orchestrator endpoints × 7 HITL touchpoints

Rows = endpoints. Columns = HITL touchpoints (per D-043 + D-044). Each cell = authority level OR `N/A` (touchpoint doesn't apply to that endpoint).

| Endpoint | #1 W1F LLM-Essentials | #2 W2T RAG-fallback | #3 W3M Plan-Day ADR | #4 W3W Multi-agent | #5 W3T LangGraph | #6 W4W OWASP LLM06 | #7 W5W AIOps |
|----------|------------------|------------------|---------------------|--------------------|-----------------|-----------------|--------------|
| `POST /draft-solicitation` | **propose-and-await** (CO confirms before publish; hallucination-flag on FAR-clause citations below confidence threshold) | N/A (not RAG-primary) | `interrupt_before_node` ADR | N/A | N/A | **propose-and-await** (irreversible — publishes to SAM.gov-equivalent) | full-auto (drafting only) |
| `POST /draft-amendment` | propose-and-await (CO confirms before publish) | N/A | `interrupt_before_node` at publish step | N/A | N/A | **propose-and-await** (FAR 15.206 amendment-acknowledgement; irreversible) | full-auto (drafting only) |
| `POST /answer-qa` | propose-and-await (CO confirms before vendor-publish) | **escalate-only** (RAG fallback at faithfulness < 0.80 → CO review queue) | `interrupt_before_node` at vendor-publish | N/A | N/A | propose-and-await | full-auto (drafting only) |
| `POST /eval/ssdd-draft` | propose-and-await (SSA confirms before sign) | escalate-only (RAG fallback) | `interrupt_before_node` ADR at SSA-review | N/A | **hard-interrupt** at SSA-review node | propose-and-await | full-auto (drafting only) |
| `POST /rag/clause-search` | N/A | escalate-only at faithfulness < 0.80 | N/A | N/A | N/A | full-auto (read-only retrieval) | full-auto |
| `POST /agent/intake-triage` | N/A | N/A | `interrupt_before_node` at TEP-routing | **propose-and-await** at TEP-routing (irreversible delegation) | hard-interrupt at TEP-routing | propose-and-await | full-auto (drafting only) |
| `POST /eval/consensus` | N/A | N/A | `interrupt_before_node` at consensus-complete | propose-and-await at consensus-complete | **hard-interrupt** at consensus-complete | propose-and-await | full-auto |
| `POST /eval/ssa-review` | N/A | N/A | N/A | N/A | hard-interrupt at award-ready | propose-and-await for narrative; **never-AI for decision** | **never-AI for decision step** (FAR 15.308 — SSA cannot delegate) |

### Pair-project AI endpoints (instantiate per pair)

| Endpoint | #1 | #2 | #3 | #4 | #5 | #6 | #7 |
|----------|----|----|----|----|----|----|----|
| `POST /<pair-endpoint-1>` | <level> | <level> | <level> | <level> | <level> | <level> | <level> |
| `POST /<pair-endpoint-2>` | <level> | <level> | <level> | <level> | <level> | <level> | <level> |
| `POST /<pair-endpoint-3>` | <level> | <level> | <level> | <level> | <level> | <level> | <level> |

## §3 — Per-endpoint detail

### `POST /draft-solicitation`

**Function:** drafts Section C SOW + Section L instructions from a requirements package + retrieved FAR/DFARS clauses.

**HITL surfaces:**
- **#1 (W1 Fri)** — hallucination-flag triggers when output cites a CFR/FAR reference with retrieval-confidence < 0.85. CO sees flagged-output banner in `/solicitations/:id/draft` UI. Audit log: `solicitation.draft.flagged`.
- **#3 (W3 Mon Plan-Day ADR)** — pair committed `interrupt_before_node` at publish step. ADR-W3-001 in catalog.
- **#6 (W4 Wed OWASP LLM06)** — Excessive-Agency defence: AI cannot publish without CO approval. Publish endpoint requires `X-Approval-Token: <human-approval-id>` header from `/admin/approvals/<id>`.
- **#7 (W5 Wed AIOps)** — Bedrock cost ceiling per-tenant; on breach, endpoint degrades to queue-mode + escalates to sys_admin.

**Authority-level summary:** `propose-and-await-approval` at publish step. `full-auto` at draft step.

**Audit-log shape:**
```
{
  "event": "solicitation.draft.created",
  "actor": "ai-orchestrator",
  "decision_authority": "ai",
  "is_draft": true,
  "agency_id": "<tenant>",
  "solicitation_id": "<id>",
  "correlation_id": "<W3C traceparent>"
}
// followed at publish-step by:
{
  "event": "solicitation.draft.published",
  "actor": "<CO user_id>",
  "decision_authority": "human",
  "ai_proposed": true,
  "approval_id": "<id>",
  "solicitation_id": "<id>",
  "correlation_id": "<W3C traceparent>"
}
```

### `POST /draft-amendment`

**Function:** drafts amendment under FAR 15.206; predicts vendor-impact (re-acknowledgement count + schedule effect).

**HITL surfaces:**
- **#1 (W1 Fri)** — hallucination-flag on FAR-clause citations.
- **#3 (W3 Mon Plan-Day ADR)** — interrupt_before_node at publish step (ADR-W3-002).
- **#6 (W4 Wed OWASP LLM06)** — Excessive-Agency: irreversible amendment publish requires CO approval + audit-logged with vendor-re-acknowledgement count prediction.

**Authority-level summary:** `propose-and-await-approval` at publish step. NEVER `full-auto` at publish (FAR 15.206 requires CO sign-off).

### `POST /eval/ssa-review`

**Function:** SSA review of the SSDD draft; outputs award decision.

**HITL surfaces:**
- **#5 (W3 Thu LangGraph)** — hard-interrupt at award-ready node. SSA receives notification; opens `/evaluation/:id/ssa-review`; either approves SSDD draft or rejects (which loops back to consensus-agent).
- **#6 (W4 Wed OWASP LLM06)** — narrative-level propose-and-await for the SSDD draft text. AI CANNOT mark the award decision.
- **#7 (W5 Wed AIOps)** — **never-AI for the decision step.** FAR 15.308 — "The Source Selection Authority shall be a single individual ... shall be responsible for the proper and efficient conduct of source selection." [FAR 15.308 Source selection decision, retrieved 2026-05-23, https://www.acquisition.gov/far/15.308]

**Authority-level summary:** `propose-and-await-approval` for narrative drafting. **`never-AI` for the decision step** — AI cannot mark `award_decision`. Any attempt to set `award_decision` via AI-orchestrator results in 403 + audit-log event `ssa.decision.ai_attempt_blocked`.

**This is the regulatory red-line of the synthesis.** Misclassification here is a P0 codex finding + Thu gate-failure trigger.

### <Repeat for remaining 5 acquire-gov endpoints + pair-project endpoints>

## §4 — Per-touchpoint narrative

### Touchpoint #1 — W1 Fri LLM Essentials (hallucination-flagging)

**Design intent:** when LLM output cites authority that retrieval can't verify, flag the output for human review before vendor-/CO-facing publication.

**Endpoints exhibiting #1:** `/draft-solicitation`, `/draft-amendment`, `/answer-qa`, `/eval/ssdd-draft`.

**Authority-level pattern:** `propose-and-await-approval` at publish step. Drafting itself stays `full-auto` because drafts are reversible (until published).

### Touchpoint #2 — W2 Thu RAG fallback

**Design intent:** when retrieval confidence falls below threshold (RAGAS faithfulness < 0.80 OR context-precision < 0.70), escalate to human queue rather than ship a guess.

**Endpoints exhibiting #2:** `/answer-qa`, `/rag/clause-search`, `/eval/ssdd-draft`.

**Authority-level pattern:** `escalate-only` — flagged output routes to CO review queue at `/admin/rag-fallback-queue`.

### Touchpoint #3 — W3 Mon Plan-Day ADR

**Design intent:** at pair-Plan-Day, pair commits HITL interrupt-node boundaries as ADRs — *which agent actions require a human gate*. Captured as code in the LangGraph state machine.

**Endpoints exhibiting #3:** all agentic endpoints (`/agent/intake-triage`, `/eval/consensus`, `/eval/ssa-review`, plus `/draft-*` endpoints wired through LangGraph).

**Authority-level pattern:** Each pair's ADR catalog (`docs/adrs/INDEX.md`) cross-references the `interrupt_before_node` placements. Pair-specific.

### Touchpoint #4 — W3 Wed Multi-agent handoffs

**Design intent:** when a supervisor agent must delegate an irreversible task to a sub-agent, require explicit human approval before delegation.

**Endpoints exhibiting #4:** `/agent/intake-triage` (TEP routing).

**Authority-level pattern:** `propose-and-await-approval` at the delegation boundary.

### Touchpoint #5 — W3 Thu LangGraph deep-dive

**Design intent:** soft-interrupts vs hard-interrupts in the LangGraph state machine. Audit-trail of human input. Resuming graphs after approval.

**Endpoints exhibiting #5:** `/agent/intake-triage`, `/eval/consensus`, `/eval/ssa-review`.

**Authority-level pattern:** hard-interrupts at irreversible transitions (consensus-complete, award-ready). Soft-interrupts (timeout-after-warning) at reversible ones.

### Touchpoint #6 — W4 Wed OWASP LLM06 Excessive Agency

**Design intent:** narrow tool scopes; HITL on irreversible actions; LLM-orchestrated actions cannot exceed the explicit grant of authority.

**Endpoints exhibiting #6:** all 8 ai-orchestrator endpoints + pair-project endpoints (universal).

**Authority-level pattern:** any endpoint with an irreversible side-effect (publish, sign, delegate, award) must be `propose-and-await-approval` OR `never-AI`. Reversible reads stay `full-auto`.

### Touchpoint #7 — W5 Wed AIOps auto-remediation

**Design intent:** what does the AI-SRE have authority to do alone vs what must escalate? The canonical case is the **auto-fix-the-audit-gap decision** — when AIOps detects an Item 2 audit-log race gap, does the platform auto-replay the missing AuditEvent, escalate to sys_admin, or page the on-call?

**Endpoints exhibiting #7:** the AIOps auto-remediation surface itself (NOT the 8 ai-orchestrator endpoints directly — but the AIOps-detector observes them).

**Authority-level pattern (per pair):**
- **Auto-replay missing AuditEvent from outbox** → `full-auto` IFF the missing event is < 5 minutes stale AND can be reconstructed from the outbox table.
- **Quarantine the pod + alert on-call** → `escalate-only`.
- **Modify production data outside the outbox replay path** → `never-AI`.

## §5 — Cross-references

- `docs/runbook.md` §A — incident-response signals + on-call action (this doc names the AUTHORITY level; runbook names the ACTION).
- `docs/security-attestation.md` §FedRAMP-AC-2 — RBAC + decision-authority enforcement.
- `docs/security-attestation.md` §FedRAMP-AU-2 — audit-log requirements per authority level.
- `docs/security-attestation.md` §OWASP-LLM06 — Excessive-Agency attestation.
- `eval/REPORT.md` §4 — HITL approval rates per endpoint (intake-triage routing, evaluator→consensus agreement, SSDD-draft approval).
- `docs/adrs/INDEX.md` — all `interrupt_before_node` ADR placements.
- W3 LangGraph state-machine code: `<pair-project>/ai-orchestrator/graphs/<flow>.py` — the actual interrupt-node placements in code.

## §6 — Known gaps + Phase 3 commitments

(Pair fills in honestly.)

- **Gap 1:** <e.g., "Auto-replay logic for Item 2 race is currently `escalate-only`; full-auto target requires the outbox table re-tested under load. Open `Finding/F-2026-W6-005`. Phase 3 ETA: week 4.">
- **Gap 2:** <e.g., "Pair-project endpoint `/foia/redact-suggest` HITL authority not yet locked — currently `propose-and-await` provisional; pair to commit final classification by Phase 3 week 2.">
- **Gap 3:** <text>

## §7 — Why this synthesis matters (the 30-second pitch)

For a Karsun manager staffing this onto a federal engagement next quarter: this doc is the single artifact that answers *"is the AI in this system doing things humans should be doing?"* — the question every federal CIO asks first and every OIG auditor probes deepest.

For a next FDE: this doc is the entry-point to the system's *authority model*. Before changing any AI behavior, you read this doc to know what authority level you're working in.

For Cohort #2: this template is the inheritance. Refine it; don't recreate from scratch.
