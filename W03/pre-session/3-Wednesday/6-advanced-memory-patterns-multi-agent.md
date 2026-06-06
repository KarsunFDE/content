---
week: W03
day: Wed
topic_slug: advanced-memory-patterns-multi-agent
topic_title: "Advanced memory patterns for multi-agent"
parent_overview: W03/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://docs.langchain.com/oss/python/concepts/memory
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.vellum.ai/blog/multi-agent-systems-building-with-context-engineering
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://mem0.ai/blog/multi-agent-memory-systems
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.augmentcode.com/guides/cross-agent-organizational-memory
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://medium.com/mongodb/why-multi-agent-systems-need-memory-engineering-153a81f8d5be
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-06-06
---

# Advanced memory patterns for multi-agent

> [!NOTE]
> **From earlier:** Mon covered single-agent memory (message history, summarization, retrieval). Today adds the second dimension: not just *how much* memory, but *whose* — and what happens when agents share it.

## 1. Learning Objectives

- Distinguish three multi-agent memory shapes — shared scratchpad, per-agent thread, supervisor-curated context — and name the workload each fits.
- Recognise memory-contamination and cross-agent-trust failure modes and apply standard mitigations.
- Choose between thread-scoped (short-term) and store-scoped (long-term) memory based on durability requirements.
- Implement per-agent thread isolation using LangGraph's checkpointer + thread_id model.

## 2. Introduction

Multi-agent memory adds a dimension single-agent memory doesn't have: not just *how much* to keep, but *whose* memory is visible to whom. Each worker needs *some* context — but exactly what, written by whom, readable by whom, and persisted in what scope is a design space with real failure modes.

Pick wrong and you get cross-agent contamination, context collapse, or trust failures. The 2026 production literature converges on three default shapes. Pick deliberately, with a documented rationale per state key.

## 3. Core Concepts

### 3.1 Three memory shapes

| Shape | Fits when | Key failure mode | Mitigation |
|-------|-----------|-----------------|------------|
| **Shared scratchpad** | Workers collaborating on one artifact | Trust contagion — early hallucination becomes "fact" | Source-tag every write (`actor`, `confidence`) |
| **Per-agent thread** | Workers must reason independently (evaluators, parallel scorers) | Reconciliation overhead at synthesis | Synthesizer as first-class node |
| **Supervisor-curated context** | Workers have heterogeneous info needs; global state too large | Compression bias; extra LLM hop | Typed schema per worker; prefer deterministic templates |

Most production systems use all three for different state keys. The discipline is making the choice explicit, not picking one globally.

### 3.2 Per-agent thread in LangGraph

Each agent gets its own `thread_id` — the agent's conversation persists in the checkpointer keyed by that thread, invisible to other agents. Workers reasoning *independently* (consensus-style scoring, parallel reviewers) should use this shape.

> [!IMPORTANT]
> **Mint thread IDs per logical task, not globally per agent.** A global per-agent `thread_id` accumulates cross-request drift across user sessions. A per-task thread starts clean every time.

### 3.3 Short-term vs long-term scope

| Scope | Lives in | Persists | Use for |
|-------|---------|---------|---------|
| Thread-scoped (short-term) | Graph state / checkpointer | One execution only | In-flight work, current-run reasoning |
| Store-scoped (long-term) | Separate memory store | Across executions | User preferences, org knowledge, learned facts |

The choice is about *durability semantics*, not capacity. Things the agent needs on the next run go in the long-term store; things relevant only to the current run stay in graph state.

### 3.4 Cross-agent trust and contamination

Shared memory creates implicit trust: Agent B reads Agent A's finding with no built-in reason to weight it differently from a system instruction — yet A's finding may be hallucinated. Standard mitigations:

- **Source tagging.** Every write carries `actor`, `confidence`, `derived_from` — downstream readers weigh provenance.
- **Namespacing.** Partition by `user_id / agent_id / run_id` — cross-tenant contamination structurally impossible.
- **Validation gates.** Before a memory becomes load-bearing downstream, a gate verifies it.

