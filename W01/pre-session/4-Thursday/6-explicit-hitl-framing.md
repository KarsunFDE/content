---
week: W01
day: Thu
topic_slug: explicit-hitl-framing
topic_title: "Explicit HITL framing — Friday's named topic"
parent_overview: W01/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 8
sources:
  - url: https://www.nist.gov/itl/ai-risk-management-framework
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://genai.owasp.org/llmrisk/llm06-excessive-agency/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/oss/python/langgraph/add-human-in-the-loop
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.anthropic.com/rsp-updates
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/oss/python/langgraph/interrupts
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# Explicit HITL framing — Friday's named topic

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define Human-in-the-Loop (HITL), Human-on-the-Loop, and full autonomy and place a given LLM action on the spectrum.
- Apply the reversibility + blast-radius framework to decide whether a specific action requires a human gate.
- Distinguish a *workflow* HITL gate (decision approval) from a *runtime* HITL interrupt (mid-execution pause).
- Identify the failure modes of "always-approve" gates (rubber-stamping) and "never-approve" workflows (silent over-action).
- Recognise the seven HITL touchpoints in the cohort's 6-week arc and the role each one plays.

## 2. Introduction

LLMs can act. The question is when they should pause and ask a human first. Industry and regulatory guidance throughout 2025–2026 — from the NIST AI Risk Management Framework to OWASP LLM06 (Excessive Agency) to Anthropic's Responsible Scaling Policy — has converged on a two-axis decision frame: *reversibility* (can the action be undone if wrong?) and *blast radius* (how many people or systems does it affect?).

Naive HITL design picks the wrong default. "Always gate everything" produces approval-fatigue: humans rubber-stamp because the cost of reading each request exceeds the rate of catching real errors. "Never gate anything" produces silent over-action — by the time the harm is visible, the agent has already shipped the bad result downstream. The correct shape is *selective*, *justified*, and *measured* gates, deployed where the framework says they're load-bearing.

This reading is the first of seven HITL touchpoints across the cohort's six weeks. Each touchpoint surfaces a different facet of the same discipline; together they constitute the cohort's HITL competency. Friday's exercise is small — each pair commits one HITL decision in their Phase 1 plan-spec — but the framework you bring to that decision is the framework you'll exercise across all seven touchpoints.

## 3. Core Concepts

### 3a. The HITL spectrum

The 2026 literature distinguishes three operating modes, not two:

- **Human-in-the-Loop (HITL).** A human approves each agent action before it executes. Slow but high-assurance. Appropriate for high-stakes, low-frequency actions.
- **Human-on-the-Loop (HOTL).** The agent acts autonomously, but a human is notified and can intervene. Faster than HITL, but the cost of a wrong action is the cost of detecting + reverting it, not zero. Appropriate when action is reversible and notification latency is short.
- **Full autonomy.** No human gate or notification. Appropriate only for low-stakes, reversible actions where the rate of correct actions vastly exceeds the cost-weighted rate of incorrect ones.

A fourth mode worth knowing is **LLM-as-judge**, where another model approves the action. This is *not* the same as HITL; it's a defensible middle ground when human gating is operationally infeasible, but it carries its own failure modes (correlated errors between the actor model and judge model).

### 3b. The reversibility × blast-radius framework

The decision frame that NIST AI RMF and OWASP LLM06 both centre on:

```
                       small blast radius          large blast radius
                    ┌─────────────────────────┬──────────────────────────┐
   reversible       │  full autonomy OK       │  HOTL — notify, allow    │
                    │  (logs, searches,       │  fast revert (cache       │
                    │   read-only queries)    │   purges, soft locks)     │
                    ├─────────────────────────┼──────────────────────────┤
   irreversible     │  HOTL — log every       │  HITL — block, require   │
                    │  action, audit          │  approval (payments,     │
                    │  (drafts, internal      │  notifications to        │
                    │   notes)                │  external parties)       │
                    └─────────────────────────┴──────────────────────────┘
```

The rule of thumb that the 2026 guides converge on: any *irreversible* or *difficult-to-reverse* action should require human approval by default, regardless of the agent's confidence. Confidence is not a substitute for reversibility — a confident wrong agent is more dangerous than an uncertain one, because confidence calibration is the very property that fails silently.

### 3c. Workflow gates vs. runtime interrupts

Two operationally different shapes of HITL exist, and conflating them causes problems:

- **Workflow gate (decision-time).** The user submits a task, the agent assembles a plan, the *plan* is presented for human approval, execution begins only on approval. Familiar from incident-response runbooks. Best for tasks where the agent's plan is the load-bearing artifact and execution itself is mechanical.
- **Runtime interrupt (action-time).** The agent executes autonomously and *pauses* at specific tool calls or decisions. Human can approve, modify, or cancel. LangGraph's `interrupt` primitive is the canonical implementation. Best for tasks where the plan can't be fully specified up front and judgment is needed mid-execution.

