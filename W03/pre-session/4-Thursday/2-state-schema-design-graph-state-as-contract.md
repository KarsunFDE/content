---
week: W03
day: Thu
topic_slug: state-schema-design-graph-state-as-contract
topic_title: "State schema design — graph state is your contract"
parent_overview: W03/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://docs.langchain.com/oss/python/langgraph/graph-api
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://langchain-ai.github.io/langgraph/concepts/low_level/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.python.org/3/library/typing.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.langchain.com/oss/python/langgraph/persistence
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# State schema design — graph state is your contract

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define a LangGraph state schema using `TypedDict`, `dataclass`, or Pydantic `BaseModel` and explain the trade-offs between the three.
- Explain why state is the "contract" between nodes — what each node is allowed to read, what it is allowed to write, and what happens when two nodes write the same field concurrently.
- Identify when a field needs a **reducer** (`Annotated[T, reducer_fn]`) and when last-writer-wins is the correct semantics.
- Distinguish input schema, output schema, and internal state in graphs where they differ.
- Recognise the failure modes that follow from a sloppy schema — silent overwrites, parallel-write exceptions, and downstream nodes coupling to "magic" untyped fields.

## 2. Introduction

Every multi-step workflow has *some* shared state — even if it is implicit. The variables that flow between steps, the intermediate results one step needs from another, the running tally one step updates while a parallel step reads it. Whether you are building a graph-based agent runtime, a Kafka stream-processor, a data pipeline, or a long-running business workflow in a state machine, the *shape* of that shared state determines what kinds of bugs you can possibly write.

LangGraph leans into this directly. Before you add a single node, you define the graph's `State` — a typed schema that becomes the input type for every node function and the constraint that every node update is checked against. The schema is not a convenience type for the IDE; it is the load-bearing contract that lets the framework provide parallel execution, time-travel debugging, deterministic replay, and resume-from-checkpoint without each developer having to invent ad-hoc serialisation for their state every time.

The same idea appears across other workflow systems with different names. AWS Step Functions calls it the "state object" passed between tasks. Temporal calls it "workflow state" and enforces determinism rules around what you can store there. Kafka Streams calls it the "state store." The principle is identical: **the workflow's state shape is the contract between its participants, and an explicit, typed contract dramatically narrows the set of bugs you can write.**

This reading is the generic foundation. The day's overview applies it to the specific evaluator → consensus → SSA-review flow you will wire today; here we are after the discipline that travels with you to any future graph-shaped system.

## 3. Core Concepts

### Schema choices: `TypedDict` vs `dataclass` vs Pydantic

LangGraph supports three schema definitions, each with a clear use-case ([source: docs.langchain.com graph-api, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/graph-api)):

- **`TypedDict`** — the recommended default. Dict-shaped, zero-overhead at runtime, fully typed for IDEs and static checkers. No default values; you must set every key when you initialise. Best for "fast, typed, no validation."
- **`dataclass`** — same shape as `TypedDict` but allows default values via `field(default=...)`. Slightly heavier than a dict but still no validation. Use when you want defaults for optional fields.
- **Pydantic `BaseModel`** — runtime validation, coercion, and recursive checks. Heavier than the other two. Use when state contains complex nested types that you want validated on every node return, not just statically.

The state is the **input schema to every node**: every node function receives the full state as its first argument and returns a dict of *just the fields it updated*. The framework merges that partial return into the running state.

### Reducers — what happens when two nodes write the same field

By default, two writes to the same field overwrite — the second one wins. That is correct for fields where only one node ever writes (e.g., `ssa_approval_status` is only ever set by the human-resume node). It is *catastrophically wrong* for fields where multiple nodes write concurrently (e.g., a `scores` dict that four parallel evaluator-nodes all need to populate).

The fix is a **reducer**: a function that takes the existing value and the new value and returns the merged result. LangGraph annotates the reducer onto the type:

```python
from typing import Annotated, TypedDict
from operator import add

class State(TypedDict):
    # Last-writer-wins (default): only set by one node.
    status: str
    # Reducer: each parallel node appends its results to the list.
    scores: Annotated[list, add]
```

