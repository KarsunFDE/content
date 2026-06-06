---
week: W03
day: Thu
topic_slug: parallel-execution-fan-out-fan-in
topic_title: "Parallel execution in LangGraph — fan-out, fan-in"
parent_overview: W03/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://docs.langchain.com/oss/python/langgraph/graph-api
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://langchain-ai.github.io/langgraph/concepts/low_level/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/oss/python/langgraph/persistence
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://research.google/pubs/pregel-a-system-for-large-scale-graph-processing/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-06-06
---

# Parallel execution in LangGraph — fan-out, fan-in

> [!NOTE]
> **From earlier:** Tue's ReAct loop stepped one tool at a time — sequential decisions. Today the loop fans out: N evaluator-agents score N proposals simultaneously, then a single aggregator sees all results. The typed state schema from file 2 is what makes that merge deterministic.

## 1. Learning Objectives

By the end of this reading, you can:

- Explain LangGraph's Pregel-inspired super-step model and the guarantee it gives about when parallel branches see each other's writes.
- Wire both static fan-out (known N at compile time) and dynamic fan-out (`Send` primitive, N decided at runtime).
- Explain why parallel writes to shared state require an explicit reducer — and what the two failure modes look like when you forget.
- Trace what survives in the checkpoint when one branch fails and others succeed.

## 2. Introduction

