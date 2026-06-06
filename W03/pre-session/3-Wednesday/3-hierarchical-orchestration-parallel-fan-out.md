---
week: W03
day: Wed
topic_slug: hierarchical-orchestration-parallel-fan-out
topic_title: "Hierarchical orchestration + parallel fan-out"
parent_overview: W03/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://docs.langchain.com/oss/python/langgraph/use-graph-api
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://machinelearningplus.com/gen-ai/langgraph-map-reduce-parallel-execution/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://gurusup.com/blog/agent-orchestration-patterns
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://aipractitioner.substack.com/p/scaling-langgraph-agents-parallelization
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://atlan.com/know/multi-agent-system-orchestration/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-06-06
---

# Hierarchical orchestration + parallel fan-out

> [!NOTE]
> **From earlier:** Topic 2 established the flat supervisor-worker shape. Today's two extensions — vertical hierarchy and horizontal fan-out — solve different growth problems on top of that foundation.

## 1. Learning Objectives

- Distinguish *hierarchical orchestration* (supervisors-of-supervisors) from *parallel fan-out* (one supervisor dispatching N workers concurrently).
- Identify the three signals that justify adding a second supervisor layer.
- Implement parallel fan-out using LangGraph's Send API and state reducers.
- Explain why reducers are mandatory for safe parallel writes — and what error you get without them.

## 2. Introduction

Once a flat supervisor exists, two growth dimensions open. **Vertical** — when the routing prompt overflows or organizational boundaries cut across the worker set, add a second supervisor layer. **Horizontal** — when one decision needs the same work done N times in parallel (18 evaluator-agent calls, 50 document summaries), fan out into N concurrent workers.

Hierarchy buys *organization*; fan-out buys *throughput*. Start flat, add fan-out when latency forces it, add hierarchy when prompt overflow or eval-surface entanglement forces it.

Today's arc: **three evaluations × six evaluators = 18 parallel scoring tasks** before fan-in to consensus — flat supervisor with parallel fan-out. Hierarchy is a refactor candidate only if the routing prompt grows past usefulness.

## 3. Core Concepts

### 3.1 Hierarchical orchestration — when one supervisor isn't enough

Three signals that justify a second layer:

| Signal | Symptom | Fix |
|--------|---------|-----|
| Routing prompt overflow | Adding any new worker degrades routing accuracy | Move worker cluster under a sub-supervisor |
| Organizational boundary | Different teams own different worker sets with different SLAs | Sub-supervisor per team boundary |
| Eval surface entanglement | Can't pinpoint a regression because supervisor + workers share one trace | Subgraph per cluster; meta-supervisor sees typed outputs only |

In LangGraph, each mid-level cluster is a compiled `StateGraph` embedded as a node in the parent. The parent's supervisor sees only the subgraph's typed output — it cannot introspect internal worker state.

### 3.2 Parallel fan-out — Send API + superstep semantics

Fan-out is implemented with the **Send API**. Inside a node, return a list of `Send(target_node, input_state)` objects; LangGraph spawns one parallel execution of `target_node` per `Send`, each with its own input state. All parallel branches complete in one **superstep** before the next superstep starts — deterministic, checkpoint-friendly.

```python
from langgraph.types import Send

def supervisor_dispatch(state):
    return [
        Send("evaluator", {"proposal_id": pid, "criteria": state["criteria"]})
        for pid in state["pending_proposals"]
    ]
```

The fan-out width is runtime-computed from state, not hard-coded in the topology.

### 3.3 Reducers — mandatory for parallel writes

Parallel branches that write to the same state key without a reducer raise `InvalidUpdateError` at runtime. Declare how writes combine:

```python
from typing import Annotated, TypedDict
import operator

class FlowState(TypedDict):
    scores: Annotated[list[dict], operator.add]   # list-concat reducer
    notes: Annotated[list[str], operator.add]
```

`Annotated[list[dict], operator.add]` tells LangGraph: when two parallel branches emit a `scores` update, concatenate the lists instead of overwriting. This is the same primitive (commutative monoid) Spark uses for `reduceByKey` and CRDTs use for safe merge.

> [!IMPORTANT]
> **Superstep guarantee.** All branches in a superstep complete before the reducer runs — the reducer always sees a complete batch. This is what makes interrupt-and-resume work cleanly across parallel branches.

### 3.4 Latency trade-off at concrete cardinality

Back-of-envelope for 18 evaluator calls at 3s each:

| Mode | Wall clock | Token cost |
|------|-----------|------------|
| Sequential | 54 s | 18× per-call |
| Parallel fan-out | ~3 s (slowest call) | 18× per-call |

Cost is identical; latency drops ~18×. The question is never "should I fan out?" but "do I have a reducer that combines results meaningfully?"

> [!TIP]
> **Cap fan-out width.** `Send` returning 10,000 items starves your inference quota. Chunk large batches with a `max_parallel` cap.

## 4. Generic Implementation

Flat supervisor with parallel fan-out — content-moderation pipeline scoring N items against a shared rubric:

