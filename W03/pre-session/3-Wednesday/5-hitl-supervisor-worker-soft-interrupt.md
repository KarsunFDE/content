---
week: W03
day: Wed
topic_slug: hitl-supervisor-worker-soft-interrupt
topic_title: "HITL #4 — supervisor → worker soft interrupt"
parent_overview: W03/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://docs.langchain.com/oss/python/langgraph/interrupts
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/oss/python/langchain/human-in-the-loop
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.langchain.com/blog/making-it-easier-to-build-human-in-the-loop-agents-with-interrupt
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.flowhunt.io/blog/human-in-the-loop-middleware-python-safe-ai-agents/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://redis.io/blog/ai-human-in-the-loop/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-06-06
---

# HITL #4 — supervisor → worker soft interrupt

> [!NOTE]
> **From earlier:** Mon's ADR named which transitions get HITL gates and cited FAR 15.308 framing. Today is the implementation of one of those gates — a *soft* interrupt at the supervisor → worker handoff, distinct from Thu's hard interrupt.

## 1. Learning Objectives

- Distinguish a *soft interrupt* (graph pauses, suggests a default, human approves/edits/rejects) from a *hard interrupt* (no default, blocks indefinitely, non-regenerative resume).
- Wire a soft interrupt using LangGraph's static `interrupt_before=[...]` and `Command(resume=...)`.
- Apply the before/after audit-event shape that captures both the proposed action and the human's edit in one row.
- Explain why auto-approve on timeout is itself a delegation — and why that makes it wrong for accountability-bearing gates.
- Articulate the four standard human-response decision types (approve / edit / reject / respond).

## 2. Introduction

**HITL #4 of 7.** Not #3 (Mon's ADR boundary design) and not #5 (Thu's hard interrupt + FAR 15.308). Today: soft advisory gate — graph pauses, supervisor proposes a default, SSA approves/edits/rejects.

A **soft interrupt** pauses where the system has a proposed action. There is a default. A **hard interrupt** has no default, blocks indefinitely; the human's judgment *is* the decision. Auto-approve on timeout is itself a delegation of the protected decision.

## 3. Core Concepts

### 3.1 Soft vs hard — four-axis comparison

| Axis | Soft interrupt | Hard interrupt |
|------|----------------|----------------|
| Default action | Yes — supervisor proposes a value | No — system has no defensible default |
| Behavior on no response | Escalate per policy (timeout → alternate reviewer) | Block; never auto-proceed |
| Human input shape | Approve / edit / reject the proposed value | Substantive answer the system cannot generate |
| Resume semantics | Regenerative (graph continues with agreed value) | Non-regenerative (human input *is* the decision) |

Soft = advisory governance. Hard = binding governance. The distinction is policy-driven: does the machine's proposal constitute a defensible default, or is the human's judgment the load-bearing artifact?

### 3.2 LangGraph mechanics — `interrupt_before` and `Command(resume=...)`

The cohort's canonical shape this week is the **static `interrupt_before=[...]` breakpoint at graph-compile time** — declared once, named at the gate boundary, paired with a checkpointer. Paused state persists; the graph resumes when re-invoked with `Command(resume=<value>)` against the same `thread_id`.

```python
from langgraph.types import Command

app = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["supervisor_decide_next_worker"],
)

def supervisor_decide_next_worker(state):
    proposed = call_router_llm(state)
    resume = state.get("hitl_resume", {})

    if resume.get("decision") == "approve":
        action = proposed
    elif resume.get("decision") == "edit":
        action = resume["override"]
    else:
        return Command(goto="abort")

    return Command(update={"next_worker": action["worker"]}, goto=action["worker"])
```

> [!TIP]
> **Static vs dynamic — canonical W3 story.** Static `interrupt_before` for named compile-time boundaries (every FAR-anchored gate this week). Dynamic `interrupt()` when the *pause condition itself* is runtime-decided ("interrupt only if refund > $500"). Both valid; cohort picks one per gate in Thu's ADR exercise.

### 3.3 The four human-response decision types

Production HITL middleware converges on four types: **Approve** (execute as-is), **Edit** (execute modified version), **Reject** (do not execute; feedback for re-planning), **Respond** (for "ask user" tools; reviewer provides the value directly).

