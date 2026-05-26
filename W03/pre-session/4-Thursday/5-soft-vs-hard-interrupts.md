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
last_verified: 2026-05-26
---

# Soft vs hard interrupts — name them, audit them differently

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define a "soft interrupt" (suggested default action, human can approve/edit/reject) and a "hard interrupt" (no default, graph blocks indefinitely until human input).
- Choose between `interrupt_before` (static breakpoint) and the dynamic `interrupt()` function based on whether the gate is structural or conditional.
- Resume a paused graph using `Command(resume=...)` and explain why the resume payload becomes the return value of `interrupt()` inside the node.
- Describe the auditing shape that distinguishes soft and hard resumes (what fields go on the audit row, what they mean).
- Explain why auto-timeout on a hard interrupt is a contract violation, regardless of how convenient it would be.

## 2. Introduction

"Pause and let a human decide" is one of those features that every workflow framework eventually grows, and they all grow it differently. Temporal calls it a signal. AWS Step Functions calls it the `WaitForTaskToken` callback pattern. Apache Airflow uses sensors. BPMN engines use user tasks. LangGraph calls it an interrupt.

What distinguishes a useful pause-for-human design from a frustrating one is the same across all these systems: the framework has to make it easy to express *two different shapes* of human gate. The first shape is "I have a suggested next move; please review and either approve, tweak, or override." This is the dominant shape for routine human-in-the-loop work — content moderation queues, expense approvals, supervisor-suggested next agent. The second shape is "this transition is irreversible or regulatorily-anchored; do not proceed without explicit human assent, and *no timeout* substitutes for that assent." This is the dominant shape for any gate where the human's act of judgement is itself the load-bearing thing being recorded — medical orders, court filings, payment captures over a threshold, regulatory sign-offs.

These two shapes look the same when you sketch them on a whiteboard ("the graph pauses, the human reviews, the graph continues") but they audit differently and they fail differently. Conflating them is the source of subtle bugs. This reading separates them carefully.

## 3. Core Concepts

### What an interrupt is at the framework level

LangGraph supports interrupts in two equivalent shapes ([source: docs.langchain.com interrupts, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/interrupts)):

