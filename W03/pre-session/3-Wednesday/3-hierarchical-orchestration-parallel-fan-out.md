---
week: W03
day: Wed
topic_slug: hierarchical-orchestration-parallel-fan-out
topic_title: "Hierarchical orchestration + parallel fan-out"
parent_overview: W03/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 10
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
last_verified: 2026-05-26
---

# Hierarchical orchestration + parallel fan-out

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish *hierarchical orchestration* (supervisors-of-supervisors) from *parallel fan-out* (one supervisor dispatching N workers concurrently) and explain when each shape earns its place.
- Identify the architectural signals that justify adding a second supervisor layer (organizational/data-residency boundaries, prompt overflow, eval scoping).
- Implement a flat supervisor with parallel fan-out using LangGraph's Send API and state reducers, and explain why reducers are mandatory for safe parallel writes.
- Articulate the latency/cost trade-offs between sequential delegation and parallel fan-out at concrete cardinality (e.g., 6 workers, 18 workers).
- Recognize the deferred-execution / superstep semantics that make LangGraph parallelism behave deterministically.

## 2. Introduction

Once a single supervisor agent exists, two independent dimensions of growth open up. The first is *vertical*: when a supervisor's routing prompt gets too long or when organizational boundaries (data residency, access control, ownership) cut across the worker set, you grow upward by adding a second supervisor that delegates to sub-supervisors. The second is *horizontal*: when one decision step needs to invoke the same kind of work many times in parallel — score 18 proposals at once, summarize 50 documents at once, run the same evaluator against three model candidates — you grow outward by *fanning out* from a single supervisor into many concurrent workers.

These two shapes are often confused because both involve "more than one worker," but they solve different problems. Hierarchy buys you *organization*; fan-out buys you *throughput*. Production multi-agent systems usually start flat (one supervisor, N sequential workers), then add fan-out when latency forces it, then add hierarchy when the prompt or the eval surface forces it. The path from flat to fully hierarchical fan-out is rarely a single jump.

The mechanical question that follows is: how do you implement either pattern in a way that stays debuggable? For fan-out, the canonical answer in LangGraph v1.0+ is the **Send API** combined with **state reducers** — the pair that turns a static graph into a dynamic map-reduce while still producing deterministic traces. For hierarchy, the canonical answer is a subgraph compiled from its own `StateGraph` and embedded as a node in the parent.

## 3. Core Concepts

### 3.1 Hierarchical orchestration: when one supervisor is not enough

A flat supervisor breaks down when any of the following are true:

- **Routing prompt overflow.** The supervisor's system prompt grows past usability because it has to encode policy for too many workers. Symptoms: regressions when adding any new worker; routing accuracy degrades as the prompt grows [1].
- **Organizational boundaries.** Workers belong to different teams with different data-access scopes, different SLAs, or different deployment cadences. A flat supervisor cannot enforce these boundaries cleanly [2].
- **Eval surface scoping.** You cannot pinpoint a regression because the supervisor's policy and the workers' implementations are entangled in one trace [3].

The mitigation is a second layer: a top-level *meta-supervisor* (sometimes called a *meta-agent* or *router*) that delegates to mid-level supervisors, each of which owns a coherent worker cluster. The meta-supervisor sees only the mid-level supervisors' structured outputs, not the leaf workers' raw traces — this is what gives hierarchies their composability [2].

```text
                    ┌────────────────┐
                    │ meta-supervisor│
                    └───────┬────────┘
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
       ┌──────────┐  ┌──────────┐  ┌──────────┐
       │supervisor│  │supervisor│  │supervisor│
       │   A      │  │   B      │  │   C      │
       └─┬──┬──┬─┘  └─┬──┬──┬─┘  └─┬──┬──┬─┘
         ▼  ▼  ▼     ▼  ▼  ▼     ▼  ▼  ▼
        w  w  w     w  w  w     w  w  w   (leaf workers)
```