Resume schema needs at minimum approve/edit/reject.

### 3.4 The before/after audit-event shape

One audit row captures both the proposed and approved state:

```json
{
  "event": "HITL_SOFT_RESUME",
  "actor_id": "user:ssa-42",
  "correlation_id": "eval-7f3a-...",
  "before": {"action": "evaluator_5", "inputs": {}},
  "after":  {"action": "evaluator_3", "inputs": {}},
  "decision": "edit",
  "ts": "2026-06-06T15:42:11Z"
}
```

Reconstructs the system's proposal *and* the human's correction in one row — required for FAR accountability and retrospective bias analysis.

### 3.5 The timeout-then-auto-approve anti-pattern

Auto-approve on timeout is itself a delegation of the decision the interrupt was protecting. For today's soft gate: surface immediately → after a configured wait, escalate to an alternate reviewer → never auto-approve. The calendar is the constraint.

## 4. Generic Implementation

Dynamic `interrupt()` shape — illustrating a runtime-conditional pause (only when refund > $500). The cohort's named gate boundaries this week use static `interrupt_before`; this shows the conditional alternative:

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command, interrupt
from langgraph.checkpoint.memory import MemorySaver

class SupportState(TypedDict):
    ticket: dict
    proposed_action: dict | None
    audit_events: list[dict]
    correlation_id: str
    resolved: bool

def risk_gate(state: SupportState) -> Command:
    proposal = state["proposed_action"]
    if not (proposal["type"] == "refund" and proposal["amount"] > 500):
        return Command(goto="execute")

    response = interrupt({
        "kind": "soft",
        "proposed_action": proposal,
        "suggested_default": "approve",
        "correlation_id": state["correlation_id"],
    })

    audit = {
        "event": "HITL_SOFT_RESUME",
        "before": proposal,
        "after": response.get("edited_action", proposal),
        "decision": response["decision"],
        "actor_id": response.get("actor_id"),
        "correlation_id": state["correlation_id"],
    }

    if response["decision"] in ("approve", "edit"):
        updated = response.get("edited_action", proposal)
        return Command(
            update={"proposed_action": updated,
                    "audit_events": state["audit_events"] + [audit]},
            goto="execute",
        )
    return Command(
        update={"audit_events": state["audit_events"] + [audit]},
        goto="re_plan",
    )

