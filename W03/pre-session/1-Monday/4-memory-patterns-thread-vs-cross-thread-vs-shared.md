---
week: W03
day: Mon
topic_slug: memory-patterns-thread-vs-cross-thread-vs-shared
topic_title: "Memory patterns — thread vs cross-thread vs shared"
parent_overview: W03/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://docs.langchain.com/oss/python/langgraph/persistence
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/oss/python/langgraph/memory
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://blog.langchain.com/memory-for-agents/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.pinecone.io/learn/series/langchain/langchain-conversational-memory/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.anthropic.com/research/building-effective-agents
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-06-06
---

# Memory patterns — thread vs cross-thread vs shared

> [!NOTE]
> **From earlier:** Topic 3 introduced the checkpointer as the primitive that turns a graph into a resumable process. This topic is the design decision: which checkpointer backend, which memory scope, and how the plan-spec names that choice today.

## 1. Learning Objectives

- Distinguish three memory scopes (per-thread, supervisor-curated, cross-thread/durable) and pick the right one for a given problem.
- Explain what a "thread" means in LangGraph and how `thread_id` shape affects multi-tenant safety.
- Justify `InMemorySaver` vs `PostgresSaver` and name the failure mode each wrong choice causes.
- Recognise the shared-scratchpad anti-pattern and prescribe the curated-view alternative.

## 2. Introduction

Most production memory accidents happen because teams adopt the demo-default (one in-memory store, one global scratchpad) and never revisit it. The result: state evaporates across restarts, tenants see each other's traces, multi-agent runs balloon with irrelevant content, and audit replays have no history.

The remedy is to treat memory as a *scope* decision, not a *backend* decision. The same agent run usually needs more than one scope: short-term state inside a conversation, durable state across days, and supervisor-curated state for multi-agent collaboration. Today's plan-spec must name the memory shape — even if the answer is "we use `InMemorySaver` Tue–Wed; switch to `PostgresSaver` Thu."

LangGraph exposes memory through two complementary primitives: **checkpointers** that persist *thread* state, and **stores** that persist data *across threads* ([LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence), retrieved 2026-05-26).

## 3. Core Concepts

### 3.1 The three scopes

| Scope | What it holds | Lifetime | LangGraph primitive |
|---|---|---|---|
| **Per-thread** | One conversation, one workflow run | Until thread completes | Checkpointer (state) |
| **Supervisor-curated** | Filtered view passed between collaborating agents | One supervised run | Sub-graphs + explicit state passing |
| **Cross-thread / durable** | Facts accumulated across days, audit history | Indefinite | Store (cross-thread) |

