---
week: W03
day: Mon
topic_slug: agents-are-state-machines-with-hitl-nodes
topic_title: "Agents are state machines with HITL nodes"
parent_overview: W03/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://docs.langchain.com/oss/python/langgraph/graph-api
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/oss/python/langgraph/interrupts
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/oss/python/langgraph/persistence
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.anthropic.com/research/building-effective-agents
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://react-lm.github.io/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-06-06
---

# Agents are state machines with HITL nodes

> [!NOTE]
> **From earlier:** Topic 2 established that irreversible actions need hard gates. This topic gives those gates a *home* — they are nodes in a state machine, not UI afterthoughts bolted onto an autonomous loop.

## 1. Learning Objectives

- State a working definition of "agent" useful at design time.
- Map agent concepts (nodes, edges, state, interrupts, checkpoints) onto state-machine vocabulary.
- Choose between three agent shapes (single-agent ReAct, supervisor-worker, hard-interrupt) for a given problem.
- Distinguish a *workflow* from an *agent* and justify which shape a problem needs.

## 2. Introduction

The word "agent" has lost most of its design-time utility. The corrective: ground it in a structure engineers already know — a **state machine** whose transitions are decided by a language model. Design questions become familiar: what are the states, transitions, persistence points, and human-entry nodes?

This framing is not arbitrary — it is the model LangGraph exposes: graphs of nodes, conditional edges, persistent state via checkpointers, and explicit interrupts ([LangGraph Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api), retrieved 2026-05-26). Anthropic uses the same vocabulary: workflows vs agents that "dynamically direct their own processes" ([Anthropic](https://www.anthropic.com/research/building-effective-agents), retrieved 2026-05-26).

Two dividends: explicit design decisions (transitions, interrupts, checkpoints) and a home for HITL — a human checkpoint is a node, not a special-case feature.

## 3. Core Concepts

### 3.1 The working definition and five primitives

> **Agent** — a state machine whose transitions are LLM-decided (typically via tool calls), with persistent state across steps and explicit points where a human can resume the graph.

| Primitive | LangGraph term | What it is |
|---|---|---|
| State | `State` (TypedDict) | Data the graph mutates as it runs |
| Step | Node | A function `(state) → updates_to_state` |
| Transition | Edge (regular or conditional) | Rule deciding which node runs next |
| Pause | Interrupt | Execution stops until external input arrives |
| Persistence | Checkpointer | Where graph state is durably stored between steps |

The LLM appears as the brain of a node (`call_model(state)`) and as the chooser of a conditional edge (tool-call routing) ([LangGraph Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api), retrieved 2026-05-26).

### 3.2 Three agent shapes — pick the right one

**Single-agent ReAct (Tue)** — one agent, Reason + Act + Observe in a loop. The warm-up shape for intake-triage because there is one decision-maker and one tool kit ([ReAct paper](https://react-lm.github.io/), retrieved 2026-05-26).

**Supervisor-worker (Wed)** — supervisor delegates to N specialised workers, collects results, synthesises. Workers receive a *filtered view* of state — not the full shared scratchpad. Evaluator → consensus flow.

**Hard-interrupt state machine (Thu)** — explicit interrupt nodes at FAR-anchored gates, checkpointed across the 18-hour SSA gap. This is the shape for transitions that are genuinely irreversible.

> [!NOTE]
> **Operational implication:** The shape choice (ReAct / supervisor-worker / hard-interrupt) must land in today's HITL #3 ADR. Mon–Tue is single-agent ReAct (one decision-maker handles intake); supervisor shape waits until Wed when the evaluator topology is defined.

### 3.3 Workflow vs agent — the Anthropic test

- **Workflow** — LLM orchestrated against a pre-defined pattern. Cheaper, more predictable, easier to debug.
- **Agent** — LLM dynamically directs its own process. More flexible, harder to test and operate.

Anthropic's guidance: reach for the workflow shape unless dynamic direction is genuinely required. Most production "agents" are workflows with one or two dynamic decision points.

> [!TIP]
> **HITL #3 anchors here.** The Plan-Day ADR names which transitions are static `interrupt_before` (hard gates, compile-time-inspectable, FAR-anchored) and which are dynamic `interrupt()` (pause condition decided at runtime). Static is the cohort's canonical W3 wiring Mon→Thu; dynamic appears in the code examples as an alternative. Both are valid LangGraph v1.x primitives — the distinction matters for today's ADR.

## 4. Generic Implementation

```python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END

class TicketState(TypedDict):
    ticket_id: str
    text: str
    category: Literal["lookup", "refund", "escalate", "unknown"]
    draft_reply: str
    resolution: str | None

def classify_ticket(state: TicketState) -> dict:
    """LLM-decided categorisation — the only node where the LLM runs."""
    return {"category": llm.classify(state["text"])}

def lookup_answer(state: TicketState) -> dict:
    """Reversible: query KB, draft reply."""
    return {"draft_reply": kb.answer(state["text"]), "resolution": "auto-resolved"}

def prepare_refund(state: TicketState) -> dict:
    """Reversible: prepares refund object, does NOT externalise."""
    return {"draft_reply": refund_engine.prepare(state["ticket_id"])}

def issue_refund(state: TicketState) -> dict:
    """IRREVERSIBLE: money leaves the account. Static interrupt sits before this."""
    receipt = refund_engine.issue(state["ticket_id"])
    return {"resolution": f"refunded:{receipt.id}"}

builder = StateGraph(TicketState)
builder.add_node("classify", classify_ticket)
builder.add_node("lookup", lookup_answer)
builder.add_node("prepare_refund", prepare_refund)
builder.add_node("issue_refund", issue_refund)
builder.add_edge(START, "classify")
builder.add_conditional_edges("classify",
    lambda s: s["category"],
    {"lookup": "lookup", "refund": "prepare_refund",
     "escalate": END, "unknown": END})
builder.add_edge("prepare_refund", "issue_refund")
builder.add_edge("lookup", END)
builder.add_edge("issue_refund", END)

# Hard gate before the irreversible step — one line at compile time.
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["issue_refund"],
)
```

Every node is a small function over `TicketState`. The LLM appears only in `classify_ticket`. The HITL gate is one line at compile, at the reversible-prepare / irreversible-execute boundary.

## 5. Real-world Patterns

**E-commerce — Shopify-style fraud-review queues.** An order moves through `pending → risk-scored → (auto-clear | manual-review | auto-reject)`. Manual-review is an interrupt: graph pauses, analyst inspects, resumes with `Command(resume=...)`. A human-review queue is structurally a queue of paused agent threads.

**Healthcare — multidisciplinary tumour boards.** An oncology coordinator routes a case to radiology, pathology, and surgery, synthesises a treatment recommendation. Treatment authorisation is a hard human gate — supervisor-worker with a hard interrupt at the externalising boundary ([Anthropic](https://www.anthropic.com/research/building-effective-agents), retrieved 2026-05-26).

**Fintech — KYC onboarding spanning days.** Document upload Monday, sanctions screening Friday — textbook checkpointed state machine. `thread_id` = case ID; checkpoint history = regulatory audit trail.

> [!TIP]
> **Cross-domain lesson:** The "pause and resume" pattern maps to a durable checkpointer in every domain — KYC case file, tumour-board packet, paused fraud queue. If your pair's workflow spans more than one business day, `PostgresSaver` is not optional.

## 6. Best Practices

- Write the state shape (`TypedDict`) before any node — it forces the question of what data moves through the graph.
- Keep nodes small and pure-over-state; side effects belong at the externalising boundary.
- Route conditional edges via pure functions over state, not side-channel reads.
- Use durable `PostgresSaver` in production — the gap from `InMemorySaver` matters the first time you resume after a deploy.
- Name `thread_id` namespaced (`f"{tenant_id}:{case_id}"`) — it is the primary key of your audit trail.

> [!WARNING]
> **Anti-pattern: `agent-equals-function-calling`.** Most tutorials conflate "agent" with single-step tool use. A genuine agent pursues a goal autonomously across *multiple* steps with persistent state and the ability to change plan mid-run. An ADR that calls a single LLM tool-call an "agent" is vibes-based architecture. Name the distinction in your HITL #3 ADR: autonomous goal pursuit vs single-step tool invocation. See slug `agent-equals-function-calling`.

## 7. Hands-on Exercise

Design the state machine for a content-moderation pipeline. Incoming posts: auto-approved (low risk), auto-removed (clear violation), or sent to a human moderator (ambiguous). A small fraction also trigger a "permanent ban" branch.

Draw: (1) the `TypedDict` (5–7 fields), (2) node list (4–6 nodes), (3) edges including the conditional after the classifier, (4) position of `interrupt_before` markers, (5) `thread_id` shape.

> [!NOTE]
> **Self-check** (30s — answer mentally before expanding)
>
> 1. Why should the `permanent_ban` node use `interrupt_before` (static) rather than `interrupt()` (dynamic)?
> 2. The human-review node uses `interrupt()` (dynamic). Why is this the *right* choice here?

<details>
<summary>Show answers</summary>

1. `permanent_ban` is a named, compile-time-known gate anchored to an irreversible externalisation (identity service ban). Static `interrupt_before` makes it inspectable at build time — any reviewer can see exactly which nodes pause before executing. Dynamic interrupt is for runtime-decided pause conditions.
2. The routing to `human_review` depends on the classifier's output — a runtime value. The pause condition is itself dynamic (which posts reach human review depends on what the LLM decides about the content). `interrupt()` inside the node is the right primitive when the pause condition isn't known until the graph is running.

</details>

## 8. Key Takeaways

- An agent is a state machine with LLM-decided transitions, persistent state, and explicit human-resume points.
- Five primitives: state, node, edge, interrupt, checkpointer — map these before coding.
- Three shapes: ReAct (Tue), supervisor-worker (Wed), hard-interrupt (Thu) — shape choice lands in today's ADR.
- Most production "agents" are workflows with one or two dynamic decision points; default to workflow unless you need dynamic direction.
- HITL is a node-level concern — not a UI feature bolted on after.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://docs.langchain.com/oss/python/langgraph/graph-api — retrieved 2026-05-26 — hot-tech
- https://docs.langchain.com/oss/python/langgraph/interrupts — retrieved 2026-05-26 — hot-tech
- https://docs.langchain.com/oss/python/langgraph/persistence — retrieved 2026-05-26 — hot-tech
- https://www.anthropic.com/research/building-effective-agents — retrieved 2026-05-26 — foundation-stable
- https://react-lm.github.io/ — retrieved 2026-05-26 — foundation-stable

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

LangGraph exposes two interrupt primitives because the use cases are genuinely different:

**Static `interrupt_before=[node_name]`** compiles the pause into the graph definition. The checkpoint before the node is written; the graph halts; resume requires an external `Command(resume=...)`. The audit trail is implicit — every checkpoint is a record. Best for known, regulatory-anchored gates.

**Dynamic `interrupt(payload)`** is called inside a node body. The payload is surfaced to whoever is driving the graph (UI, API, human operator). Resume delivers the human's input as the return value of `interrupt()`. Best for runtime-conditional pauses ("only interrupt if proposed credit > $500").

The two primitives compose: a graph can have static gates on named nodes AND dynamic gates inside nodes for finer-grained conditions. The W3 canonical wiring uses static gates for the FAR-anchored transitions and introduces dynamic interrupt Thu for the SSA authority check.

Senior FDEs should also read: LangGraph docs on `Command(resume=..., update=...)` — the `update` field lets the human modify graph state as part of resuming, not just supply a decision. This matters for SSDD review flows where the SSA edits the evaluation narrative before approving.

</details>

Last verified: 2026-06-06