In LangGraph this is implemented by compiling each mid-level subtree as its own `StateGraph` and embedding the compiled subgraph as a node in the parent graph [1]. The subgraph has its own state schema; the parent only sees what the subgraph chooses to emit.

### 3.2 Parallel fan-out: one supervisor, N concurrent workers

Fan-out is the orthogonal axis. The supervisor decides *once* that a batch of identical-shape work needs to happen, then dispatches all instances in parallel. The work all completes before the supervisor's next decision.

LangGraph implements this with the **Send API**. Inside a node, you return a list of `Send(target_node, input_state)` objects; LangGraph spawns one parallel execution of `target_node` per `Send`, each with its own input state. When all parallel branches complete, their results merge into the parent state via *reducers* [4][5].

```python
from langgraph.types import Send

def supervisor_dispatch(state):
    # Decide to score every pending proposal in parallel
    return [
        Send("evaluator", {"proposal_id": pid, "criteria": state["criteria"]})
        for pid in state["pending_proposals"]
    ]
```

The parallel branches all run concurrently in what LangGraph calls a **superstep** — the framework's deterministic execution model where every branch in a superstep completes before the next superstep starts, so reducers always see a complete batch of writes [5][6].

### 3.3 Reducers — why parallel writes need them

If two nodes run in parallel and both write to the same state key without a reducer, LangGraph raises `InvalidUpdateError`. Overwriting only works when nodes run sequentially [4]. For parallel writes, you must declare *how* writes combine:

```python
from typing import Annotated, TypedDict
import operator

class FlowState(TypedDict):
    pending_proposals: list[str]
    scores: Annotated[list[dict], operator.add]   # reducer: list concat
    notes: Annotated[list[str], operator.add]
```

The `Annotated[list[dict], operator.add]` declaration tells LangGraph: when two parallel branches both emit a `scores` update, concatenate the lists instead of overwriting. The `add_messages` reducer is the same idea specialized for message lists.

This is not a LangGraph quirk — it is the same conceptual primitive (a commutative monoid) that Spark uses for `reduceByKey`, that CRDTs use for safe merge, and that Flink uses for keyed state. The reducer makes parallel state safe by construction [3][6].

### 3.4 Deferred execution and superstep semantics

LangGraph's parallel execution is **deferred** — nothing in a superstep starts until the previous superstep finishes. This guarantees three properties:

1. **Determinism.** The reducer always sees a complete batch of writes; nothing arrives "after" the merge.
2. **Trace coherence.** The LangSmith trace shows all parallel branches as siblings of the same supervisor step, not interleaved.
3. **Checkpointability.** A checkpoint between supersteps represents a consistent global state, which is what makes interrupt-and-resume work cleanly [6].

The cost is that the slowest branch sets the superstep latency — there is no early-out on a partial result. If you need early-out semantics, you express that with a *cancellation* primitive inside the worker, not at the graph level.

### 3.5 Latency and cost at concrete cardinality

A back-of-envelope for the daily evaluation flow gives the feel:

- Sequential: 18 evaluator calls × 3 seconds each = **54 s wall clock**, 18× per-call token cost.
- Parallel fan-out: 18 evaluator calls in parallel ≈ **3 s wall clock** (slowest call), 18× per-call token cost, plus reducer overhead (negligible) [4].

Cost stays the same; latency drops by ~18×. The question is never "should I fan out?" but "do I have a reducer that combines the results meaningfully?" If you do, fan out. If you don't, sequential is fine.

## 4. Generic Implementation

A flat supervisor with parallel fan-out using the Send API. The scenario is generic: a content-moderation pipeline that needs to score N pieces of user-generated content against a shared rubric (no federal-acquisitions overlap).

