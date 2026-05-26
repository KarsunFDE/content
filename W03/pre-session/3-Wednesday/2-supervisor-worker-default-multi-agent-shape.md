---
week: W03
day: Wed
topic_slug: supervisor-worker-default-multi-agent-shape
topic_title: "Supervisor-worker — the default multi-agent shape"
parent_overview: W03/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://docs.langchain.com/oss/python/langgraph/workflows-agents
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.anthropic.com/engineering/multi-agent-research-system
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://github.com/langchain-ai/langgraph-supervisor-py
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.databricks.com/blog/multi-agent-supervisor-architecture-orchestrating-enterprise-ai-scale
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://blog.bytebytego.com/p/how-anthropic-built-a-multi-agent
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# Supervisor-worker — the default multi-agent shape

## 1. Learning Objectives

By the end of this reading, the learner can:

- Describe the three roles inside a supervisor-worker multi-agent topology (supervisor, workers, optional terminator/synthesizer) and what each is allowed to read and write.
- Explain why the supervisor-worker pattern is the default starting shape for production multi-agent systems and when a single agent is still the right call.
- Articulate the *delegation contract* — the per-arrow agreement of inputs, outputs, and firing conditions between supervisor and worker.
- Identify the two common supervisor-worker failure modes (infinite delegation loops, unbounded prompt growth) and the standard mitigations.
- Recognize the supervisor-as-tool-caller pattern as the canonical LangGraph implementation in v1.0+.

## 2. Introduction

When a single agent's prompt grows past a useful size, when one agent ends up juggling tools that need wildly different reasoning styles, or when you cannot tell from the trace which capability regressed, you have outgrown a single-agent architecture. The industry's default next step is a **supervisor-worker** topology: one coordinating agent (the supervisor) that decides which specialized worker to invoke next, and a small set of workers that each do one narrow thing well.

The pattern is not new — it is the same hierarchical-task-network shape that classical AI planning, distributed systems supervisors (Erlang/OTP), and microservice orchestration have used for decades. What changed in 2025–2026 is that the *supervisor* is now an LLM whose policy is expressed in a prompt rather than a state machine, and the *workers* are agents with their own context windows and tools. This combination — symbolic delegation plus learned reasoning at each level — is what powers production systems like Anthropic's Research, Klarna's customer-service triage, and Uber's internal AI tooling [1][2].

