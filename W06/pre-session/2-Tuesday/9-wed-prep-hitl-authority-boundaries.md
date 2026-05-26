---
week: W06
day: Tue
topic_slug: wed-prep-hitl-authority-boundaries
topic_title: "Wed-prep — HITL authority boundaries (Wed's capstone framing)"
parent_overview: W06/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://genai.owasp.org/llmrisk/llm06-sensitive-information-disclosure/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.elementum.ai/blog/human-in-the-loop-agentic-ai
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.kiteworks.com/regulatory-compliance/human-in-the-loop-ai-compliance/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.strata.io/blog/agentic-identity/practicing-the-human-in-the-loop/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://agentcenter.cloud/blogs/enterprise-ai-agent-governance-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# Wed-prep — HITL authority boundaries (Wed's capstone framing)

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define the four authority levels (`full-auto` / `propose-and-await-approval` / `escalate-only` / `never-AI`) and place actions into the appropriate level based on reversibility and consequence.
- Distinguish the three oversight models (Human-in-the-Loop / Human-on-the-Loop / Human-over-the-Loop) and explain when each fits.
- Map agent actions to authority levels using a decision matrix the audit-oriented reader can verify.
- Recognise OWASP LLM06 Excessive Agency and its three root causes (excessive functionality, excessive permissions, excessive autonomy).
- Build a HITL-authority-boundaries document that lets a reviewer see "what does this system let AI do alone, and what doesn't it?" in 5 minutes.

## 2. Introduction

A capstone document for any AI-assisted system is the *HITL authority boundaries* matrix — the page that names, for each AI-touched action in the system, what the AI is permitted to do without human review and what it must escalate. For an audit-oriented audience, this is often the single most-read document; it answers the question every FedRAMP reviewer, every model-risk committee, and every Karsun-style manager asks: *"what does this system let the AI do alone, and what doesn't it?"*