> [!TIP]
> For the Wed–Fri evaluator flow: use **per-agent thread for evaluator-agents** (isolated scoring — they shouldn't see each other's rationale until consensus) and **supervisor-curated context for the consensus-agent** (the supervisor passes panel results with framing). Defend the pick today; it's in your morning scaffolding.

## 4. Generic Implementation

Per-agent-thread isolation for parallel independent scorers (job-application portfolio review — not federal-acquisitions):

```python
import uuid
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send, Command
from langgraph.checkpoint.postgres import PostgresSaver
import operator

class PortfolioState(TypedDict):
    portfolio: dict
    rubric: dict
    scores: Annotated[list[dict], operator.add]   # reducer — parallel writes safe
    final_decision: str | None

def scorer(input_state: dict):
    # Each scorer runs in its own thread — no cross-scorer visibility
    score = independent_review(
        portfolio=input_state["portfolio"],
        rubric=input_state["rubric"],
        reviewer_role=input_state["reviewer_role"],
    )
    return {"scores": [{"reviewer": input_state["reviewer_role"], "score": score,
                        "thread_id": input_state["thread_id"]}]}

def supervisor(state: PortfolioState):
    if not state["scores"]:
        return [
            Send("scorer", {
                "portfolio": state["portfolio"],
                "rubric": state["rubric"],
                "reviewer_role": role,
                "thread_id": str(uuid.uuid4()),   # per-task thread ID
            })
            for role in ["tech_reviewer", "design_reviewer", "comms_reviewer"]
        ]
    return Command(goto="synthesizer")

def synthesizer(state: PortfolioState):
    decision = aggregate_scores(state["scores"])
    return Command(update={"final_decision": decision}, goto=END)

graph = StateGraph(PortfolioState)
graph.add_node("supervisor", supervisor)
graph.add_node("scorer", scorer)
graph.add_node("synthesizer", synthesizer)
graph.add_edge(START, "supervisor")
graph.add_edge("scorer", "supervisor")

checkpointer = PostgresSaver.from_conn_string(POSTGRES_URL)
app = graph.compile(checkpointer=checkpointer)
```

Three scorers run in parallel via Send; each gets its own `thread_id` minted per task — no cross-scorer drift. `scores` uses `operator.add` for safe parallel writes. Synthesizer is the only place threads' outputs combine.

## 5. Real-world Patterns

**E-commerce — per-customer memory namespacing.** A marketplace assistant namespaces memory as `user_id:{cid}/agent_id:{aid}/run_id:{rid}`. Browsing, recommendation, and cart agents share within a customer namespace but cannot cross-read into other customers' memories.

**Healthcare — independent radiology reviewers.** Each modality reviewer (CT, MRI, X-ray) reasons in its own thread, preventing confirmation bias. The aggregator surfaces all three threads to the radiologist rather than reconciling into a single number.

**Fintech — supervisor-curated fraud analysis.** The lead agent holds the full transaction history; each worker (velocity-rules, graph-features, behavioral-anomalies) receives a curated, deterministically-templated subset — prompts stay stable and short.

## 6. Best Practices

- **Pick the memory shape per state key, with a documented rationale.** "Shared scratchpad because collaboration; per-thread because independence; curated because heterogeneous needs."
- **Always namespace long-term memory.** `user_id / agent_id / run_id` minimum — cross-tenant contamination without namespacing is easy.
- **Source-tag shared writes.** `actor`, `confidence`, `derived_from`, `timestamp` — downstream readers can then weigh provenance.
- **Use database-backed checkpointers in production.** In-memory loses state on restart, hostile to soft interrupts and long-running flows.

