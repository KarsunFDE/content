---
week: W03
day: Thu
topic_slug: soft-vs-hard-interrupts
topic_title: "Soft vs hard interrupts — name them, audit them differently"
parent_overview: W03/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://docs.langchain.com/oss/python/langgraph/interrupts
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/oss/python/langgraph/persistence
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/oss/python/langgraph/durable-execution
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://langchain-ai.github.io/langgraph/concepts/low_level/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-06-06
---

# Soft vs hard interrupts — name them, audit them differently

> [!NOTE]
> **From earlier:** Wed's HITL #4 was a *soft* interrupt — the supervisor proposed the next worker; the SSA approved or edited. Today the hard pattern lands: no proposal, no default, no auto-proceed. Same framework call; different contract.

## 1. Learning Objectives

By the end of this reading, you can:

- Define soft interrupt (suggested default, human can approve/edit/reject) and hard interrupt (no default, graph blocks indefinitely).
- Choose between `interrupt_before` (static breakpoint) and dynamic `interrupt()` based on whether the gate is structural or conditional.
- Resume a paused graph with `Command(resume=...)` and explain what that value becomes inside the node.
- Describe the audit-row fields that distinguish a soft resume from a hard resume.
- Explain why auto-timeout on a hard interrupt is a contract violation.

## 2. Introduction

Every workflow framework grows a "pause for human" primitive: Temporal calls it a signal; AWS Step Functions calls it `WaitForTaskToken`; LangGraph calls it an interrupt. Two structurally different shapes look identical on a whiteboard but audit and fail differently.

Soft: proposes a default; the human approves, edits, or rejects. Hard: no default, blocks indefinitely — the human's act of judgement is the load-bearing artifact.

Conflating them produces subtle bugs: soft gates that should be hard auto-approve under load; hard gates block on trivial decisions. Today you name them, see their audit-row shapes, and wire the hard pattern for HITL #5.

## 3. Core Concepts

### 3.1 Framework primitives — static vs dynamic

LangGraph supports two interrupt forms ([source: docs.langchain.com interrupts, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/interrupts)):

- **`interrupt_before` / `interrupt_after`** — declared at compile time. Pauses before (or after) the named node on *every* run. Use when the gate is structural.
- **Dynamic `interrupt()` function** — called inside the node body, conditionally. Use when only *some* runs require human review (e.g., confidence < 0.7, or transactions over $10 000).

Both require a checkpointer and a `thread_id`. When resumed via `Command(resume=value)`, `value` becomes the return value of `interrupt()` — the resume *passes data back into the workflow*.

> [!TIP]
> **Choose `interrupt_before` for structural gates; dynamic `interrupt()` for conditional ones.** A gate that fires on every run belongs in compile-time config; one that fires only sometimes belongs in node logic.

### 3.2 Soft interrupt — the supervised-suggestion shape