The cohort exercises both. W3 Thu's LangGraph deep-dive is the canonical runtime-interrupt training. W1 Fri's plan-spec HITL decision is a workflow gate. Knowing which shape you're designing matters because they have different observability requirements (the workflow gate's audit trail is "approver + decision + timestamp"; the runtime interrupt's audit trail is "state snapshot + interrupt point + resume action").

### 3d. The seven HITL touchpoints in this cohort

The cohort arc weaves HITL through seven explicit touchpoints, each adding a facet to the discipline:

1. **W1 Fri — LLM Essentials.** Today. Each pair commits one HITL decision in the Phase 1 plan-spec. This is the framework primer.
2. **W2 Thu — RAG fallback.** When retrieval fails (no relevant documents found), what is the HITL fallback? Escalate to a human researcher? Return a "cannot answer" with a deflection? The decision shape is "how does the agent handle its own ignorance?"
3. **W3 Mon — Plan-Day ADR.** Document the HITL design as an Architecture Decision Record. Why this gate, here, with this approval policy?
4. **W3 Wed — Multi-agent handoffs.** When agent A finishes and agent B begins, where is the human gate? Between every handoff? Only at high-stakes ones? The decision shape is "HITL across agent boundaries."
5. **W3 Thu — LangGraph deep-dive.** The technical anchor. Implement `interrupt` in a working agent; design the resume-from-state semantics.
6. **W4 Wed — OWASP LLM06 (Excessive Agency).** HITL re-asserted as a security control. Excessive agency is a documented vulnerability class; HITL is one of the prescribed mitigations.
7. **W5 Wed — AIOps authority boundaries.** When an AI-SRE agent can auto-remediate, where are the authority boundaries? What blast radius can it touch alone? When must it page a human? The decision shape is "HITL for operational autonomy."

Each touchpoint builds on the prior. By W5 Wed the cohort can articulate a position on AIOps authority because they spent W1–W4 building the framework.

### 3e. The two failure modes to avoid

**Approval fatigue (rubber-stamping).** Humans approve too many low-information requests, so when a high-information request arrives they approve it on autopilot. The Machine Learning Mastery guide describes this as the dominant failure mode for over-gated systems. Mitigation: gate fewer things, but gate them genuinely. Measure the rate of "approvals that took less than N seconds to read" — that rate is your rubber-stamp rate.

**Silent over-action.** The system acts on a wrong agent decision, the action propagates to external systems, and by the time anyone notices, the damage is partially irrecoverable. Mitigation: instrument the *output* side, not just the *approval* side — if an agent action triggers a downstream side effect, log the effect at the boundary so post-hoc detection is possible. HOTL with strong logging beats no HITL.

## 4. Generic Implementation

A worked example outside federal acquisitions — a customer-success agent for a SaaS vendor:

The agent can take five actions:

| Action | Reversibility | Blast radius | Decision |
|---|---|---|---|
| Look up a customer's recent tickets | reversible (read-only) | small | full autonomy |
| Draft an internal note for the CSM | reversible (deletable) | small | full autonomy |
| Send the customer a follow-up email | hard to reverse (already sent) | medium (one customer) | HITL gate |
| Issue a service credit to the customer | reversible (revocable) | medium (one customer, finance audit) | HOTL — notify + auto-rollback if challenged |
| Escalate to executive sponsor | hard to reverse (relationship signal) | large (cross-org visibility) | HITL gate |

A LangGraph-style runtime-interrupt implementation (pseudo-code, framework-agnostic):

```python
def agent_step(state):
    next_action = plan_next_action(state)

    if requires_hitl(next_action):
        # Pause execution; return state to caller; wait for resume
        return {"status": "awaiting_approval",
                "action": next_action,
                "rationale": next_action.justification,
                "blast_radius": next_action.blast_radius,
                "reversibility": next_action.reversibility}

    if is_hotl(next_action):
        notify_human(next_action)              # fire-and-forget
        result = execute(next_action)
        log_for_post_hoc_review(next_action, result)
        return {"status": "executed", "result": result}

    # Full autonomy path
    result = execute(next_action)
    return {"status": "executed", "result": result}


def requires_hitl(action) -> bool:
    return action.reversibility == "hard" and action.blast_radius >= "medium"

def is_hotl(action) -> bool:
    return action.reversibility != "hard" and action.blast_radius >= "medium"
```

The decision logic is intentionally simple — two predicates, four classifications. Production systems extend this with confidence scoring, per-user role policies, and time-of-day rules, but the core frame stays the same: reversibility × blast radius drives the gate, not raw model confidence.

## 5. Real-world Patterns

