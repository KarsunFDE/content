---
week: W03
day: Thu
topic_slug: checkpointing-persistence-memorysaver-vs-postgressaver
topic_title: "Checkpointing + persistence — MemorySaver vs PostgresSaver"
parent_overview: W03/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://docs.langchain.com/oss/python/langgraph/persistence
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://reference.langchain.com/python/langgraph.checkpoint.postgres/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/oss/python/langgraph/durable-execution
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/oss/python/langgraph/interrupts
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-06-06
---

# Checkpointing + persistence — MemorySaver vs PostgresSaver

> [!NOTE]
> **From earlier:** File 3's fan-out showed that partial-failure survivors are stored as pending writes. Those pending writes have to go *somewhere* durable to survive a restart. That somewhere is today's topic — the checkpointer.

## 1. Learning Objectives

By the end of this reading, you can:

- Explain what a LangGraph checkpoint is, when it is written, and what it contains.
- Pick the right checkpointer for dev, unit-test, and production contexts.
- Configure `PostgresSaver` correctly — `autocommit`, `row_factory`, `setup()`, and pool sizing.
- Design a `thread_id` scheme that prevents cross-tenant state leakage in a multi-tenant system.

## 2. Introduction

A workflow that does not persist its state cannot survive anything — a deploy, a crash, or a human deciding to come back tomorrow. The moment you want "resume after restart," "human review hours later," or "do not re-bill if the LLM call times out," the persistence layer is load-bearing.

LangGraph separates this concern cleanly: the graph definition has nothing to do with persistence. Persistence is configured by passing a *checkpointer* to `graph.compile(checkpointer=...)`. The same graph runs with `MemorySaver` in unit tests and `PostgresSaver` in production — node code does not change. The SSA's 18-hour gap is only survivable because `PostgresSaver` keeps the checkpoint alive across FastAPI restarts.

## 3. Core Concepts

### 3.1 What gets persisted

LangGraph writes a checkpoint at the end of every super-step ([source: docs.langchain.com persistence, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/persistence)). Each checkpoint is a `StateSnapshot` containing the full graph state, nodes that ran, `thread_id`, `checkpoint_id`, pending writes from partial failures, and metadata.