A soft interrupt proposes a default action in state; the human approves, edits, or rejects. Audit row (HITL #4 shape from Wed):

```python
{
    "action": "HITL_SOFT_RESUME",
    "actor_id": "user:reviewer-jdoe",
    "node": "supervisor_decide_next_worker",
    "before": {"proposed_action": "fan_out_evaluator_scoring"},
    "after": {"approved_action": "fan_out_evaluator_scoring", "edits": {}},
    "rationale": "Looks right — proceed.",
    "correlation_id": "<from EvaluationState.audit_correlation_id>",
    "ts": "2026-06-10T09:00:00Z"
}
```

The human's act graduates a system proposal to an approved action. The graph could proceed without review, but the sign-off is the value.

### 3.3 Hard interrupt — the no-default shape

No fallback. The decision field is unwritten until the human's resume payload fills it. Audit row (HITL #5 shape — today):

```python
{
    "action": "HITL_HARD_RESUME",
    "actor_id": "user:ssa-mhardy",
    "node": "ssa_review_ssdd",
    "before": {"ssdd_draft": "<full draft at consensus>", "ssa_approval_status": "pending"},
    "after": {"ssa_approval_status": "approved", "rationale": "<SSA text — required>"},
    "correlation_id": "<threaded>",
    "ts": "2026-06-11T09:00:00Z",
    "far_citation": "15.308"
}
```

Three differences from soft: no `proposed_action`; rationale required not optional; `far_citation` anchors why the gate exists.

> [!IMPORTANT]
> **`far_citation` is the defensibility anchor.** Without it the row is a transition log. With it, it is a regulatory artifact — an OIG auditor asking "why did this transition fire?" gets a citation, not a shrug.

### 3.4 No auto-timeout; replay safety

**Hard interrupt timeout** — "if no response in 24 h, default to reject" is a contract violation. The timeout substitutes an automated decision for the human's judgement. Correct pattern: escalate; the workflow stays paused.

**Resume replay** — when `Command(resume=value)` fires, code that ran before `interrupt()` re-runs ([source: docs.langchain.com durable-execution, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/durable-execution)). Wrap non-deterministic operations in `task` boundaries so results are recorded on first run and replayed from record.

## 4. Generic Implementation

Content-moderation pipeline: soft interrupt for routine review; hard for monetisation-impacting actions:

```python
from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt, Command
from typing import TypedDict

class ModerationState(TypedDict):
    item_id: str
    classifier_label: str
    proposed_action: str | None
    final_action: str | None
    impacts_monetisation: bool

def soft_review(state: ModerationState) -> dict:
    proposed = "remove" if state["classifier_label"] == "violation" else "keep"
    human = interrupt({
        "kind": "soft",
        "proposed_action": proposed,
        "context": {"item_id": state["item_id"]},
    })
    return {"final_action": human["action"]}

def hard_gate_monetisation(state: ModerationState) -> dict:
    if not state["impacts_monetisation"]:
        return {}
    human = interrupt({
        "kind": "hard",
        "context": {"item_id": state["item_id"], "proposed_action": state["final_action"]},
        "policy_citation": "platform-policy-3.2",
        "rationale_required": True,
    })
    if not human.get("rationale"):
        raise ValueError("Hard gate requires written rationale")
    return {
        "final_action": human["decision"],
        "senior_reviewer_id": human["actor_id"],
        "senior_rationale": human["rationale"],
    }

builder = StateGraph(ModerationState)
builder.add_node("soft_review", soft_review)
builder.add_node("hard_gate", hard_gate_monetisation)
builder.add_edge(START, "soft_review")
builder.add_edge("soft_review", "hard_gate")
builder.add_edge("hard_gate", END)
graph = builder.compile(checkpointer=checkpointer)
```

The framework call is identical — `interrupt(payload)`. The contract differs: soft ships a proposal; hard requires rationale and cites a policy.

## 5. Real-world Patterns

**Fintech — wire transfer dual-control.** Correspondent banks use two-step approval above a threshold: soft with sampled review below; hard gate above — no default, no timeout, rationale captured.

> [!NOTE]
> **Cross-domain lesson:** The soft/hard distinction maps to regulatory tiers across every domain. The audit-row shape difference is what makes the distinction legible to regulators — same code primitive, different contract.

**Healthcare — clinical decision support.** EHR soft alerts dismiss with one-click rationale. Hard stops on high-risk orders block until a second oncologist signs off. Same UI metaphor; structurally different gates with different audit trails.

## 6. Best Practices

- **Name them differently in code and logs.** `HITL_SOFT_RESUME` vs `HITL_HARD_RESUME` makes filtering straightforward.
- **Require rationale on hard-gate resumes; reject without it.**
- **Include a policy citation on every hard-gate audit row.**
- **Never auto-timeout a hard interrupt.** Escalate; the workflow stays paused.
- **`interrupt_before` for structural gates; dynamic `interrupt()` for conditional ones.**

> [!WARNING]
> **Anti-pattern: `langgraph-v0x-run-resume`.** LangGraph v0.x used `.run()` and a separate `.resume()` method. Sources on the open internet — blog posts, tutorials, Stack Overflow answers from 2023–2024 — still show this pattern. In LangGraph v1.0, the resume mechanism is `Command(resume=value)` passed to `graph.invoke()`. If you see `.resume()` called directly, or `.run()` used as the primary invocation path, the source is v0.x. Cite the v1.x docs at `docs.langchain.com/oss/python/langgraph/interrupts`.

## 7. Hands-on Exercise

For a "loan approval" workflow — eligibility check → risk scoring → disbursement — classify each gate as soft or hard, sketch its audit-row schema, and say what happens when no human responds for 24 hours: (a) loan officer reviews edge-case applications; (b) every loan above $250 k requires a senior underwriter's individual sign-off; (c) customer must e-sign before disbursement; (d) fraud-ops reviews fraud-model flags.

> [!NOTE]
> **Self-check** (30 s — answer mentally before expanding)
>
> 1. Gate (b) — senior underwriter sign-off above $250 k. Should it use `interrupt_before` or dynamic `interrupt()`? Why?
> 2. You add a 24-hour auto-reject timeout to a hard gate. The auditor asks "who rejected this loan?" and the system answers "system." What is the compliance problem?

<details>
<summary>Show answers</summary>

1. Dynamic `interrupt()` inside the node body, guarded by `if state["loan_amount"] > 250_000`. The gate is conditional — only some runs hit it. `interrupt_before` would pause every loan regardless of amount, which is wrong. Use static `interrupt_before` only when the gate is structural (every run through that node requires human review).
2. The audit row records "system" as the actor. This means no human made the rejection decision — an automated process substituted for the senior underwriter's independent judgement. For a regulatory gate (the analogue of FAR 15.308) this is the exact violation the gate was designed to prevent. The compliance problem: the decision is not defensible as independent human judgement, and any later dispute requires reconstructing what actually happened from incomplete records.

</details>

## 8. Key Takeaways

- Soft interrupt: proposal in state, human approves/edits/rejects; auto-timeout may be acceptable depending on policy.
- Hard interrupt: no proposal, no default, blocks indefinitely; auto-timeout is a contract violation for policy-anchored gates.
- Both use the same framework primitive; the audit-row contract distinguishes them.
- `Command(resume=value)` passes `value` as the return of `interrupt()` inside the node; code before the interrupt re-runs on resume.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://docs.langchain.com/oss/python/langgraph/interrupts — retrieved 2026-05-26 — hot-tech
- https://docs.langchain.com/oss/python/langgraph/persistence — retrieved 2026-05-26 — hot-tech
- https://docs.langchain.com/oss/python/langgraph/durable-execution — retrieved 2026-05-26 — hot-tech
- https://langchain-ai.github.io/langgraph/concepts/low_level/ — retrieved 2026-05-26 — hot-tech

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

**The direction of the hard-gate principal.** In the loan example, gate (c) — customer e-sign — is hard "from the human side" (no default for the customer) but the *workflow* can legitimately cancel after N days because the customer is the principal making a choice about their own loan, not a system actor exercising regulatory authority. The distinction: when the hard gate exists to protect a third party or a regulatory requirement (underwriter sign-off, SSA non-delegation), auto-timeout is a violation. When the hard gate exists because the principal must affirmatively consent (customer e-sign), a bounded cancellation window is legitimate business logic. A strong senior answer recognises this nuance and names the *direction* of the gate's authority.

**Interrupt payloads and type safety.** The `interrupt(payload)` call accepts any dict. The resume `Command(resume=value)` also accepts any value. Both are untyped at the framework level. Senior practice: define a Pydantic model for both the interrupt payload and the resume payload, validate on entry to the node, and raise `ValueError` with a descriptive message if validation fails. This catches malformed resume payloads (e.g., missing rationale on a hard gate) at the boundary rather than deep in node logic.

</details>

Last verified: 2026-06-06
