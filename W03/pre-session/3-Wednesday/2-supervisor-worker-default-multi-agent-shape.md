---
week: W03
day: Wed
topic_slug: supervisor-worker-default-multi-agent-shape
topic_title: "Supervisor-worker — the default multi-agent shape"
parent_overview: W03/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://docs.langchain.com/oss/python/langgraph/workflows-agents
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.anthropic.com/engineering/multi-agent-research-system
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://github.com/langchain-ai/langgraph-supervisor-py
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.databricks.com/blog/multi-agent-supervisor-architecture-orchestrating-enterprise-ai-scale
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://blog.bytebytego.com/p/how-anthropic-built-a-multi-agent
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-06-06
---

# Supervisor-worker — the default multi-agent shape

> [!NOTE]
> **From earlier:** Mon's ADR exercise named which supervisor → worker transitions get HITL gates. Today those transitions become code — the supervisor-worker pattern is the topology they live in.

## 1. Learning Objectives

- Describe the three roles (supervisor, workers, optional synthesizer) and what each reads and writes.
- Explain the *delegation contract* — the per-arrow agreement on inputs, outputs, and firing conditions.
- Identify the two common failure modes (infinite loops, unbounded prompt growth) and standard mitigations.
- Recognize the supervisor-as-tool-caller as the canonical LangGraph v1.0 implementation shape.

## 2. Introduction

When a single agent's prompt outgrows usefulness — too many tools, wildly different reasoning styles, regressions that can't be pinpointed — the standard next step is a **supervisor-worker** topology: one coordinating agent that decides which specialist to call next, and workers that each do one narrow thing well.

The pattern is not new: hierarchical task networks, Erlang/OTP supervisors, and microservice orchestration have used this shape for decades. What changed is that the supervisor is now an LLM whose routing policy is a prompt, and workers are agents with their own context windows. What you gain is *separability* — evaluate, tune, and debug each worker in isolation. What you pay is latency (~1.5–2× wall clock) and token cost (~2–3×). Start with a single agent; move to supervisor-worker only when your eval forces the upgrade.

## 3. Core Concepts

### 3.1 The three roles

| Role | Responsibility | LangGraph shape |
|------|---------------|-----------------|
| **Supervisor** | Decides which worker to call next; declares done | LLM node emitting `next_worker` tool calls |
| **Worker** | Executes one narrow task; reads scoped state; writes results | Plain function node; no cross-worker calls |
| **Synthesizer** (optional) | Turns accumulated worker outputs into the final response | Separate node; own prompt + eval surface |

Workers do not call each other. All coordination flows through the supervisor — that single constraint is what makes multi-agent traces tractable.

### 3.2 The delegation contract

The load-bearing primitive is the **delegation contract** — the per-arrow agreement naming four things: (1) **Inputs** — the state slice the worker sees; (2) **Outputs** — the state keys the worker writes; (3) **Firing condition** — when the supervisor invokes this worker; (4) **Completion signal** — how the supervisor knows the worker finished and what to read.

Without an explicit contract, the trace becomes a guessing game.

### 3.3 Supervisor-as-tool-caller (canonical LangGraph v1.0 shape)

In LangGraph v1.0+, each worker is registered as a **tool the supervisor can call**. The supervisor's LLM emits a tool call; a `ToolNode` translates it into a transition to the corresponding worker node; the worker's return lands in shared state. This reuses the model's existing tool-calling capability — every delegation is a logged tool call, making routing visible and testable.

> [!IMPORTANT]
> **Not all problems need a supervisor.** A flat supervisor with N workers adds latency and cost that are only *earned* when your eval pinpoints a routing or specialization need. Pre-emptive hierarchy is the `supervisor-pattern-as-default` anti-pattern — pick topology to the problem.

### 3.4 Two common failure modes

- **Infinite delegation loops.** Supervisor keeps routing to the same worker because output never satisfies the completion condition. Mitigation: hard `max_iterations` cap at compile + a "good enough" threshold in the routing prompt.
- **Unbounded prompt growth.** Worker outputs accumulate in the supervisor's context until the routing call blows the window. Mitigation: curate outputs before re-entry — summarize, or store externally and pass handles.

## 4. Generic Implementation

Minimal supervisor-worker graph using LangGraph v1.0 primitives (no `Chain` class, no LCEL pipe):

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command

class FlowState(TypedDict):
    request: str
    notes: list[str]
    next_worker: str | None
    final: str | None

def worker_search(state: FlowState) -> Command:
    finding = do_search(state["request"])
    return Command(update={"notes": state["notes"] + [f"search: {finding}"]})

def worker_analyze(state: FlowState) -> Command:
    summary = analyze(state["notes"])
    return Command(update={"notes": state["notes"] + [f"analysis: {summary}"]})

def supervisor(state: FlowState) -> Command:
    decision = call_router_llm(state)   # returns "search" | "analyze" | "done"
    if decision == "done":
        return Command(update={"next_worker": None}, goto="terminator")
    return Command(update={"next_worker": decision}, goto=decision)

def terminator(state: FlowState) -> Command:
    answer = synthesize(state["notes"])
    return Command(update={"final": answer}, goto=END)