> [!WARNING]
> **Anti-pattern: `cross-agent-memory-leak`.** When agents share a scratchpad without source tagging or access controls, a hallucinated or biased finding by one agent becomes authoritative "context" for all subsequent agents. This is the contamination failure mode. In fed-acq evaluation flows, a poisoned scratchpad could propagate a fabricated CPAR rating or vendor score through the entire consensus pipeline undetected. Source-tag every write; add a validation gate before any scratchpad entry becomes load-bearing.

## 7. Hands-on Exercise

You are designing a **multi-agent novel-editing system** (not federal-acquisitions). Four agents: *plot agent* (inconsistencies), *character agent* (voice/consistency), *prose agent* (sentence edits), *consolidator* (final report).

For each state key below, pick a memory shape and write a one-sentence justification:
1. The current chapter draft being reviewed.
2. The list of inconsistencies flagged by the plot agent.
3. The character agent's working memory of established character voices (persists across chapters).
4. The accumulated suggested edits feeding into the consolidator.

> [!NOTE]
> **Self-check** (30s — answer mentally before expanding)
>
> 1. Two evaluator-agents share a single `thread_id`. What failure does this introduce?
> 2. You use LLM summarization to curate the supervisor's context blob for each worker. What's the risk compared to deterministic templating?

<details>
<summary>Show answers</summary>

1. Cross-request drift and cross-evaluator contamination — each evaluator sees the other's intermediate reasoning, violating the independence requirement for consensus-style scoring. Fix: mint a unique `thread_id` per evaluator per logical task.
2. LLM summarization introduces compression bias (the LLM's filtering criteria become an unintentional ranking), adds latency (~1 extra LLM hop per delegation), and produces unstable traces (same input → different summary). Prefer deterministic templates for known-structure worker inputs; reserve LLM summarization for genuinely unstructured cases.

</details>

## 8. Key Takeaways

- Three shapes: shared scratchpad (collaboration), per-agent thread (independence), supervisor-curated context (heterogeneous needs).
- Shared memory creates implicit trust — source tagging, namespacing, and validation gates restore the discipline.
- Thread-scoped = ephemeral (one execution); store-scoped = durable (cross-execution). Choose by durability semantics.
- Database-backed checkpointer is required for production HITL flows that span minutes or more.

> [!NOTE]
> **W5 interlock.** The `audit_correlation_id` threaded through graph state today becomes the W3C `traceparent` trace ID in W5's OpenTelemetry integration. Name it consistently now.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://docs.langchain.com/oss/python/concepts/memory — retrieved 2026-05-26 — hot-tech
- https://www.vellum.ai/blog/multi-agent-systems-building-with-context-engineering — retrieved 2026-05-26 — hot-tech
- https://mem0.ai/blog/multi-agent-memory-systems — retrieved 2026-05-26 — hot-tech
- https://www.augmentcode.com/guides/cross-agent-organizational-memory — retrieved 2026-05-26 — hot-tech
- https://medium.com/mongodb/why-multi-agent-systems-need-memory-engineering-153a81f8d5be — retrieved 2026-05-26 — hot-tech

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

**Memory as a domain object — the W5 interlock.** Treating `audit_correlation_id` and `EvaluationState` memory as domain objects (schema-versioned, tested) today creates the foundation for W5's OTel integration. The correlation ID threaded through graph state becomes the W3C `traceparent` trace ID propagated through OpenTelemetry spans — same primitive, two levels of the stack. Senior FDEs designing today's memory schema should anticipate this interlock and name the field consistently (`audit_correlation_id` or `correlation_id`) across all nodes and audit events.

**Supervisor-curated context and the "right to know" principle.** In federal-acquisitions contexts, the "right to know" principle is not just an access-control concern — it is a FAR compliance concern. Evaluator-agents should not see other evaluators' scores before consensus (FAR 15.303 independence of evaluation). Per-agent thread isolation is the structural implementation of this principle; it's not just a performance optimisation.

</details>

Last verified: 2026-06-06
