---
week: W03
day: Mon
topic_slug: memory-patterns-thread-vs-cross-thread-vs-shared
topic_title: "Memory patterns — thread vs cross-thread vs shared"
parent_overview: W03/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 11
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
last_verified: 2026-05-26
---

# Memory patterns — thread vs cross-thread vs shared

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish three memory scopes (per-thread, supervisor-curated, cross-thread/durable) and pick the right one for a given problem.
- Explain what a "thread" means in an agent framework and how `thread_id` shape affects multi-tenant safety.
- Justify when to use an in-memory checkpointer vs a durable backend like Postgres, and what failure mode the wrong choice causes.
- Recognise the shared-scratchpad anti-pattern in multi-agent designs and prescribe the curated-view alternative.
- Name a defensible "we use X now, switch to Y by date Z" memory plan that can survive a code review.

## 2. Introduction

Memory is the part of agent design where most production accidents happen. Not because the patterns are exotic — they are not — but because most teams adopt the *demo-default* memory shape (one in-memory store, one global scratchpad) and never revisit it before going to production. The result is the predictable cluster of bugs: state evaporating across a restart, tenants seeing each other's traces, multi-agent runs ballooning to thousands of tokens of irrelevant scratchpad, and audit replays that have no continuous history because the checkpointer never wrote anything durable.

The remedy is to treat memory as a *scope* decision, not a *backend* decision. The same agent run usually needs more than one scope: short-term state inside a single conversation, durable state across days when a workflow pauses for human review, and supervisor-curated state when multiple agents collaborate. These scopes map onto different primitives in current orchestration frameworks, and picking the right primitive per scope is the design move that separates a demo from a system you can ship.

LangGraph and similar frameworks expose memory through two complementary primitives: **checkpointers** that persist *thread* state, and **stores** that persist data *across threads*. Knowing which one to reach for in which case is the whole game ([LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence), retrieved 2026-05-26).

## 3. Core Concepts

### 3.1 The three scopes

| Scope | What it holds | Lifetime | Primitive (LangGraph) |
|---|---|---|---|
| **Per-thread** | One conversation, one workflow run | Until thread completes | Checkpointer (state) |
| **Supervisor-curated** | A filtered view passed between collaborating agents | One supervised run | Sub-graphs + explicit state passing |
| **Cross-thread / durable** | Facts a user accumulates across days, audit history | Indefinite | Store (cross-thread) |