The built-in `add` reducer concatenates lists. For dicts you typically write a small merge reducer. For chat-message history, LangGraph ships `add_messages`, which knows how to deduplicate by message ID rather than blindly appending.

**The rule of thumb:** any field that more than one node writes needs a reducer. Skip the reducer and you get one of two failure modes — either silent overwrite (the last write wins, the others vanish) or a parallel-update exception at runtime. Both are nasty to debug because the *first* failing run usually looks like it worked.

### Input schema vs output schema vs internal state

By default the graph's input, internal, and output schemas are the same `State` type. You can split them ([source: docs.langchain.com graph-api, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/graph-api)):

- **Input schema** — what the *caller* must pass when invoking the graph.
- **Internal state** — what nodes pass between themselves (often a superset of input, with computed fields).
- **Output schema** — what the graph returns to the caller.

Splitting them matters when the state has dozens of fields but only three are caller-relevant. Without the split, the caller sees the full internal mess and tends to depend on it.

### Why the schema is "the contract"

A node's type signature is `def my_node(state: State) -> dict`. Two things follow:

1. **What the node can read** — every key in `State`. If a key isn't in the schema, no node can use it. This rules out "let me just stash this here for later" anti-patterns.
2. **What the node can write** — only the keys it returns. Other keys are untouched. The framework merges the partial return into the running state using the reducer (or last-writer-wins) for each key.

So the schema simultaneously documents the data model, constrains what nodes can mutate, and provides the merge semantics. That is the contract. A graph with a sloppy schema is a graph where the contract is implicit, and an implicit contract is a contract every developer renegotiates in their head when they read the code.

## 4. Generic Implementation

A non-Karsun example: an e-commerce order-fulfilment workflow. The state shape declares what every step in the order pipeline reads and writes:

```python
from typing import Annotated, TypedDict
from operator import add

def merge_dict(left: dict, right: dict) -> dict:
    """Reducer for parallel writes to a dict field."""
    return {**left, **right}

class OrderFulfilmentState(TypedDict):
    # Set once by the caller; never re-written.
    order_id: str
    customer_id: str

    # Written by a single node each. Default last-writer-wins is fine.
    payment_status: str          # "pending" | "authorized" | "captured" | "failed"
    inventory_status: str        # "reserved" | "out_of_stock"
    fulfilment_status: str       # "pending" | "shipped" | "delivered"

    # Written by multiple parallel nodes (one per warehouse). Needs a reducer.
    warehouse_responses: Annotated[dict, merge_dict]

    # Append-only audit trail. Each step appends one entry. Needs a reducer.
    events: Annotated[list, add]
```

Reading this declaration tells you everything about the workflow's shape without looking at a single node:

- The caller supplies `order_id` + `customer_id` — those are inputs, never touched again.
- Three single-writer status fields — clean ownership, no reducer needed.
- Two multi-writer fields — `warehouse_responses` for parallel warehouse-availability checks, `events` for an audit log that every step appends to.

A new engineer can answer "where does `payment_status` get set?" by grepping the codebase for `"payment_status":` in node return values. That grep is only useful because the schema constrains *every* write to go through a typed return. Without the schema you'd have to read every node body.

## 5. Real-world Patterns

**Fintech — payment-processing state machines (Stripe).** Stripe's PaymentIntent object is, in effect, a typed state schema for the multi-step payment flow. Fields like `status`, `next_action`, and `last_payment_error` are written by specific transitions; integrators code against the published schema and the documented state diagram. When a competitor's gateway publishes "the response object varies depending on the path through the flow," integrators end up writing fragile defensive parsing. Typed state is the difference between integration code that survives upgrades and code that breaks on every API revision.

**Healthcare — HL7 FHIR workflow tasks.** FHIR's `Task` resource defines a typed schema for clinical workflow steps — `status`, `intent`, `for`, `owner`, `input`, `output`. When an order-entry system hands a task to an imaging system, both sides validate against the FHIR schema. Multiple downstream systems can read the same task without coordinating; each only writes the fields it owns. The schema is the contract that lets two vendors' systems collaborate on the same patient encounter without a shared codebase.