```python
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send, Command
import operator

class ModerationState(TypedDict):
    items_to_review: list[dict]
    rubric: dict
    findings: Annotated[list[dict], operator.add]   # reducer — parallel writes safe
    decision: str | None

def supervisor(state: ModerationState):
    if not state["findings"]:
        return [
            Send("score_one_item", {"item": item, "rubric": state["rubric"]})
            for item in state["items_to_review"]
        ]
    return Command(goto="synthesizer")

def score_one_item(input_state: dict):
    score = call_scoring_llm(input_state["item"], input_state["rubric"])
    return {"findings": [{"item_id": input_state["item"]["id"], "score": score}]}

def synthesizer(state: ModerationState):
    decision = aggregate(state["findings"])
    return Command(update={"decision": decision}, goto=END)

graph = StateGraph(ModerationState)
graph.add_node("supervisor", supervisor)
graph.add_node("score_one_item", score_one_item)
graph.add_node("synthesizer", synthesizer)
graph.add_edge(START, "supervisor")
graph.add_edge("score_one_item", "supervisor")
app = graph.compile()
```

`score_one_item` receives a narrow input dict — that's the delegation contract enforced at the framework level. The reducer on `findings` is what keeps parallel writes safe.

## 5. Real-world Patterns

**E-commerce personalization.** Production stacks fan out across inventory availability, shipping estimation, recommendation refresh, and price-tier evaluation in parallel per session. Latency drops from ~600 ms sequential to ~150 ms parallel (slowest worker). Routing prompt stays short because it orchestrates four named workers.

**Healthcare — radiology multi-reader.** One study, N modality-reader workers (CT, MRI, X-ray) each in isolation. Adding a new modality = adding one worker. Hierarchical orchestration kicks in when department-level supervisors (Radiology, Pathology, Cardiology) each own a cluster and a meta-supervisor handles cross-department review.

## 6. Best Practices

- **Start flat. Add fan-out when latency forces it. Add hierarchy when prompt overflow or organizational boundaries force it.**
- **Always declare a reducer on any state key written by parallel branches.** Missing reducer = runtime `InvalidUpdateError`.
- **For dynamic fan-out, use Send API — not edge multiplication.** `Send` lets the count be runtime-computed.
- **Cap fan-out width.** Unbounded `Send` returns exhaust inference quotas.
- **Embed subgraphs for hierarchy, not raw conditional branching.** Each subgraph has its own schema and trace.

> [!WARNING]
> **Anti-pattern: `fan-out-without-aggregator`.** Parallel branches that write back to shared state without a reducer will silently race or raise at runtime. Every fan-out needs a corresponding reducer declaration on every shared state key the branches write. Symptom: works fine in single-threaded dev, fails under concurrent load.

## 7. Hands-on Exercise

You are designing a **news-article fact-check pipeline** for a fintech-news aggregator (not federal-acquisitions). The pipeline must: (1) extract all factual claims in the article; (2) for each claim, look up corroborating sources and score plausibility in parallel; (3) synthesize an article-level confidence score and a list of flagged claims.

Draw the topology, naming: where fan-out happens, the state schema (which keys need reducers), the `Send` invocation shape, a `max_parallel` cap with justification, and one sentence explaining why you did *not* introduce hierarchy.

> [!NOTE]
> **Self-check** (30s — answer mentally before expanding)
>
> 1. Two parallel evaluator branches both write to `scores: list[dict]` without a reducer. What does LangGraph do?
> 2. Your fan-out is returning 500 `Send` items. What single mitigation prevents inference-quota starvation?

<details>
<summary>Show answers</summary>

1. LangGraph raises `InvalidUpdateError` at runtime. Declare `scores: Annotated[list[dict], operator.add]` to make concurrent writes safe via list-concat.
2. Add a `max_parallel` cap and chunk the input: dispatch at most N `Send` items per superstep, process the next chunk after the first superstep completes.

</details>

> [!NOTE]
> **Cost stays identical to sequential.** Fan-out saves latency, not tokens — every parallel branch still pays its own inference cost. The win is wall-clock time, not budget.

## 8. Key Takeaways

- Hierarchy solves organization and prompt scaling; fan-out solves throughput and latency.
- Parallel branches require reducers — without them, concurrent writes raise `InvalidUpdateError`.
- Send API gives runtime-computed fan-out width; static edge multiplication is fixed topology.
- Latency is set by the slowest branch; cost is identical to sequential.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://docs.langchain.com/oss/python/langgraph/use-graph-api — retrieved 2026-05-26 — hot-tech
- https://machinelearningplus.com/gen-ai/langgraph-map-reduce-parallel-execution/ — retrieved 2026-05-26 — hot-tech
- https://gurusup.com/blog/agent-orchestration-patterns — retrieved 2026-05-26 — hot-tech
- https://aipractitioner.substack.com/p/scaling-langgraph-agents-parallelization — retrieved 2026-05-26 — hot-tech
- https://atlan.com/know/multi-agent-system-orchestration/ — retrieved 2026-05-26 — hot-tech

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

**Deferred execution and trace coherence.** LangGraph's superstep model is *deferred* — nothing in a superstep starts until the previous one finishes. This buys three properties: determinism (reducer always sees a complete batch), trace coherence (LangSmith shows parallel branches as siblings, not interleaved), and checkpointability (a checkpoint between supersteps is a consistent global state, enabling clean interrupt-and-resume). The cost is that the slowest branch sets wall-clock latency with no early-out on a partial result. If early-out semantics are required, express them with a cancellation primitive inside the worker, not at the graph level.

**CTE vs Send for hierarchical fan-out.** When a sub-supervisor itself needs to fan out, you can nest `Send` returns inside a subgraph's supervisor node. The outer superstep waits for the entire subgraph to complete before the meta-supervisor's next step. This composes cleanly: the meta-supervisor never sees intermediate fan-out state, only the typed output the subgraph emits at completion.

</details>

Last verified: 2026-06-06