**Fintech — automated investment portfolio rebalancing:** A robo-advisor's agent recommends portfolio rebalances. Below a threshold (e.g., 2% of portfolio value), the agent acts autonomously with email notification to the customer (HOTL). Above the threshold, it queues the recommendation for advisor review (HITL). The threshold is calibrated against the rate at which customers challenge unilateral changes — the goal is approval that catches real disputes without rubber-stamping.

**Healthcare — automated prior-authorisation drafting:** A medical-billing platform drafts prior-authorisation requests from clinical notes. The draft is *always* HITL — a clinician must approve before submission. The rubber-stamp risk is real: clinicians under time pressure approve drafts unread. The platform's response is to surface drafts where the agent's confidence is *low* prominently and de-emphasise high-confidence drafts to a one-click approve flow. The HITL discipline is selective attention, not uniform approval.

**E-commerce — automated refund decisions:** A retail platform's customer-service agent can issue refunds up to $50 autonomously (logged + spot-audited), between $50 and $500 with manager HOTL approval (notification + 1-hour challenge window before payment clears), and above $500 with synchronous HITL. The graduated structure matches financial exposure to approval friction.

**Gaming — automated community moderation:** A live-service game's moderation agent flags content. For temporary actions (mute, message delete), it acts autonomously with appeal queue (full autonomy + appeal HOTL). For account-affecting actions (bans, restrictions), it routes to a human moderator (HITL). The reversibility frame applies cleanly: mute is reversible, ban is socially irreversible even if technically revocable.

## 6. Best Practices

- Start with the reversibility × blast-radius matrix for every agent action; do not start from "what feels safe."
- Gate fewer things, but gate them genuinely — measure your rubber-stamp rate as a leading indicator of approval-fatigue.
- Log the *effect* of every HOTL action at the downstream boundary, not just the agent's intent — silent over-action is the failure mode HOTL most often hides.
- Distinguish workflow gates from runtime interrupts in your design — they need different audit trails and resume semantics.
- Treat HITL as a security control, not just a quality control — it's a documented mitigation for OWASP LLM06 (Excessive Agency).
- Calibrate gates over time — the right threshold is empirical and shifts as model and prompt change.
- Document HITL decisions in an ADR — future-you needs to understand why this gate exists at this exact action.

## 7. Hands-on Exercise

**Task (10–15 min):** For your pair's Phase 1 plan-spec, name *one* HITL decision and write it up in three short paragraphs:

1. **Which action requires the gate.** Be specific — not "outputs from the agent," but the named tool call, endpoint, or step.
2. **Why this action.** Apply the reversibility × blast-radius frame. Why is this combination load-bearing enough to gate, and the other actions in the same plan-spec not?
3. **What the gate looks like operationally.** Is it a workflow gate (approve the plan) or a runtime interrupt (pause at the tool call)? Who is the approver role? What's the audit trail?

**What good looks like:** The decision can be defended in 90 seconds to the rest of the cohort. The reversibility and blast-radius classifications are explicit, not implicit. The approver role is a *role* (not "the team") and is reachable within the workflow's latency budget. If you find yourself writing "this is just safer," go back to the matrix — that's the answer of someone who hasn't engaged the framework.

## 8. Key Takeaways

- *What's the difference between HITL, HOTL, and full autonomy?* HITL blocks until approval; HOTL acts but notifies for challenge; full autonomy acts silently. The cost of detection-and-revert determines which fits.
- *How do I decide whether to gate a given action?* Reversibility × blast radius. Irreversible large-blast-radius actions are gated by default; everything else is justified case-by-case.
- *What's the difference between a workflow gate and a runtime interrupt?* The workflow gate approves a plan up front; the runtime interrupt pauses mid-execution. Different audit trails, different resume semantics.
- *What are the two dominant failure modes of HITL design?* Rubber-stamping (gate too much, humans stop reading) and silent over-action (gate too little, harm propagates).
- *How does HITL connect across the cohort's six weeks?* Seven touchpoints — each adds a facet, from plan-spec gating in W1 to AIOps authority in W5.

## Sources

1. [AI Risk Management Framework (NIST)](https://www.nist.gov/itl/ai-risk-management-framework) — retrieved 2026-05-26
2. [LLM06: Excessive Agency (OWASP GenAI Top 10)](https://genai.owasp.org/llmrisk/llm06-excessive-agency/) — retrieved 2026-05-26
3. [Add human-in-the-loop controls (LangGraph official docs)](https://docs.langchain.com/oss/python/langgraph/add-human-in-the-loop) — retrieved 2026-05-26
4. [Responsible Scaling Policy (Anthropic)](https://www.anthropic.com/rsp-updates) — retrieved 2026-05-26
5. [Interrupts in LangGraph (official docs)](https://docs.langchain.com/oss/python/langgraph/interrupts) — retrieved 2026-05-26

Last verified: 2026-05-26
