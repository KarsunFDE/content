---
week: W03
day: Thu
topic_slug: hitl-5-hard-interrupt-far-15308-audit-row
topic_title: "HITL #5 — LangGraph hard interrupt + the FAR 15.308 audit row"
parent_overview: W03/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 15
sources:
  - url: https://www.acquisition.gov/far/15.308
    retrieved_on: 2026-05-26
    recency_category: federal-regulatory
  - url: https://docs.langchain.com/oss/python/langgraph/interrupts
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/oss/python/langgraph/persistence
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/oss/python/langgraph/durable-execution
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.acquisition.gov/far/5.705
    retrieved_on: 2026-05-26
    recency_category: federal-regulatory
last_verified: 2026-05-26
---

# HITL #5 — LangGraph hard interrupt + the FAR 15.308 audit row

## 1. Learning Objectives

By the end of this reading, the learner can:

- Explain why a regulatorily-anchored gate (a rule that *requires* a human's independent judgement) maps cleanly to a LangGraph hard interrupt and not to a soft one.
- Design an audit row schema for a hard-interrupt resume that names the regulation, the human actor, the inputs the human saw, and the decision the human made.
- Articulate the "non-regeneration on resume" rule and explain why auto-regenerating draft content invalidates the audit trail.
- Trace the two-hard-gate pattern (decision gate + publication gate) and explain why each is structurally distinct.
- Use `graph.get_state_history(thread_id)` as an audit-replay surface and explain what a reviewer can reconstruct from it.

## 2. Introduction

Some human gates exist for convenience — *we want a human to review this so the model does not make a mistake*. Other human gates exist because a rule *requires* a human's judgement, and substituting any automated process for that judgement is itself a violation of the rule. The two look similar from the workflow runtime's perspective but they audit completely differently.

The Federal Acquisition Regulation (FAR) provides a clean, durable example of the second kind. FAR 15.308 — Source Selection Decision — reads, in part: *"While the SSA may use reports and analyses prepared by others, the source selection decision shall represent the SSA's independent judgment. The source selection decision shall be documented…"* ([source: acquisition.gov, retrieved 2026-05-26](https://www.acquisition.gov/far/15.308)). The plain reading: the human decision-maker's individual judgement is the thing being recorded, not a system-generated analysis they signed off on. A platform that auto-completes the decision when the SSA does not respond, or that regenerates the underlying analysis after the SSA started reviewing, has effectively replaced her judgement with the system's. That is the kind of substitution the rule was designed to prevent.

This reading uses FAR 15.308 as the worked example because it is *unusually clean* as a teaching artifact. Every rule of the same shape elsewhere (medical, legal, financial) maps onto the same audit-row design. The framework move that makes this possible is the LangGraph **hard interrupt** — the gate that blocks indefinitely, ships no default, and produces an audit row containing the regulation that put it there.

## 3. Core Concepts

### What FAR 15.308 actually says, and what follows for software

The clause has three load-bearing words: *independent*, *judgment*, *documented* ([source: acquisition.gov, retrieved 2026-05-26](https://www.acquisition.gov/far/15.308)). Each maps to a technical constraint:

- **"Independent"** — the SSA's decision must not be delegated, not to another person and not to an automated process. A timeout that auto-resumes is delegation to a timer.
- **"Judgment"** — the SSA is exercising discretion, not approving a precomputed answer. The system can prepare materials; it cannot pre-decide.
- **"Documented"** — there must be a record. The record is what later auditors review. The audit row *is* the documentation.

The compliance question is therefore not "did the SSA click approve?" but rather "is there a defensible record that the SSA, individually, judged the proposals against the criteria in the solicitation and recorded her rationale?" The system's job is to make that record possible and tamper-evident.

### Why this is a hard interrupt, not a soft one

A soft interrupt ships a default. If no human responds, the default is what happens — or the human's role is to lightly review a precomputed answer. **Neither shape satisfies FAR 15.308.** Shipping a default would let the system make the decision when the SSA does not respond. Letting the SSA rubber-stamp a precomputed answer would make the system's analysis the judgement, not hers.

A hard interrupt has no default. The graph blocks. The state contains the materials the SSA needs (the proposals, the consensus evaluation, the draft SSDD) but the *decision field* is unwritten until the SSA's resume payload fills it. There is no timer that resumes the graph in her absence. There is no fallback path the framework can take. The block is the architectural expression of "her decision is the thing that proceeds the workflow."

### The two-hard-gate pattern

Real procurement workflows usually have *two* hard gates, not one, and they exist for different reasons:

1. **Decision gate** — the SSA approves or rejects the source-selection decision. Anchored by FAR 15.308.
2. **Publication gate** — once the award is recorded, the public award notice gets published (FAR Part 5 / FAR 5.301-5.705 family — postaward publication procedures, [source: acquisition.gov, retrieved 2026-05-26](https://www.acquisition.gov/far/5.705)). This is irreversible in the sense that once the notice is out, it has been seen by the offerors and the public.

The two gates need separate human assent because they answer different questions. The first is "is this the right choice?" The second is "now that we have chosen, do we make it public, with all that entails?" Both are hard interrupts; the audit rows cite different sections of the FAR.

In LangGraph this is two `interrupt_before` entries:

```python
app = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["ssa_review_ssdd", "record_award"],
)
```

> [!instructor-review]
> **Static `interrupt_before=[...]` vs dynamic `interrupt()` — canonical story for W3.** Both primitives are valid in LangGraph v1.x. The programme's canonical wiring Mon→Tue→Wed→Thu is **static `interrupt_before`** for the named, FAR-anchored gate boundaries — simpler, inspectable at compile time, matches the war-room walkthroughs, and what production codebases everywhere still use. That is the shape used above for `ssa_review_ssdd` and `record_award`: both gates are named, compile-time-known boundaries, so static is the right primitive. The LangGraph docs (retrieved 2026-05-26) describe **dynamic `interrupt()`** as the docs-canonical alternative when the *pause condition itself* is runtime-decided; Wed's file 5 §4 generic-implementation example shows that shape for the refund-over-$500 case. Cross-reference: identical callout lands in W03 Mon overview + Mon files 3 & 7 + Wed file 5 + Thu overview — the canonical story is consistent across the week.

### The non-regeneration rule on resume

The most subtle bug pattern with hard interrupts on regulatorily-anchored gates is well-meaning automatic regeneration. The flow looks innocuous: the SSA does not click approve immediately, hours pass, the underlying regulatory corpus is re-indexed in the meantime ("we got an update from acquisition.gov this morning"), the SSA clicks approve, and the system "freshens" the draft document before recording it.

This is a contract violation, even though the system has done extra work to be helpful. **The SSA's judgement was applied to a specific draft on her screen.** A draft regenerated after her review is not the draft she judged. The audit row reads "SSA approved at 09:00" but the artifact reads "draft generated at 09:01." A later reviewer cannot reconstruct what she saw.

The rule:

- **Snapshot the draft into graph state at the moment it is ready for review.** The draft is now a field of the state, not a function of upstream data.
- **On resume, do not re-invoke any model.** The resume *only* transitions state — it records the human's decision against the snapshot already in state. No RAG retrieval, no LLM call.
- **The audit row stores both the snapshot and the decision.** A reviewer can replay what the SSA saw.

This is the architectural cousin of the "determinism and consistent replay" rule from the LangGraph durable-execution docs ([source: docs.langchain.com durable-execution, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/durable-execution)). Non-deterministic operations (LLM calls, RAG retrievals) belong in earlier nodes that *write into state*; the resume node should be pure decision-recording.

### The audit row schema — what makes it defensible

A defensible audit row for a hard-interrupt resume has these fields:

| Field | Source | Why it matters |
|---|---|---|
| `action` | enum, "HITL_HARD_RESUME" | Distinguishes from soft resumes for filtering/reporting. |
| `actor_id` | authenticated user submitting the resume | The named individual who made the decision. Never "system". |
| `node` | name of the interrupted node | Tells the reviewer which gate this row records. |
| `before` | snapshot of state at the interrupt | What the human saw — the draft, the evaluation, the proposals. |
| `after` | resume payload | What the human decided + rationale. |
| `rationale` | required string from the resume | The human's own words. Required, not optional. |
| `correlation_id` | threaded from upstream | Lets the reviewer connect to other events in the same workflow. |
| `ts` | timestamp | When the decision was made. |
| `far_citation` (or analogous `policy_citation`) | the rule that requires this gate | The defensibility anchor. |

The reason the citation field is load-bearing: when an auditor asks "why did the platform let this transition fire?", the answer is the regulation that *required* the human gate. Without that field the row reads as a transition log. With it, the row reads as a regulatory artifact.

### `graph.get_state_history(thread_id)` — the replay surface

LangGraph persists every checkpoint, and `graph.get_state_history(thread_id)` returns them in order ([source: docs.langchain.com persistence, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/persistence)). For an audit, this means a reviewer can replay every state the workflow passed through — every checkpoint, every interrupt, every resume — without re-running any model or any non-deterministic operation. The persisted state *is* the record.

For a regulatorily-anchored workflow this is the single most useful surface. A later reviewer asking "show me everything this evaluation went through" can be served from the history without re-executing anything.

## 4. Generic Implementation

A non-federal-acquisitions example: a securities trading platform's compliance gate on a large block trade. Rule analogue: many jurisdictions require a registered compliance officer to approve trades above a notional threshold before the order is routed to an exchange. The human's individual judgement is the load-bearing thing; no auto-timeout substitutes.

```python
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command
from typing import TypedDict

class BlockTradeState(TypedDict):
    trade_id: str
    desk_id: str                          # multi-tenant key
    notional_usd: float
    risk_snapshot: dict | None            # pre-trade risk numbers — snapshotted
    compliance_decision: str | None       # set only on resume
    compliance_rationale: str | None
    compliance_officer_id: str | None
    audit_correlation_id: str

def compute_pre_trade_risk(state: BlockTradeState) -> dict:
    # Heavy computation: VaR, exposure, counterparty risk.
    # Snapshot the result into state so the resume does not need to recompute.
    snapshot = run_risk_model(state["trade_id"])
    return {"risk_snapshot": snapshot}

def compliance_review(state: BlockTradeState) -> dict:
    # This node body runs only after Command(resume=...) provides the decision.
    # No model call here, no recomputation — pure state transition.
    return {}    # decision fields are set by the resume payload

builder = StateGraph(BlockTradeState)
builder.add_node("compute_pre_trade_risk", compute_pre_trade_risk)
builder.add_node("compliance_review", compliance_review)
builder.add_node("route_to_exchange", route_to_exchange)

builder.add_edge(START, "compute_pre_trade_risk")
builder.add_edge("compute_pre_trade_risk", "compliance_review")
builder.add_edge("compliance_review", "route_to_exchange")
builder.add_edge("route_to_exchange", END)

app = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["compliance_review", "route_to_exchange"],
)
```

When the compliance officer resumes:

```python
resume_payload = {
    "compliance_decision": "approved",
    "compliance_rationale": "Reviewed VaR and counterparty exposure; within mandate.",
    "compliance_officer_id": "user:compliance-mhardy",
}
app.invoke(Command(resume=resume_payload), config={"configurable": {"thread_id": f"{state['desk_id']}:{state['trade_id']}"}})
```

The audit row emitted on resume cites the firm's policy (e.g., "policy-3.2: block-trade compliance gate") in the same shape FAR 15.308 occupies for the federal-procurement case. The risk numbers in `risk_snapshot` are what the officer reviewed; they are not recomputed on resume.

Carrying the snapshot in state rather than recomputing on resume is the rule that travels with any hard-gate design: the human reviewed a specific artifact, and the audit row must let a later reader reconstruct it.

## 5. Real-world Patterns

**Healthcare — chemotherapy order verification.** Many EHR systems implement a hard stop for chemotherapy orders above a dose threshold. The order, the calculated dose-per-body-surface-area, and the prior-history flags are snapshotted; a second oncologist must approve before transmission to the infusion centre. The system does not refresh the patient's lab values mid-review. The audit row cites the policy and names the second oncologist by identity. Conceptually identical to FAR 15.308: load-bearing human judgement, no default, snapshot-at-review.

**Financial markets — large-order best-execution sign-off.** US broker-dealers must document best-execution analyses for certain large institutional orders under FINRA and SEC rules. The compliance record names the supervisor who reviewed the routing decision, the data they reviewed, and the rationale. The data — venue quotes, execution costs, fill probabilities — is a snapshot, not a live computation. If the snapshot is reconstructed after the fact from then-current data the audit is no longer defensible.

**Pharmaceutical clinical trials — investigator sign-off on adverse-event reports.** Clinical trial systems require the principal investigator to sign off on adverse-event reports with a recorded rationale within a regulatory window. The system can prepare the report; the investigator's judgement is what makes it a regulatory filing. The audit captures the report state at the moment of signature; a subsequent edit creates a new versioned filing with its own signature row. Snapshot, hard gate, citation, named human actor.

## 6. Best Practices

- **For any policy- or regulation-anchored gate, the audit row must include the citation.** A row that names the rule is a regulatory artifact; one that does not is merely a log.
- **Snapshot the materials into graph state at the moment of review.** Reading them at resume time invites tampering and breaks reconstructability.
- **Resume nodes are pure state transitions — no LLM calls, no RAG retrievals.** Anything non-deterministic belongs in upstream nodes that already wrote into state.
- **`actor_id` on the resume payload is the authenticated human, never `"system"`.**
- **Require rationale on resume and reject the payload without it.**
- **Use `graph.get_state_history(thread_id)` as the audit-replay surface.** Build any review UI on top of it rather than reinventing event sourcing.
- **Two structurally distinct gates get two `interrupt_before` entries.**

## 7. Hands-on Exercise

**Whiteboarding exercise (15 min).** Design the audit-row schema for a generic "executive sign-off on a board resolution" workflow. The workflow has these hard gates:

1. The CFO must approve the financial impact assessment before the resolution is added to the board agenda.
2. The board chair must approve the final resolution language before it is published.

For each gate:

- Name the policy citation (use a synthetic one; "board-policy-7.1" etc.).
- List the fields the audit row should carry.
- Identify what materials get snapshotted into state before the gate.
- Decide what happens if 14 days pass without a response.

**What good looks like.** Audit rows for both gates have `action`, `actor_id`, `node`, `before` (snapshot), `after` (decision + rationale), `correlation_id`, `ts`, `policy_citation`. The snapshot for gate 1 is the financial impact assessment text; for gate 2 it is the resolution text. The 14-day non-response action is *not* auto-approve — it is "escalate" (notify the company secretary; surface the stalled item in the next board meeting agenda). A weak answer auto-approves on timeout, conflating the work-item ageing with the workflow decision.

## 8. Key Takeaways

- Why does FAR 15.308 (or any "independent judgement required" rule) map to a hard interrupt and not a soft one? (LO1)
- What fields must a hard-interrupt audit row carry to be defensible, and which field is the *regulatory* anchor? (LO2)
- What is the "non-regeneration on resume" rule and what does it prevent? (LO3)
- Why are there *two* hard gates (decision + publication) rather than one, and what audit row does each emit? (LO4)
- If an auditor asks "show me every state this workflow passed through," what API call do I make and what does it return? (LO5)

## Sources

1. [FAR 15.308 — Source Selection Decision (acquisition.gov)](https://www.acquisition.gov/far/15.308) — retrieved 2026-05-26
2. [FAR 5.705 — Publicizing Postaward (acquisition.gov)](https://www.acquisition.gov/far/5.705) — retrieved 2026-05-26
3. [LangGraph Interrupts — interrupt_before, Command(resume=…), hard-gate model](https://docs.langchain.com/oss/python/langgraph/interrupts) — retrieved 2026-05-26
4. [LangGraph Persistence — checkpoints, `get_state_history`, replay surface](https://docs.langchain.com/oss/python/langgraph/persistence) — retrieved 2026-05-26
5. [LangGraph Durable Execution — determinism and consistent replay](https://docs.langchain.com/oss/python/langgraph/durable-execution) — retrieved 2026-05-26

Last verified: 2026-05-26