What you give up to move from one agent to a team of agents is real: total latency rises (one extra LLM hop per delegation step), token spend climbs (each worker pays for its own prompt and reasoning), and you gain a new failure surface (the supervisor's routing policy itself). What you get is *separability* — each worker can be evaluated, prompted, and iterated on in isolation; regressions can be pinpointed to one specialist; new capabilities can be added by appending a worker rather than rewriting the whole prompt.

The Wednesday day-arc uses this pattern to scaffold a real evaluation flow, but the shape applies almost everywhere multi-agent systems are deployed. This reading covers the generic architecture; the daily overview handles the Karsun-specific framing.

## 3. Core Concepts

### 3.1 The three roles

A supervisor-worker topology has at minimum two roles, often three:

- **Supervisor** — A node whose only job is to choose the next worker (or to declare the task done). It reads global state, reasons about which capability is needed next, and emits a delegation message. It does **not** execute domain logic itself. In LangGraph, this is a node whose function calls an LLM with a routing prompt and writes a `next_worker` field on the graph state [3].
- **Worker** — A node specialized for one task. Workers read a scoped subset of state (per the delegation contract), call tools, and write results back. Workers do not call each other; all coordination flows through the supervisor [4].
- **Terminator / synthesizer** *(optional)* — A final node that turns the workers' accumulated outputs into the response the caller actually wanted. Sometimes this is collapsed into the supervisor's last call; in systems with non-trivial synthesis (Anthropic's Research compiler, for example), it is its own node [2].

### 3.2 The delegation contract

The single most important primitive is the *delegation contract* — the agreement on what crosses the arrow from supervisor to worker. A complete contract names four things:

1. **Inputs** — the slice of state the worker sees. (Filtered, summarized, or raw.)
2. **Outputs** — the state keys the worker is allowed to write.
3. **Firing condition** — when the supervisor decides to invoke this worker (encoded in the supervisor's routing prompt).
4. **Completion signal** — how the supervisor knows the worker is done and what value to read.

When contracts are explicit, multi-agent systems are tractable to debug. When they are implicit, the trace becomes a guessing game.

### 3.3 The supervisor-as-tool-caller pattern

In LangGraph v1.0+, the canonical implementation expresses worker invocation as **tool calls**: each worker is registered as a tool the supervisor can call. The supervisor's LLM emits a tool call, a `ToolNode` (or `create_handoff_tool` helper) translates it into a transition to the corresponding worker node, and the worker's return value lands in shared state [3]. This reuses the model's existing tool-calling capability — there is no new abstraction to learn — and it makes the supervisor's reasoning visible (every delegation is a logged tool call).

```text
                 ┌────────────────────┐
   user input ──▶│    supervisor      │
                 │ (LLM + routing     │
                 │  prompt; emits     │
                 │  tool calls)       │
                 └─────────┬──────────┘
                           │  tool_call("worker_a", inputs=...)
                ┌──────────┼──────────┐
                ▼          ▼          ▼
          ┌────────┐ ┌────────┐ ┌────────┐
          │worker A│ │worker B│ │worker C│
          └───┬────┘ └───┬────┘ └───┬────┘
              │          │          │
              └──────────┼──────────┘
                         │  results written back to shared state
                         ▼
                ┌────────────────────┐
                │    supervisor      │ (next iteration: delegate or terminate)
                └────────────────────┘
```

### 3.4 When the pattern fits — and when it doesn't

The Databricks production-architecture guidance is direct: *use supervisor only when a single agent is no longer sufficient* [4]. The trigger conditions are roughly:

- Your agent has more than ~10 tools and the routing prompt is getting unwieldy.
- Different tasks need different system prompts or different reasoning depths.
- Evaluations cannot pinpoint which capability regressed because everything lives in one trace.

If a single agent is hitting your eval target and the workload is homogeneous, the supervisor pattern adds latency (~1.5–2× wall clock) and cost (~2–3× tokens) for no benefit [1].

### 3.5 Two common failure modes

- **Infinite delegation loops.** The supervisor keeps routing to the same worker because the worker's output doesn't satisfy its completion condition. Standard mitigation: a hard iteration cap (`max_iterations` on graph compile) plus a "good enough" threshold in the routing prompt so the supervisor terminates on near-misses rather than chasing perfection [3].
- **Unbounded prompt growth.** Each worker's output accumulates in the supervisor's context until the routing call blows past the context window. Standard mitigation: a *curation step* — either summarize worker outputs before they re-enter the supervisor's prompt, or store full outputs in a shared memory keyed by worker ID and surface only handles to the supervisor [2].

## 4. Generic Implementation

A minimal supervisor-worker graph using LangGraph v1.0 primitives (no `Chain` class, no LCEL pipe foundation — plain Python function composition):

```python
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command

# Shared state — every node reads and writes here
class FlowState(TypedDict):
    request: str
    notes: list[str]          # append-only; workers add findings
    next_worker: str | None   # supervisor sets this
    final: str | None         # terminator writes this

# --- Workers: each is a plain function over state ---
def worker_search(state: FlowState) -> Command:
    # narrow task: do one thing and return
    finding = do_search(state["request"])  # imagine a tool call here
    return Command(update={"notes": state["notes"] + [f"search: {finding}"]})

def worker_analyze(state: FlowState) -> Command:
    summary = analyze(state["notes"])
    return Command(update={"notes": state["notes"] + [f"analysis: {summary}"]})

# --- Supervisor: LLM picks next worker or terminates ---
def supervisor(state: FlowState) -> Command:
    # Call the LLM with a routing prompt; it returns one of:
    # "search", "analyze", or "done".
    decision = call_router_llm(state)        # returns a literal
    if decision == "done":
        return Command(update={"next_worker": None}, goto="terminator")
    return Command(update={"next_worker": decision}, goto=decision)

def terminator(state: FlowState) -> Command:
    # Synthesize the final answer from accumulated notes
    answer = synthesize(state["notes"])
    return Command(update={"final": answer}, goto=END)

# --- Wire the graph ---
graph = StateGraph(FlowState)
graph.add_node("supervisor", supervisor)
graph.add_node("search", worker_search)
graph.add_node("analyze", worker_analyze)
graph.add_node("terminator", terminator)

graph.add_edge(START, "supervisor")
# Every worker returns to the supervisor for the next decision
graph.add_edge("search", "supervisor")
graph.add_edge("analyze", "supervisor")

app = graph.compile()
```

Each piece is doing one thing: the supervisor only routes, the workers only execute one task, the terminator only synthesizes. The delegation contract is *visible* — `worker_search` reads `state["request"]`, writes `state["notes"]`, fires when the supervisor returns `"search"`. No worker calls another worker directly.

## 5. Real-world Patterns

**Fintech — fraud-triage supervisor (Stripe-style).** Production payment-fraud systems often use a supervisor-worker shape where a triage supervisor routes incoming transactions among a velocity-rules worker, a graph-features worker (looking at counterparty relationships), and an LLM-explanation worker that produces the analyst-facing rationale. Workers cannot see each other's outputs until the supervisor synthesizes them, which prevents one signal from biasing another. Production benchmarks reported by orchestration-pattern surveys show 60–70% reductions in analyst review time when workers are well-scoped [5].

**Healthcare — clinical decision support.** Patient-summary systems built on Anthropic's orchestrator-worker blueprint use a lead agent that spawns parallel workers for (a) labs review, (b) imaging review, (c) prior-history review, with a final synthesizer that produces the unified summary [2]. Each worker has its own context window and tool access; the lead reconciles their findings with explicit citations. This is the same pattern Anthropic published for their internal Research product and reported a 90.2% improvement over single-agent Claude on their internal eval [2].

**Customer-service triage (Klarna).** Klarna's production deployment uses one supervisor that routes among refund, account-issue, and shipping-status workers. Each worker is single-purpose with a small tool set; the supervisor's prompt encodes the routing policy and the escalation rules to a human. The reported pattern is that adding a new capability (say, gift-card support) is just adding a worker and one routing example, not retraining the whole prompt [1].

**E-commerce — product-catalog Q&A.** Mid-2026 production case studies from the LangGraph community describe catalog-search systems where a supervisor routes among an inventory-lookup worker, a recommendations worker, and a comparison-builder worker, with the synthesizer producing the final shopper-facing answer. The reported win is *evaluability*: when comparisons started regressing on a model upgrade, the team could pinpoint the comparison worker's prompt without touching the rest of the graph [3].

## 6. Best Practices

- **Start with a single agent. Move to supervisor-worker only when your eval pinpoints a routing or specialization need** — don't pre-emptively split.
- **Make the delegation contract explicit in code.** Every worker declares the state keys it reads and writes; surface this in a docstring or type alias rather than leaving it to the supervisor's prompt to enforce.
- **Always set `max_iterations` (or equivalent) on the compiled graph.** Infinite delegation loops are the #1 failure mode in production [3].
- **Keep workers stateless beyond their input contract.** A worker that maintains its own memory across invocations becomes another distributed-systems problem; let the graph state own all durable memory.
- **Curate worker outputs before they re-enter the supervisor's context.** Either summarize, or store full outputs externally and pass handles only.
- **Log every delegation as a structured event.** Tool-call-shaped invocations make this free; if you implement the supervisor with literal branching, emit a structured event manually.
- **Resist hierarchies until they earn their place.** A supervisor-of-supervisors is justified when you have 30+ workers or distinct organizational boundaries; below that, a flat supervisor is simpler and faster.

## 7. Hands-on Exercise

**Architecture-drawing prompt (20 min).** You are designing an *interview-preparation assistant* for a coding-interview platform (think a leetcode-meets-tutor product, no federal-acquisitions overlap). The product needs to: (a) pick a problem matched to the user's skill level, (b) provide a hint if the user is stuck, (c) review the user's submitted solution for correctness and style, and (d) suggest a follow-up problem.

On a whiteboard or in a markdown file, draw the supervisor-worker topology. Your diagram must label:

- The supervisor's role and the input it receives.
- Each worker, with one-sentence scope.
- The delegation contract for each supervisor → worker arrow (inputs, outputs, firing condition).
- Where the synthesizer (if any) sits.
- The state schema (which keys exist, who writes to each).
- One named failure mode you've designed against (e.g., infinite hint loop) and how the topology mitigates it.

**What good looks like:** Four workers (problem-picker, hint-giver, reviewer, next-problem-suggester) with clearly disjoint scopes. The supervisor's routing logic is expressible in three bullet points. The delegation contracts are explicit — no worker reads state it shouldn't. A `max_hints` counter on state caps the hint-giver loop. The reviewer's output is structured (e.g., `{"correct": bool, "style_notes": [...]}`) so the supervisor doesn't need to re-parse natural language to decide the next step.

## 8. Key Takeaways

- *What are the three roles in a supervisor-worker topology, and what is each allowed to do?* (Supervisor routes, workers execute one narrow task, optional synthesizer composes the final answer.)
- *Why is the supervisor-as-tool-caller pattern the canonical LangGraph v1.0+ implementation?* (Reuses model tool-calling; makes delegation visible as logged events; no new abstraction.)
- *What is the delegation contract and why does it matter for debuggability?* (Per-arrow agreement on inputs/outputs/firing/completion; without it, the trace is a guessing game.)
- *What are the two most common supervisor-worker failure modes and the standard mitigations?* (Infinite delegation loops → max-iterations + good-enough threshold; unbounded prompt growth → curation step or external memory.)
- *When should you NOT move from a single agent to supervisor-worker?* (When a single agent is hitting eval targets on a homogeneous workload — the latency and cost overhead is not earned.)

## Sources

1. [LangGraph Supervisor + Deep Agents: Production Guide (buildmvpfast)](https://www.buildmvpfast.com/blog/langgraph-supervisor-deep-agents-multi-agent-patterns-2026) — retrieved 2026-05-26
2. [How we built our multi-agent research system (Anthropic Engineering)](https://www.anthropic.com/engineering/multi-agent-research-system) — retrieved 2026-05-26
3. [Workflows and agents — LangChain/LangGraph official docs](https://docs.langchain.com/oss/python/langgraph/workflows-agents) — retrieved 2026-05-26
4. [Supervisor Agent Architecture: Orchestrating Enterprise AI at Scale (Databricks Blog)](https://www.databricks.com/blog/multi-agent-supervisor-architecture-orchestrating-enterprise-ai-scale) — retrieved 2026-05-26
5. [How Anthropic Built a Multi-Agent Research System (ByteByteGo)](https://blog.bytebytego.com/p/how-anthropic-built-a-multi-agent) — retrieved 2026-05-26

Last verified: 2026-05-26