The naming varies by framework — sometimes "short-term" and "long-term" are used — but the three-scope shape is the same. The LangChain blog post *Memory for agents* divides it into "what" memory holds (semantic, episodic, procedural) and "how" it is updated, but the operational scopes still fall into these three ([LangChain blog — Memory for agents](https://blog.langchain.com/memory-for-agents/), retrieved 2026-05-26).

### 3.2 A thread is a unit of resumable execution

In LangGraph, a thread is a single graph invocation identified by a `thread_id`. The checkpointer writes graph state on every step, keyed by that ID. If the graph hits an interrupt and the user comes back tomorrow, you resume by invoking the graph with the same `thread_id` and the checkpointer reads the state back ([LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence), retrieved 2026-05-26).

Two design decisions follow:

- **Thread granularity** — what real-world thing does one thread correspond to? One ticket, one conversation, one onboarding case, one drafting session? Pick the smallest unit that has a coherent state.
- **Thread ID namespace** — a flat UUID is fine for a single-tenant prototype, but in multi-tenant systems the convention is `f"{tenant_id}:{case_id}"` so it is impossible to accidentally resume one tenant's thread against another's data.

### 3.3 Checkpointer vs Store — they are not the same thing

This is the most common confusion. Both are forms of memory but they serve different scopes:

- **Checkpointer** writes *graph state* on every step, keyed by `thread_id`. It is what lets you resume an interrupted graph. It is *not* meant to hold facts you want to read across threads.
- **Store** is a key-value+search interface for facts that outlive any single thread — user preferences, account-level rules, an analyst's historical decisions. Stores are organised by *namespaces*, not by `thread_id` ([LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence), retrieved 2026-05-26).

A graph can be compiled with both at the same time. You pass `checkpointer=...` and `store=...` to `builder.compile(...)`. Nodes receive the store as a parameter so they can read and write cross-thread data without smuggling it through the graph state.

### 3.4 The three checkpointer options and when each is right

- **`InMemorySaver`** — process memory only. Right for unit tests, local dev, demos. Wrong the moment your process restarts or your workflow needs to survive a deploy.
- **`PostgresSaver`** / `AsyncPostgresSaver` — durable, transactional, queryable. The default for any production workflow with interrupts, multi-day pauses, or audit requirements ([LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence), retrieved 2026-05-26).
- **`SQLite` / other community backends** — useful for single-node deployments or self-hosted tooling, but rare in cloud production.

The shape of a Postgres-backed checkpointer is three tables: `checkpoints` (one row per step), `checkpoint_blobs` (the serialised state values), and `checkpoint_writes` (pending writes between checkpoints). These tables *are* the audit history of every run, queryable with normal SQL.

### 3.5 Supervisor-curated memory — the anti-shared-scratchpad rule

When multiple agents collaborate, the naive design gives them all read-write access to a single shared scratchpad. This blows up for three reasons:

- **Token bloat.** Every agent reads every other agent's working notes; context windows fill up with noise.
- **Traceability collapse.** When something goes wrong, you cannot say which agent wrote which fact — they all touched the same data.
- **Tenant leakage risk.** If two simultaneous runs reach the same shared scratchpad through a misconfigured key, one tenant's data appears in another's trace.

The supervisor-worker correction is: the supervisor *curates* what each worker sees and what gets merged back into shared state. Workers receive a filtered view, not the full graph state. Anthropic's *Building Effective Agents* makes the same point in a different vocabulary: each agent should have a clear, narrow contract about what it consumes and emits ([Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents), retrieved 2026-05-26).

### 3.6 The history API — why durable checkpointers earn their keep

Once you are on a durable checkpointer, you get an inspection API for free. `graph.get_state_history(config)` returns every step the thread has taken; you can iterate it, find a specific checkpoint, and even *resume from an earlier checkpoint* with edits — the "time-travel" pattern that makes regulated audit replays tractable ([LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence), retrieved 2026-05-26).

## 4. Generic Implementation

A worked example outside federal-acquisitions: an online tutoring platform that pairs a learner with an AI tutor and a periodic human reviewer.

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
    """
    Cross-thread memory: read durable facts about the learner that exist
    across many sessions. The store is namespaced by learner_id, so we
    never accidentally read another learner's data.
    """
    namespace = ("learners", state["learner_id"], "preferences")
    facts = store.search(namespace, limit=10)
    return {"learner_preferences": [f.value for f in facts]}

def tutor_turn(state) -> dict:
    """Per-thread memory: the running transcript lives in graph state."""
    reply = llm.respond(state["current_question"], state["transcript"])
    state["transcript"].append({"role": "tutor", "content": reply})
    return {"transcript": state["transcript"]}

def maybe_request_human_review(state) -> dict:
    """If the conversation drifts, mark for human review (interrupt sits after)."""
    flagged = quality_check(state["transcript"])
    return {"review_pending": flagged}

def persist_learner_preference(state, *, store) -> dict:
    """Update the cross-thread store at the end of a session."""
    namespace = ("learners", state["learner_id"], "preferences")
    store.put(namespace, key="last-topic", value={"topic": state["current_question"]})
    return {}

builder = StateGraph(TutoringState)
builder.add_node("load_facts", get_relevant_learner_facts)
builder.add_node("tutor", tutor_turn)
builder.add_node("review_gate", maybe_request_human_review)
builder.add_node("persist", persist_learner_preference)
builder.add_edge(START, "load_facts")
builder.add_edge("load_facts", "tutor")
builder.add_edge("tutor", "review_gate")
builder.add_edge("review_gate", "persist")
builder.add_edge("persist", END)

# Production: durable checkpointer + durable store.
checkpointer = PostgresSaver.from_conn_string(DB_URL)
store = PostgresStore.from_conn_string(DB_URL)
graph = builder.compile(
    checkpointer=checkpointer,
    store=store,
    interrupt_before=["persist"],   # pause for human review when flagged
)

# Invocation: thread_id namespaces the per-session state; store namespaces
# the cross-session facts.
config = {"configurable": {"thread_id": f"tenant-acme:{session_id}"}}
graph.invoke(initial_state, config=config)
```

Notice the separation: the *transcript for this session* lives in graph state, written through the checkpointer keyed by `thread_id`. The *learner's persistent preferences* live in the store, namespaced by `learner_id`. Two different scopes, two different primitives, one graph.

## 5. Real-world Patterns

**E-commerce — multi-day cart abandonment recovery.** A retail platform's cart-recovery workflow pauses overnight to send a follow-up email, then resumes when the shopper clicks the link. The thread is the cart; the `thread_id` is `f"{tenant_id}:{cart_id}"`; the checkpointer is Postgres-backed because the workflow spans days. The cross-thread store holds the shopper's lifetime preferences (size, brand, opt-in status) used to personalise the email. Two scopes, two primitives, one workflow ([LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence), retrieved 2026-05-26).

**Healthcare — patient-journey state machines.** Care pathways that span weeks (referral → consult → procedure → follow-up) are textbook resumable threads. The thread is the referral case; per-thread state holds the working record of this encounter; cross-thread store holds the patient's medical history and allergies. Modern hospital systems are explicit about the boundary because clinical data governance does not allow a per-visit cache to silently drift from the canonical record.

**Gaming — competitive-ladder coaching agents.** A coaching agent that reviews a player's matches and gives advice needs both: per-thread state for "this coaching session," cross-thread store for "what we have discussed before and what the player's weaknesses are over the last 30 days." Without the cross-thread layer, the coach starts from scratch every session — which players experience as the assistant being amnesiac ([LangChain blog — Memory for agents](https://blog.langchain.com/memory-for-agents/), retrieved 2026-05-26).

**Fintech — KYC onboarding case files.** A KYC pipeline that takes a customer through document upload, address verification, and sanctions screening over five business days *requires* a durable checkpointer; an `InMemorySaver` would lose state on every deploy. The cross-thread store holds the institution's standing rules (jurisdiction lists, screening tiers) that every case must apply. Production runs always pair the two primitives, and the audit-readout is implemented as `get_state_history(config)` over the case's `thread_id`.

## 6. Best Practices

- Pick the thread granularity before the backend — what real-world unit corresponds to one resumable run?
- Namespace `thread_id` with tenant from the first commit (`f"{tenant_id}:{case_id}"`); retrofitting multi-tenant safety into flat UUIDs is painful.
- Default to a durable checkpointer (`PostgresSaver`) for anything that touches a human gate, a multi-day workflow, or an audit requirement.
- Use the store, not the checkpointer, for facts that outlive any single thread — checkpointers are not designed for cross-thread reads.
- For multi-agent designs, give each worker a filtered view of state; never give all workers read-write to a shared scratchpad.
- Pin a memory-migration date in the ADR: "we use `InMemorySaver` for the prototype, switch to `PostgresSaver` by W3 Thu" is defensible; "we will figure out memory later" is not.
- Use `get_state_history(config)` as the audit-replay surface — a durable checkpointer plus this API is your built-in audit trail.

## 7. Hands-on Exercise

**Code task (12–15 minutes).** Take the tutoring-platform snippet in §4 and modify it to support a *team-tutoring* mode where two AI tutors (a math tutor and a physics tutor) collaborate under a supervisor for a single learner session. The supervisor routes each learner question to one of the workers based on the question's topic.

You must:

1. Add a `supervisor_route` node that decides which worker gets the question (math vs physics).
2. Add the two worker nodes; each worker should see only the *current question* and *its own past turns* from the transcript — not the other worker's turns.
3. Keep the cross-thread store untouched — both workers can still read the learner's preferences.
4. Decide whether the workers' decisions need a soft gate, a hard gate, or no gate, and justify your choice in one comment.

**What good looks like.** Each worker has its own slice of the transcript (filtered by `role` field) rather than a shared scratchpad. The supervisor's routing decision is a pure function over the current question — not a side-channel read from the store. You did not add a hard gate (no irreversible side effects in tutoring), but you did add a soft gate (the supervisor surfaces a confidence flag) when the routing is ambiguous. The `thread_id` is still namespaced by tenant, and you can explain why the store contents are *not* worker-namespaced (the learner is one person; both workers should see the same preferences).

## 8. Key Takeaways

- Can you name the three memory scopes and pick the right primitive for each in your favourite agent framework?
- Can you explain why a `thread_id` should be namespaced by tenant from the start, and what bug you avoid by doing so?
- Can you justify the choice between `InMemorySaver` and `PostgresSaver` for a given workflow in one paragraph?
- Can you describe the shared-scratchpad anti-pattern and prescribe the curated-view alternative?
- Can you point to the API (`get_state_history`) that turns a durable checkpointer into an audit-replay surface?

## Sources

1. [LangGraph Persistence — Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/persistence) — retrieved 2026-05-26
2. [LangGraph Memory overview — Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/memory) — retrieved 2026-05-26
3. [Memory for agents — LangChain blog](https://blog.langchain.com/memory-for-agents/) — retrieved 2026-05-26
4. [Conversational Memory for LLMs with LangChain — Pinecone](https://www.pinecone.io/learn/series/langchain/langchain-conversational-memory/) — retrieved 2026-05-26
5. [Building Effective AI Agents — Anthropic](https://www.anthropic.com/research/building-effective-agents) — retrieved 2026-05-26

Last verified: 2026-05-26