**E-commerce — Shopify Flow / order tagging.** Shopify's Flow automation builds workflows where each step reads from the order object and writes back named tags or metafields. Because the metafield namespace is typed and versioned, two unrelated automations can append tags to the same order without stepping on each other. Without that namespacing, merchants would see automations randomly clobber each other's writes — the same failure mode parallel LangGraph nodes have without reducers.

**Logistics — AWS Step Functions for parcel routing.** Step Functions executions pass a JSON state object between tasks. Best-practice AWS guidance is to define the JSON schema up front and validate it at task boundaries, because once you have parallel branches (the `Parallel` state) you need explicit merge logic in the same shape as a LangGraph reducer. Teams that skip the schema discipline discover, weeks in, that two parallel branches both write to `result` and one is silently lost.

## 6. Best Practices

- **Start with `TypedDict`; only reach for Pydantic when you genuinely need runtime validation.** Pydantic is heavier on every state mutation; for most graphs the type-checker is enough.
- **Name fields by *who writes them*, not by *what they are*.** `consensus_complete` is clearer than `complete`. `ssa_approval_status` is clearer than `status`. The name carries the ownership.
- **Annotate every multi-writer field with a reducer at the moment you introduce it.** Adding a reducer later means migrating every checkpoint already in the database.
- **Keep the schema flat where possible.** Deeply nested state is hard to merge with reducers and hard to read in a debugger. Prefer separate top-level keys over a single nested dict.
- **Split input/output schemas when the internal state has more than ~8 keys.** Callers do not need to see the audit trail you keep internally.
- **Treat schema changes as breaking changes.** Existing checkpoints in your persistence layer were written against the old shape. Either migrate or version the schema.
- **Document each field with a one-line comment naming the writer node(s).** The schema is the contract; a contract without annotation is a contract no one reads.

## 7. Hands-on Exercise

**Whiteboarding exercise (15 min).** Design the state schema for a generic "expense-report approval" workflow. The workflow has these steps:

1. `submit` — employee submits a report (sets `report_id`, `submitter_id`, `line_items`).
2. `categorise` — three parallel category-classifier nodes each tag the report with a category proposal.
3. `manager_review` — manager reviews and either approves or rejects.
4. `finance_check` — finance system validates against the budget.
5. `pay` — payment system disburses (or marks rejected).

Draw the `TypedDict` declaration. For every field:
- Name its writer node(s).
- Decide whether it needs a reducer; justify the choice.
- Identify any field that could be moved out of the input schema (caller doesn't need to set it).

**What good looks like.** A clean answer has: `report_id`/`submitter_id`/`line_items` in the input schema only; an `Annotated[dict, merge_dict]` field for the three parallel categorisers' proposals; a single-writer `manager_decision` field (no reducer); a single-writer `finance_validation` field; a single-writer `payment_status` field; and an `Annotated[list, add]` `events` audit trail that every node appends to. A weak answer either misses the reducer on the parallel writes, lumps everything into one input schema, or invents fields that no node actually owns.

## 8. Key Takeaways

- What is the difference between `TypedDict`, `dataclass`, and `BaseModel` as graph state, and when do I pick each? (LO1)
- Why is the state schema "the contract" between nodes, and what does that constraint *give* me as a framework user? (LO2)
- When does a field need a reducer, and what are the two failure modes when I forget? (LO3)
- When is splitting input/output schemas worth the extra type definitions? (LO4)
- What bugs does an implicit, untyped shared state make easy to write, that an explicit schema makes hard? (LO5)

## Sources

1. [LangGraph Graph API — State, Schemas, Reducers (v1.x docs)](https://docs.langchain.com/oss/python/langgraph/graph-api) — retrieved 2026-05-26
2. [LangGraph low-level concepts — Graphs, StateGraph, message-passing model](https://langchain-ai.github.io/langgraph/concepts/low_level/) — retrieved 2026-05-26
3. [Python typing module — `TypedDict`, `Annotated`](https://docs.python.org/3/library/typing.html) — retrieved 2026-05-26
4. [LangGraph Persistence — checkpoints, threads, super-steps](https://docs.langchain.com/oss/python/langgraph/persistence) — retrieved 2026-05-26

Last verified: 2026-05-26
