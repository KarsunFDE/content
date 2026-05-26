---
template: pre-session-reading
week: W03
day: Thu
phase: Agentic
topic: "State machine fundamentals + LangGraph HITL deep-dive (HITL #5 — technical anchor)"
estimated_total_minutes: 60
last_verified: 2026-05-23
fde_situations: [3, 4, 5, 7, 11, 12]
tech: [LangGraph, LangChain v1.0, Bedrock InvokeModel, LangSmith, PostgresSaver, MemorySaver]
sources_research_briefs: [research/langgraph-state-machines-checkpointing.md, research/langgraph-interrupt-before.md]
author: instructor
---

# Pre-session reading — State machines + LangGraph HITL deep-dive (HITL #5)

Week 3, Day Thu. Estimated total time on task: ~60 minutes. Last verified: 2026-05-23.

## 1. Why this matters — this is the technical anchor day for HITL

Thursday is HITL #5 — the **LangGraph deep-dive technical anchor**. By end of day the evaluator → consensus → SSA-review state machine should be running end-to-end against the acquire-gov stack with `interrupt_before` nodes at every hard-gate transition. Soft vs hard interrupts get named and exercised. Checkpointing + persistence is mandatory because the SSA may not approve the SSDD draft until the next morning — the graph state has to survive the gap.

This is also the last day new tech lands in W3. Friday is gate day; no new concepts. Everything you ship today is what you defend Friday.

## 2. Core concept in 5 minutes — graph state is your contract

LangGraph's mental model: your agent flow is a directed graph; each node is a function that reads + writes a typed state object; edges (including conditional ones) describe transitions. The state is **the contract** — every node knows what fields exist and which it's allowed to write.

For the evaluator-flow:

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages

class EvaluationState(TypedDict):
    evaluation_id: str
    proposal_ids: list[str]
    evaluator_scores: dict[str, dict]   # proposal_id -> {evaluator_id -> score}
    consensus_complete: bool
    ssdd_draft: str | None
    ssa_approval_status: str            # "pending" | "approved" | "rejected" | None
    audit_correlation_id: str           # threaded through every node (Item 6 fix)
    messages: Annotated[list, add_messages]
```

The graph:

```
START
  -> supervisor_decide
       -> evaluator_score_proposal (parallel fan-out, one per proposal)
       -> consensus_aggregate                (waits for all evaluators)
       -> [interrupt_before] ssa_review_ssdd  (HARD GATE — FAR 15.308)
       -> [interrupt_before] record_award     (HARD GATE — FAR 5.705)
       -> END
```

Two hard interrupts. Both are FAR-anchored. Both write `AuditEvent` rows with `actor_id = human_reviewer` on resume.

## 3. Soft vs hard interrupts — name them, audit them differently

Soft and hard interrupts mean different things at the framework level *and* the audit level:

- **Soft interrupt** — `interrupt_before` with a *suggested* default action. The human can approve, reject, or *edit* the proposed action. The graph state captures the human's edit if any. (This is HITL #4's shape: supervisor proposes the next worker; human approves/edits before fire.)
- **Hard interrupt** — `interrupt_before` with **no default**. The graph blocks indefinitely until human input arrives. There is no auto-resume timeout. (This is HITL #5's anchor: SSA SSDD approval. The SSA *cannot* be auto-resumed by a timeout because FAR 15.308 says the SSA's independent judgment cannot be delegated — including to a timer.)

Both write audit rows. The schema differs:

```python
# Soft interrupt resume — audit_event
{
  "action": "HITL_SOFT_RESUME",
  "actor_id": "user:co-jdoe",
  "node": "supervisor_decide_next_worker",
  "before": {"proposed_action": "fan_out_evaluator_scoring"},
  "after":  {"approved_action":  "fan_out_evaluator_scoring", "edits": {}},
  "correlation_id": "<threaded from EvaluationState.audit_correlation_id>",
  "ts": "..."
}

