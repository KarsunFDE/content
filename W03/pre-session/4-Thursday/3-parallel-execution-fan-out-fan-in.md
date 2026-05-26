---
week: W03
day: Thu
topic_slug: parallel-execution-fan-out-fan-in
topic_title: "Parallel execution in LangGraph — fan-out, fan-in"
parent_overview: W03/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 10
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
last_verified: 2026-05-26
---

# Parallel execution in LangGraph — fan-out, fan-in

## 1. Learning Objectives

By the end of this reading, the learner can:

- Explain LangGraph's Pregel-inspired super-step execution model and how it makes fan-out / fan-in deterministic.
- Wire a graph that fan-outs from one node into N parallel branches and fan-ins to a single aggregator using conditional edges and the `Send` primitive.
- Explain why parallel writes to the same state field require an explicit reducer.
- Trace the order of events when a fan-out branch fails partway and the others succeed — what is persisted, what re-runs.
- Distinguish "static fan-out" (known branches at compile time) from "dynamic fan-out" (branches determined by node return value at run time).

## 2. Introduction

Parallel execution is one of those features that looks easy on paper and is full of sharp edges in practice. "Run these N branches at the same time and then merge their results" is a one-line statement that hides race conditions, partial failures, ordering questions, and merge-conflict resolution. Most workflow systems either give you ad-hoc parallelism (you start threads yourself, you handle the joining yourself, you debug your own race conditions) or a heavyweight parallel construct that does the join for you but makes the surrounding code awkward.

LangGraph sits in the middle. Its execution model is borrowed from Google's Pregel system for large-scale graph processing ([source: Pregel paper, retrieved 2026-05-26](https://research.google/pubs/pregel-a-system-for-large-scale-graph-processing/)) and brings two ideas with it. First, execution proceeds in discrete **super-steps**: at each super-step, every active node runs (possibly in parallel), then the framework collects their state updates and applies them as a batch before the next super-step begins. Second, the merge semantics for concurrent writes are part of the *type* of each state field (the reducer pattern from yesterday's reading), not part of the node code. That separation is what makes fan-out / fan-in tractable in LangGraph where it is gnarly in plain Python.

This reading is the generic foundation for today's wiring: how to fan-out from a supervisor to N parallel branches, how to fan-in to an aggregator that sees all their results, what the framework guarantees on partial failure, and the gotchas you only learn by hitting them.

## 3. Core Concepts

### The super-step execution model

LangGraph's runtime ticks in super-steps ([source: docs.langchain.com graph-api and low_level concepts, retrieved 2026-05-26](https://langchain-ai.github.io/langgraph/concepts/low_level/)):

- At the start of a super-step, every node that has at least one new incoming message is **active**.
- Active nodes run their function, potentially in parallel with one another, and return state updates.
- The framework collects all returned updates and applies them via each field's reducer (or last-writer-wins for non-reduced fields).
- A checkpoint is written at the end of the super-step.
- The next super-step starts with whichever nodes now have new incoming messages.

This is not just an implementation detail; it gives you a strong guarantee — **all updates from a super-step are visible together to the next super-step**, never interleaved. Two parallel evaluator nodes do not see each other's intermediate writes. They both read the *previous* super-step's state and both write into the *next* super-step's state via the reducer.

### Static fan-out — known branches at compile time

The simplest case: you know at graph-build time that you have N branches. You add N edges from the source node to N branch nodes and one edge from each branch back to the aggregator. The runtime fires all N branches in the same super-step:

```python
graph.add_edge("classify", "branch_a")
graph.add_edge("classify", "branch_b")
graph.add_edge("classify", "branch_c")
graph.add_edge("branch_a", "aggregate")
graph.add_edge("branch_b", "aggregate")
graph.add_edge("branch_c", "aggregate")
```

The aggregator runs in the super-step *after* all three branches complete. It sees the merged state with all three branch contributions already reduced into the relevant field.

### Dynamic fan-out — branches determined at runtime via `Send`

The more interesting case: the number of branches depends on the state. For example, you do not know how many proposals the upstream node will produce until it runs. LangGraph provides the `Send` primitive — a node can return a list of `Send("target_node", payload)` objects, and the runtime spawns one execution of `target_node` per `Send`, each with its own payload (`from langgraph.types import Send`):

