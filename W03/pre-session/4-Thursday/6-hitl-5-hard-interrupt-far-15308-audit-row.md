---
week: W03
day: Thu
topic_slug: hitl-5-hard-interrupt-far-15308-audit-row
topic_title: "HITL #5 — LangGraph hard interrupt + the FAR 15.308 audit row"
parent_overview: W03/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 11
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
last_verified: 2026-06-06
---

# HITL #5 — LangGraph hard interrupt + the FAR 15.308 audit row

> [!NOTE]
> **From earlier:** File 5 named the two interrupt shapes and their audit-row differences. This file is the application: HITL #5, the programme's mid-point hard-gate anchor, grounded in FAR 15.308 — the cleanest "independent judgement required" rule in the federal acquisition canon.

## 1. Learning Objectives

By the end of this reading, you can:

- Explain why a regulation requiring "independent judgement" maps to a hard interrupt and not a soft one.
- Design a defensible audit-row schema for a hard-interrupt resume: regulation cited, human actor named, inputs snapshotted, decision recorded.
- Apply the non-regeneration rule: snapshot draft content into graph state before the gate; never re-invoke a model on resume.
- Trace the two-hard-gate pattern (`ssa_review_ssdd` + `record_award`) and explain why each is structurally distinct.
- Use `graph.get_state_history(thread_id)` as an OIG audit-replay surface.

## 2. Introduction

