---
template: pre-session-reading
week: W03
day: Thu
phase: Agentic
topic: "State machine fundamentals + LangGraph HITL deep-dive (HITL #5 — technical anchor)"
estimated_total_minutes: 70
last_verified: 2026-05-26
fde_situations: [3, 4, 5, 7, 11, 12]
tech: [LangGraph, LangChain v1.0, Bedrock InvokeModel, LangSmith, PostgresSaver, MemorySaver]
sources_research_briefs: [research/langchain-v1-20260522.md]
author: instructor
---

# Pre-session reading — W3 Thu — State machines + LangGraph HITL deep-dive (HITL #5)

Week 3, Day Thu. Estimated total time on task: ~70 minutes. Last verified: 2026-05-26.

## Why this matters

Thursday is **HITL #5 — the LangGraph deep-dive technical anchor**. By 17:00, the evaluator → consensus → SSA-review state machine ships end-to-end against the acquire-gov stack with `interrupt_before` nodes at every hard-gate transition (the static-breakpoint primitive the cohort has been wiring since Mon — kept canonical for FAR-anchored gates this week). Soft vs hard interrupts get named and exercised. PostgresSaver persists the graph state across an 18-hour gap — because the SSA may not click approve until tomorrow morning, and FAR 15.308 says her independent judgment cannot be delegated, including to a timeout.