app = graph.compile(checkpointer=MemorySaver())
config = {"configurable": {"thread_id": "ticket-123"}}
result = app.invoke({"ticket": {}, "correlation_id": "..."}, config)
# reviewer responds:
final = app.invoke(
    Command(resume={"decision": "edit", "edited_action": {}, "actor_id": "user:reviewer-42"}),
    config,
)
```

`interrupt(payload)` persists state and surfaces the payload to the reviewer. `Command(resume=...)` carries their decision back into the `interrupt()` return value.

## 5. Real-world Patterns

**Healthcare — prior-auth review.** Insurance systems soft-interrupt when the agent selects a denial code. Clinician reviews proposal + policy citation, then approves, edits, or rejects. Before/after audit shape is HIPAA-required.

**Fintech — wire-transfer review.** Banks soft-interrupt agent-proposed transfers above threshold. AML compliance explicitly disallows auto-approve on timeout.

**SaaS infra — DLT tool review.** Destructive operations (drop table, terminate cluster) use soft interrupts on every invocation — the canonical LangChain HITL middleware use case.

## 6. Best Practices

- **Name the interrupt's shape (soft vs hard) and its policy basis before writing code.** Put it in the docstring and the runbook.
- **Use static `interrupt_before` for named gate boundaries; reach for dynamic `interrupt()` only when the pause condition is runtime-decided.**
- **Capture before/after in one audit row.** Two separate events lose the causal link.
- **Never auto-approve on timeout for accountability-bearing gates.** Escalate to an alternate reviewer.
- **Persist state through a database-backed checkpointer in production.** In-memory loses state on restart — hostile to interrupts that span more than a few minutes.

> [!WARNING]
> **Anti-pattern: `soft-interrupt-without-context`.** An interrupt payload that omits the proposal's rationale, context, or correlation ID forces the reviewer to go fetch additional data before deciding. This creates review latency and reviewer errors. Make the interrupt payload self-contained: include everything the reviewer needs — proposal, rationale, context, correlation ID — in the single payload object surfaced to their queue.

## 7. Hands-on Exercise

Build a LangGraph node `proposal_gate` that: (1) receives `{"type": str, "value": float, "metadata": dict}`; (2) conditionally interrupts only when `type == "expense"` and `value > 1000`; (3) surfaces a payload with `type`, `value`, `metadata`, and `suggested_default = "approve"`; (4) accepts resume `{"decision": "approve"|"edit"|"reject", "edited_value": float|None, "actor_id": str}`; (5) produces a before/after audit event; (6) routes to `execute` on approve/edit and `aborted` on reject.

> [!NOTE]
> **Self-check** (30s — answer mentally before expanding)
>
> 1. The pause condition is "interrupt only when value > 1000." Should you use static `interrupt_before` or dynamic `interrupt()`? Why?
> 2. A soft interrupt auto-approves after 5 minutes of no response. Why is this wrong for an accountability-bearing gate?

<details>
<summary>Show answers</summary>

1. Dynamic `interrupt()` inside the node — because the pause condition is runtime-decided (depends on `value` from state). Static `interrupt_before` fires unconditionally at compile-named boundaries; it would pause on every proposal regardless of value.
2. Auto-approve on timeout is itself a delegation of the decision the interrupt was protecting. For accountability-bearing gates (FAR-anchored, compliance-required), the human's judgment is the load-bearing artifact — not the system's default. The correct policy is escalate to an alternate reviewer; the calendar is the constraint, not the graph's patience.

</details>

> [!IMPORTANT]
> **Before/after in one row, not two events.** Two separate audit events lose the causal link between the proposed action and the human's decision. One `HITL_SOFT_RESUME` row with both `before` and `after` fields is the accountability-grade shape.

## 8. Key Takeaways

- Soft = default + non-blocking escalation; hard = no default + never auto-proceed. Distinction is policy-driven.
- Static `interrupt_before` for named compile-time gate boundaries; dynamic `interrupt()` for runtime-conditional pauses.
- Four response types: approve / edit / reject / respond.
- Before/after audit row in one event — captures proposal and decision together for accountability.

> [!NOTE]
> **Thu's hard interrupt.** HITL #5 wires the non-regenerative gate — no default, FAR 15.308 audit row, blocks until the SSA acts. Today's soft gate is the precursor; Thu's is the binding counterpart.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://docs.langchain.com/oss/python/langgraph/interrupts — retrieved 2026-05-26 — hot-tech
- https://docs.langchain.com/oss/python/langchain/human-in-the-loop — retrieved 2026-05-26 — hot-tech
- https://www.langchain.com/blog/making-it-easier-to-build-human-in-the-loop-agents-with-interrupt — retrieved 2026-05-26 — hot-tech
- https://www.flowhunt.io/blog/human-in-the-loop-middleware-python-safe-ai-agents/ — retrieved 2026-05-26 — hot-tech
- https://redis.io/blog/ai-human-in-the-loop/ — retrieved 2026-05-26 — hot-tech

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

**Static `interrupt_before` vs dynamic `interrupt()` — full decision matrix.** Static `interrupt_before` declared at compile time is inspectable, auditable, and produces deterministic graph topology — the gate is always there, always named, shows up in the LangGraph visualization. The downside: it over-pauses when the gate should be conditional. Dynamic `interrupt()` is flexible but harder to audit because the pause condition is buried in node logic. For the programme's FAR-anchored gates, static is preferred because the gate is non-conditional by policy — every supervisor → worker delegation at that boundary must surface for review. Reserve dynamic `interrupt()` for genuinely conditional pauses where the condition derives from runtime state.

**Non-regenerative resume and what it means for state.** Thu's hard interrupt uses non-regenerative resume: `Command(resume=human_judgment)` where the human's input is the final value passed forward, not a prompt to the LLM for further reasoning. This means the graph's next node receives the human's raw input in state — the LLM is not re-invoked to "interpret" the human's answer. Senior FDEs implementing the hard gate should ensure the downstream node is written to consume the human's input directly, not to feed it back into a routing LLM first.

</details>

Last verified: 2026-06-06