The LangChain blog *Memory for agents* divides memory into semantic, episodic, and procedural; operationally the three-scope shape is the same ([LangChain blog](https://blog.langchain.com/memory-for-agents/), retrieved 2026-05-26).

### 3.2 Thread granularity and `thread_id` namespace

A thread is a single graph invocation identified by a `thread_id`. Two design decisions:

- **Thread granularity** — what real-world thing does one thread correspond to? One ticket, one evaluation, one case? Pick the smallest coherent unit.
- **Thread ID namespace** — a flat UUID works for single-tenant prototypes. Multi-tenant systems must use `f"{tenant_id}:{case_id}"` from the first commit; retrofitting is painful ([LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence), retrieved 2026-05-26).

> [!TIP]
> **Operational implication:** Name your `thread_id` namespace in today's plan-spec ADR entry — even one line. A bare UUID is a multi-tenant accident waiting to happen. For the pair-project use `f"{pair_slug}:{case_id}"` from day one.

### 3.3 Checkpointer vs Store — not the same thing

Both are memory, but different scopes:

- **Checkpointer** writes *graph state* on every step, keyed by `thread_id`. Resumes interrupted graphs. Not for cross-thread reads.
- **Store** is a key-value + search interface for facts that outlive any single thread — preferences, rules, historical decisions. Organised by namespaces.

A graph is compiled with both: `builder.compile(checkpointer=..., store=...)`. Nodes receive the store as a parameter to read and write cross-thread data without smuggling it through graph state.

> [!IMPORTANT]
> **Memory scope decision in today's plan-spec.** The ADR does not need a production backend today — but it must name the choice and defer it explicitly. *"We use `InMemorySaver` Tue–Wed; switch to `PostgresSaver` Thu when hard-interrupt gates land"* is defensible. *"We'll figure out memory later"* is not. The W2 §0 retro likely surfaces a multi-tenant filtering gap — the memory plan is where that carry-forward lands.

## 4. Generic Implementation

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.store.postgres import PostgresStore
from typing import TypedDict

class TutoringState(TypedDict):
    learner_id: str
    session_id: str
    current_question: str
    transcript: list[dict]
    review_pending: bool

def get_relevant_learner_facts(state, *, store) -> dict:
    # Cross-thread: facts about the learner across many sessions.
    namespace = ("learners", state["learner_id"], "preferences")
    facts = store.search(namespace, limit=10)
    return {"learner_preferences": [f.value for f in facts]}

def tutor_turn(state) -> dict:
    # Per-thread: running transcript lives in graph state.
    reply = llm.respond(state["current_question"], state["transcript"])
    state["transcript"].append({"role": "tutor", "content": reply})
    return {"transcript": state["transcript"]}

def persist_learner_preference(state, *, store) -> dict:
    # Update the cross-thread store at session end.
    namespace = ("learners", state["learner_id"], "preferences")
    store.put(namespace, key="last-topic",
              value={"topic": state["current_question"]})
    return {}

builder = StateGraph(TutoringState)
builder.add_node("load_facts", get_relevant_learner_facts)
builder.add_node("tutor", tutor_turn)
builder.add_node("persist", persist_learner_preference)
builder.add_edge(START, "load_facts")
builder.add_edge("load_facts", "tutor")
builder.add_edge("tutor", "persist")
builder.add_edge("persist", END)

checkpointer = PostgresSaver.from_conn_string(DB_URL)
store = PostgresStore.from_conn_string(DB_URL)
graph = builder.compile(checkpointer=checkpointer, store=store,
                        interrupt_before=["persist"])

# thread_id namespaces the per-session state.
config = {"configurable": {"thread_id": f"tenant-acme:{session_id}"}}
graph.invoke(initial_state, config=config)
```

Session transcript lives in graph state (per-thread checkpointer). Learner preferences live in the store (cross-thread). Two scopes, two primitives, one graph.

## 5. Real-world Patterns

**E-commerce — multi-day cart abandonment recovery.** A cart-recovery workflow pauses overnight, resumes when the shopper clicks the link. `thread_id` = `f"{tenant_id}:{cart_id}"`; checkpointer is Postgres-backed; store holds lifetime preferences. Two scopes, two primitives, one workflow ([LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence), retrieved 2026-05-26).

**Fintech — KYC onboarding case files.** A KYC pipeline spanning five business days requires a durable checkpointer; `InMemorySaver` loses state on every deploy. The cross-thread store holds institutional rules (jurisdiction lists, screening tiers). Audit readout is `get_state_history(config)` over the case's `thread_id`.

> [!NOTE]
> **Cross-domain lesson:** The durable-checkpointer requirement is a correctness concern, not performance. Any workflow with a human gate that outlasts a process restart needs `PostgresSaver`. The pair-project's HITL #3 gate fires across the 18-hour SSA gap — plan the backend today.

## 6. Best Practices

- Pick thread granularity before the backend — what real-world unit corresponds to one resumable run?
- Namespace `thread_id` with tenant from the first commit; retrofitting is painful.
- Default to `PostgresSaver` for anything with a human gate, multi-day span, or audit requirement.
- Use the store, not the checkpointer, for facts that outlive any single thread.
- For multi-agent designs, give each worker a filtered view of state; never give all workers read-write to a shared scratchpad.
- Pin a memory-migration date in the ADR: *"InMemorySaver through W3 Wed; PostgresSaver by Thu"* is defensible.

> [!WARNING]
> **Anti-pattern: `chat-history-as-memory`.** Stuffing the full conversation history into graph state and passing it to every node is the single-agent shared-scratchpad problem. Context windows fill with irrelevant turns; token cost balloons; traceability collapses. Use the Store for cross-session facts; keep per-thread state lean. See slug in `known-bad-patterns.yml`.

## 7. Hands-on Exercise

Take the tutoring-platform snippet in §4 and modify it for a *team-tutoring* mode: two AI tutors (math and physics) collaborate under a supervisor for a single learner session. The supervisor routes each question to one worker based on topic.

Requirements: (1) add a `supervisor_route` node, (2) add two worker nodes each seeing only the current question and their own past turns, (3) keep the cross-thread store unchanged, (4) decide whether workers' decisions need a gate and justify in one comment.

> [!NOTE]
> **Self-check** (30s — answer mentally before expanding)
>
> 1. Each worker should see only its own past turns from the transcript. What's the simplest implementation — filter by `role` field in the node, or split the `TypedDict` into two separate state fields?
> 2. Should the store namespace include the worker's subject (math/physics) or just the learner ID? Why?

<details>
<summary>Show answers</summary>

1. Filter by `role` field inside the worker node — this avoids changing the state shape and keeps the supervisor in control of what each worker sees. Splitting the TypedDict creates two parallel state trees the supervisor must merge, which is more complex for the same outcome.
2. Just the learner ID. Both workers are tutoring the *same* learner; they should read the same preferences (learning style, past topics). Namespacing by subject would prevent the physics tutor from knowing the learner prefers visual explanations — a preference that applies to both subjects.

</details>

## 8. Key Takeaways

- Three scopes: per-thread (checkpointer), supervisor-curated (filtered sub-graph state), cross-thread (store).
- `thread_id` is the primary audit-trail key — namespace it with tenant from day one.
- `InMemorySaver` for dev/tests; `PostgresSaver` for anything with human gates, multi-day spans, or audit requirements.
- Shared scratchpad in multi-agent designs causes token bloat, traceability collapse, and tenant leakage.
- `get_state_history(config)` over a durable checkpointer is your built-in audit-replay surface.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://docs.langchain.com/oss/python/langgraph/persistence — retrieved 2026-05-26 — hot-tech
- https://docs.langchain.com/oss/python/langgraph/memory — retrieved 2026-05-26 — hot-tech
- https://blog.langchain.com/memory-for-agents/ — retrieved 2026-05-26 — hot-tech
- https://www.pinecone.io/learn/series/langchain/langchain-conversational-memory/ — retrieved 2026-05-26 — foundation-stable
- https://www.anthropic.com/research/building-effective-agents — retrieved 2026-05-26 — foundation-stable

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

The Postgres checkpointer schema is three tables: `checkpoints` (one row per step), `checkpoint_blobs` (serialised state values), and `checkpoint_writes` (pending writes between checkpoints). These tables are the audit history of every run, queryable with normal SQL. Senior FDEs should know that `checkpoint_writes` is the table to watch for debugging interrupted graphs — a write that never promoted to a checkpoint indicates a crash mid-step.

The Store API (`store.search(namespace, query=..., limit=N)`) supports semantic search when backed by a vector-capable Postgres extension (pgvector). This matters for the pair-project: if the cross-thread store holds historical CO decisions (e.g., past routing choices on similar proposals), a semantic search over the store can seed the LLM's context with relevant precedents. This is the bridge between W2's RAG layer and W3's agent memory — the RAG corpus is a cross-thread store at programme scale.

Multi-tenant isolation: the store namespace is the isolation boundary. A misconfigured namespace key (e.g., accidentally using a bare `case_id` without the `tenant_id` prefix) is how tenant A's stored facts leak into tenant B's context. Audit the namespace construction in code review before any multi-tenant deploy.

</details>

Last verified: 2026-06-06
