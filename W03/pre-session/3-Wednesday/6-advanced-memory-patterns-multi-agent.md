---
week: W03
day: Wed
topic_slug: advanced-memory-patterns-multi-agent
topic_title: "Advanced memory patterns for multi-agent"
parent_overview: W03/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 10
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
last_verified: 2026-05-26
---

# Advanced memory patterns for multi-agent

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish three multi-agent memory shapes — *shared scratchpad*, *per-agent thread*, and *supervisor-curated context* — and name the workload each fits.
- Recognize memory-contamination and cross-agent-trust failure modes and apply standard mitigations (namespacing, access control, source tagging).
- Choose between thread-scoped (short-term) and store-scoped (long-term) memory based on durability and cross-session requirements.
- Implement a per-agent thread pattern using LangGraph's checkpointer + thread_id model.
- Articulate when *isolation* (no shared memory at all) is the right answer.

## 2. Introduction

Memory in a single-agent system is roughly one question: how do you keep enough conversation history that the agent can reason coherently, without overflowing the context window or losing important earlier facts? The answers are well-known — message history, summarization, retrieval — and Monday's reading covered them at single-agent scope.

Multi-agent memory introduces a second dimension: not just *how much* memory, but *whose* memory. When a supervisor delegates to workers, the workers each need *some* context to do their jobs — but exactly what context, written by whom, readable by whom, and persisted in what scope, becomes a design space with real failure modes. Pick wrong and you get cross-agent contamination (one agent's incorrect fact poisons every other agent's reasoning), context collapse (the worker can't recover the framing it needs), or trust failures (an agent treats another agent's emission as authoritative when it shouldn't be) [1][2].

The 2026 production literature converges on three default shapes for cross-agent memory: shared scratchpad, per-agent thread, and supervisor-curated context. Each fits a particular kind of workload; the picks compose (a multi-agent system can use all three for different state keys). The discipline is to pick deliberately, not by default.

## 3. Core Concepts

### 3.1 Shared scratchpad

**Shape.** All agents read and write a common state key (often a list of structured findings). LangGraph implements this directly via shared graph state with a list-concat reducer; many other frameworks expose it as a `SharedContext` object with thread-safe `context_read()` / `context_write()` tools [3].

**Fits when.** Workers are *collaborating* on a shared artifact: research agents accumulating findings on a single topic, content-generation agents progressively refining a draft, planning agents elaborating a single plan. The scratchpad is the artifact in flight.

**Failure modes.**

- *Leakage* — one agent's incorrect or biased finding influences every subsequent agent's reasoning. Once written, it is hard to retract.
- *Race conditions* — when parallel agents both append without a reducer, you lose writes or hit `InvalidUpdateError`.
- *Trust contagion* — agents tend to treat anything in the scratchpad as authoritative; an early agent's hallucination becomes "fact" for later agents [4].

**Mitigations.** Source-tag every write (`{"finding": ..., "actor": "worker_a", "confidence": 0.7}`) so later agents can weigh provenance; use list-concat reducers (append-only, never overwrite); periodic curation by a dedicated "scratchpad curator" agent that summarizes and dedupes [4].

### 3.2 Per-agent thread

**Shape.** Each agent has its own isolated message history. In LangGraph this is implemented via a checkpointer plus a per-agent `thread_id` — the agent's conversation persists in the checkpointer keyed by its thread, and other agents cannot read it [1].

**Fits when.** Workers should reason *independently* — evaluators scoring the same artifact, multiple model candidates reviewing the same output, parallel branches whose mutual influence would be a bug. The classic case is consensus-style scoring: you want N independent opinions, not one opinion echoed N times.

**Failure modes.**

- *Reconciliation overhead* — when threads are fully isolated, the synthesizer has to do real work to combine outputs.
- *Wasted shared context* — if all threads need the same base context (a document, a rubric), reloading it per-thread wastes tokens; usually solved by having the supervisor pre-load and pass a frozen subset.
- *Cross-session drift* — if `thread_id`s persist across user requests, agents can accumulate stale state; usually solved by minting thread IDs per logical task [1].

**Mitigations.** Mint `thread_id`s per logical task (not per agent instance globally). Use a database-backed checkpointer in production (Postgres, Redis) rather than in-memory. Treat the synthesizer as a first-class node, not an afterthought.

### 3.3 Supervisor-curated context

**Shape.** The supervisor maintains a global view of state, and for each worker delegation, it constructs a per-worker context blob with exactly what that worker needs — filtered, summarized, or annotated. Each worker sees a *curated* slice, not the raw global state [2][5].

**Fits when.** Workers have heterogeneous information needs and would be confused by raw state; or the global state is too large to pass to every worker (token cost) and selective passing is necessary; or worker prompts need agent-specific framing that the worker shouldn't reconstruct itself.

**Failure modes.**

- *Compression bias* — the supervisor's filtering criteria become an unintentional ranking; workers don't know what they're not seeing.
- *Extra LLM hop* — naive implementations call an LLM to summarize per delegation, doubling the latency budget.
- *Drift between supervisor's mental model and worker's mental model* — the supervisor compresses; the worker decompresses; the compressions don't match (this is the lost-context-across-supervisor-boundary anti-pattern from the prior reading).

**Mitigations.** Use a typed schema for the worker-input contract — the supervisor must populate every field, the worker must consume every field. When summarization is needed, prefer deterministic templated summaries to LLM-summarized blurbs; reserve LLM summarization for cases where the structure is genuinely unknown.

### 3.4 Composition — most production systems use all three

The three shapes are not exclusive — production multi-agent systems typically use *all three* for different state keys [2][5]:

- A shared scratchpad for collaborative artifacts.
- Per-agent threads for independent worker reasoning.
- Supervisor-curated context blobs for delegation inputs.

The discipline is to make the choice explicit per state key, with a documented rationale: "this is a shared scratchpad because workers are collaborating on it; this is per-thread because we need independent opinions; this is supervisor-curated because workers need different slices."

### 3.5 Short-term vs long-term memory

Crosscutting all three shapes is the **scope** dimension:

- **Short-term (thread-scoped, ephemeral).** Lives in the graph state for one execution; persisted by the checkpointer; gone at flow end. The default for in-flight work [1].
- **Long-term (store-scoped, durable).** Lives in a separate memory store (often namespaced); persists across executions. Used for cross-session user preferences, accumulated organizational knowledge, learned facts about external systems [1].

The choice is not about memory capacity but about *durability semantics*. Things the agent will need on the next run go in the long-term store; things relevant only to the current run go in graph state.

### 3.6 Cross-agent trust and contamination

The newer concern in the 2026 production literature is that **shared memory creates implicit trust** [4][5]. When Agent A writes a finding and Agent B reads it, B has no built-in reason to treat it differently from a system instruction — yet A's finding may be hallucinated, biased, or maliciously injected (in adversarial settings).

Standard mitigations:

- **Source tagging.** Every memory write carries `actor`, `confidence`, and `derived_from` fields. Downstream readers treat memories from different sources differently [4].
- **Namespacing.** Memory stores partition by tenant/user/agent so cross-tenant contamination is structurally impossible [3][5].
- **Access control.** Per-agent read/write permissions per memory namespace; some agents can write a key, others can only read it [4].
- **Validation gates.** Before a memory becomes "load-bearing" for downstream agents, a separate gate verifies it.

These are distributed-systems disciplines (provenance, ACLs, validation) applied to memory rather than to RPC.

## 4. Generic Implementation

A per-agent-thread implementation for parallel independent scorers, using LangGraph's checkpointer + thread_id model. The scenario is generic: three independent reviewers scoring a job applicant's portfolio (no federal-acquisitions overlap).

```python
import uuid
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send, Command
from langgraph.checkpoint.postgres import PostgresSaver  # production-grade
import operator

class PortfolioState(TypedDict):
    portfolio: dict
    rubric: dict
    # Reducer-bearing key — parallel scorers concat their scores
    scores: Annotated[list[dict], operator.add]
    final_decision: str | None

# Each scorer node — receives its own slice + a unique thread_id
def scorer(input_state: dict):
    # input_state contains a per-invocation thread_id minted by the supervisor.
    # The scorer's internal LLM calls are checkpointed under THIS thread,
    # so other scorers do not see this scorer's intermediate reasoning.
    score = independent_review(
        portfolio=input_state["portfolio"],
        rubric=input_state["rubric"],
        reviewer_role=input_state["reviewer_role"],
        thread_id=input_state["thread_id"],
    )
    return {"scores": [{
        "reviewer": input_state["reviewer_role"],
        "score": score,
        "thread_id": input_state["thread_id"],
    }]}

def supervisor(state: PortfolioState):
    if not state["scores"]:
        # Mint per-reviewer thread IDs to isolate their reasoning
        return [
            Send("scorer", {
                "portfolio": state["portfolio"],
                "rubric": state["rubric"],
                "reviewer_role": role,
                "thread_id": str(uuid.uuid4()),   # one thread per reviewer-task
            })
            for role in ["tech_reviewer", "design_reviewer", "comms_reviewer"]
        ]
    return Command(goto="synthesizer")

def synthesizer(state: PortfolioState):
    # Combine independent scores into a decision
    decision = aggregate_scores(state["scores"])
    return Command(update={"final_decision": decision}, goto=END)

graph = StateGraph(PortfolioState)
graph.add_node("supervisor", supervisor)
graph.add_node("scorer", scorer)
graph.add_node("synthesizer", synthesizer)

graph.add_edge(START, "supervisor")
graph.add_edge("scorer", "supervisor")
graph.add_edge("synthesizer", END)

# Production: persist to a real database, not in-memory
checkpointer = PostgresSaver.from_conn_string(POSTGRES_URL)
app = graph.compile(checkpointer=checkpointer)
```

What each piece does:

- **Three independent scorers** run in parallel via Send; each receives its own input slice with no view of the others' work (per-agent-thread isolation).
- **`thread_id` minted per scorer-task** — not a global per-reviewer ID. Each portfolio review starts fresh threads so no cross-portfolio drift.
- **`scores` carries a list-concat reducer** — parallel writes are safe.
- **`synthesizer` is the only place threads' outputs combine** — explicit reconciliation, not implicit cross-thread reading.
- **Postgres checkpointer** — production durability so a soft-interrupt on one scorer doesn't lose the others' work.

For a shared-scratchpad shape, you would replace the per-scorer thread isolation with a single `findings: Annotated[list[dict], operator.add]` state key that all workers read and write. For supervisor-curated context, you would have the supervisor LLM-summarize global state before passing it into each `Send`.

## 5. Real-world Patterns

**E-commerce — per-customer memory namespacing.** A mid-2026 marketplace deployed a multi-agent shopping assistant where each customer's memory is namespaced (`user_id:{cid}/agent_id:{aid}/run_id:{rid}`). Browsing-agent, recommendation-agent, and cart-agent share access within a customer's namespace but cannot cross-read into other customers' memories. The namespacing is the structural mitigation against cross-customer contamination [3].

**Healthcare — independent radiology reviewers.** Multi-reader diagnostic systems use per-agent-thread isolation: each modality reviewer (CT, MRI, X-ray) reasons in its own thread with no view of others' findings until the synthesizer aggregates. The clinical reason is that independent opinions are by design — shared scratchpad would create confirmation bias across modalities [5]. The aggregator surfaces all three threads in the radiologist's review UI rather than reconciling them into a single number.

**Fintech — supervisor-curated context for fraud analysis.** A fraud-detection multi-agent system uses supervisor-curated context: the lead agent has access to the full transaction history, but each worker (velocity-rules, graph-features, behavioral-anomalies) receives a curated subset matched to its analysis style. The reported architectural win is that the workers' prompts can be stable and short — they never see context they don't need; the curation is done deterministically (template-based) rather than via LLM summarization [2].

**SaaS — research-assistant shared scratchpad with provenance.** A B2B research-assistant product uses a shared scratchpad where each finding is tagged with `actor`, `confidence`, and `derived_from`. The reported lesson from a 2026 production retro: in early versions, agents treated all findings as equally authoritative and a single hallucination poisoned downstream reasoning. After adding the source-tagging discipline, downstream agents could weigh findings differently and the contamination rate dropped substantially [4].

## 6. Best Practices

- **Pick the memory shape per state key, with a documented rationale.** "Shared scratchpad because collaboration; per-thread because independence; curated because heterogeneous needs."
- **Always namespace long-term memory.** `user_id / agent_id / run_id / app_id` (or a tenant-equivalent) — cross-tenant contamination is too easy without it [3].
- **Source-tag shared writes.** `actor`, `confidence`, `derived_from`, `timestamp`. Downstream readers can then weigh provenance instead of treating everything as equally authoritative.
- **Mint thread IDs per logical task, not globally per agent.** Global per-agent threads accumulate cross-request drift; per-task threads are clean by default.
- **Use database-backed checkpointers in production.** In-memory checkpointers are fine for dev; they lose state on restart, which is hostile to soft interrupts and long-running flows.
- **Resist LLM-summarization for supervisor-curated context unless structure is genuinely unknown.** Deterministic templates are faster, cheaper, and produce stable traces.
- **Treat memory as a domain object** — version it, schema it, test it. "Memory contamination" is a real failure class, not a hypothetical [4].

## 7. Hands-on Exercise

**Architecture-decision prompt (15 min).** You are designing a *multi-agent novel-editing system* (no federal-acquisitions overlap). Four agents collaborate: a *plot agent* identifies inconsistencies, a *character agent* checks character voice/consistency, a *prose agent* suggests sentence-level edits, and a *consolidator* produces the final reviewer report.

For each of the following state keys, pick a memory shape (shared scratchpad / per-agent thread / supervisor-curated context / long-term store) and write one-sentence justifications:

1. The current chapter draft being reviewed.
2. The list of inconsistencies flagged by the plot agent.
3. The character agent's working memory of established character voices (this persists across chapters in the same novel).
4. The supervisor's per-agent delegation packets (what each agent sees when invoked).
5. The accumulated suggested edits feeding into the consolidator.
6. The author's previously stated stylistic preferences (these persist across novels).

**What good looks like:** (1) Shared scratchpad — all agents read the same chapter. (2) Shared scratchpad with source-tagging — agents may want to reference each other's flagged inconsistencies, but with provenance. (3) Long-term store namespaced per novel — durable, cross-chapter, isolated to one novel. (4) Supervisor-curated context — each agent gets a focused packet; the supervisor's curation is template-based. (5) Per-agent thread inputs combined by the consolidator — the agents reason independently to avoid one's suggestion biasing another's. (6) Long-term store namespaced per author — durable, cross-novel.

## 8. Key Takeaways

- *What are the three default multi-agent memory shapes and the workload each fits?* (Shared scratchpad → collaboration; per-agent thread → independence; supervisor-curated context → heterogeneous needs.)
- *Why does shared memory create implicit trust and what mitigates it?* (Downstream agents have no built-in reason to discount earlier agents' writes; source tagging, namespacing, access control, and validation gates restore the trust discipline.)
- *What is the difference between thread-scoped and store-scoped memory?* (Thread-scoped is ephemeral and gone at flow end; store-scoped is durable and persists across executions — the choice is about durability semantics, not capacity.)
- *Why mint thread IDs per logical task rather than per agent globally?* (Global per-agent threads accumulate cross-request drift; per-task threads are clean by default.)
- *Why is database-backed checkpointing essential for production HITL flows?* (Interrupts that span more than a few minutes need durable persistence; in-memory loses state on restart.)

## Sources

1. [Memory overview — LangChain/LangGraph official docs](https://docs.langchain.com/oss/python/concepts/memory) — retrieved 2026-05-26
2. [How to Build Multi-Agent AI Systems With Context Engineering (Vellum)](https://www.vellum.ai/blog/multi-agent-systems-building-with-context-engineering) — retrieved 2026-05-26
3. [How to Design Multi-Agent Memory Systems for Production (Mem0)](https://mem0.ai/blog/multi-agent-memory-systems) — retrieved 2026-05-26
4. [Cross-Agent Organizational Memory: How Knowledge Compounds (Augment Code)](https://www.augmentcode.com/guides/cross-agent-organizational-memory) — retrieved 2026-05-26
5. [Why Multi-Agent Systems Need Memory Engineering (MongoDB)](https://medium.com/mongodb/why-multi-agent-systems-need-memory-engineering-153a81f8d5be) — retrieved 2026-05-26

Last verified: 2026-05-26