FAR 15.308 reads in part: *"the source selection decision shall represent the SSA's independent judgment…and shall be documented"* ([source: acquisition.gov, retrieved 2026-05-26](https://www.acquisition.gov/far/15.308)). Three words carry the load: **independent** (not delegated to a timer), **judgment** (the SSA exercises discretion — the system prepares, cannot pre-decide), **documented** (the audit row *is* the documentation).

The compliance question is not "did the SSA click approve?" It is: "is there a defensible record that the SSA individually judged the proposals?" Today you build the code that makes that record possible.

## 3. Core Concepts

### 3.1 Why hard, not soft

Soft interrupts ship a default — if no human responds, the default fires, or the human rubber-stamps a precomputed answer. Neither satisfies FAR 15.308: the default lets the system decide in the SSA's absence; rubber-stamping makes the system's analysis the judgement.

Hard interrupt: `ssa_approval_status` is unwritten until the SSA's resume payload fills it. No timer. Her decision is what advances the workflow.

> [!IMPORTANT]
> **`actor_id` is the SSA, not the system.** `actor_id = f"user:{ssa_user_id}"`. `"actor_id = system"` on a hard-gate row is the same as no gate from an audit perspective.

### 3.2 The two-hard-gate pattern

```python
app = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["ssa_review_ssdd", "record_award"],
)
```

| Gate | Node | FAR anchor | Question it answers |
|------|------|-----------|---------------------|
| Decision | `ssa_review_ssdd` | FAR 15.308 | "Is this the right choice?" |
| Publication | `record_award` | FAR 5.705 | "Do we make it public?" |

Both hard; both emit audit rows with different `far_citation` values.

### 3.3 The non-regeneration rule on resume

The subtlest hard-interrupt bug: the SSA reviews at 09:00, the FAR clause library re-indexes at 09:30, she approves at 10:00, the system regenerates the draft. Contract violation: **her judgement was applied to the 09:00 draft.**

Rule: (1) snapshot `ssdd_draft` into graph state at consensus completion; (2) on resume, no model call; (3) the audit row stores both snapshot and decision so a reviewer can reconstruct what the SSA saw.

### 3.4 The defensible audit-row schema

| Field | Value / source | Why |
|-------|---------------|-----|
| `action` | `"HITL_HARD_RESUME"` | Distinguishes from soft resumes |
| `actor_id` | Authenticated user — never `"system"` | Named individual |
| `before` | State snapshot at interrupt | What the human saw |
| `after` | Resume payload | Decision + rationale |
| `rationale` | Required string | Human's own words |
| `far_citation` | `"15.308"` or `"5.705"` | Regulation requiring this gate |

### 3.5 `graph.get_state_history` — the replay surface

`graph.get_state_history(thread_id)` returns every checkpoint in order ([source: docs.langchain.com persistence, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/persistence)). An OIG auditor gets every state the evaluation passed through — every interrupt and resume, FAR citations attached — without re-executing anything.

> [!TIP]
> **`get_state_history` is your UI data source.** Build the "review history" view directly on checkpoint data — the checkpoint store is already the event store. A separate event log duplicates it and risks consistency drift.

## 4. Generic Implementation

Securities trading compliance gate — compliance officer must approve large block trades before exchange routing:

```python
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command
from typing import TypedDict

class BlockTradeState(TypedDict):
    trade_id: str
    desk_id: str
    notional_usd: float
    risk_snapshot: dict | None
    compliance_decision: str | None
    compliance_rationale: str | None
    compliance_officer_id: str | None
    audit_correlation_id: str

def compute_pre_trade_risk(state: BlockTradeState) -> dict:
    snapshot = run_risk_model(state["trade_id"])
    return {"risk_snapshot": snapshot}

def compliance_review(state: BlockTradeState) -> dict:
    return {}   # decision fields set by the resume payload

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

resume_payload = Command(resume={
    "compliance_decision": "approved",
    "compliance_rationale": "Reviewed VaR and counterparty exposure; within mandate.",
    "compliance_officer_id": "user:compliance-mhardy",
})
app.invoke(
    resume_payload,
    config={"configurable": {"thread_id": f"{state['desk_id']}:{state['trade_id']}"}},
)
```

The risk snapshot is what the officer reviewed — not recomputed on resume. The firm's policy citation occupies the same position FAR 15.308 does in the federal case.

## 5. Real-world Patterns

**Healthcare — chemotherapy hard stop.** EHR systems snapshot the order and dose; a second oncologist approves before transmission without refreshing lab values mid-review. Audit row cites policy, names the oncologist — structurally identical to FAR 15.308.

> [!NOTE]
> **Cross-domain lesson:** Snapshot-then-gate appears wherever a human must sign off on a *specific artifact* — not "the latest version." Financial best-execution and pharma adverse-event filings share this invariant: the auditable record must match what the human actually reviewed.

**Financial markets — best-execution sign-off.** Broker-dealers document best-execution analyses under FINRA/SEC rules. Reconstructing data from then-current quotes after the fact makes the audit indefensible — same reason the SSA's draft must not regenerate on resume.

## 6. Best Practices

- **Policy citation on every hard-gate row.**
- **Snapshot materials before the gate.**
- **Resume nodes are pure state transitions.** No LLM calls, no RAG.
- **`actor_id` is the authenticated human, never `"system"`.**
- **Require rationale; reject the resume payload without it.**
- **Two distinct gates = two `interrupt_before` entries.**

> [!WARNING]
> **Anti-pattern: `interrupt-as-pause-only`.** `interrupt()` is not a pause button. The value passed to `Command(resume=value)` becomes the return of `interrupt()` inside the node — the resume payload is the human's decision, not an "OK" signal. A hard gate where the node ignores the resume payload and treats it as a bare acknowledgement misses the point: the decision must drive the next state transition, not just unblock the graph.

## 7. Hands-on Exercise

Design the audit-row schema for a "board resolution" workflow: CFO approves the financial impact assessment before the agenda; board chair approves final resolution language before publication. For each gate: name the policy citation, list audit-row fields, identify what gets snapshotted, and say what happens if 14 days pass without a response.

> [!NOTE]
> **Self-check** (30 s — answer mentally before expanding)
>
> 1. SSA reviews the SSDD draft at 09:00. The FAR clause library is re-indexed at 09:30. She clicks approve at 10:00. Should the system regenerate the draft before recording the approval?
> 2. Your audit row has `actor_id = "system"` because resume was submitted via an automated webhook. What is the compliance problem?

<details>
<summary>Show answers</summary>

1. No. The SSA's independent judgement was applied to the draft that existed at 09:00. Regenerating it at 10:01 — even with fresher data — means the artifact the system records is not the artifact the SSA reviewed. The audit row would read "SSA approved at 10:00" but the document would be the 10:01 version. A later reviewer cannot reconstruct what the SSA actually saw. Snapshot the draft at consensus completion; never regenerate on resume.
2. The compliance problem: no human made this decision. The audit row records "system" as the approver of a gate that exists precisely because FAR 15.308 requires the SSA's *individual, independent* judgement. An automated webhook submitting the resume means the system approved itself — the non-delegation rule is violated. The `actor_id` must be the authenticated human who took the action, populated from their authenticated session, not from the calling process.

</details>

## 8. Key Takeaways

- FAR 15.308's three words — independent, judgment, documented — each map to a technical constraint: no delegation, no precomputed answer, defensible record.
- Hard interrupt = no default, blocks indefinitely, `far_citation` on the audit row.
- Two hard gates: `ssa_review_ssdd` (FAR 15.308) + `record_award` (FAR 5.705).
- `graph.get_state_history(thread_id)` is the OIG audit-replay surface.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://www.acquisition.gov/far/15.308 — retrieved 2026-05-26 — federal-regulatory
- https://www.acquisition.gov/far/5.705 — retrieved 2026-05-26 — federal-regulatory
- https://docs.langchain.com/oss/python/langgraph/interrupts — retrieved 2026-05-26 — hot-tech
- https://docs.langchain.com/oss/python/langgraph/persistence — retrieved 2026-05-26 — hot-tech
- https://docs.langchain.com/oss/python/langgraph/durable-execution — retrieved 2026-05-26 — hot-tech

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

**`get_state_history` as a UI primitive.** For a production HITL UI, `graph.get_state_history(thread_id)` is the data source for the "review history" view — every state the workflow passed through, with timestamps and which node ran. Build the review UI on top of this rather than maintaining a separate event log. The checkpoint store *is* the event store; duplicating it is a consistency risk. Senior FDEs who wire this surface Friday have a credible answer to "how does your OIG review UI work?"

**Tamper evidence for checkpoints.** The checkpoint store is the single source of truth for the audit trail. If an attacker can write to the Postgres checkpoint tables directly, they can alter the audit record. Production defences: (1) the database user the app connects with should have INSERT/SELECT only on the checkpoint tables — no UPDATE or DELETE; (2) append-only audit_events in a separate table (sole writer: `log_event()`) provides a second, independent record; (3) `LANGGRAPH_STRICT_MSGPACK=true` prevents crafted payloads from exploiting deserialisation. These three together make the checkpoint record tamper-evident at the application level.

</details>

Last verified: 2026-06-06