> [!instructor-review]
> **Static `interrupt_before=[...]` vs dynamic `interrupt()` — canonical story for W3.** The programme's canonical primitive Mon→Tue→Wed→Thu is **static `interrupt_before`** for the FAR-anchored gates — it is the simpler, more inspectable primitive for the named gate boundaries in `evaluation_flow.py`. The LangGraph docs (retrieved 2026-05-26) describe **dynamic `interrupt()`** as the docs-canonical alternative for production HITL where the pause condition is itself dynamic. Today's ADR exercise (per pair) names which primitive each of the two hard gates uses: static `interrupt_before` (recommended, matches the wiring you've practised all week) OR dynamic `interrupt()` (acceptable if the pair can defend why a runtime-conditional pause is warranted at that gate). Do not let a cohort member read the LangChain blog and conclude the curriculum is stale — surface this nuance live.

This is also the **last day new tech lands in W3**. Friday is gate day; no new concepts. Everything you ship today is what you defend tomorrow. Mid-Programme Gate countdown: 1 day.

---

## 1. State schema design — graph state is your contract (10 min)

LangGraph's mental model: your agent flow is a directed graph; each node is a function that reads + writes a typed state object; edges (including conditional ones) describe transitions. The state is **the contract** — every node knows what fields exist and which it's allowed to write.

For the evaluator-flow against acquire-gov:

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages

class EvaluationState(TypedDict):
    evaluation_id: str
    agency_id: str                      # multi-tenant thread-key (Item 10 fix)
    proposal_ids: list[str]
    evaluator_scores: dict[str, dict]   # proposal_id -> {evaluator_id -> score}
    consensus_complete: bool
    ssdd_draft: str | None              # snapshot at consensus; never regenerated
    ssa_approval_status: str            # "pending" | "approved" | "rejected"
    audit_correlation_id: str           # threaded through every node (Item 6 fix)
    messages: Annotated[list, add_messages]
```

`Annotated[list, add_messages]` is the reducer pattern — when parallel evaluator nodes write to the same field, the reducer merges. Without a reducer, parallel writes either overwrite or raise. **State without a schema is a bug magnet at multi-agent scale.** Today the schema is locked before any node code lands.

## 2. Parallel execution in LangGraph — fan-out, fan-in (8 min)

Multiple evaluator-agents scoring different proposals at the same time before fan-in to consensus. In LangGraph v1.x this is just adding multiple edges from one node — the runtime handles concurrency. The load-bearing primitive is the **state reducer**: when 4 evaluator nodes write to `evaluator_scores` concurrently, the reducer (or `Annotated[dict, merge_reducer]`) deterministically merges.

Practical wiring against `EvaluationState.evaluator_scores`:

```python
graph.add_node("supervisor_decide", supervisor_decide)
graph.add_node("evaluator_score_proposal", evaluator_score_proposal)
graph.add_node("consensus_aggregate", consensus_aggregate)

graph.add_edge(START, "supervisor_decide")
# fan-out: supervisor returns N proposal_ids; runtime spawns N evaluator runs in parallel
graph.add_conditional_edges("supervisor_decide", route_to_evaluators)
# fan-in: all N evaluators converge here
graph.add_edge("evaluator_score_proposal", "consensus_aggregate")
```

This builds on Wed's *parallel fan-out* topic at code-implementation depth. The Wed pattern was the *shape*; today is the *wiring*.

## 3. Checkpointing + persistence — MemorySaver vs PostgresSaver (12 min)

The dev/prod split is not optional today — it's what makes hard interrupts viable.

- **Dev:** `MemorySaver` (alias for `InMemorySaver` in current LangGraph) — in-process dict; lost on restart. Fine for `pytest` + local dev only.
- **Production-shape:** `PostgresSaver` — backed by Postgres tables (`checkpoints`, `checkpoint_blobs`, `checkpoint_writes`); survives FastAPI restarts; survives the 18-hour SSA gap. Requires `.setup()` once to create the tables.

```python
from langgraph.checkpoint.postgres import PostgresSaver
from psycopg_pool import ConnectionPool

DB_URI = "postgresql://acquire_gov:...@postgres:5432/acquire_gov"
pool = ConnectionPool(DB_URI, max_size=20)
checkpointer = PostgresSaver(pool)
checkpointer.setup()  # one-time table creation

app = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["ssa_review_ssdd", "record_award"],
)
```

**Thread key shape:** `thread_id = f"{agency_id}:{evaluation_id}"` — not just `evaluation_id`. If two agencies collide on an internal int, careless namespacing is Item 10 multi-tenant leakage. SSA approves Agency A's draft; your resume reads Agency B's checkpoint; you cross-publish. **Add `agency_id` to the thread key.**

Without persistence you'd need to keep the FastAPI process up for 18 hours waiting for the SSA — operationally absurd, and one CI restart breaks it. PostgresSaver is the persistence layer that makes hard interrupts work.

## 4. Soft vs hard interrupts — name them, audit them differently (10 min)

Soft and hard interrupts mean different things at the framework level *and* the audit level. The cohort exercised soft yesterday (HITL #4); today is the hard pattern.

- **Soft interrupt** — `interrupt_before` with a *suggested* default action. The human can approve, reject, or *edit* the proposed action. The graph state captures the edit if any. (HITL #4's shape: supervisor proposes the next worker; human approves/edits before fire.)
- **Hard interrupt** — `interrupt_before` with **no default**. The graph blocks indefinitely until human input arrives via `Command(resume=...)`. **There is no auto-resume timeout.** A timeout that auto-resumes a FAR-anchored gate is itself a regulatory violation (FAR 15.308 says independent judgment cannot be delegated — and delegating to a timer is delegation).

Both write audit rows. The schemas differ:

```python
# Soft interrupt resume (HITL #4 shape from Wed)
{
  "action": "HITL_SOFT_RESUME",
  "actor_id": "user:co-jdoe",
  "node": "supervisor_decide_next_worker",
  "before": {"proposed_action": "fan_out_evaluator_scoring"},
  "after":  {"approved_action":  "fan_out_evaluator_scoring", "edits": {}},
  "correlation_id": "<from EvaluationState.audit_correlation_id>",
  "ts": "..."
}

