---
week: W03
day: Wed
topic_slug: hitl-supervisor-worker-soft-interrupt
topic_title: "HITL #4 — supervisor → worker soft interrupt"
parent_overview: W03/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 14
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
last_verified: 2026-05-26
---

# HITL #4 — supervisor → worker soft interrupt

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish a *soft interrupt* (graph pauses, suggests a default, human approves/edits/rejects) from a *hard interrupt* (graph blocks indefinitely, no default, non-regenerative resume) — and name which use cases each shape fits.
- Implement the soft-interrupt pattern in LangGraph v1.0+ using the static `interrupt_before=[...]` breakpoint primitive (the cohort's canonical shape Mon→Thu) and the `Command(resume=...)` API, with dynamic `interrupt()` named as the alternative when a runtime-conditional pause is warranted.
- Explain why the timeout-then-auto-approve pattern is itself a delegation and therefore the wrong shape for any soft interrupt that exists for accountability reasons.
- Apply the *before/after audit-event* shape that captures both the proposed action and the human's edit in a single audit row.
- Articulate the four standard human-response decision types (approve, edit, reject, respond) and which downstream graph behavior each produces.

## 2. Introduction

**This is HITL #4 in the programme's seven-touchpoint thread — distinct from HITL #3 (Mon: ADR-level interrupt-node-boundaries discussion, *no implementation*) and HITL #5 (Thu: hard-interrupt + FAR 15.308 audit row in the LangGraph state machine, *non-regenerative resume*). Today's pattern is implementation of a soft, *advisory* gate — graph pauses, supervisor proposes a default, human approves/edits/rejects. The shape is mechanically the same as Thu's hard gate but the policy and the resume semantics are different.**

Human-in-the-loop (HITL) is not one design — it is a family of designs varying along several axes: where the pause happens, what shape the human input takes, whether a default is suggested, what happens if no human is available, and whether the action is reversible. The two ends of the spectrum are *soft interrupts* and *hard interrupts*.

A **soft interrupt** is a graph pause at an action boundary where the system has a proposed action and is asking for review before executing it. There is a default; the human can approve, edit, or reject. If reasonably designed, the soft interrupt does not block forever — it surfaces to the appropriate UI, waits the human's natural response window, and either gets a decision or escalates per policy. The classic use case is *tool-call review*: the agent has decided to send an email, write to a database, or fire a webhook, and an operator wants to see the call before it goes out [1][2].

A **hard interrupt** is a graph pause where no default exists, the system cannot legitimately proceed without the human's substantive input, and the resume is non-regenerative — the human's judgment is *the* answer, not a prompt for further AI reasoning. Hard interrupts exist for accountability reasons: the law, the contract, or the policy says "this decision shall not be made by a machine." Auto-approve on timeout is not a legitimate behavior for a hard interrupt because it is itself a delegation of the very decision the interrupt was protecting [3].

This reading covers the soft-interrupt shape. Mechanically, both shapes share the same LangGraph primitive family — static `interrupt_before=[...]` breakpoints at compile time (the cohort's canonical wiring this week) plus the `Command(resume=...)` resume API, with dynamic `interrupt()` as the alternative when the pause condition itself is runtime-decided — but they differ in their policy, their UI, and their defaults. Conflating the two is the most common HITL design mistake; spelling out the differences up front is the cheapest insurance.

## 3. Core Concepts

### 3.1 Soft vs hard — the four-axis comparison

| Axis | Soft interrupt | Hard interrupt |
|------|----------------|----------------|
| Default action | Yes — supervisor proposes a value | No — system has no defensible default |
| Behavior on no response | Escalate per policy (timeout, retry, alternate reviewer) | Block; never auto-proceed |
| Human input shape | Approve / edit / reject the proposed value | Substantive answer the system cannot generate itself |
| Resume semantics | Regenerative (graph continues with the agreed value) | Non-regenerative (the human's input *is* the decision, not a prompt) |

Soft interrupts are *advisory governance*; hard interrupts are *binding governance*. The distinction is policy-driven: it lives in whether the action being gated is one a machine could in principle propose well, or one where the human's judgment is by definition load-bearing [1][3].

### 3.2 The LangGraph mechanics — `interrupt_before` and `Command(resume=...)`

LangGraph v1.0+ exposes two interrupt primitives. The cohort's canonical shape this week is the **static `interrupt_before=[...]` breakpoint at graph-compile time** — declared once, named at the gate boundary, paired with a checkpointer. Paused state persists; the graph resumes when re-invoked with `Command(resume=<value>)` against the same `thread_id` [1]. Today's soft gate (supervisor → worker delegation) is wired exactly this way:

```python
from langgraph.types import Command
from langgraph.graph import StateGraph

# ... build the graph, then at compile time declare the gate boundary ...
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["supervisor_decide_next_worker"],
)

def supervisor_decide_next_worker(state):
    # The supervisor's LLM has proposed a delegation. The graph already paused
    # *before* this node ran; resume payload is in state when execution arrives here.
    proposed = call_router_llm(state)   # e.g., {"worker": "evaluator_3", "input": {...}}
    resume = state.get("hitl_resume", {})  # filled by Command(resume=...) on re-invoke

    if resume.get("decision") == "approve":
        action = proposed
    elif resume.get("decision") == "edit":
        action = resume["override"]
    else:
        return Command(goto="abort")

    return Command(update={"next_worker": action["worker"]}, goto=action["worker"])
```

> [!instructor-review]
> **Static `interrupt_before=[...]` vs dynamic `interrupt()` — canonical story for W3.** Mon→Wed→Thu canonically teaches **static `interrupt_before`** as the gate primitive: it is the simpler, more inspectable shape for the FAR-anchored gates the cohort wires this week; production codebases everywhere still use it; and it is what the war-room walkthroughs + Thu hard-gate live-coding ship against. The current LangGraph docs (retrieved 2026-05-26) describe **dynamic `interrupt()`** as the docs-canonical alternative when the *pause condition itself* is runtime-decided (e.g., "interrupt only if the proposed refund > $500"). Both shapes are valid in LangGraph; the cohort picks one per gate during Thu's ADR exercise. Surface the nuance live — do not let a learner read the LangChain blog and conclude the curriculum is stale. *(Cross-reference: identical callout lands in W03 Mon overview + Mon files 3 & 7 + Thu overview + Thu file 6 — the canonical story is consistent across the week.)*

Static `interrupt_before` is the right primitive when the gate boundary is *named and known at compile time* — which is true for every FAR-anchored gate in `evaluation_flow.py` this week. Dynamic `interrupt()` becomes the better choice when the *condition for pausing* is itself dynamic (interrupt only when the proposal crosses a runtime-computed threshold). Today's supervisor → worker delegation gate is a named boundary, so static `interrupt_before` is the canonical shape; §4's generic implementation shows the dynamic-`interrupt()` shape as the conditional alternative for cases where the pause condition is runtime-decided.

### 3.3 The four human-response decision types

Production HITL middleware (LangChain's, OpenAI Agents SDK's, and most production custom shapes) converges on the same four response decisions [2][4]:

1. **Approve** — execute the proposed action as-is. The most common path.
2. **Edit** — execute a modified version of the proposed action. The reviewer changes inputs, target, or parameters; the graph proceeds with the edited value.
3. **Reject** — do not execute. The reviewer either provides feedback (which the system surfaces to the agent for re-planning) or aborts the flow.
4. **Respond** — for "ask user" style tools, the reviewer directly provides the value the tool would have produced (e.g., asking the user for a missing piece of information).

Your soft-interrupt resume schema should at minimum allow approve/edit/reject; respond is only relevant for tools whose purpose is to query the user.

### 3.4 The before/after audit-event shape

The soft interrupt creates a unique audit requirement: you need to record what was *proposed* and what was *approved* (or edited, or rejected). The canonical shape is one audit row with both states:

```json
{
  "event": "HITL_SOFT_RESUME",
  "actor_id": "user:reviewer-42",
  "correlation_id": "request-7f3a-...",
  "before": {"action": "evaluator_5", "inputs": {...}},
  "after":  {"action": "evaluator_3", "inputs": {...}},
  "decision": "edit",
  "ts": "2026-05-26T15:42:11Z"
}
```

This shape lets you reconstruct *both* the system's proposed behavior and the human's correction in a single row — useful for retrospective bias analysis (how often does the human override the supervisor and on what cases?) and for accountability (the audit log shows the human is the actor, not the system) [5].

### 3.5 The timeout-then-auto-approve anti-pattern

A common temptation in soft-interrupt design is "if no response in N minutes, auto-approve so the graph doesn't block." For a *purely advisory* soft interrupt (the human's review is a quality check, not an accountability gate), this is sometimes defensible. For a soft interrupt that exists for accountability reasons, **auto-approve on timeout is itself a delegation of the very decision the interrupt was protecting**.

The Wed soft interrupt sets up Thu's hard interrupt, which is non-negotiably accountability-bearing (the FAR/regulatory framing makes the human-in-the-loop a legal requirement, not an optimization). Defaulting to auto-approve under any timeout in that flow would be illegitimate. The discipline starts on Wed, even though the soft shape allows more flexibility in principle [3][5].

The right escalation policy for a soft interrupt with accountability semantics is:

- Surface the pending interrupt to the assigned reviewer immediately.
- After a configured wait, escalate to an alternate reviewer (or a reviewer pool).
- Never auto-approve. The graph waits for a human; the calendar is the constraint, not the graph's patience.

### 3.6 Distinguishing this from #3 and #5 — explicit cross-reference

In the programme's seven-touchpoint HITL thread, today's gate is **HITL #4** — and it is structurally distinct from both #3 and #5:

- **This is HITL #4 — NOT #3.** HITL #3 (Mon Plan Day) was an *ADR-level boundary design exercise* — deciding *which* transitions in the topology get gates, with FAR citation or reversibility argument, *no implementation*. Today is the implementation of one of those soft gates against a real supervisor → worker flow.
- **This is HITL #4 — NOT #5.** HITL #5 (Thu) is a *hard interrupt* with **no default**, **non-regenerative resume**, and the **FAR 15.308 audit row** — the SSA's independent judgment is the load-bearing artifact. Today's #4 is a *soft interrupt* with a *suggested default* the human can approve, edit, or reject; the supervisor's proposal is the load-bearing artifact, and the human's role is review.

The pattern of *naming what shape this gate is and why* before implementing it is the discipline; you should be able to answer "what kind of interrupt is this, what's the resume semantics, and which touchpoint in the seven-thread is it?" for every gate you add to a graph.

## 4. Generic Implementation

A complete generic soft-interrupt example using the **dynamic `interrupt()` shape** — illustrating the case where the *pause condition itself* is runtime-decided (refunds only pause above $500). This is the shape the cohort reaches for when a static compile-time gate would over-pause; for the named gate boundaries in this week's evaluator-flow, the canonical wiring is static `interrupt_before` per §3.2. Domain: a customer-support agent that wants human review before issuing a refund over $500.

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

def classify_and_propose(state: SupportState) -> Command:
    # Agent decides what action to take on the ticket
    proposed = run_agent(state["ticket"])  # e.g., {"type": "refund", "amount": 650}
    return Command(update={"proposed_action": proposed}, goto="risk_gate")

def risk_gate(state: SupportState) -> Command:
    proposal = state["proposed_action"]
    needs_review = (
        proposal["type"] == "refund" and proposal["amount"] > 500
    )
    if not needs_review:
        return Command(goto="execute")

    # Conditional soft interrupt — only when the proposal crosses the threshold
    response = interrupt({
        "kind": "soft",
        "proposed_action": proposal,
        "rationale": "Refund over $500 — please review",
        "suggested_default": "approve",
        "ticket_id": state["ticket"]["id"],
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

    if response["decision"] == "approve":
        return Command(
            update={"audit_events": state["audit_events"] + [audit]},
            goto="execute",
        )
    elif response["decision"] == "edit":
        return Command(
            update={
                "proposed_action": response["edited_action"],
                "audit_events": state["audit_events"] + [audit],
            },
            goto="execute",
        )
    else:  # reject
        return Command(
            update={"audit_events": state["audit_events"] + [audit]},
            goto="re_plan",
        )

def execute(state: SupportState) -> Command:
    # Perform the (possibly edited) action
    do_action(state["proposed_action"])
    return Command(update={"resolved": True}, goto=END)

def re_plan(state: SupportState) -> Command:
    # Reject path: the reviewer's feedback feeds back into the agent
    return Command(goto="classify_and_propose")

graph = StateGraph(SupportState)
graph.add_node("classify_and_propose", classify_and_propose)
graph.add_node("risk_gate", risk_gate)
graph.add_node("execute", execute)
graph.add_node("re_plan", re_plan)
graph.add_edge(START, "classify_and_propose")

# Checkpointer is required — interrupt persists state across the pause
app = graph.compile(checkpointer=MemorySaver())

# Caller side — pause surfaces; resume happens later with Command
config = {"configurable": {"thread_id": "ticket-123"}}
result = app.invoke({"ticket": {...}, "correlation_id": "..."}, config)
# ... reviewer responds ...
final = app.invoke(
    Command(resume={"decision": "edit", "edited_action": {...}, "actor_id": "user:reviewer-42"}),
    config,
)
```

Notes on what each piece does:

- **`risk_gate`** decides *whether* to interrupt based on state — this is the conditional dynamic shape, not a compile-time `interrupt_before`. Refunds under $500 never pause.
- **`interrupt(payload)`** persists graph state and surfaces `payload` to whatever monitors the run.
- **`Command(resume=...)`** carries the human's decision back into the `interrupt()` call's return value.
- **The audit row** captures both before and after in one event.
- **`re_plan`** is the reject path — the agent gets another turn rather than the flow ending.

## 5. Real-world Patterns

**Healthcare — prior-authorization review (Cigna-style).** Insurance prior-auth systems use soft interrupts when the agent has selected a denial code based on policy match. A clinician reviewer sees the proposed denial, the policy citation, and the case context, then approves, edits the code, or rejects with a comment that re-prompts the agent. The before/after audit shape is regulatory-required (HIPAA + state insurance regs). Soft, not hard, because the agent can produce a defensible default; the clinician's role is review, not generation [4].

**Fintech — wire-transfer review.** Banks use soft interrupts when an agent has proposed a wire transfer above an internal-policy threshold (varies by bank, often around $10k). The pause surfaces to a compliance officer's queue; the officer approves or edits the destination/amount; the audit row captures both sides. Auto-approve on timeout is explicitly disallowed by anti-money-laundering compliance policy at most banks, even though the interrupt shape is soft (the system *could* produce a defensible action — the human's role is to verify, not to originate) [5].

**E-commerce — pricing-update review.** A mid-2026 production case study from a marketplace pricing system described soft interrupts on agent-proposed price changes >5% from baseline. The reviewer queue surfaced about 200 proposals/day; reviewers approve, edit, or reject; the audit log is used both for retrospective bias analysis (where does the agent over-correct?) and for vendor compliance reports. The system was deliberately designed without timeout-auto-approve to maintain pricing-integrity guarantees [4].

**SaaS infra — DLT (delete-large-thing) tool review.** Internal platform tools that perform destructive operations (drop a table, terminate a cluster, mass-delete logs) use soft interrupts on every invocation. The agent proposes the action; an on-call engineer reviews; approve/edit/reject. The pattern is documented in LangChain's HITL middleware docs as the canonical use case [2].

## 6. Best Practices

- **Name the interrupt's shape (soft vs hard) and its policy basis (advisory vs accountability) before you write the code.** This shows up in the docstring and the on-call runbook.
- **Use static `interrupt_before=[...]` for *named* gate boundaries; reach for dynamic `interrupt()` only when the *pause condition itself* is runtime-decided.** The cohort's canonical wiring this week is static — it's simpler, inspectable at compile time, and matches every FAR-anchored gate in the evaluator-flow. The dynamic shape is the right tool when the gate fires conditionally on runtime state (the §4 generic example) [1].
- **Capture before/after in one audit row.** Two separate events lose the causal link between proposal and decision.
- **Never auto-approve on timeout for accountability-bearing soft interrupts.** Escalate to an alternate reviewer instead; the calendar is the constraint.
- **Persist state through a real checkpointer (database-backed in production, not in-memory).** Interrupts that span more than a few minutes need durable persistence.
- **Make the interrupt payload self-contained.** Include everything the reviewer needs to decide — proposal, rationale, context, correlation ID. Do not require the reviewer to go fetch additional data.
- **Surface interrupts via the reviewer's actual workflow (their queue, their UI), not a generic admin panel.** Soft interrupts that don't reach the reviewer get auto-rejected by neglect.

## 7. Hands-on Exercise

**Code task (15 min).** Build a LangGraph node `proposal_gate` that:

1. Receives a proposal of shape `{"type": str, "value": float, "metadata": dict}`.
2. Conditionally interrupts only when `type == "expense"` and `value > 1000`.
3. On interrupt, surfaces a payload including `type`, `value`, `metadata`, and a `suggested_default` of `"approve"`.
4. Accepts a resume value of shape `{"decision": "approve" | "edit" | "reject", "edited_value": float | None, "actor_id": str}`.
5. Produces an audit event with before/after fields.
6. Routes to `execute` on approve/edit and to `aborted` on reject.

Use `langgraph.types.interrupt` and `Command`. You can stub `execute` and `aborted` as no-op nodes that print their name.

**What good looks like:** Because the *pause condition itself* is runtime-decided (only `type == "expense" AND value > 1000` interrupts), the dynamic `interrupt()` function is the right primitive here — static `interrupt_before` would over-pause on every proposal. The conditional gate fires *only* when both conditions hold, leaving cheap expenses non-interrupted. The audit event includes both the original proposal and the (possibly edited) approved value. The function returns a `Command(update=..., goto=...)` rather than raising or returning a raw dict. The reject path produces a structured audit row before routing to `aborted` (so rejection is logged, not silent). Bonus: you parameterize the threshold rather than hard-coding 1000. (Contrast this with the named gate boundaries in the evaluator-flow this week, where static `interrupt_before` at compile time is the canonical shape because the gate always fires at that boundary — see §3.2.)

## 8. Key Takeaways

- *What separates a soft interrupt from a hard interrupt?* (Soft has a default and a non-blocking escalation policy; hard has no default and never auto-proceeds — and the difference is policy-driven, not mechanical.)
- *When do you use static `interrupt_before` vs dynamic `interrupt()`, and what is the cohort's canonical W3 wiring?* (Static `interrupt_before` at compile time for named gate boundaries — the canonical W3 shape for all FAR-anchored gates Mon→Thu. Dynamic `interrupt()` inside a node when the *pause condition itself* is runtime-decided, e.g., "interrupt only if refund > $500." Both are valid LangGraph primitives.)
- *What are the four standard human-response decision types and what does each route to?* (Approve → execute; edit → execute with modified value; reject → re-plan or abort; respond → return the value directly to a tool.)
- *Why is auto-approve on timeout the wrong default for accountability-bearing soft interrupts?* (It is itself a delegation of the very decision the interrupt was protecting; the calendar is the constraint, not the graph.)
- *What does the before/after audit-event shape buy you?* (Reconstructable proposal-vs-decision history in one row; supports retrospective bias analysis and accountability reporting.)

## Sources

1. [Interrupts — LangChain/LangGraph official docs](https://docs.langchain.com/oss/python/langgraph/interrupts) — retrieved 2026-05-26
2. [Human-in-the-loop — LangChain official docs](https://docs.langchain.com/oss/python/langchain/human-in-the-loop) — retrieved 2026-05-26
3. [Making it easier to build human-in-the-loop agents with interrupt (LangChain Blog)](https://www.langchain.com/blog/making-it-easier-to-build-human-in-the-loop-agents-with-interrupt) — retrieved 2026-05-26
4. [Human in the Loop Middleware in Python (FlowHunt)](https://www.flowhunt.io/blog/human-in-the-loop-middleware-python-safe-ai-agents/) — retrieved 2026-05-26
5. [AI Human in the Loop: Production Oversight Patterns (Redis Blog)](https://redis.io/blog/ai-human-in-the-loop/) — retrieved 2026-05-26

Last verified: 2026-05-26
