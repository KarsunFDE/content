---
week: W03
day: Mon
topic_slug: agents-are-state-machines-with-hitl-nodes
topic_title: "Agents are state machines with HITL nodes"
parent_overview: W03/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 13
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
last_verified: 2026-05-26
---

# Agents are state machines with HITL nodes

## 1. Learning Objectives

By the end of this reading, the learner can:

- State a working definition of "agent" that is useful at design time, not just at marketing time.
- Map agent concepts (nodes, edges, state, interrupts, checkpoints) onto familiar state-machine vocabulary.
- Choose between three common agent shapes (single-agent ReAct, supervisor-worker, hard-interrupt state machine) for a given problem.
- Distinguish a *workflow* from an *agent* and justify which shape a given problem needs.
- Explain why human-in-the-loop (HITL) is a node-level concern in this model, not a UI-level afterthought.

## 2. Introduction

The word "agent" has been used so loosely in the past two years that it has lost most of its design-time utility. To get it back, the most useful move is to ground the concept in a structure engineers already know how to reason about: a **state machine** whose transitions are decided by a language model. Once you see an agent that way, the design questions become familiar — what are the states, what are the transitions, where is the state persisted, where can a human enter the loop, and what happens when the process is paused and resumed days later.

This framing is not arbitrary. It is exactly the model the dominant orchestration framework (LangGraph) chose to expose to developers: graphs of nodes, conditional edges, persistent state via checkpointers, and explicit interrupts that pause and resume execution ([LangGraph Graph API overview](https://docs.langchain.com/oss/python/langgraph/graph-api), retrieved 2026-05-26). It is also consistent with the framing Anthropic uses in *Building Effective Agents*, which distinguishes deterministic *workflows* from *agents* that "dynamically direct their own processes and tool usage" ([Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents), retrieved 2026-05-26).

Treating an agent as a state machine pays two compounding dividends. First, it forces explicit design decisions (which transitions, which interrupts, which checkpoints) where the alternative is implicit autonomy. Second, it gives HITL a *home*: a human checkpoint is not a special-case UI feature bolted onto a black box — it is a node in the graph, with state before and state after, just like any other node.

## 3. Core Concepts

### 3.1 The working definition

> **Agent** — a state machine whose transitions are LLM-decided (typically via tool calls), with persistent state across steps and explicit points where a human can resume the graph.

Three things are doing work in that definition:

- **State machine** — finite, inspectable, replayable.
- **LLM-decided transitions** — the *choice* of transition comes from a model call; the transition itself and its side effects are normal code.
- **Persistent state + resumable interrupts** — the process can pause and resume from the same point days later. This is what separates an agent from a long-running script.

### 3.2 The five primitives

Almost every agent framework exposes the same five primitives under different names:

| Primitive | LangGraph term | What it is |
|---|---|---|
| State | `State` (TypedDict) | The data the graph mutates as it runs |
| Step | Node | A function `(state) -> updates_to_state` |
| Transition | Edge (regular or conditional) | The rule that decides which node runs next |
| Pause | Interrupt | A point where execution stops until external input arrives |
| Persistence | Checkpointer | Where the graph state is durably stored between steps |

The LLM enters in two places: as the brain of a node (`call_model(state)` is just a node like any other) and as the chooser of a transition (a conditional edge that routes based on the model's tool-call output) ([LangGraph Graph API overview](https://docs.langchain.com/oss/python/langgraph/graph-api), retrieved 2026-05-26).

### 3.3 ReAct — the canonical single-agent shape

ReAct (Reason + Act + Observe) is the simplest non-trivial agent: a loop where the model reasons about the next step, calls a tool, observes the result, and decides whether to continue or stop ([ReAct paper page](https://react-lm.github.io/), retrieved 2026-05-26). As a state machine:

```
[start] -> call_model -> (decide) -> call_tool -> observe -> call_model -> ... -> [end]
```

The conditional edge after `call_model` inspects the model's output: if it produced a tool call, route to `call_tool`; if it produced a final answer, route to `[end]`. The loop terminates when the model says "I'm done" or when a budget (steps, tokens, time) is exhausted. This is the warm-up shape for intake-style triage workflows because it has *one* decision-maker and *one* set of tools.

### 3.4 Supervisor-worker — the canonical multi-agent shape

When the workload decomposes naturally into sub-tasks with different specialisations, the supervisor-worker pattern wraps a supervisor agent around N worker agents. The supervisor sees the user's intent, decides which worker(s) to invoke, collects results, and synthesises an answer. Each worker is itself a state machine — often a ReAct loop with a specialised tool kit.

```
                       [supervisor]
                      /     |       \
              [worker A] [worker B] [worker C]
                      \     |       /
                       [synthesise]
```

The supervisor *does not* let workers see each other's scratchpads — that is the dominant anti-pattern. Workers receive a filtered view of state the supervisor has prepared for them, and they return their results to the supervisor, who decides what to share next ([LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence), retrieved 2026-05-26).

### 3.5 Hard-interrupt state machine — the HITL-anchored shape

When some transitions are irreversible (publish to a public registry, mutate a system of record, externalise a notification), the agent's state machine grows explicit interrupt nodes that pause execution until a human resumes the graph. Two implementations of the same idea exist in the current LangGraph surface area:

**Static interrupt — the cohort's canonical W3 wiring (Mon→Thu):**

```python
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["release_step"],
)
```

**Dynamic interrupt — alternative when the pause condition itself is runtime-decided:**

```python
from langgraph.types import interrupt, Command

def review_node(state: State) -> dict:
    # Pause the graph; surface a payload to whatever is driving the run.
    decision = interrupt({"proposal": state["draft"]})
    # Resume only when the driver supplies a Command(resume=...) value.
    return {"draft": decision["edited_draft"], "approver": decision["actor"]}
```

> [!instructor-review]
> **Static `interrupt_before=[...]` vs dynamic `interrupt()` — canonical story for W3.**
> Both primitives are valid in LangGraph v1.x. The programme's canonical wiring Mon→Tue→Wed→Thu is **static `interrupt_before`** for the named, FAR-anchored gate boundaries: simpler, inspectable at compile time, what the war-room walkthroughs ship against, and what production codebases everywhere still use. The LangGraph docs (retrieved 2026-05-26) describe **dynamic `interrupt()`** as the docs-canonical alternative when the *pause condition itself* is runtime-decided (e.g., "interrupt only when proposed refund > $500"). Wed's soft-interrupt deep-dive (file 5) shows static at §3.2 and dynamic at §4 with the contrast spelled out. Thu's hard-interrupt deep-dive (file 6) lands on static for the FAR 15.308 + FAR 5.705 gates. Surface this nuance to learners — do not let them think the curriculum is stale.

### 3.6 Checkpointers turn the graph into a resumable process

The checkpointer persists the graph state at every step. In development that's an in-memory store; in production it's typically Postgres-backed. The two consequences that matter for design:

- A graph that paused at an interrupt yesterday can resume today, against the same `thread_id`, with no state loss.
- The full history of a thread is inspectable via APIs like `graph.get_state_history(thread_id)`, which is the audit-replay surface for any workflow that crosses a regulatory boundary ([LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence), retrieved 2026-05-26).

### 3.7 Workflow vs agent — and why "agent" is not always the right answer

Anthropic's *Building Effective Agents* offers a useful working distinction:

- **Workflow** — an LLM is orchestrated against a *pre-defined* pattern (sequence, parallel, conditional branch).
- **Agent** — the LLM *dynamically* directs its own process: it decides what to do next based on what it has seen so far.

Workflows are cheaper, more predictable, easier to debug. Agents are more flexible but harder to test and operate. Anthropic's guidance: reach for the workflow shape unless dynamic direction is genuinely required ([Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents), retrieved 2026-05-26). Most production "agents" are workflows with one or two genuinely dynamic decision points.

## 4. Generic Implementation

A worked example outside the federal-acquisitions domain: a logistics dispatcher that triages incoming customer-support tickets for a regional courier service. Tickets either resolve themselves (lookup), require a human dispatcher (irreversible refund), or escalate to a partner carrier.

```python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END

class TicketState(TypedDict):
    ticket_id: str
    text: str
    category: Literal["lookup", "refund", "escalate", "unknown"]
    draft_reply: str
    approver_id: str | None
    resolution: str | None

def classify_ticket(state: TicketState) -> dict:
    """LLM-decided categorisation. Returns updates to merge into state."""
    category = llm.classify(state["text"])  # the LLM call
    return {"category": category}

def lookup_answer(state: TicketState) -> dict:
    """Reversible: query knowledge base, draft a reply."""
    draft = kb.answer(state["text"])
    return {"draft_reply": draft, "resolution": "auto-resolved"}

def prepare_refund(state: TicketState) -> dict:
    """Reversible up to here: prepares the refund object but does NOT issue it."""
    draft = refund_engine.prepare(state["ticket_id"])
    return {"draft_reply": draft}

def issue_refund(state: TicketState) -> dict:
    """IRREVERSIBLE: money leaves the account.
    Static interrupt sits *before* this node — see compile() below.
    """
    receipt = refund_engine.issue(state["ticket_id"])
    return {"resolution": f"refunded:{receipt.id}"}

def route_after_classification(state: TicketState) -> str:
    """Conditional edge — pure function over state."""
    return state["category"]  # "lookup" | "refund" | "escalate" | "unknown"

builder = StateGraph(TicketState)
builder.add_node("classify", classify_ticket)
builder.add_node("lookup", lookup_answer)
builder.add_node("prepare_refund", prepare_refund)
builder.add_node("issue_refund", issue_refund)
builder.add_node("escalate", lambda s: {"resolution": "escalated-to-carrier"})

builder.add_edge(START, "classify")
builder.add_conditional_edges("classify", route_after_classification, {
    "lookup": "lookup",
    "refund": "prepare_refund",
    "escalate": "escalate",
    "unknown": "escalate",
})
builder.add_edge("prepare_refund", "issue_refund")
builder.add_edge("lookup", END)
builder.add_edge("issue_refund", END)
builder.add_edge("escalate", END)

# Hard gate before the irreversible step.
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["issue_refund"],
)
```

Notice the three properties of the state-machine framing: every node is a small pure-ish function over `TicketState`; the LLM appears only inside `classify_ticket` (and only the choice of category — not the routing rule itself); the HITL gate is a single line at the compile step, sitting on the boundary between reversible (prepare) and irreversible (issue).

## 5. Real-world Patterns

**E-commerce — Shopify-style fraud-review queues.** Modern fraud-detection pipelines are state machines: an order moves through `pending` → `risk-scored` → (`auto-clear` | `manual-review` | `auto-reject`). The "manual review" branch is an interrupt: the graph pauses, an analyst inspects, and resumes with `Command(resume="clear")` or `Command(resume="reject")`. A queue of human-review work is structurally identical to a queue of paused agent threads.

**Healthcare — clinical-decision-support tumour boards.** Multi-disciplinary tumour boards model exactly the supervisor-worker pattern: an oncology coordinator (supervisor) routes a case to radiology, pathology, and surgery (workers), collects their assessments, and synthesises a treatment recommendation that returns to the patient's care team. Modern decision-support software increasingly mirrors this topology, with an LLM supervisor proposing a draft synthesis the human board reviews — but the irreversible "treatment authorisation" step remains a hard human gate ([Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents), retrieved 2026-05-26, on the supervisor-worker pattern).

**Gaming — matchmaking and report-review pipelines.** Competitive-gaming platforms run two distinct state machines at scale: a matchmaking workflow (deterministic, no LLM, no human-in-the-loop) and a player-report-review pipeline (LLM-classified, supervisor-worker with a human moderator on the irreversible "permanent ban" branch). The split mirrors the workflow-vs-agent distinction: matchmaking is a workflow because the pattern is pre-defined; report review is an agent because the LLM's tool-call decisions depend on the specific report content.

**Fintech — KYC verification with checkpointed sessions.** Onboarding pipelines that span days (document upload Monday, address verification Wednesday, sanctions screening Friday) are state-machine-with-checkpointer in every dimension. The checkpointer survives the multi-day pause; resume happens when the user clicks the email link. Production KYC platforms treat the `thread_id` as the case ID, and the checkpoint history *is* the regulatory audit trail.

## 6. Best Practices

- Write the state shape (the `TypedDict`) before writing any node — the shape forces the question of what data the graph actually mutates.
- Keep nodes small and pure-over-state — side effects belong at the externalising boundary, not scattered across every node.
- Route conditional edges by *pure functions over state*, not by side-channel reads (a sleeping anti-pattern is to read a database inside an edge function).
- Persist with a real checkpointer in production (`PostgresSaver` or equivalent), not the in-memory dev saver — the difference matters the first time you need to resume after a deploy.
- Use static `interrupt_before` for clarity at named gate boundaries; use dynamic `interrupt()` when the *condition* for pausing is itself dynamic.
- Name the `thread_id` deliberately and namespaced (e.g., `f"{tenant_id}:{case_id}"`) — it is the primary key of your audit trail.
- Apply the workflow-vs-agent test before reaching for an agent: if the decision graph is fixed, write a workflow; reserve agents for genuinely dynamic decision-making.

## 7. Hands-on Exercise

**Whiteboarding prompt (12–15 minutes).** Design the state machine for a content-moderation pipeline at an online forum. Incoming posts can be: auto-approved (low risk), auto-removed (clear policy violation), or sent to a human moderator (ambiguous). A small fraction of posts also trigger a "permanent ban" branch that is itself moderated.

Draw:

1. The `TypedDict` for the graph state (5–7 fields).
2. The node list (4–6 nodes).
3. The edges, including the conditional edge after the classifier.
4. The position of any `interrupt_before` markers.
5. The `thread_id` shape — what gets concatenated, and why.

Expected components:

- A `classify_post` node that calls the LLM and returns `{risk_score, category}`.
- A conditional edge after `classify_post` that routes to `auto_approve`, `auto_remove`, or `human_review` based on the category.
- A `human_review` node that uses `interrupt()` to pause; resume payload contains the moderator's decision.
- A `permanent_ban` node sitting behind an `interrupt_before` static gate (irreversible action — externalises to identity service).
- A `thread_id` of the form `f"{forum_id}:{post_id}"` so the audit trail is multi-tenant safe.

**What good looks like.** Your state shape contains only data that actually moves through the graph — no transient fields, no UI-only flags. There is at most one hard gate (the permanent-ban interrupt); the human-review interrupt is dynamic because the routing depends on classifier output the LLM produced. The `thread_id` is namespaced by tenant. The total node count is four to six — if you have ten, you have over-decomposed.

## 8. Key Takeaways

- Can you state the working definition of "agent" as a state machine with LLM-decided transitions, persistent state, and explicit human-resume points — and explain what each phrase rules out?
- Can you map nodes, edges, state, interrupts, and checkpointers onto your favourite agent framework's vocabulary?
- Can you choose between single-agent ReAct, supervisor-worker, and hard-interrupt state machine for a given problem and defend the choice in one paragraph?
- Can you apply the Anthropic workflow-vs-agent test and explain why most production "agents" are mostly workflows?
- Can you place an interrupt at the right boundary — between the reversible-prepare step and the irreversible-execute step — instead of at the start of the flow?

## Sources

1. [LangGraph Graph API overview — Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/graph-api) — retrieved 2026-05-26
2. [LangGraph Interrupts — Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/interrupts) — retrieved 2026-05-26
3. [LangGraph Persistence — Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/persistence) — retrieved 2026-05-26
4. [Building Effective AI Agents — Anthropic](https://www.anthropic.com/research/building-effective-agents) — retrieved 2026-05-26
5. [ReAct: Synergizing Reasoning and Acting in Language Models](https://react-lm.github.io/) — retrieved 2026-05-26

Last verified: 2026-05-26