# Hard interrupt resume (HITL #5 shape — today)
{
  "action": "HITL_HARD_RESUME",
  "actor_id": "user:ssa-mhardy",
  "node": "ssa_review_ssdd",
  "before": {"ssdd_draft": "<full draft text>", "ssa_approval_status": "pending"},
  "after":  {"ssdd_draft": "<possibly-edited text>", "ssa_approval_status": "approved",
             "rationale": "<SSA text>"},
  "correlation_id": "<threaded>",
  "ts": "...",
  "far_citation": "15.308"
}
```

The **`far_citation` field on hard-interrupt rows is what makes the OIG audit trail defensible**: when OIG asks *"why did the platform let this transition fire?"*, the answer is the regulation that *required* the human gate. Without that field, the row is a transition log; with it, it's a regulatory artifact.

## 5. HITL #5 — LangGraph hard interrupt + the FAR 15.308 audit row (15 min)

**This is the seventh-touchpoint thread's mid-programme anchor.** HITL #5 is structurally distinct from #3 (Mon ADR-level) and #4 (Wed soft interrupt) — it's the *hard, FAR-anchored, OIG-replayable* shape.

The SSA's anchor question (war-room D4): *"It's 17:00 — 18 hours after consensus completed. When I click approve, is your system going to record what I approved, or regenerate the draft against today's RAG corpus because the FAR was amended this morning?"*

Two hard interrupts in the evaluator-flow. Both are FAR-anchored. Both write audit rows on resume.

```
START
  -> supervisor_decide
       -> evaluator_score_proposal (parallel fan-out, one per proposal)
       -> consensus_aggregate                (waits for all evaluators)
       -> [interrupt_before] ssa_review_ssdd  (HARD GATE — FAR 15.308)
       -> [interrupt_before] record_award     (HARD GATE — FAR 5.705)
       -> END
```

The four design moves the SSA's question forces:

1. **Snapshot the SSDD draft into graph state at consensus completion.** Draft text becomes part of `EvaluationState.ssdd_draft`. The state IS the truth.
2. **Never re-invoke Bedrock on resume.** Resume just transitions state. No LLM call. No RAG retrieval. "Freshening" the draft IS the FAR 15.308 violation.
3. **Two hard interrupts, no timeout.** `ssa_review_ssdd` (FAR 15.308: SSA independent judgment) + `record_award` (FAR 5.705: award publication is irreversible). Both block indefinitely.
4. **`actor_id` is the SSA, not the system.** The graph didn't approve; the human did. `actor_id = f"user:{ssa_user_id}"`. This is the non-delegation rule made auditable.

`graph.get_state_history(thread_id)` is the **OIG audit-replay surface**. If OIG asks *"show me every state this evaluation passed through,"* this is the answer — every checkpoint, every interrupt, every resume, in order, with the FAR citation attached to each hard gate.

## 6. Workflow resiliency + debugging LangGraph with LangSmith (8 min)

What happens when Bedrock 429s mid-graph? **Checkpointing means resume-from-last-good-node, not restart.** The runtime catches the exception, the last good checkpoint persists, you redrive from `Command(resume=...)` — no double-billing on already-completed evaluator scores.

Debugging multi-step traces in LangSmith — how interrupts appear in the trace UI: the trace **pauses at the interrupt node and resumes when `Command(resume=...)` fires**, with a visible gap that may span hours. Tomorrow's defense uses these traces as evidence — "LangSmith trace screenshots ready" is on the Phase 1 Defense rubric.

```python
# Capture trace metadata on every node for the resiliency story
from langsmith import traceable

@traceable(run_type="chain", name="evaluator_score_proposal")
def evaluator_score_proposal(state: EvaluationState) -> dict:
    ...