The 2025–2026 governance literature is converging on this discipline. The Human-Governed Automation Loop framework describes "decision delegation boundaries" as dynamically evaluated multi-dimensional constraints — model confidence, downstream impact, contextual sensitivity, historical reliability ([Elementum AI on HITL agentic AI, retrieved 2026-05-26](https://www.elementum.ai/blog/human-in-the-loop-agentic-ai)). Enterprise governance guides describe explicit "authority matrices" mapping agent actions to action types, value thresholds, and confidence levels ([AgentCenter on Enterprise AI Agent Governance 2026, retrieved 2026-05-26](https://agentcenter.cloud/blogs/enterprise-ai-agent-governance-2026)). OWASP's LLM06 Excessive Agency category in the 2025 Top 10 list specifies the failure mode this discipline prevents ([OWASP LLM06, retrieved 2026-05-26](https://genai.owasp.org/llmrisk/llm06-sensitive-information-disclosure/)).

The bridge concept is the four authority levels named in the overview. They function as a *controls vocabulary* (see the controls-language reading) — a small, finite, well-defined set of states that every AI-touched action maps to. The vocabulary lets the audit-oriented reader make a single judgement per action ("is the assigned authority level appropriate for this action's reversibility?") rather than re-reasoning from first principles each time.

This reading covers the discipline generically. The day's overview applies it to the W6 Wed capstone document `docs/hitl-authority-boundaries.md`; this reading defines the underlying technique.

## 3. Core Concepts

### 3.1 The four authority levels

The vocabulary:

1. **`full-auto`** — the AI acts without human review for this action. Reserved for reversible, low-stakes, high-volume actions where downstream impact is bounded and recovery is fast.
2. **`propose-and-await-approval`** — the AI drafts; a human reviews and approves before the action takes effect. The default for any action with downstream consequence. The human approval event is itself an audit-trail row that references the AI proposal.
3. **`escalate-only`** — the AI does not act; it identifies a condition and surfaces it to a human owner with a recommendation. The human decides whether to take any action at all, and what action.
4. **`never-AI`** — a class of actions the system is structurally prohibited from taking, typically because a regulation, policy, or contract assigns the decision to a named role. The AI may *inform* but never act in this class.

Each action in the system is assigned exactly one level. The assignment is published in the matrix. The matrix is the *contract* between the system and its audit-oriented readers.

### 3.2 The three oversight models

The HITL literature also distinguishes three broader oversight models ([Strata 2026 Guide to HITL, retrieved 2026-05-26](https://www.strata.io/blog/agentic-identity/practicing-the-human-in-the-loop/); [Kiteworks on HITL Compliance, retrieved 2026-05-26](https://www.kiteworks.com/regulatory-compliance/human-in-the-loop-ai-compliance/)):

- **Human-in-the-Loop (HITL)** — a human must approve or authorize the action *before* execution. The system pauses at defined checkpoints. Maps to `propose-and-await-approval` at the action level.
- **Human-on-the-Loop (HOTL)** — the AI acts autonomously; a human monitors output and intervenes *after the fact*. Maps to `full-auto` with monitoring at the action level. Acceptable for medium-risk, reversible actions where speed matters.
- **Human-over-the-Loop** — humans set the policies and boundaries; the AI operates independently *within them*. The oversight is at the policy level, not the action level. Maps to a *system* assigned `full-auto` for many actions but where the policy boundaries themselves are continuously reviewed by humans.

The action-level authority assignment (the four levels) sits inside whichever oversight model the system is operating under. A system can use HOTL for low-stakes actions and HITL for high-stakes actions within the same product.

### 3.3 Decision factors for assigning authority levels

Five factors converged on in the 2026 enterprise-governance literature:

- **Reversibility** — can the action be undone, and at what cost? Irreversible skews toward `propose-and-await-approval` or higher.
- **Downstream impact scope** — wider impact skews toward higher oversight.
- **Material consequence** — financial, regulatory, contractual, or safety effects skew toward `propose-and-await-approval` minimum.
- **Model confidence** — some platforms route dynamically (high-confidence `full-auto`, low-confidence to approval). Requires mature observability; avoid early.
- **Regulatory or policy assignment** — when a clause assigns a decision to a named role (e.g., FAR 15.308's SSA independent-judgment requirement), the mapping is `never-AI`.

The matrix should *show* the driving factor so reviewers can verify the logic.

### 3.4 OWASP LLM06 — Excessive Agency

OWASP LLM06 (2025 v2.0) names three root causes ([OWASP LLM06, retrieved 2026-05-26](https://genai.owasp.org/llmrisk/llm06-sensitive-information-disclosure/)):

- **Excessive functionality** — agent has tools beyond its purpose. Mitigation: minimum tool inventory.
- **Excessive permissions** — agent runs with broader credentials than needed. Mitigation: least privilege; per-agent service accounts; JIT ephemeral tokens.
- **Excessive autonomy** — agent executes high-impact actions without HITL. Mitigation: `propose-and-await-approval` as default for consequential actions ([Indusface, retrieved 2026-05-26](https://www.indusface.com/learning/owasp-llm-excessive-agency/)).

The boundaries matrix directly addresses the third root cause and surfaces the first two — mapping every action lets the team review needed capabilities and required credentials.

### 3.5 The matrix structure

A workable structure for the boundary document:

```
docs/hitl-authority-boundaries.md

§1 System summary (one paragraph — what does this system do)
§2 Oversight model (HITL / HOTL / Human-over-the-Loop; per-action exceptions)
§3 The matrix:

| Endpoint / action       | Authority level             | Driving factor       | Approval role         | Audit-row pattern    |
|-------------------------|-----------------------------|----------------------|-----------------------|----------------------|
| /draft-response         | propose-and-await-approval  | Material consequence | Reviewing analyst     | proposed + approved  |
| /classify-intent        | full-auto                   | Reversible, low      | (none)                | acted (with monitor) |
| /escalate-anomaly       | escalate-only               | Confidence + scope   | On-call security      | recommended          |
| /post-ledger            | never-AI                    | Regulation (X §Y)    | Authorised role only  | (no AI action row)   |
| ... [one row per action]

§4 Tool-and-permission inventory per agent (LLM06 functionality + permissions)
§5 Open questions / dynamic-confidence considerations
```

Five columns, one row per action. The reader scans the matrix and immediately sees: which actions are `full-auto` (the surface area for excessive-autonomy concerns); which actions are `propose-and-await-approval` (the HITL touchpoints); which actions are `escalate-only` (the human-recommendation surface); which actions are `never-AI` (the regulatory or policy boundaries).

### 3.6 Per-action implementation patterns

- **`full-auto`:** action takes effect; AI is the actor on the audit row; monitoring alerts fire on anomalies.
- **`propose-and-await-approval`:** AI writes a proposal record (not the final action); a human reviews and approves via UI; on approval, the action takes effect with *two* audit rows — proposal + approval. LangGraph and similar frameworks support this via state-managed interrupts ([OWASP LLM06, retrieved 2026-05-26](https://genai.owasp.org/llmrisk/llm06-sensitive-information-disclosure/)).
- **`escalate-only`:** AI writes an escalation record with a recommendation; the human-owner queue receives it; the human decides the action.
- **`never-AI`:** AI may inform (surface context, draft analysis) but the action is performed by the human via a separate non-AI surface. Audit trail has the human as sole actor; AI artifacts are linked but not chained.

The implementation matters because it determines whether the audit-trail walkthrough surfaces HITL approval cleanly. A `propose-and-await-approval` action that doesn't generate both rows defeats the purpose.

## 4. Generic Implementation

A worked example outside federal acquisitions: a generic enterprise IT-operations platform with an AI assistant that helps the on-call SRE.

**System summary:** an AI assistant ingests alerts, proposes remediation actions, and (with approval) executes them via the platform's automation interface.

**Oversight model:** HITL for any action that modifies infrastructure state; HOTL for advisory / read-only actions; Human-over-the-Loop for the policy that defines which alert types the assistant is permitted to engage with at all.

**The matrix:**

| Endpoint / action | Authority level | Driving factor | Approval role | Audit-row pattern |
|---|---|---|---|---|
| `/summarise-alert` | `full-auto` | Read-only; reversible | (none) | acted (with monitor) |
| `/suggest-runbook` | `full-auto` | Advisory only | (none) | acted |
| `/restart-process` | `propose-and-await-approval` | Material; reversible-but-disruptive | On-call SRE | proposed + approved |
| `/scale-cluster-up` | `propose-and-await-approval` | Cost + capacity | On-call SRE | proposed + approved |
| `/scale-cluster-down` | `propose-and-await-approval` | Risk of saturation | On-call SRE | proposed + approved |
| `/rollback-deploy` | `propose-and-await-approval` | High blast radius | On-call SRE + engineering manager | proposed + 2-of-2 approval |
| `/escalate-to-incident-commander` | `escalate-only` | Confidence + impact | Incident commander | recommended |
| `/page-executive` | `escalate-only` | Material | Incident commander only | recommended |
| `/post-public-status-update` | `never-AI` | Brand + legal | Comms lead only | (no AI action row) |
| `/issue-customer-credits` | `never-AI` | Financial; contractual | Authorised role only | (no AI action row) |

Reading top to bottom: `full-auto` is confined to read-only / advisory. Every infrastructure-modifying action requires approval; `/rollback-deploy` requires *two* approvers. Public communications and financial actions are `never-AI` for regulatory and contractual reasons.

**Tool-and-permission inventory:** the `ops-assistant-agent` has four tools — `read-metrics`, `read-runbooks`, `read-alert-history`, `propose-action`. It does *not* have `execute-action`, `post-status`, or `issue-credit`. Service account `svc-ops-assistant` has IAM permissions `read:metrics`, `read:runbooks`, `write:proposals` only. Per LLM06: bounded functionality, least privilege, no autonomous write path.

**Audit-trail consequence:** every `/restart-process` event generates two rows — the proposal (actor: `svc-ops-assistant`) and the approval (actor: SRE user ID, `hitl_approval_target_id` pointing back to the proposal). The walkthrough surfaces both in order.

## 5. Real-world Patterns

**Aviation autopilot.** Commercial autopilots operate across authority levels: `full-auto` for cruise heading and altitude (pilots monitor); `full-auto` with strict envelope for autoland (outside envelope reverts to pilot); escalate-to-pilot for abnormal conditions; `never-automation` for declaring emergency or requesting priority. The autopilot's authority matrix has been formalised over decades and is the model AI-governance literature now adapts.

**Hospital pharmacy — medication-order verification.** Pharmacy automation screens orders for interactions, allergies, dosing. It can `full-auto` reject obviously bad orders; `propose-and-await-approval` for ambiguous cases; `escalate-only` for new-drug or unusual dosing; `never-AI` for controlled-substance release (licensed pharmacist only). Four-level mapping, exact.

**Trading systems — pre-trade risk checks.** Equity-trading platforms layer authority: ultra-low-latency small orders `full-auto`; mid-sized go through `propose-and-await-approval` against risk algorithms; large blocks require explicit trader authorisation; certain order types require trader-and-supervisor approval; insider-restricted symbols are `never-automation`. Action-by-action assignment driven by reversibility, size, and regulation.

**Customer-support refund tiers.** Support platforms expose tiered refund authority to AI agents: low-dollar refunds `full-auto`; mid-range `propose-and-await-approval`; above threshold `escalate-only` to a manager; fraud-flagged or dispute-in-progress accounts `never-AI`. Dollar threshold is the driving factor; approver role varies by tier.

## 6. Best Practices

- **Default to `propose-and-await-approval`** for any action with downstream consequence. Promotions to `full-auto` are intentional, evidence-based, and reviewable; demotions are easy to forget.
- **Make the matrix the canonical document.** When implementation, runbook, attestation, or eval report references HITL behaviour, they cite the matrix — not duplicate it.
- **Scope every agent's tool inventory and permissions to the minimum required** (LLM06 functionality and permissions root causes). Review on a cycle.
- **Generate two audit rows for every `propose-and-await-approval` event** — the proposal and the approval, with bidirectional linkage.
- **Document `never-AI` boundaries with the citation** (regulation, policy, contract clause). The reviewer can verify the boundary is real and current.
- **Review the matrix when the system gains new tools, endpoints, or integrations.** Authority levels are not set-and-forget; they evolve with the system.
- **Show the driving-factor column.** A matrix without driving factors is unverifiable; a matrix with them lets the reviewer dispute logic productively.

## 7. Hands-on Exercise

**Time:** 15 minutes. **Format:** matrix draft.

You are designing the HITL authority-boundaries matrix for a generic AI assistant embedded in a corporate HR platform. The assistant helps managers draft performance-review summaries, suggests training resources, and can update employee records on request.

**Steps:**

1. **List 8–10 actions** the assistant can perform. Include at least one each of: a read-only action, a draft-content action, an employee-record-update action, an HR-policy-decision action.
2. **Assign each action to one of the four authority levels.** Use the driving factors from §3.3 (reversibility, downstream impact, material consequence, regulatory/policy assignment).
3. **For each `propose-and-await-approval` action, name the approver role.** For each `never-AI` action, name the regulatory or policy reason.
4. **Identify which OWASP LLM06 root cause your matrix most actively prevents** and explain in one sentence.

**Self-check:**

- Is every action mapped to *exactly one* level (not "depends on situation")?
- Does the matrix include at least one `never-AI` action with a policy or regulation citation?
- For `propose-and-await-approval` actions, does the approver role have appropriate authority for the action's impact?
- Would a reviewer reading the matrix in 5 minutes know what the assistant is and is not permitted to do alone?

**What good looks like:** 8–10 rows, each with action, authority level, driving factor, approver role (or "n/a"), and citation if applicable. The matrix shows clear gradient: read actions are `full-auto`, draft actions are `full-auto` (drafts are inherently reversible) or `propose-and-await-approval` if they auto-publish, record-updates are `propose-and-await-approval` with the affected employee's manager as approver, HR-policy decisions (termination, compensation changes) are `never-AI` with citation to HR policy. If your matrix has *no* `never-AI` actions, you've likely missed a regulatory or policy constraint — most enterprise AI systems have at least one. Add it.

## 8. Key Takeaways

- Can I name the four authority levels (`full-auto`, `propose-and-await-approval`, `escalate-only`, `never-AI`) and which actions in my system map to each?
- Does my system's HITL behaviour follow the principle of "default to `propose-and-await-approval` for any action with downstream consequence"?
- Have I scoped my agents' tool inventory and permissions to the minimum required (OWASP LLM06 functionality and permissions)?
- Do `propose-and-await-approval` actions generate two audit rows with bidirectional linkage?
- Does my HITL-authority-boundaries document let a reviewer answer "what does this system let AI do alone?" in 5 minutes?

## Sources

1. [OWASP LLM06: Excessive Agency (OWASP Gen AI Security Project)](https://genai.owasp.org/llmrisk/llm06-sensitive-information-disclosure/) — retrieved 2026-05-26
2. [Human-in-the-Loop Agentic AI: When You Need Both (Elementum AI)](https://www.elementum.ai/blog/human-in-the-loop-agentic-ai) — retrieved 2026-05-26
3. [Human in the Loop: AI Compliance (Kiteworks)](https://www.kiteworks.com/regulatory-compliance/human-in-the-loop-ai-compliance/) — retrieved 2026-05-26
4. [Human-in-the-Loop: A 2026 Guide to AI Oversight (Strata)](https://www.strata.io/blog/agentic-identity/practicing-the-human-in-the-loop/) — retrieved 2026-05-26
5. [Enterprise AI Agent Governance: A 2026 Framework (AgentCenter)](https://agentcenter.cloud/blogs/enterprise-ai-agent-governance-2026) — retrieved 2026-05-26

Last verified: 2026-05-26