"Run N branches at the same time and merge their results" hides race conditions, partial-failure semantics, and merge-conflict resolution. LangGraph's execution model is borrowed from Google's Pregel system ([source: Pregel paper, retrieved 2026-05-26](https://research.google/pubs/pregel-a-system-for-large-scale-graph-processing/)): discrete super-steps (all active nodes run, then the framework merges updates before the next step) and reducer-annotated fields (merge semantics are part of the type, not the node code).

Today: four parallel evaluator-agents score proposals in the same super-step; the consensus aggregator runs in the next with all four results already merged.

## 3. Core Concepts

### 3.1 The super-step execution model

At each super-step, every node with a pending incoming message runs — possibly in parallel. The framework collects all returned updates and applies them via each field's reducer (or last-writer-wins). A checkpoint is written. The next super-step begins ([source: langchain-ai.github.io low_level, retrieved 2026-05-26](https://langchain-ai.github.io/langgraph/concepts/low_level/)).

The load-bearing guarantee: **all updates from a super-step are visible together to the next super-step.** Two parallel evaluator nodes read the previous super-step's state and both write into the next via the reducer. No locking needed in the node code.

> [!TIP]
> **The super-step boundary is your unit of atomicity.** All parallel branches within a step commit together — this is what makes fan-out safe without explicit locks.

### 3.2 Static vs dynamic fan-out

**Static fan-out** — N is known at build time. Add N edges from the source to N branch nodes:

```python
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send
from typing import Annotated, TypedDict
from operator import add

class EvalState(TypedDict):
    proposal_ids: list[str]
    evaluator_scores: Annotated[dict, lambda a, b: {**a, **b}]
    final_result: dict | None

# Static: three fixed branches.
graph.add_edge("supervisor", "eval_branch_a")
graph.add_edge("supervisor", "eval_branch_b")
graph.add_edge("supervisor", "eval_branch_c")
graph.add_edge("eval_branch_a", "aggregate")
graph.add_edge("eval_branch_b", "aggregate")
graph.add_edge("eval_branch_c", "aggregate")
```

**Dynamic fan-out** — N depends on runtime state. Use `Send`: a node returns a list of `Send("target", payload)` objects; the runtime spawns one execution per `Send`.

```python
def route_to_evaluators(state: EvalState) -> list[Send]:
    return [
        Send("evaluator_score_proposal", {"proposal_id": pid})
        for pid in state["proposal_ids"]
    ]

graph.add_conditional_edges("supervisor_decide", route_to_evaluators, ["evaluator_score_proposal"])
graph.add_edge("evaluator_score_proposal", "consensus_aggregate")
```

All worker invocations run in the same super-step; `consensus_aggregate` runs in the next with the merged `evaluator_scores`.

### 3.3 Partial-failure semantics

When one branch raises an exception while others succeed, LangGraph persists the successful branches' writes as **pending writes** at the end of the (failed) super-step ([source: docs.langchain.com persistence, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/persistence)). Resume re-runs only the failed branch. At LLM-call prices, this matters: four parallel evaluators, one 429 — resume bills you for one call, not four.

> [!IMPORTANT]
> **Fan-in requires a reducer.** The aggregator runs once, in the super-step *after* all parallel branches. For it to see all N contributions, branches must write to a reduced field. Last-writer-wins silently loses all but one branch's result — and the first failing run usually looks correct because the surviving branch happened to be last.

## 4. Generic Implementation

Price-comparison engine fanning out to three supplier APIs:

```python
from typing import Annotated, TypedDict
from operator import add
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send

class QuoteState(TypedDict):
    sku: str
    suppliers_to_query: list[str]
    quotes: Annotated[list, add]       # parallel branches append here
    final_quote: dict

def planner(state: QuoteState) -> dict:
    return {"suppliers_to_query": ["acme", "globex", "initech"]}

def route_to_suppliers(state: QuoteState) -> list[Send]:
    return [
        Send("query_supplier", {"sku": state["sku"], "supplier": s})
        for s in state["suppliers_to_query"]
    ]

def query_supplier(state: dict) -> dict:
    price = fetch_price(state["sku"], state["supplier"])
    return {"quotes": [{"supplier": state["supplier"], "price": price}]}

def aggregate(state: QuoteState) -> dict:
    cheapest = min(state["quotes"], key=lambda q: q["price"])
    return {"final_quote": cheapest}

builder = StateGraph(QuoteState)
builder.add_node("planner", planner)
builder.add_node("query_supplier", query_supplier)
builder.add_node("aggregate", aggregate)
builder.add_edge(START, "planner")
builder.add_conditional_edges("planner", route_to_suppliers, ["query_supplier"])
builder.add_edge("query_supplier", "aggregate")
builder.add_edge("aggregate", END)
graph = builder.compile()
```

`quotes` is `Annotated[list, add]` so each supplier's result appends cleanly. If `globex` 503s while `acme` and `initech` succeed, the checkpoint contains both working quotes; resume only re-runs `globex`.

## 5. Real-world Patterns

**E-commerce — search enrichment.** Storefronts fan out a query to multiple ranking signals (relevance, popularity, personalisation, inventory) in parallel. Each writes to a shared scoring map; the merge is deterministic because the merge function is a published contract.

> [!NOTE]
> **Cross-domain lesson:** Partial-write preservation on branch failure appears in every mature fan-out system — LangGraph checkpoints, Temporal task queues, and AWS Step Functions Map state all preserve successful siblings' work when one branch crashes.

**Gaming — matchmaking (Riot Games).** Multiplayer matchmakers fan out to region pools, then fan in by ping + skill + queue-time. Preserving partial results when one region times out is the same pattern as LangGraph's partial-write preservation.

## 6. Best Practices

- **Use static edges when N is known at build time; `Send` when N depends on the run.**
- **Every parallel-write field gets a reducer at the moment it is declared.**
- **Branches return lists (or dicts) to be reduced, not bare scalars.**
- **Treat the aggregator as the only node that reads the full merged collection.**
- **Set per-branch timeouts at the node level, not the whole-graph level.**

> [!WARNING]
> **Anti-pattern: `parallel-without-aggregator`.** Fanning out to N branches without a typed aggregator node — expecting each branch to self-aggregate by writing to the same last-writer-wins field — silently drops N-1 branches' work. This is especially dangerous in fan-out LLM workflows where you are paying for every branch's model call. Define the aggregator node and the reducer before wiring any branch edges.

## 7. Hands-on Exercise

Implement a `fan_out_with_aggregation` graph. Accept `{"inputs": [1, 2, 3, 4]}`. Fan out via `Send` to one worker per integer; each worker returns `{"results": [<int_squared>]}`. Fan in to an aggregator returning `{"total": <sum of squared values>}`. Acceptance criterion: invocation produces `{"total": 30}` without exception.

> [!NOTE]
> **Self-check** (30 s — answer mentally before expanding)
>
> 1. What happens if you annotate `results` as `results: list` (no reducer) instead of `results: Annotated[list, add]` when running with 4 parallel workers?
> 2. The `aggregate` node's body calls `sum(state["results"])`. When exactly does it run relative to the worker nodes?

<details>
<summary>Show answers</summary>

1. Last-writer-wins: only the last worker's single result survives. The other three are silently discarded. The total will be wrong (one squared value instead of four summed) but no exception is raised.
2. `aggregate` runs in the super-step *after* all four workers complete. It sees the fully merged `results` list because the framework applied the reducer from all four workers' writes before starting the next super-step.

</details>

## 8. Key Takeaways

- Super-step model: all active nodes run together; updates merge before the next step; no branch sees another's writes mid-step.
- Static fan-out for known N; `Send` for runtime-determined N.
- Every parallel-write field needs a reducer — forget it and you get silent data loss.
- Partial-failure: successful branches' writes persist in the checkpoint; only the failed branch re-runs on resume.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://docs.langchain.com/oss/python/langgraph/graph-api — retrieved 2026-05-26 — hot-tech
- https://langchain-ai.github.io/langgraph/concepts/low_level/ — retrieved 2026-05-26 — hot-tech
- https://docs.langchain.com/oss/python/langgraph/persistence — retrieved 2026-05-26 — hot-tech
- https://research.google/pubs/pregel-a-system-for-large-scale-graph-processing/ — retrieved 2026-05-26 — foundation-stable

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

**`Send` payload scoping.** When `Send("worker", payload)` fires, `payload` becomes the worker node's initial state for that invocation — not the full graph state. This is intentional: each branch should see only its slice of the work. If you want a branch to see the full state plus a per-branch identifier, include the relevant fields explicitly in the payload. Branches that read the full pending list from shared state and self-assign slots are a common source of double-processing bugs.

**Super-step boundaries and LangSmith.** In LangSmith traces, each super-step appears as a distinct group of runs executing in the same timestamp window. The visual gap between super-steps is the time the framework spent merging updates and writing the checkpoint. When debugging a slow fan-out, look at whether the gap between super-steps is larger than expected — it may indicate the reducer is doing expensive work (e.g., merging a very large dict) or the checkpoint write is slow (Postgres under load).

**Cost at fan-out scale.** 4 parallel evaluators each calling Bedrock = 4 model invocations per fan-out super-step. If the evaluator flow runs 10 times per day, that is 40 Bedrock calls just for evaluation. Wire per-node cost instrumentation (today's file 8) before you fan out to production load — the fan-out multiplier is where cost surprises originate.

</details>

Last verified: 2026-06-06