graph = StateGraph(FlowState)
graph.add_node("supervisor", supervisor)
graph.add_node("search", worker_search)
graph.add_node("analyze", worker_analyze)
graph.add_node("terminator", terminator)
graph.add_edge(START, "supervisor")
graph.add_edge("search", "supervisor")
graph.add_edge("analyze", "supervisor")
app = graph.compile()
```

The delegation contract is visible in code: `worker_search` reads `state["request"]`, writes `state["notes"]`, fires on `"search"`. No worker calls another worker.

## 5. Real-world Patterns

> [!TIP]
> **Separability is the core win.** Each worker can be evaluated, prompted, and iterated on in isolation — regressions pinpoint to one specialist, new capabilities attach as one more worker.

**Fintech — fraud triage.** A supervisor routes among velocity-rules, graph-features, and LLM-explanation workers. Workers can't see each other's outputs until synthesis — preventing signal bias. Production surveys report 60–70% analyst review-time reductions when workers are well-scoped.

**Anthropic Research.** A lead agent spawns parallel workers for labs, imaging, and history review; a synthesizer reconciles findings with explicit citations. Reported 90.2% improvement over single-agent Claude on their internal eval.

## 6. Best Practices

- **Start with a single agent.** Move to supervisor-worker only when your eval pinpoints a routing or specialization need.
- **Make the delegation contract explicit in code.** TypedDict or Pydantic model; surface which keys the worker reads and writes.
- **Always set `max_iterations` on the compiled graph.** Infinite delegation loops are the #1 production failure mode.
- **Curate worker outputs before re-entering the supervisor's context.** Summarize or store externally; pass handles, not full transcripts.
- **Log every delegation as a structured event.** Tool-call-shaped invocations make this free.

> [!WARNING]
> **Anti-pattern: `supervisor-pattern-as-default`.** Not all multi-agent problems need a supervisor. Using a supervisor pre-emptively on a homogeneous, single-concern workload adds 1.5–2× latency and 2–3× token cost for no benefit. Pick the simplest topology that passes your eval. `supervisor-pattern-as-default` is an operative slug — recognize it by name.

## 7. Hands-on Exercise

You are designing an **interview-prep assistant** (not federal-acquisitions). The product must: (a) pick a problem matched to skill level, (b) give a hint if the user is stuck, (c) review the submitted solution, (d) suggest a follow-up.

Draw the supervisor-worker topology naming: each worker + one-sentence scope; delegation contract per arrow (inputs, outputs, firing condition); one failure mode you've designed against; the state schema.

> [!NOTE]
> **Self-check** (30s — answer mentally before expanding)
>
> 1. A supervisor fires its routing LLM after every single state mutation. Which failure mode is this, and what's the standard fix?
> 2. Two workers both write to the same state key without a reducer. What happens in LangGraph?

<details>
<summary>Show answers</summary>

1. Chatty handoffs — the supervisor re-invokes between every step rather than every decision boundary. Fix: add explicit `goto` from workers back to supervisor only after a unit of work completes; add a "re-plan only when X condition" clause to the routing prompt.
2. LangGraph raises `InvalidUpdateError` on concurrent writes to a non-reducer key. Declare `Annotated[list[T], operator.add]` or an equivalent reducer on any key two parallel branches write to.

</details>

> [!NOTE]
> **LangGraph v1.0 posture (D-033).** No `Chain` class, no LCEL pipe as foundation. Supervisor-as-tool-caller uses plain Python function composition — `supervisor` calls a routing LLM, emits a tool call, `ToolNode` transitions to the worker node. That's the canonical v1.0 shape.

## 8. Key Takeaways

- Supervisor routes, workers execute one narrow task, synthesizer (optional) composes the final answer.
- Delegation contract (inputs / outputs / firing / completion) makes multi-agent traces debuggable.
- Supervisor-as-tool-caller is the canonical LangGraph v1.0 shape — each delegation is a logged tool call.
- Move to supervisor-worker only when your eval forces it; latency and token cost are real.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://docs.langchain.com/oss/python/langgraph/workflows-agents — retrieved 2026-05-26 — hot-tech
- https://www.anthropic.com/engineering/multi-agent-research-system — retrieved 2026-05-26 — hot-tech
- https://github.com/langchain-ai/langgraph-supervisor-py — retrieved 2026-05-26 — hot-tech
- https://www.databricks.com/blog/multi-agent-supervisor-architecture-orchestrating-enterprise-ai-scale — retrieved 2026-05-26 — hot-tech
- https://blog.bytebytego.com/p/how-anthropic-built-a-multi-agent — retrieved 2026-05-26 — hot-tech

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

**Routing accuracy vs invocation rhythm.** A common supervisor-tuning trap is optimizing for routing accuracy (does the supervisor pick the right worker?) while ignoring invocation rhythm (does it fire at the right cadence?). A 90%-accurate supervisor that fires 10× too often is worse in production than an 80%-accurate one at the right rhythm — the former costs 10× more and produces unreadable traces. Profile both dimensions independently.

**Subgraph embedding for organizational boundaries.** When teams own distinct worker clusters (different codebases, different deployment cadences), embed each cluster as a compiled LangGraph subgraph node in the parent. The parent supervisor sees only the subgraph's typed output — it cannot introspect internal worker state. This is the hierarchical-orchestration shape and is covered in the next topic file.

**Delegation contract as a test fixture.** Each worker's typed input contract doubles as a test fixture: you can unit-test each worker in isolation by passing a minimal TypedDict input without invoking the supervisor or the graph. This is the same pattern as hexagonal architecture's "port" — the contract is both the integration seam and the test interface.

</details>

Last verified: 2026-06-06