```

When Bedrock returns a 429, LangSmith records the retry; when the human resumes, the trace timeline shows the gap clearly. Defending Friday without these traces is defending narrative without evidence.

## 7. Cost + latency management — per-node instrumentation (7 min)

Every evaluator-agent invocation is a Bedrock call. Multi-agent flows multiply call counts — what was 2 calls last week is 4N+2 this week for N proposals. Instrument per-node, not just per-flow.

Per-node metadata to log into LangSmith (OTel GenAI Semantic Conventions vocabulary, `gen_ai.*` namespace):

- `gen_ai.usage.input_tokens` + `gen_ai.usage.output_tokens`
- `gen_ai.request.model` (which Claude model on Bedrock)
- Wall-clock latency per node
- Tool-call count per node
- Projected cost (Bedrock per-token pricing × token count)

**Two thresholds to alert on:**

1. **Per-evaluation cost ceiling** — if projected cost > $X, escalate to CO before continuing. Band shape (`$X`–`$Y` per evaluation) is a pair-decision tonight; the specific dollar values are deliberately deferred. [!instructor-review] *Confirm specific Bedrock cost band via `/web-research` Sunday-before W3 Mon — the value of the deferred figure is the live re-verification; do not pre-commit to a band that may have shifted.*
2. **Per-node latency p95** — if `consensus_aggregate` p95 > 60s, the consensus prompt is too expensive. Coach toward decomposition (smaller per-evaluator summaries, then aggregate).

This is the foundation for W5's **AIOps cost-as-signal** topic — wire it well today and Week 5 builds on it instead of refactoring it.

---

## What you'll ship today (EOD Thu 17:00)

- `ai-orchestrator/agents/evaluation_flow.py` — full `StateGraph` with 4 named nodes + 2 `interrupt_before` configurations.
- `PostgresSaver` wired against acquire-gov Postgres (new schema: `langgraph_checkpoints`). Migration in `services/evaluation-service/db/migrations/`. Thread keys = `f"{agency_id}:{evaluation_id}"`.
- Tests: (a) graph runs end-to-end without interrupts (`MemorySaver` unit-test mode), (b) hard interrupt at `ssa_review_ssdd` blocks correctly + resumes with `Command(resume=...)`, (c) audit_event rows fire on resume with correct `far_citation`, (d) **restart test passing** — FastAPI Ctrl+C → restart → resume reads same draft text (not regenerated).
- LangSmith traces showing the full flow with interrupt-pause + resume captured.
- Cost + latency instrumentation in LangSmith metadata (token-per-node + latency-per-node).
- Codex Adversarial Review on the PR (Near-full strictness per D-034).

## Two questions to come in with tomorrow morning

1. Your Thu PR ships the SSDD-draft flow with `PostgresSaver`. The SSA approves the draft 18 hours after it was generated, but the underlying RAG corpus (W2's FAR/DFARS clause library) was re-indexed in the meantime. Should the SSA see the *original* draft or a *regenerated* draft? Defend with audit-trail reasoning. (The SSA's anchor question. Answer it before you code.)
2. The Item 6 correlation-ID fix is *partial* this week (W3 threads it through the agent flow but the wider W3C `traceparent` rollout is W5). Where does the partial fix leave gaps in the cross-service trace? Name two specific cross-service hops where the correlation ID *won't* flow.

## Anti-patterns to avoid today (reinforcement — these map to topics 3–7 above)

- **Regenerating draft content on resume "to keep it fresh."** That IS the FAR 15.308 violation. Audit row records what was approved; auto-regen breaks the contract.
- **Auto-timing-out hard interrupts.** Hard interrupts are FAR-anchored. A timeout that auto-resumes is a regulatory violation, not a feature.
- **`thread_id = evaluation_id` (no agency prefix).** Item 10 multi-tenant leakage waiting to fire.
- **`actor_id = "system"` on resume.** The actor is the SSA. `user:{ssa_user_id}`. Non-delegation made auditable.
- **Treating interrupts as exceptions.** Interrupts are *normal flow*, not error handling. Code defensively against resume happening hours later, not seconds.
- **Skipping the LangSmith traces.** Tomorrow's defense uses them as evidence. Defending Phase 1 without trace screenshots will struggle.
- **Skipping the restart test.** It's the test that proves FAR 15.308 compliance. Without it, the audit row schema is hopeful, not verified.

## Glossary refresh

- **`StateGraph`** — LangGraph's primary builder. Nodes are `add_node`'d; edges are `add_edge` or `add_conditional_edges`.
- **`interrupt_before`** — list of node names that pause the graph before executing. Resumed via `Command(resume=...)`.
- **`Checkpointer`** — interface for persisting graph state. `MemorySaver` (dev), `PostgresSaver` (prod), `SqliteSaver` (small prod).
- **`thread_id`** — the persistence key. One per logical agent run. For us: `f"{agency_id}:{evaluation_id}"`.
- **`Command`** — the resume payload. Carries `resume=<value>` (hard interrupts) or `update=<state_dict>` (state edits on resume).
- **State `reducer`** — function that merges concurrent state writes (parallel evaluator nodes writing to `evaluator_scores`). `Annotated[dict, merge_reducer]` syntax.
- **HITL audit trail** — audit_events emitted on every interrupt resume. Cohort builds the schema today.
- **`far_citation`** — field on hard-interrupt audit rows. Specific FAR clause that requires the human gate. Defensibility for OIG.
- **`graph.get_state_history(thread_id)`** — OIG audit-replay surface. Every checkpoint, every interrupt, every resume.

## Friday is gate day — short framing (read last, 3 min)

- Defense format: 45 min/pair (15-min demo + 10-min Agency CIO Q&A + 10-min OIG Q&A + 10-min per-individual architecture defense). Tier-aware: Senior FDE defends production-shape; Entry FDE defends applied-recognition.
- Scope: cumulative W1–W3 (LLM essentials, RAG, agentic).
- Bring: working demo of the evaluator-flow with at least one hard-interrupt resume captured live; LangSmith trace screenshots; ADR catalog covering the three HITL touchpoints this week (#3 Mon ADR, #4 Wed soft, #5 Thu hard); the §0 retro on W2 plan.
- Tomorrow's pre-session is short on purpose — the work tomorrow morning *is* the defense itself.

## Optional deep-dive (not required)

- [LangGraph — Subgraphs](https://docs.langchain.com/oss/python/langgraph/subgraphs) (~10 min read), retrieved 2026-05-26 via /web-research. Useful if you want to defend "we could decompose this further" Friday — not required.

## Sources

- LangGraph Interrupts (v1.x docs — covers `interrupt()`, `Command(resume=...)`, `interrupt_before`, checkpointer requirement), docs.langchain.com, retrieved 2026-05-26 via /web-research. <https://docs.langchain.com/oss/python/langgraph/interrupts>
- LangGraph Persistence (v1.x docs — InMemorySaver / MemorySaver alias, SqliteSaver, PostgresSaver/AsyncPostgresSaver, `.setup()` requirement), docs.langchain.com, retrieved 2026-05-26 via /web-research. <https://docs.langchain.com/oss/python/langgraph/persistence>
- PostgresSaver API reference (langgraph-checkpoint-postgres v1.x — table schema, `setup()`, `list`/`put` methods), reference.langchain.com, retrieved 2026-05-26 via /web-research. <https://reference.langchain.com/python/langgraph.checkpoint.postgres/PostgresSaver>
- FAR 15.308 Source Selection Decision ("the source selection decision shall represent the SSA's independent judgment"), acquisition.gov, retrieved 2026-05-26 via /web-research. <https://www.acquisition.gov/far/15.308>
- LangChain v1.0 research brief (langchain v1.3.0, langgraph v1.2.0, no breaking changes until 2.0 — D-033 hygiene), `research/langchain-v1-20260522.md`, 2026-05-22. Internal pipeline brief.
- OpenTelemetry GenAI Semantic Conventions (`gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens` / `gen_ai.request.model` namespace), opentelemetry.io, retrieved 2026-05-26 via /web-research. <https://opentelemetry.io/docs/specs/semconv/gen-ai/>

Last verified: 2026-05-26
