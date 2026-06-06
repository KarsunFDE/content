---
week: W03
day: Thu
topic_slug: state-schema-design-graph-state-as-contract
topic_title: "State schema design — graph state is your contract"
parent_overview: W03/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 11
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
last_verified: 2026-06-06
---

# State schema design — graph state is your contract

> [!NOTE]
> **From earlier:** Wed's supervisor-worker pattern passed state between agents implicitly. Today that implicit state becomes an explicit, typed contract — the foundation every node in today's flow depends on.

## 1. Learning Objectives

By the end of this reading, you can:

- Define a LangGraph state schema with `TypedDict` and explain when `dataclass` or Pydantic is the right upgrade.
- Explain why state is the contract between nodes — what each node can read and write, what happens on concurrent writes.
- Identify when a field needs a `reducer` (`Annotated[T, fn]`) versus last-writer-wins semantics.
- Distinguish input schema, output schema, and internal state, and know when splitting them matters.

## 2. Introduction

Every multi-step workflow has shared state — whether it is explicit or not. LangGraph makes it explicit by design: before you add a single node, you define a typed `State` class. Every node receives the full state as its first argument and returns only the fields it changed. The framework merges that partial return into the running state using each field's declared merge semantics.

This is not a convenience feature. The schema enables parallel execution, time-travel debugging, deterministic replay, and resume-from-checkpoint. The same principle appears in AWS Step Functions (state object between tasks), Temporal (workflow state), and Kafka Streams (state stores). **An explicit, typed contract narrows the set of bugs you can write.**

Today's schema anchors `EvaluationState` — the contract every evaluator node, the consensus aggregator, and the SSA-review hard gate reads and writes.

## 3. Core Concepts

### 3.1 Schema options: TypedDict, dataclass, Pydantic

LangGraph supports three schema styles ([source: docs.langchain.com graph-api, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/graph-api)):

| Style | Runtime overhead | Default values | Validation |
|-------|:----------------:|:--------------:|:----------:|
| `TypedDict` | None | No | Static only |
| `dataclass` | Minimal | Yes (`field(default=...)`) | Static only |
| Pydantic `BaseModel` | Per-mutation | Yes | Runtime |

**Rule of thumb:** start with `TypedDict`. Reach for Pydantic only when you need runtime validation on state mutations — it costs a parse on every node return.

### 3.2 Reducers — what happens when two nodes write the same field

By default, two writes to the same field last-writer-wins. Correct for single-owner fields (`ssa_approval_status`); wrong for any field multiple nodes write concurrently.

The fix: annotate the field with a reducer function. LangGraph calls the reducer to merge the existing value with each new write:

```python
from typing import Annotated, TypedDict
from operator import add
from langgraph.graph.message import add_messages

class EvaluationState(TypedDict):
    evaluation_id: str
    agency_id: str                        # multi-tenant thread-key
    proposal_ids: list[str]
    evaluator_scores: Annotated[dict, lambda a, b: {**a, **b}]  # parallel writes merge
    consensus_complete: bool
    ssdd_draft: str | None                # snapshotted at consensus — never regenerated
    ssa_approval_status: str              # "pending" | "approved" | "rejected"
    audit_correlation_id: str
    messages: Annotated[list, add_messages]
```