# Hard interrupt resume — audit_event
{
  "action": "HITL_HARD_RESUME",
  "actor_id": "user:ssa-mhardy",
  "node": "ssa_review_ssdd",
  "before": {"ssdd_draft": "<full draft text>", "ssa_approval_status": "pending"},
  "after":  {"ssdd_draft": "<possibly-edited text>", "ssa_approval_status": "approved", "rationale": "<SSA text>"},
  "correlation_id": "<threaded>",
  "ts": "...",
  "far_citation": "15.308"
}
```

The `far_citation` field on hard interrupts is what makes the OIG audit trail defensible: when OIG asks "why did the platform let this transition fire?", the answer is the regulation that requires the human gate.

## 4. Checkpointing + persistence — the dev/prod split

- **Dev:** `MemorySaver` — in-process dict. Lost on restart. Fine for `pytest` + local dev.
- **Production-shape:** `PostgresSaver` — backed by Postgres tables (`checkpoints`, `checkpoint_blobs`, `checkpoint_writes`). Threads keyed by `evaluation_id`. Survives restarts. Required for hard interrupts (SSA approval may come 18+ hours later).

The persistence is what makes hard interrupts viable. Without it, you'd have to keep the FastAPI process up until the SSA shows up — operationally absurd.

## 5. What to read or watch tonight

- [LangGraph — Persistence and checkpointing](https://langchain-ai.github.io/langgraph/concepts/persistence/) (~12 min read), published 2025-11-08 (verified 2026-05-23 via /web-research). Focus on `Checkpointer` interface + the difference between `MemorySaver` and `PostgresSaver`. The Postgres setup is what your Thu afternoon practical wires.
- [LangGraph — Add human-in-the-loop with interrupt](https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/add-human-in-the-loop/) (~15 min read), published 2026-01-12 (verified 2026-05-23 via /web-research). Focus on the `Command(resume=...)` pattern + how `interrupt_before` differs from `interrupt_after`. This is the exact API the SSA-approval gate uses.
- [LangSmith — Debugging multi-step traces](https://docs.smith.langchain.com/observability/how_to_guides/trace_with_langgraph) (~10 min read), retrieved 2026-05-23 via /web-research. Focus on how interrupts appear in the trace UI — the trace pauses at the interrupt node + resumes when `Command(resume=...)` fires.
- [LangGraph — Time travel](https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/time-travel/) (~8 min read), published 2025-12-20 (verified 2026-05-23 via /web-research). Focus on `graph.get_state_history(thread_id)` — the OIG audit-replay surface. *If OIG asks "show me every state this evaluation passed through," this is the answer.*
- *(optional)* [Anthropic — Long-running agent workflows](https://www.anthropic.com/engineering/building-effective-agents) (~10 min focused on the "workflows vs agents" section), published 2024-12-19 (verified 2026-05-23 via /web-research). Useful background on why state machines beat unstructured agent loops at this kind of work.

## 6. Two questions to come in with tomorrow morning

1. Your Thu PR ships the SSDD-draft flow with `PostgresSaver`. The SSA approves the draft 18 hours after it was generated, but the underlying RAG corpus (W2's FAR/DFARS clause library) was re-indexed in the meantime. Should the SSA see the *original* draft or a *regenerated* draft? Defend either answer with audit-trail reasoning.
2. The Item 6 correlation-ID fix is *partial* this week (W3 threads it through the agent flow but the wider W3C `traceparent` rollout is W5). Where does the partial fix leave gaps in the cross-service trace? Name two specific cross-service hops where the correlation ID *won't* flow.

## 7. Glossary refresh (terms you'll hear today)

- **`StateGraph`** — LangGraph's primary builder. Nodes are `add_node`'d; edges are `add_edge` or `add_conditional_edges`.
- **`interrupt_before`** — list of node names that pause the graph before executing. Resumed via `Command(resume=...)`.
- **`Checkpointer`** — interface for persisting graph state. `MemorySaver` (dev), `PostgresSaver` (prod), `SqliteSaver` (small prod).
- **`thread_id`** — the persistence key. One `thread_id` per logical agent run (for us: one per `evaluation_id`).
- **`Command`** — the resume payload. Can carry `resume=<value>` (for hard interrupts that need human-supplied data) or `update=<state_dict>` (for edits to state on resume).
- **State `reducer`** — function that merges concurrent state writes (when parallel evaluator nodes write to the same `evaluator_scores` dict). `Annotated[dict, merge_reducer]` syntax.
- **HITL audit trail** — the audit_events emitted on every interrupt resume. Cohort builds the schema today.
- **FAR citation field** — on hard-interrupt audit rows, the specific FAR clause that requires the human gate. Defensibility for OIG.

## 8. What you'll ship today

- `ai-orchestrator/agents/evaluation_flow.py` — full `StateGraph` with the 4 named nodes + 2 `interrupt_before` configurations.
- `PostgresSaver` wired against the existing acquire-gov Postgres (new schema: `langgraph_checkpoints`). Migration in `services/evaluation-service/db/migrations/`.
- Tests: (a) graph runs end-to-end without interrupts in unit-test mode, (b) hard interrupt at `ssa_review_ssdd` blocks correctly + resumes with `Command(resume=...)`, (c) audit_event rows fire on resume with the correct `far_citation`.
- LangSmith traces showing the full flow with interrupt-pause + resume.
- Cost + latency instrumentation: token-per-node + latency-per-node logged to LangSmith metadata.

## 9. Anti-patterns to avoid today

- **Treating interrupts as exceptions.** Interrupts are *normal flow*, not error handling. Code defensively against the resume happening hours later, not seconds.
- **Forgetting the `correlation_id` thread.** Every node write must include it. Item 6 is *partially* fixed today; the discipline starts now even though the full W3C `traceparent` rollout is W5.
- **Auto-timing-out hard interrupts.** Hard interrupts are *FAR-anchored*. A timeout that auto-resumes the graph is a regulatory violation, not a feature.
- **Re-generating draft content on resume "to keep it fresh."** If the SSA approves the SSDD draft 18h later, the audit trail says *what was approved*. Auto-regeneration breaks that contract. (This is Q1 of tomorrow's prep questions.)
- **Skipping the LangSmith traces.** Friday's defense uses them as evidence. Cohort defending Phase 1 without trace screenshots will struggle.

## 10. Friday gate framing (light, 5 min — do this last)

- Defense format: 45 min/pair (15-min demo + 10-min Agency CIO Q&A + 10-min OIG Q&A + 10-min per-individual architecture defense). Tier-aware: Senior FDE defends production-shape; Entry FDE defends applied-recognition.
- Scope: cumulative W1–W3 (LLM essentials, RAG, agentic).
- Bring: working demo of the evaluator-flow with at least one hard-interrupt resume captured live; LangSmith trace screenshots; ADR catalog covering the three HITL touchpoints this week; the §0 retro on W2 plan.

## 11. Optional deep-dive (not required to participate)

- [LangGraph — Subgraphs](https://langchain-ai.github.io/langgraph/concepts/subgraphs/) (~10 min read), published 2026-02-18 (verified 2026-05-23 via /web-research). Useful if you want to defend "we could decompose this further" Friday — not required.