```python
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send, Command
import operator

class ModerationState(TypedDict):
    items_to_review: list[dict]      # incoming UGC batch
    rubric: dict                     # shared scoring criteria
    # Reducer: parallel branches concat their findings here
    findings: Annotated[list[dict], operator.add]
    decision: str | None             # supervisor's final synthesis

def supervisor(state: ModerationState):
    # If we have no findings yet, fan out to score every item
    if not state["findings"]:
        return [
            Send("score_one_item", {
                "item": item,
                "rubric": state["rubric"],
            })
            for item in state["items_to_review"]
        ]
    # Findings are in — synthesize and terminate
    return Command(goto="synthesizer")

def score_one_item(input_state: dict):
    # Each parallel branch runs this with its own narrow input
    item = input_state["item"]
    rubric = input_state["rubric"]
    score = call_scoring_llm(item, rubric)
    return {"findings": [{"item_id": item["id"], "score": score}]}

def synthesizer(state: ModerationState):
    # Aggregate findings into the final moderation decision
    decision = aggregate(state["findings"])
    return Command(update={"decision": decision}, goto=END)

graph = StateGraph(ModerationState)
graph.add_node("supervisor", supervisor)
graph.add_node("score_one_item", score_one_item)
graph.add_node("synthesizer", synthesizer)

graph.add_edge(START, "supervisor")
graph.add_edge("score_one_item", "supervisor")   # all parallel branches return here

app = graph.compile()
```

What each piece is doing:

- **`Annotated[list[dict], operator.add]`** on `findings` is what makes parallel writes safe — without it, two branches both writing to `findings` would raise `InvalidUpdateError`.
- **`Send("score_one_item", ...)`** in `supervisor` dispatches N parallel instances dynamically — the count is computed at runtime from `state["items_to_review"]`, not hard-coded in the graph topology.
- **`score_one_item`** receives its own narrow input dict (not the full state) — this is the *delegation contract* enforced at the framework level.
- All parallel branches converge back into `supervisor` in the next superstep; the reducer merges their `findings` writes deterministically.
- The synthesizer is a separate node so its prompt and eval surface stay isolated from the supervisor's routing prompt.

## 5. Real-world Patterns

**E-commerce — order-personalization fan-out (Shopify-style).** Production personalization stacks fan out from a single supervisor across (a) inventory availability, (b) shipping-time estimation, (c) recommendation refresh, and (d) price-tier evaluation in parallel per customer session. The supervisor's job becomes coordination only; each parallel worker has its own context. Latency drops from sequential (~600 ms total) to parallel (~150 ms, set by the slowest worker), and the supervisor's routing prompt stays short because it only orchestrates four well-named workers [3].

**Healthcare — radiology multi-reader.** Imaging diagnostic systems implement parallel fan-out where the supervisor dispatches N specialist-reader workers (one per modality: CT, MRI, X-ray) on the same study, then a synthesizer reconciles findings. The reported architectural win is that adding a new modality is adding one worker, not rewriting the supervisor — and the eval surface for each modality reader is isolated [2]. Hierarchical orchestration becomes justified once you have department-level supervisors (Radiology, Pathology, Cardiology) reporting to a meta-supervisor for cross-department case review.

**Logistics — last-mile route optimization.** Carriers like UPS-style operations route a single dispatch decision through a supervisor that fans out to (a) traffic-pattern worker, (b) driver-availability worker, (c) package-priority worker, all in parallel; the supervisor reduces their outputs into a single per-route assignment. Reported in 2026 production case studies as the canonical fan-out shape — the supervisor never tries to reason about all three concerns simultaneously [4].

**Gaming — procedural-content generation.** Game-content pipelines (think roguelike level generation with LLM-authored variants) use hierarchical orchestration: a meta-supervisor coordinates level-design supervisors, each of which delegates to room-builder and creature-placer workers within their level. The two-layer hierarchy lets one team iterate on room generation independently of another team's creature logic, while the meta-supervisor enforces level-wide constraints like difficulty progression [1][2].

## 6. Best Practices