Skip the reducer on `evaluator_scores` and you get one of two failure modes: silent overwrite (only one evaluator's result survives) or a parallel-update exception. Both are nasty because the first run often looks correct.

> [!TIP]
> **Declare the reducer at the moment you introduce the field.** Adding one later means migrating every checkpoint in the database — treat it as a schema migration from day one.

### 3.3 Input / output / internal schema split

By default input, internal, and output schemas are the same type. Splitting matters when the state has many internal fields callers do not need to see or supply. LangGraph supports separate `input_schema` and `output_schema` at `StateGraph(State, input=InputType, output=OutputType)`. Use it when internal state grows beyond ~8 keys — callers should not couple to audit fields they never write.

> [!TIP]
> **Name fields by who writes them.** `consensus_complete` is clearer than `complete`; `ssa_approval_status` is clearer than `status`. The name carries ownership — a new engineer can grep for `"ssa_approval_status":` in node return values and immediately know which node owns it.

## 4. Generic Implementation

E-commerce order-fulfilment — same schema discipline outside federal-acq:

```python
from typing import Annotated, TypedDict
from operator import add

def merge_dict(left: dict, right: dict) -> dict:
    return {**left, **right}

class OrderFulfilmentState(TypedDict):
    # Set once by caller — never re-written (no reducer needed).
    order_id: str
    customer_id: str

    # Single-writer status fields — last-writer-wins is correct.
    payment_status: str          # "pending" | "authorized" | "captured" | "failed"
    inventory_status: str        # "reserved" | "out_of_stock"
    fulfilment_status: str       # "pending" | "shipped" | "delivered"

    # Multiple parallel warehouse nodes write here — reducer required.
    warehouse_responses: Annotated[dict, merge_dict]

    # Every node appends one audit entry — reducer required.
    events: Annotated[list, add]
```

Reading this declaration tells you the workflow's shape without opening a single node body: two input-only keys, three single-writer status keys, two multi-writer keys needing reducers. A new engineer auditing "who writes `payment_status`?" greps node returns — that grep is only useful because the schema constrains every write through a typed return.

## 5. Real-world Patterns

**Fintech — Stripe PaymentIntent.** Stripe's `PaymentIntent` is a typed schema for the multi-step payment flow. Integrators code against the published field set; "the response varies by path" forces fragile defensive parsing. Typed state is the difference.

> [!NOTE]
> **Cross-domain lesson:** Every long-running multi-step workflow — payment, clinical task, order saga — uses an explicit typed schema as its participant contract. The pattern predates LangGraph; LangGraph just applies it to LLM nodes.

**Healthcare — HL7 FHIR Task.** FHIR's `Task` resource defines a typed schema for clinical workflow steps: `status`, `intent`, `owner`, `input`, `output`. Two vendors collaborate on the same encounter without a shared codebase because the schema is the contract.

## 6. Best Practices

- **Start with `TypedDict`; reach for Pydantic only when runtime validation is genuinely needed.**
- **Annotate every multi-writer field with a reducer the moment you introduce it.** Adding one later means migrating every checkpoint already in the database.
- **Keep the schema flat where possible.** Deeply nested state is hard to merge with reducers and hard to read in a debugger.
- **Split input/output schemas when internal state has more than ~8 keys.**
- **Treat schema changes as breaking changes.** Existing checkpoints were written against the old shape.

> [!WARNING]
> **Anti-pattern: `state-as-untyped-dict`.** Using a plain `dict` as graph state (no `TypedDict`, no reducers) compiles and runs — until a parallel branch silently overwrites another's write and the bug appears hours later in production data. An untyped dict is an implicit contract that every developer renegotiates in their head when they read the code. Define the schema first; it is cheaper than debugging silent overwrites at 02:00.

## 7. Hands-on Exercise

Design the `TypedDict` state schema for an "expense-report approval" workflow: `submit` → `categorise` (three parallel classifiers) → `manager_review` → `finance_check` → `pay`. For every field: name the writer node(s), decide if a reducer is needed, and identify which fields belong in the input schema only.

> [!NOTE]
> **Self-check** (30 s — answer mentally before expanding)
>
> 1. The `categorise` step has three parallel nodes each writing a category proposal. What happens if you forget the reducer?
> 2. `manager_decision` is written by exactly one node. Does it need a reducer?

<details>
<summary>Show answers</summary>

1. Last-writer-wins: only one of the three proposals survives, silently. The other two are lost with no error. The run appears to work until you notice the final category is always from the same classifier.
2. No. Last-writer-wins is correct for single-owner fields — adding a reducer would be unnecessary complexity with no benefit.

</details>

## 8. Key Takeaways

- `TypedDict` is the recommended default; `dataclass` adds defaults; Pydantic adds runtime validation — pick the lightest option that meets your needs.
- State is the contract: constrains what nodes can read and write, provides merge semantics, documents the data model in one place.
- Every multi-writer field needs a reducer; skip it and you get silent overwrite or runtime exceptions.
- Splitting input/output schemas keeps callers decoupled from internal audit state.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://docs.langchain.com/oss/python/langgraph/graph-api — retrieved 2026-05-26 — hot-tech
- https://langchain-ai.github.io/langgraph/concepts/low_level/ — retrieved 2026-05-26 — hot-tech
- https://docs.python.org/3/library/typing.html — retrieved 2026-05-26 — foundation-stable
- https://docs.langchain.com/oss/python/langgraph/persistence — retrieved 2026-05-26 — hot-tech

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

**Schema evolution and checkpoint migration.** When you change a `TypedDict` field name or add a reducer to an existing field, every checkpoint in your persistence layer was written against the old shape. LangGraph will attempt to deserialise the old checkpoint against the new schema — added fields get `None`, removed fields are dropped, but changed reducer semantics produce subtle bugs: old checkpoints that were last-writer-wins will not be retroactively merged.

Production strategy: version the schema. Name the new `StateV2` distinctly, migrate existing threads explicitly, run both schemas in parallel during migration. The same pattern applies to Temporal workflow versioning and Kafka consumer-group schema evolution. The lesson: treat the checkpoint schema with the same care as a database migration — because it is one.

**Input/output schema granularity.** LangGraph's `input_schema` and `output_schema` can be separate Pydantic models or `TypedDict`s. A common senior-FDE move is to define a thin `InputState` with just the caller-supplied fields, keep the full internal schema for node-to-node passing, and define a thin `OutputState` with just the caller-relevant result. This prevents callers from accidentally coupling to fields like `audit_correlation_id` that are internal bookkeeping.

</details>

Last verified: 2026-06-06
