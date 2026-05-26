---
week: W03
day: Thu
topic_slug: checkpointing-persistence-memorysaver-vs-postgressaver
topic_title: "Checkpointing + persistence — MemorySaver vs PostgresSaver"
parent_overview: W03/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 13
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
last_verified: 2026-05-26
---

# Checkpointing + persistence — MemorySaver vs PostgresSaver

## 1. Learning Objectives

By the end of this reading, the learner can:

- Explain what a LangGraph checkpoint is, when it is written, and what data it contains.
- Compare `MemorySaver`/`InMemorySaver`, `SqliteSaver`, and `PostgresSaver` and pick the right one for dev, test, and production.
- Configure a `PostgresSaver` against a real Postgres database — including the `setup()` table-creation step and the `autocommit=True` + `row_factory=dict_row` requirements.
- Design a `thread_id` namespacing scheme that prevents cross-tenant state leakage in a multi-tenant system.
- Identify which workflow features (human-in-the-loop, time travel, fault tolerance, memory) require persistence and which do not.

## 2. Introduction

A long-running workflow that does not persist its state is a workflow that cannot survive anything — not a deploy, not a crash, not a human deciding to come back tomorrow. This is true whether the workflow is a multi-step AI agent, a Temporal saga, a Kafka stream-processor, or an AWS Step Function. The moment you want any of "resume after restart," "human review hours later," "replay this run to debug it," or "do not double-charge if the LLM call times out," you have walked into the territory where the persistence layer is load-bearing rather than nice-to-have.

LangGraph splits this concern out cleanly. The graph definition has nothing to do with persistence; the persistence behaviour is configured by passing a *checkpointer* to `graph.compile(checkpointer=...)`. The same graph can run with an in-memory checkpointer in unit tests, a SQLite checkpointer on a laptop, and a Postgres checkpointer in production — the node code does not change. This separation is the same pattern Django's database backends use, and the same pattern Temporal uses for its history store. It is worth internalising because it travels with you to any workflow framework: **the persistence backend is a deployment concern, not a design concern.**

The implication for today is direct: the `MemorySaver` you use in `pytest` and the `PostgresSaver` you wire against the production database are interchangeable from the graph's point of view. The only thing that changes is the cost of getting them wrong.

## 3. Core Concepts

### What gets persisted

LangGraph writes a checkpoint at the end of every super-step ([source: docs.langchain.com persistence, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/persistence)). Each checkpoint is a `StateSnapshot` containing:

- The full graph state at that point.
- The set of nodes that ran in that super-step.
- The `thread_id` and a unique `checkpoint_id`.
- Pending writes from any nodes that completed during a partially-failed super-step.
- Metadata (timestamp, source — "loop"/"input"/"update"/"fork").

This snapshot is what enables four distinct features ([source: docs.langchain.com persistence, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/persistence)):

1. **Human-in-the-loop** — the graph can pause on an interrupt, the human can come back hours later, and the resumed run reads the saved state.
2. **Memory across runs** — multiple invocations on the same `thread_id` see the previous state.
3. **Time travel** — `graph.get_state_history(thread_id)` returns every checkpoint, and you can resume from any of them.
4. **Fault tolerance** — when a node fails, the successful nodes from the same super-step have their pending writes persisted, so a resume does not re-run them.

### Checkpointer options

LangGraph ships several checkpointers, each appropriate for a different runtime context:

| Checkpointer | Backing store | Use case |
|---|---|---|
| `InMemorySaver` (aliased as `MemorySaver`) | Python dict in process | Unit tests, notebook experimentation, dev mode |
| `SqliteSaver` / `AsyncSqliteSaver` | SQLite file | Single-process apps, laptop demos, lightweight deployments |
| `PostgresSaver` / `AsyncPostgresSaver` | Postgres | Production: multi-process, multi-instance, durable across restarts |

The interface is the same across all three: `.get`, `.put`, `.list`, `.get_state_history`. Swapping is a one-line change at compile time.

### Wiring `PostgresSaver` correctly