- **Static (`interrupt_before` / `interrupt_after`)** — declared at compile time. The graph pauses before (or after) the named node every time execution reaches it. Useful when the gate is structural to the workflow — *every* run through this point requires human review.
- **Dynamic (`interrupt()` function)** — called inside a node body, possibly conditionally. The graph pauses only when the function actually runs. Useful when the gate is conditional — *some* runs require human review based on state (e.g., only flag for human when the model's confidence is below 0.7, or only when the transaction is over $10,000).

Both require a checkpointer (otherwise there is nowhere to save the paused state) and a `thread_id` (otherwise the framework cannot match the resume to the right pause). When the graph is resumed via `Command(resume=value)`, that `value` becomes the return value of the `interrupt()` call from inside the node — so the resume *passes data back into the workflow*.

### Soft interrupt — the "supervised suggestion" shape

A soft interrupt is structurally identical to a hard interrupt — same `interrupt_before` or `interrupt()` call — but the *semantics* are different. The graph proposes a default action; the human can approve it, edit it, or reject it. The state usually includes a `proposed_action` field that the node populated, and the resume payload includes the human's decision.

Typical resume audit row for a soft interrupt:

```python
{
    "action": "HITL_SOFT_RESUME",
    "actor_id": "user:reviewer-id",
    "node": "router_decide_next_step",
    "before": {"proposed_action": "<original suggestion>"},
    "after": {
        "approved_action": "<final action, possibly edited>",
        "edits": {"some_field": "new_value"}  # only if human modified
    },
    "rationale": "<optional human note>",
    "correlation_id": "<threaded>",
    "ts": "..."
}
```

The audit value is "this human reviewed this suggestion and made it final." The graph could in principle proceed without the human (the proposed action is already in state), but the human's review is what graduates it from "system proposal" to "approved action."

### Hard interrupt — the "no default" shape

A hard interrupt has no fallback. The graph *cannot* proceed without explicit human input. There is no proposed action embedded in the resume contract — or if there is, it is not authoritative; only what the human submits is.

Typical resume audit row for a hard interrupt:

```python
{
    "action": "HITL_HARD_RESUME",
    "actor_id": "user:authorised-decision-maker",
    "node": "irreversible_gate_node",
    "before": {"<full snapshot of inputs at the gate>"},
    "after": {
        "human_decision": "approve" | "reject",
        "human_rationale": "<required text>"
    },
    "correlation_id": "<threaded>",
    "ts": "...",
    "policy_citation": "<the rule that requires this gate>"
}
```

Notice three differences from the soft row:

1. **No `proposed_action`** — there is nothing for the human to rubber-stamp.
2. **Rationale is required, not optional** — the human must record *why*.
3. **A `policy_citation` field** — the gate exists because some rule requires it. The citation makes the audit trail defensible during later review.

### Why no auto-timeout on a hard interrupt

It is tempting to add a "if no response in 24h, default to reject" timeout to a hard interrupt, especially as workflows accumulate stale pauses. **For hard interrupts that exist because of a policy or regulatory requirement, this is a contract violation.** The point of the gate is that the human's judgement is the load-bearing thing. A timeout that auto-resumes is the system substituting an automated decision for the human one — exactly what the gate was designed to prevent.

The right pattern is to **escalate** (notify someone else) and **age out the work item** (mark the *task* as overdue), not to auto-resume the workflow. The workflow stays paused until a human resumes it.

Soft interrupts are different — if no human reviews the suggestion, sometimes auto-applying the default is fine. But this is a per-gate decision, not a framework default.

### How resumes interact with the rest of the graph

When `Command(resume=value)` fires, the graph resumes from the interrupt point with `value` as the return of the `interrupt()` call. Critically ([source: docs.langchain.com durable-execution, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/durable-execution)):

- **The framework replays from the last super-step boundary, not the exact line.** Any code that ran before `interrupt()` within the node will re-run on resume.
- **Non-deterministic operations should be inside tasks** so their results are recorded and not re-executed on replay. Otherwise an external API call that was already made will be re-issued.
- **State written by other nodes in the same super-step (before the interrupt) is already persisted.** Those nodes do not re-run.

This is the "determinism and consistent replay" rule. For soft interrupts, where the suggested action may have been the result of an LLM call, you want the LLM call recorded in a task so the resume does not re-call the model just to recompute a suggestion the human has already reviewed.

## 4. Generic Implementation

A non-Karsun example: a content-moderation pipeline. Soft interrupt for routine queue review; hard interrupt for "remove a user's monetisation status," which is an account-impacting action requiring senior reviewer sign-off.

```python
from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt, Command
from typing import TypedDict

class ModerationState(TypedDict):
    item_id: str
    classifier_label: str          # set upstream by an ML model
    proposed_action: str | None    # set by the soft gate
    final_action: str | None       # set by the human resume
    impacts_monetisation: bool     # set upstream

# --- Soft gate: every queued item is reviewed; the system proposes an action ---

def soft_review(state: ModerationState) -> dict:
    proposed = "remove" if state["classifier_label"] == "violation" else "keep"
    human = interrupt({
        "kind": "soft",
        "proposed_action": proposed,
        "context": {"item_id": state["item_id"]},
    })
    # Human returned either the proposal, or an edit, or a reject.
    return {"final_action": human["action"]}

# --- Hard gate: monetisation-impacting actions need an additional sign-off ---

def hard_gate_monetisation(state: ModerationState) -> dict:
    if not state["impacts_monetisation"]:
        return {}   # no gate needed, skip
    human = interrupt({
        "kind": "hard",
        "context": {
            "item_id": state["item_id"],
            "proposed_action": state["final_action"],
        },
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

Two things worth noticing:

- The *framework call* is the same in both cases — `interrupt(payload)`. The *contract* differs: the soft case ships a proposal; the hard case requires a rationale and cites a policy.
- The hard gate is implemented dynamically (inside the node body, guarded by `if not state["impacts_monetisation"]`) rather than via `interrupt_before`, because only some items need it. Use `interrupt_before` instead when the gate is structural (always-on).

## 5. Real-world Patterns

**Fintech — high-value wire transfer dual-control.** US correspondent banks use a two-step approval for wires above a threshold: one operator initiates, a second operator must approve before SWIFT submission. The second approval is a hard gate — there is no default, no timeout, and the rationale is captured for audit. Below the threshold, the system may auto-approve with sampled human review (the soft pattern). The fundamentally different audit shape is what makes the difference legible to regulators.

**Healthcare — clinical decision support overrides.** EHR systems display a soft alert ("this drug interacts with another the patient is on") that the prescriber can dismiss with a one-click rationale. The same EHR has hard stops on certain high-risk orders (e.g., a chemotherapy order over a dose threshold), where the system *will not transmit* without a second oncologist's sign-off. The same UI metaphor (a popup with action buttons) hides two very different gates underneath.

**E-commerce — return-fraud holds.** Amazon and similar marketplaces process most returns automatically. When the system detects a pattern consistent with return fraud, the return enters a hard gate that requires a specialist's review and a written disposition before refund. Below the fraud-risk threshold, returns either auto-process or queue for sampled human review (soft gate with a default-allow). The line between the two is policy, not technical capability.

**Aerospace — flight-software change approval.** Spacecraft software uplinks go through a multi-stage review where most changes pass a soft gate (the change-board meeting where defaults are usually accepted) and a small set of changes (anything affecting flight-critical systems) go through a hard gate where the chief engineer's individual sign-off is recorded by name. The hard gate's purpose is to make accountability identifiable in the post-mortem of any incident.

## 6. Best Practices

- **Name the two interrupts differently in code and audit logs.** `HITL_SOFT_RESUME` vs `HITL_HARD_RESUME` makes downstream filtering and reporting straightforward.
- **For hard interrupts, require a rationale field on resume and reject the resume payload without it.** Optional rationale gets skipped; mandatory rationale is the only way the audit row is complete.
- **Include a policy citation field on hard-gate audit rows.** When a reviewer later asks "why did the platform require human input here?", the citation is the answer.
- **Never auto-timeout a hard interrupt.** Escalate the task, not the workflow.
- **Wrap LLM calls used inside an interrupt node in a `task` so they are not re-issued on resume.** Avoids double-billing and non-deterministic re-runs.
- **Set the `actor_id` from the *authenticated* user submitting the resume, not from the system.** "actor_id = system" on a human gate is the same as no gate.
- **Use `interrupt_before` for structural gates; use dynamic `interrupt()` for conditional gates.** Mixing them confuses both readers and the audit trail.

## 7. Hands-on Exercise

**Whiteboarding exercise (12 min).** For a generic "loan approval" workflow with three steps — eligibility check, risk scoring, and disbursement — identify which human gates are soft and which are hard. For each:

1. State whether it is soft or hard and why.
2. Sketch the resume audit row schema.
3. Say what should happen if no human responds for 24 hours.

The four candidate gates:

- (a) After the eligibility check, a loan officer reviews edge-case applications flagged by the rules engine.
- (b) After risk scoring, every loan above $250k requires a senior underwriter's individual sign-off.
- (c) Before disbursement, the customer must e-sign the loan agreement.
- (d) Before disbursement, fraud-ops reviews any application flagged by the fraud model.

**What good looks like.** (a) is soft — the rules engine proposes a decision, the loan officer reviews; on no-response, default-defer to a queue. (b) is hard — the dollar threshold drives a policy gate; no timeout, escalate the work item. (c) is hard *from the human side* (no default for the customer) but the workflow can have a cancel-after-N-days timeout, because the customer is the principal, not a system actor. (d) is hard — fraud holds do not auto-clear; escalate. A weak answer marks (b) and (c) the same; a strong answer notices that the *direction* of the hard-gate principal matters.

## 8. Key Takeaways

- What is the difference between a soft and a hard interrupt at the audit-row level — what fields are present in one but not the other? (LO1, LO4)
- When do I prefer `interrupt_before` over a dynamic `interrupt()` call, and vice versa? (LO2)
- How does `Command(resume=...)` flow into the paused node, and what happens to non-deterministic code that ran inside the node before the interrupt? (LO3)
- Why is auto-timeout on a hard interrupt a contract violation, and what is the right pattern to handle stale gates? (LO5)
- How do I decide, for a given human gate I am designing, whether it is soft or hard? (LO1, LO5)

## Sources

1. [LangGraph Interrupts — interrupt(), Command, interrupt_before, resume model (v1.x docs)](https://docs.langchain.com/oss/python/langgraph/interrupts) — retrieved 2026-05-26
2. [LangGraph Persistence — checkpointer requirement, thread_id semantics](https://docs.langchain.com/oss/python/langgraph/persistence) — retrieved 2026-05-26
3. [LangGraph Durable Execution — replay, determinism, tasks](https://docs.langchain.com/oss/python/langgraph/durable-execution) — retrieved 2026-05-26
4. [LangGraph low-level concepts — super-step boundary semantics](https://langchain-ai.github.io/langgraph/concepts/low_level/) — retrieved 2026-05-26

Last verified: 2026-05-26