- **Start flat. Add fan-out when latency forces it. Add hierarchy when prompt overflow or organizational boundaries force it.** Never pre-emptively hierarchical [3].
- **Always declare a reducer on any state key written by parallel branches.** A missing reducer surfaces as a runtime `InvalidUpdateError`; do not learn this lesson in production [4].
- **Pick reducer semantics that match the data shape.** Lists → `operator.add` (concat). Messages → `add_messages` (dedupe + append). Sets/dicts → custom reducer (commutativity matters).
- **For dynamic fan-out, use the Send API, not edge multiplication.** `Send` lets the count be runtime-computed; edge multiplication requires static topology.
- **Cap fan-out width.** Unbounded fan-out (Send returns 10,000 items) will starve your inference quota. Cap with a `max_parallel` and chunk if needed.
- **Embed subgraphs for hierarchy, not raw conditional branching.** A compiled subgraph has its own state schema and trace, which is what makes the meta-supervisor's view coherent.
- **Profile before parallelizing.** If each worker is 50 ms, fan-out's overhead can be larger than the savings; the win is real only when per-worker latency dominates.

## 7. Hands-on Exercise

**Architecture-drawing prompt (15 min).** You are designing a *news-article fact-check pipeline* for a fintech-news aggregator (no federal-acquisitions overlap). For each incoming article, the pipeline must:

1. Identify all factual claims in the article (one extraction step).
2. For each claim, look up corroborating sources from a curated database, score plausibility, and flag contradictions (this needs to happen in parallel across claims — articles have 5–30 claims).
3. Synthesize an article-level confidence score and a list of flagged claims for human review.

On a whiteboard or in a markdown file, draw the topology. Your diagram must label:

- Where you use parallel fan-out and where you use sequential.
- The state schema, including which keys need reducers and what reducer each key uses.
- The exact shape of the `Send` invocation (what input state each parallel branch receives).
- A `max_parallel` cap and your justification for the chosen value.
- One sentence explaining why you did **not** introduce hierarchy.

**What good looks like:** The extraction step is sequential (one supervisor decision, one worker call). The per-claim verification step uses `Send` to fan out across claims, with `findings: Annotated[list[dict], operator.add]` as the reducer-bearing state key. `max_parallel = 10` with chunked batches if an article has more, with a justification grounded in inference-quota math (not arbitrary). No hierarchy because the worker set is small and homogeneous; the supervisor's prompt encodes three branches and fits comfortably.

## 8. Key Takeaways

- *What architectural growth dimension does hierarchical orchestration solve, and what does parallel fan-out solve?* (Hierarchy → organization and prompt scaling; fan-out → throughput and latency.)
- *Why do parallel branches require reducers on shared state keys?* (Without a reducer, concurrent writes raise `InvalidUpdateError`; reducers turn writes into commutative monoid operations.)
- *What does the Send API give you that static edge multiplication doesn't?* (Runtime-computed fan-out width — the number of parallel branches can depend on data, not just topology.)
- *What is the latency profile of parallel fan-out and what sets it?* (Determined by the slowest branch — no early-out on partial results; cost is unchanged from sequential.)
- *Under what conditions should you escalate from a flat supervisor to a hierarchical one?* (Routing prompt overflow, organizational/data-residency boundaries, eval-surface entanglement — not pre-emptively.)

## Sources

1. [Use the graph API — LangChain/LangGraph official docs](https://docs.langchain.com/oss/python/langgraph/use-graph-api) — retrieved 2026-05-26
2. [Agent Orchestration Patterns: Swarm vs Mesh vs Hierarchical (Gurusup)](https://gurusup.com/blog/agent-orchestration-patterns) — retrieved 2026-05-26
3. [How to Orchestrate Multi-Agent AI Systems at Scale in 2026 (Atlan)](https://atlan.com/know/multi-agent-system-orchestration/) — retrieved 2026-05-26
4. [LangGraph Map-Reduce: Parallel Execution with Send API (machinelearningplus)](https://machinelearningplus.com/gen-ai/langgraph-map-reduce-parallel-execution/) — retrieved 2026-05-26
5. [Scaling LangGraph Agents: Parallelization, Subgraphs, and Map-Reduce Trade-Offs (AI Practitioner)](https://aipractitioner.substack.com/p/scaling-langgraph-agents-parallelization) — retrieved 2026-05-26

Last verified: 2026-05-26