The PostgresSaver has two requirements that are *easy to get wrong* and produce surprising failures ([source: reference.langchain.com langgraph.checkpoint.postgres, retrieved 2026-05-26](https://reference.langchain.com/python/langgraph.checkpoint.postgres/)):

1. **`.setup()` must be called once** to create the three backing tables (`checkpoints`, `checkpoint_blobs`, `checkpoint_writes`). Without it, your first write fails with a "relation does not exist" error.
2. **If you build the connection yourself rather than using `from_conn_string`, you must pass `autocommit=True` and `row_factory=dict_row`.** Missing the first means `setup()` does not commit; missing the second means every read fails with `TypeError: tuple indices must be integers or slices, not str`. The library accesses rows by name, so the cursor must return dicts.

The recommended wiring for production is a connection pool:

```python
from langgraph.checkpoint.postgres import PostgresSaver
from psycopg_pool import ConnectionPool
from psycopg.rows import dict_row

DB_URI = "postgresql://app:password@postgres:5432/app_db"

pool = ConnectionPool(
    DB_URI,
    max_size=20,
    kwargs={"autocommit": True, "row_factory": dict_row},
)
checkpointer = PostgresSaver(pool)
checkpointer.setup()   # one-time, idempotent — safe to call on every startup

app = graph.compile(checkpointer=checkpointer)
```

For async apps (FastAPI), the equivalent is `AsyncPostgresSaver` + `AsyncConnectionPool`. The same `.setup()` semantics apply.

### `thread_id` is the namespace key

Every invocation of a graph compiled with a checkpointer must pass a `thread_id` in the config ([source: docs.langchain.com interrupts, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/interrupts)):

```python
config = {"configurable": {"thread_id": "..."}}
graph.invoke({"input": "..."}, config=config)
```

The `thread_id` is the *primary key* under which state is stored. Two invocations with the same `thread_id` share state; two with different `thread_id`s are isolated. That sounds obvious until you build a multi-tenant system.

**Multi-tenant rule:** if your graph runs the same workflow for multiple tenants, the `thread_id` must include the tenant identifier. `thread_id = workflow_run_id` is broken — two tenants can generate the same internal `workflow_run_id` independently (especially if you autoincrement). Resuming tenant A's run reads tenant B's checkpoint. The pattern that works:

```python
thread_id = f"{tenant_id}:{workflow_run_id}"
```

This composes naturally with whatever identifier scheme you already use; the colon is a convention, not a framework requirement. The point is that the namespace must be globally unique across tenants.

### The `LANGGRAPH_STRICT_MSGPACK` knob

Checkpoint payloads are msgpack-serialised. If the database is ever compromised, an attacker could write a crafted payload that, on deserialise, instantiates an unintended class. The library ships a strict-mode opt-in ([source: reference.langchain.com checkpoint.postgres, retrieved 2026-05-26](https://reference.langchain.com/python/langgraph.checkpoint.postgres/)):

```bash
LANGGRAPH_STRICT_MSGPACK=true
```

Or programmatically pass `allowed_msgpack_modules` when constructing the checkpointer. Strict mode restricts deserialisation to known-safe types. **For any production deployment this should be on.** It is off by default for backward compatibility, which is a documentation hazard.

## 4. Generic Implementation

Non-Karsun example: a customer-support workflow that handles a multi-day case. The customer opens a ticket, the agent gathers information, escalates to engineering, waits for a response (possibly 48 hours), then resolves. Each ticket is one workflow run, persisted across the multi-day gap.

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

# (graph nodes defined elsewhere)

def build_graph_with_persistence():
    pool = ConnectionPool(
        DB_URI,
        max_size=10,
        kwargs={"autocommit": True, "row_factory": dict_row},
    )
    checkpointer = PostgresSaver(pool)
    checkpointer.setup()

    builder = StateGraph(TicketState)
    # builder.add_node(...) etc.
    return builder.compile(
        checkpointer=checkpointer,
        interrupt_before=["wait_for_engineering"],
    )

graph = build_graph_with_persistence()

# Multi-tenant thread_id: (org, ticket).
def run_ticket(org_id: str, ticket_id: str, payload: dict):
    config = {"configurable": {"thread_id": f"{org_id}:{ticket_id}"}}
    return graph.invoke(payload, config=config)
```

Why this matters in practice:

- The customer-support process can sit paused at `wait_for_engineering` for 48 hours. The FastAPI process restarts twice in that window. The state is intact because it lives in Postgres, not in a Python dict in a process that may not exist anymore.
- Two orgs that happen to autoincrement to `ticket_id = 4711` independently do not collide because the `org_id` prefix isolates their checkpoint namespaces.
- An engineer can call `graph.get_state_history("acme:4711")` and see every step the ticket has been through — useful for both debugging and replying to customer questions about timeline.

## 5. Real-world Patterns

**Fintech — Stripe payment intents and idempotency.** Stripe's PaymentIntent API persists the state of a multi-step payment across the customer's session, the 3DS challenge, the webhook callback, and the eventual capture. The state is keyed by the PaymentIntent ID, which incorporates the merchant account (Stripe's equivalent of multi-tenancy isolation). The persistence is what makes the workflow safe to resume after the browser tab closes during 3DS — exactly the same shape as LangGraph resuming after an interrupt.

**Logistics — Temporal-based order fulfilment.** Companies like Doordash and Snap run order-fulfilment workflows on Temporal, which uses an event-sourced history store as its persistence layer. The workflow code is plain function calls; the framework records every step's inputs and outputs so a resume on a different worker can replay deterministically. The lesson that transfers to LangGraph: **workflow code must be deterministic given persisted inputs.** Non-deterministic operations (random IDs, current time, external API calls) belong inside tasks/nodes that the framework records, not inline in the workflow logic.

**Healthcare — multi-day clinical workflow on FHIR Task resources.** FHIR-based imaging workflows persist the Task state in the FHIR server; the imaging workstation picks up the task hours or days later. The persistence layer is the FHIR server itself; the workflow code is a thin orchestration on top. Replace "FHIR Task" with "LangGraph checkpoint" and the architecture is structurally the same.

**Gaming — save-game systems.** Single-player RPGs persist the world state to local disk at save points and on game-quit. The same data structure must reload across versions of the game (forward-compat) and across player machines (portability). The lessons that map back: schema evolution matters (existing checkpoints will outlive a single deploy), serialisation safety matters (a corrupted save file should not RCE the game), and naming the save slot matters (it is the equivalent of `thread_id`).

## 6. Best Practices

- **Use `MemorySaver` for tests and notebooks; `PostgresSaver` for everything that survives a restart.** Do not put a SQLite file in production; concurrent writes will lock.
- **Always call `.setup()` on first start.** It is idempotent. Putting it in your app startup is the safe default.
- **When building a Postgres connection yourself, pass `autocommit=True` and `row_factory=dict_row` — both are required, both produce silent failures when missed.**
- **Compose `thread_id` from tenant + workflow identity.** `f"{tenant_id}:{workflow_id}"` is the minimum; add more namespacing if you have nested tenancy.
- **Enable `LANGGRAPH_STRICT_MSGPACK=true` in production.** Treat it as a default-on security control.
- **Treat the checkpoint schema as long-lived data.** Migrating it requires planning the same way migrating any other production table does.
- **Set a connection pool sized to your concurrency.** `max_size=20` is a reasonable starting point for a FastAPI app on a single host; tune from there.

## 7. Hands-on Exercise

**Code task (15 min).** Wire a `PostgresSaver` against a local Postgres instance and prove a graph survives a process restart:

1. Build a trivial two-node graph that increments a counter in state.
2. Compile it with `PostgresSaver` against a local Postgres URL.
3. Invoke it once with `thread_id = "demo:1"`. Confirm the counter is `1`.
4. Kill the Python process. Restart.
5. Re-invoke with the same `thread_id`. The counter should now be `2`, not `1`.

**What good looks like.** Acceptance criteria: the post-restart invocation reads the prior counter. The most common mistake is forgetting `.setup()` (first run errors out) or forgetting `autocommit=True` (setup appears to run but the next process sees no tables). A correct solution uses `ConnectionPool` with both kwargs, calls `setup()` once at start, and demonstrates the persistence behaviour with a `print(state["counter"])` after each invoke.

## 8. Key Takeaways

- What is a checkpoint, when does LangGraph write one, and what does it contain? (LO1)
- For each of dev / unit-test / production, which checkpointer do I pick and why? (LO2)
- What are the four things that can go wrong when wiring `PostgresSaver`, and which line of code fixes each? (LO3)
- Why does `thread_id = workflow_run_id` not work in a multi-tenant system, and what is the simplest fix? (LO4)
- Of human-in-the-loop, memory, time travel, and fault tolerance — which are enabled solely by the checkpointer being present? (LO5)

## Sources

1. [LangGraph Persistence — checkpoints, threads, super-steps, pending writes (v1.x docs)](https://docs.langchain.com/oss/python/langgraph/persistence) — retrieved 2026-05-26
2. [langgraph.checkpoint.postgres API reference — setup(), autocommit, row_factory, strict msgpack](https://reference.langchain.com/python/langgraph.checkpoint.postgres/) — retrieved 2026-05-26
3. [LangGraph Durable Execution — determinism, replay, task boundaries](https://docs.langchain.com/oss/python/langgraph/durable-execution) — retrieved 2026-05-26
4. [LangGraph Interrupts — thread_id semantics, resume model](https://docs.langchain.com/oss/python/langgraph/interrupts) — retrieved 2026-05-26

Last verified: 2026-05-26