This enables four features: **HITL** (pause, return hours later), **memory across runs** (same `thread_id` sees prior state), **time travel** (`graph.get_state_history`), and **fault tolerance** (siblings' writes survive a branch failure).

> [!TIP]
> **The checkpointer is a deployment concern, not a design concern.** Node code never references it — you swap `MemorySaver` for `PostgresSaver` at compile time without touching a single node.

### 3.2 Checkpointer options

| Checkpointer | Backing store | Use case |
|---|---|---|
| `InMemorySaver` / `MemorySaver` | Python dict in-process | Unit tests, notebooks, dev mode |
| `SqliteSaver` / `AsyncSqliteSaver` | SQLite file | Single-process apps, laptop demos |
| `PostgresSaver` / `AsyncPostgresSaver` | Postgres | Production: multi-process, durable across restarts |

The interface is identical across all three. Swapping is a one-line change at compile time.

### 3.3 Wiring PostgresSaver correctly

Two requirements that are easy to miss ([source: reference.langchain.com langgraph.checkpoint.postgres, retrieved 2026-05-26](https://reference.langchain.com/python/langgraph.checkpoint.postgres/)):

1. **`.setup()` must be called once** to create the three backing tables (`checkpoints`, `checkpoint_blobs`, `checkpoint_writes`). Without it, the first write fails with "relation does not exist."
2. **`autocommit=True` and `row_factory=dict_row` are both required** when building the connection yourself. Missing `autocommit` means `setup()` does not commit. Missing `row_factory` means every read fails with `TypeError: tuple indices must be integers or slices, not str`.

```python
from langgraph.checkpoint.postgres import PostgresSaver
from psycopg_pool import ConnectionPool
from psycopg.rows import dict_row

DB_URI = "postgresql://acquire_gov:...@postgres:5432/acquire_gov"
pool = ConnectionPool(
    DB_URI,
    max_size=20,
    kwargs={"autocommit": True, "row_factory": dict_row},
)
checkpointer = PostgresSaver(pool)
checkpointer.setup()   # idempotent — safe to call on every startup

app = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["ssa_review_ssdd", "record_award"],
)
```

For async FastAPI apps use `AsyncPostgresSaver` + `AsyncConnectionPool`. Same `.setup()` semantics.

### 3.4 `thread_id` is the namespace key

Every invocation passes a `thread_id` in config. Two invocations with the same `thread_id` share state; different `thread_id`s are isolated. In a multi-tenant system, `thread_id` must include the tenant identifier ([source: docs.langchain.com interrupts, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/interrupts)):

```python
# Wrong: two agencies can both generate evaluation_id = 4711 internally.
thread_id = evaluation_id

# Correct: globally unique across tenants.
thread_id = f"{agency_id}:{evaluation_id}"
```

Careless namespacing is multi-tenant leakage: Agency A's SSA resumes against Agency B's checkpoint.

> [!IMPORTANT]
> **`LANGGRAPH_STRICT_MSGPACK=true` in production.** Checkpoint payloads are msgpack-serialised. Strict mode restricts deserialisation to known-safe types. It is off by default — treat it as a required security control in any production deployment.

## 4. Generic Implementation

Multi-day customer-support ticket — paused while waiting for an engineering response:

```python
from typing import Annotated, TypedDict
from operator import add
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.postgres import PostgresSaver
from psycopg_pool import ConnectionPool
from psycopg.rows import dict_row

class TicketState(TypedDict):
    ticket_id: str
    customer_id: str
    status: str
    engineering_response: str | None
    events: Annotated[list, add]

def build_graph():
    pool = ConnectionPool(
        DB_URI,
        max_size=10,
        kwargs={"autocommit": True, "row_factory": dict_row},
    )
    checkpointer = PostgresSaver(pool)
    checkpointer.setup()
    builder = StateGraph(TicketState)
    # nodes added here...
    return builder.compile(
        checkpointer=checkpointer,
        interrupt_before=["wait_for_engineering"],
    )

def run_ticket(org_id: str, ticket_id: str, payload: dict):
    config = {"configurable": {"thread_id": f"{org_id}:{ticket_id}"}}
    return graph.invoke(payload, config=config)
```

The ticket can sit paused at `wait_for_engineering` for 48 hours while the FastAPI process restarts twice. State is intact because it lives in Postgres, not a Python dict in a dead process.

## 5. Real-world Patterns

**Fintech — Stripe PaymentIntent.** Stripe persists PaymentIntent state across the 3DS challenge and webhook callback, keyed by `PaymentIntentId` with merchant-account scoping. Resume after a browser-tab close mid-3DS is the same shape as LangGraph resuming after an interrupt.

**Logistics — Temporal-based fulfilment.** Temporal uses an event-sourced history store. The lesson: workflow code must be deterministic given persisted inputs — same rule applies in LangGraph (file 7).

> [!NOTE]
> **Cross-domain lesson:** Every durable workflow system — Stripe, Temporal, FHIR Tasks — separates persistence from workflow logic. You configure where state goes; node code never knows which backing store is active.

## 6. Best Practices

- **`MemorySaver` for tests and notebooks; `PostgresSaver` for everything that survives a restart.**
- **Always call `.setup()` on startup.** It is idempotent — put it in the app lifespan.
- **Pass both `autocommit=True` and `row_factory=dict_row`.** Both are required; both produce silent failures when missed.
- **Compose `thread_id` from tenant + workflow identity.** `f"{tenant_id}:{workflow_id}"` is the minimum.
- **Set a connection pool sized to your concurrency.** `max_size=20` is a reasonable starting point.

> [!WARNING]
> **Anti-pattern: `memorysaver-in-prod`.** Using `MemorySaver` in production is a common first-draft mistake — the graph compiles and the HITL interrupt appears to work. Then the FastAPI process restarts and every paused workflow loses its state. The SSA cannot resume; the evaluation must restart from scratch. Wire `PostgresSaver` before wiring any interrupt.

## 7. Hands-on Exercise

Wire `PostgresSaver` against a local Postgres instance. Prove a graph survives a restart: build a two-node counter graph, compile with `PostgresSaver`, invoke with `thread_id = "demo:1"` (counter = 1), kill the process, restart, re-invoke with the same `thread_id` — counter should be 2, not 1.

> [!NOTE]
> **Self-check** (30 s — answer mentally before expanding)
>
> 1. You wire `PostgresSaver` but forget to call `.setup()`. What error do you see on the first invoke?
> 2. Your graph runs correctly in tests with `MemorySaver`. You swap to `PostgresSaver` in production and every read fails with `TypeError: tuple indices must be integers or slices, not str`. What did you forget?

<details>
<summary>Show answers</summary>

1. `psycopg.errors.UndefinedTable` (or similar "relation does not exist") on the first checkpoint write. The three backing tables (`checkpoints`, `checkpoint_blobs`, `checkpoint_writes`) do not exist until `.setup()` runs.
2. You forgot `row_factory=dict_row` on the connection pool kwargs. `PostgresSaver` accesses checkpoint rows by column name; without `dict_row`, psycopg returns tuples and the name-based access fails with `TypeError`.

</details>

## 8. Key Takeaways

- LangGraph writes a checkpoint at the end of every super-step; enables HITL, memory, time travel, and fault tolerance.
- `MemorySaver` for tests; `PostgresSaver` for production — same graph, one-line swap at compile time.
- `PostgresSaver` requires `autocommit=True`, `row_factory=dict_row`, and `setup()` — all three; all silent failures when missed.
- `thread_id` must include tenant identity in multi-tenant systems to prevent cross-tenant state leakage.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://docs.langchain.com/oss/python/langgraph/persistence — retrieved 2026-05-26 — hot-tech
- https://reference.langchain.com/python/langgraph.checkpoint.postgres/ — retrieved 2026-05-26 — hot-tech
- https://docs.langchain.com/oss/python/langgraph/durable-execution — retrieved 2026-05-26 — hot-tech
- https://docs.langchain.com/oss/python/langgraph/interrupts — retrieved 2026-05-26 — hot-tech

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

**Checkpoint schema evolution.** The three backing tables (`checkpoints`, `checkpoint_blobs`, `checkpoint_writes`) are created by `.setup()` and are long-lived production data. When LangGraph releases a new minor version that changes the checkpoint schema, you may need to run a migration. The `setup()` call is idempotent for the *current* schema but does not automatically migrate old rows. Treat the checkpoint tables with the same care as any other production Postgres migration: version-control the migration, test in staging, and have a rollback plan.

**`LANGGRAPH_STRICT_MSGPACK` in depth.** Checkpoint payloads are msgpack-serialised Python objects. Without strict mode, deserialisation can instantiate arbitrary Python classes if an attacker controls the checkpoint store (compromised Postgres). Strict mode restricts allowed types to a safe whitelist: primitives, dicts, lists, and the LangGraph-internal types. Pass `allowed_msgpack_modules` to the checkpointer constructor to extend the whitelist for custom types in your state schema. In practice, if your `TypedDict` fields are all primitives and standard Python collections, you will not need to extend the list.

**AsyncPostgresSaver and FastAPI lifespan.** For production FastAPI apps, use `AsyncPostgresSaver` + `AsyncConnectionPool`. Wire both inside the `@asynccontextmanager` lifespan so the pool is created once on startup, `setup()` runs once, and the pool is closed on shutdown. Do not create the pool inside a request handler — connection pool creation is expensive and not thread-safe when done concurrently.

</details>

Last verified: 2026-06-06