```python
from langgraph.types import Send

def route_to_workers(state: State) -> list[Send]:
    return [
        Send("worker_node", {"item_id": item_id})
        for item_id in state["pending_items"]
    ]

graph.add_conditional_edges("router", route_to_workers, ["worker_node"])
graph.add_edge("worker_node", "aggregate")
```

All worker invocations run in the same super-step (the runtime parallelises them), and `aggregate` runs in the next super-step. The `Send` payload becomes the input to the worker's first read of state for *that* branch — useful when each branch should see only its slice of the work, not the whole list.

### Fan-in and the reducer requirement

Fan-in is where most teams get burned. The aggregator typically does not run N times; it runs once, in the super-step *after* the parallel branches. For the aggregator to see all N branch contributions, the branches must write to a *reduced* field — last-writer-wins would silently lose all but one of the writes.

The typical shape:

```python
class State(TypedDict):
    # Each parallel branch appends one entry; the reducer concatenates.
    worker_results: Annotated[list, add]
```

Branches return `{"worker_results": [<one_entry>]}` (note the *list*, not the bare entry — the reducer expects lists to concatenate). The aggregator reads the full list.

### Partial-failure semantics

What happens if branch B raises an exception while A and C succeed? LangGraph stores **pending writes** ([source: docs.langchain.com persistence, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/persistence)) — the successful updates from A and C are persisted at the end of the (failed) super-step, even though the super-step itself did not complete. When you resume, A and C are not re-run; only B re-runs. This is one of the load-bearing reliability features of the framework: parallel branches do not lose work because a sibling failed.

You can opt out by configuring the checkpointer differently, but the default is "preserve the successful branches' work."

## 4. Generic Implementation

A non-Karsun example: a price-comparison engine that fans out to multiple supplier APIs in parallel and aggregates the responses into a single quote.

```python
from typing import Annotated, TypedDict
from operator import add
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send

class QuoteState(TypedDict):
    sku: str
    suppliers_to_query: list[str]            # set by the planner node
    quotes: Annotated[list, add]             # parallel branches append here
    final_quote: dict                        # set by the aggregator

def planner(state: QuoteState) -> dict:
    # Decide which suppliers to query based on the SKU.
    return {"suppliers_to_query": ["acme", "globex", "initech"]}

def route_to_suppliers(state: QuoteState) -> list[Send]:
    # Dynamic fan-out: one branch per supplier the planner picked.
    return [
        Send("query_supplier", {"sku": state["sku"], "supplier": s})
        for s in state["suppliers_to_query"]
    ]

def query_supplier(state: dict) -> dict:
    # 'state' here is the Send payload, not the full graph state.
    supplier = state["supplier"]
    price = fetch_price(state["sku"], supplier)   # external API call
    return {"quotes": [{"supplier": supplier, "price": price}]}

def aggregate(state: QuoteState) -> dict:
    # Runs once after all parallel branches complete.
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

Notes on what each piece is doing:

- `planner` decides at runtime which suppliers to query. The list goes into state.
- `route_to_suppliers` is the dynamic fan-out — one `Send` per supplier.
- `query_supplier` runs once per `Send`, each with its own payload. The framework runs them in parallel.
- `quotes` is `Annotated[list, add]` so the parallel branches' append-of-one writes concatenate cleanly.
- `aggregate` runs in the next super-step with the merged list.

If `globex` returns 503 while `acme` and `initech` succeed, the checkpoint after the super-step contains the `acme` and `initech` quotes; a resume only re-runs the `globex` branch.

## 5. Real-world Patterns

**E-commerce — search-result enrichment (Algolia, Elastic).** Storefronts fan out a search query to multiple ranking signals (text relevance, popularity, personalisation, inventory) in parallel and fan in to a single ranked list. Each signal runs in its own thread and writes to a shared scoring map; the final merge is deterministic because the merge function is a published part of the ranker contract. Sites that skip the explicit merge end up with non-deterministic search results between deployments.

**Gaming — matchmaking services.** Multiplayer matchmakers fan out to multiple region pools to find candidate players (West Coast, EU, APAC) in parallel, then fan in to pick the best fit. Backend engineers at companies like Riot have publicly described the pattern: each pool returns a candidate slate, the aggregator scores them on ping + skill + queue-time, the winner is picked. The interesting design move is preserving partial results when one region's pool service times out — the match still completes from the other regions' candidates rather than failing entirely.

**Logistics — shipping-rate calculation.** Multi-carrier shipping services (ShipStation, EasyPost) fan out a rate request to UPS, FedEx, USPS, and regional carriers in parallel. Each carrier API has its own latency profile; the slowest dominates the wall-clock time. The fan-in aggregator typically returns whichever carriers responded within a deadline rather than waiting forever. This deadline-bounded fan-in is the same shape as a LangGraph aggregator that reads from a list reducer — the aggregator sees whoever finished.

**Healthcare — diagnostic-screening pipelines.** Imaging vendors fan out a CT scan to multiple specialised classifiers (lung-nodule detection, cardiac-event detection, bone-fracture detection) in parallel and fan in to a unified report. Each classifier writes its findings into a `findings` list with a reducer; the report generator runs once over the consolidated list. Critically, when one classifier model crashes, the others still produce findings — the partial-write preservation pattern is what makes the system safe to keep running while a single model is investigated.

## 6. Best Practices

- **Choose static fan-out when N is known at build time; choose `Send` when N depends on the run.** Static is easier to read in the graph diagram; `Send` is necessary when the count varies.
- **Every parallel-write field gets a reducer at the moment it is declared.** Adding one later means migrating every existing checkpoint.
- **Branches return *lists* (or dicts) to be reduced, not bare scalars.** `{"results": [item]}` not `{"results": item}` — the reducer expects the unit it is merging.
- **Treat the aggregator as the only node that reads the full merged collection.** Branches should not depend on each other's writes within the same super-step — they cannot see them.
- **Set per-branch timeouts at the node level, not the whole-graph level.** A 30-second graph timeout hides a 28-second slow branch that should be optimised.
- **Use `Send` payloads to give each branch its slice of the work.** Letting every parallel branch read the full pending list and pick its own slot is a recipe for double-processing.
- **Trace branch identity in logs.** Without an identifier in the log line, debugging "which of the 12 parallel branches threw" is a guessing game.

## 7. Hands-on Exercise

**Code task (15 min).** Implement a generic `fan_out_with_aggregation` graph in plain Python (no domain assumptions). The graph should:

1. Accept a list of integers.
2. Fan out to N parallel workers (one per integer) that each return `{"results": [<int_squared>]}`.
3. Fan in to an aggregator that returns `{"total": <sum of all squared values>}`.

Use `Send` for the fan-out. Use `Annotated[list, add]` on the results field. Acceptance criteria: invoking with `{"inputs": [1, 2, 3, 4]}` produces `{"total": 30}`, and the graph compiles + runs without exception.

**What good looks like.** A correct answer has: a `TypedDict` with `inputs: list[int]`, `results: Annotated[list, add]`, `total: int`; a router node that emits `[Send("worker", {"x": x}) for x in state["inputs"]]`; a worker node that reads its payload and returns `{"results": [x*x]}`; an aggregator that returns `{"total": sum(state["results"])}`. A subtly wrong answer omits the reducer annotation — the run still appears to work for small N because the last write wins by chance, but the total is wrong.

## 8. Key Takeaways

- What does "super-step" mean and what guarantee does it give me about when parallel branches see each other's writes? (LO1)
- When do I use static edges and when do I use `Send`? (LO2)
- What exactly do I need to do to a state field so N branches can write to it concurrently without losing work? (LO3)
- When one branch fails and the others succeed, what does the checkpoint contain on resume? (LO4)
- How does dynamic fan-out (number of branches decided at runtime) differ from static? (LO5)

## Sources

1. [LangGraph Graph API — Nodes, Edges, Send, Conditional Edges (v1.x docs)](https://docs.langchain.com/oss/python/langgraph/graph-api) — retrieved 2026-05-26
2. [LangGraph low-level concepts — super-steps, Pregel, message passing](https://langchain-ai.github.io/langgraph/concepts/low_level/) — retrieved 2026-05-26
3. [LangGraph Persistence — pending writes, fault tolerance, checkpoint boundaries](https://docs.langchain.com/oss/python/langgraph/persistence) — retrieved 2026-05-26
4. [Pregel: A System for Large-Scale Graph Processing (Google Research)](https://research.google/pubs/pregel-a-system-for-large-scale-graph-processing/) — retrieved 2026-05-26

Last verified: 2026-05-26
