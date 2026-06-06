---
week: W03
day: Thu
topic_slug: workflow-resiliency-debugging-with-langsmith
topic_title: "Workflow resiliency + debugging LangGraph with LangSmith"
parent_overview: W03/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://docs.langchain.com/oss/python/langgraph/durable-execution
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/oss/python/langgraph/persistence
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.smith.langchain.com/observability/concepts
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/oss/python/langgraph/interrupts
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-06-06
---

# Workflow resiliency + debugging LangGraph with LangSmith

> [!NOTE]
> **From earlier:** Tue's observability topic introduced tracing individual tool calls. Today the trace spans a multi-hour interrupt gap — the LangSmith timeline shows the pause and the resume as a single thread. That visual is your Friday defense evidence.

## 1. Learning Objectives

By the end of this reading, you can:

- Explain LangGraph's fault-tolerance model: what survives a node failure, what survives a process restart, and what re-runs on resume.
- Distinguish transient failures (retry-friendly) from logical failures (state-fixup required) and configure per-node retry policies accordingly.
- Use LangSmith's project / trace / thread / run hierarchy to locate the failing step in a multi-step workflow.
- Identify which operations require `task` wrappers to stay replay-safe.
- Read a LangSmith trace that crosses an interrupt and explain what the gap means.

## 2. Introduction

Plain Python's default answer to "what goes wrong" is bad: a half-finished workflow leaves state somewhere, discovered when a downstream system notices a missing record. The minimum viable improvement is durable state plus structured tracing: persisted progress so resumption is possible, observable history so debugging is possible.

LangGraph keeps the workflow alive across failures; LangSmith makes them legible. The resiliency story for "Bedrock 429'd at evaluator step 3, we resumed" is only credible if the trace shows the retry and the resume — Friday's defense requires those screenshots.

> [!IMPORTANT]
> **LangSmith is preview integration today (D-031).** Deep eval workflows — RAGAS, judge-model scoring — are W5 material. Today: wire the trace, capture the interrupt-gap visual, confirm token-per-node metadata appears. Do not build eval pipelines yet.

## 3. Core Concepts

### 3.1 Fault tolerance at the super-step boundary

LangGraph checkpoints at the end of each super-step ([source: docs.langchain.com persistence, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/persistence)). When a node fails, successful nodes' pending writes from the same super-step are persisted. Resume re-runs only the failed branch — pass `None` as input to read state from the checkpointer and continue.

```python
try:
    graph.invoke(payload, config={"configurable": {"thread_id": "eval:4711"}})
except SomeError:
    pass
# Resume — picks up from last good checkpoint.
graph.invoke(None, config={"configurable": {"thread_id": "eval:4711"}})
```

At LLM-call prices, not re-running successful evaluator branches matters.

### 3.2 Determinism and the `task` boundary

When a workflow resumes, it replays from the most recent super-step checkpoint ([source: docs.langchain.com durable-execution, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/durable-execution)). **Any code that ran before the failure inside the node re-runs on resume.**

For idempotent operations (a `GET`) this is fine. For non-idempotent operations (a Bedrock call, a `POST`), it causes double-execution. The fix: wrap in `task` boundaries — result recorded on first run, replayed from record on resume.

```python
from langgraph.func import task

@task
def score_proposal(proposal_id: str, criteria: dict) -> dict:
    return bedrock_client.invoke_model(...)

def evaluator_node(state: EvaluationState) -> dict:
    result = score_proposal(state["proposal_id"], state["criteria"]).result()
    return {"evaluator_scores": {state["proposal_id"]: result}}
```

> [!TIP]
> **If the operation has a cost or a side-effect, wrap it in `@task`.** GET requests that are cheap and idempotent can skip the wrapper; Bedrock calls and database writes cannot.

### 3.3 Transient vs logical failures + LangSmith hierarchy

| Failure type | Example | Right response |
|---|---|---|
| Transient | Bedrock 429, network blip | Retry with backoff |
| Logical | LLM output fails validation, 4xx malformed input | Surface to human; retry won't help |

Configure per-node retry via `retry` on `add_node`: `max_attempts`, `retry_on` (exception types), backoff. Retry-looping a logical failure wastes calls and delays diagnosis.

LangSmith has four levels ([source: docs.smith.langchain.com observability concepts, retrieved 2026-05-26](https://docs.smith.langchain.com/observability/concepts)): **Project** → **Trace** → **Thread** (traces sharing a `thread_id`) → **Run**. When the graph hits an interrupt the trace pauses; on `Command(resume=...)` a new run appears with the gap visible ([source: docs.langchain.com interrupts, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/interrupts)). **That gap visual is Friday's defense evidence.** Reuse the same `thread_id` on resume so LangSmith groups both invocations into one thread.

## 4. Generic Implementation

Data-enrichment pipeline with retry and `task` wrappers:

```python
from langgraph.graph import StateGraph, START, END
from langgraph.func import task
from langgraph.checkpoint.postgres import PostgresSaver
from langsmith import traceable
from typing import TypedDict

class EnrichState(TypedDict):
    record_id: str
    address: str
    geocode: dict | None
    demographics: dict | None
    sentiment: dict | None

@task
def call_geocode(address: str) -> dict:
    return geocode_api.lookup(address)

@task
def call_demographics(coords: dict) -> dict:
    return demographics_api.lookup(coords)

@task
def call_sentiment(text: str) -> dict:
    return sentiment_api.classify(text)

@traceable(run_type="chain", name="enrich_record")
def enrich(state: EnrichState) -> dict:
    g = call_geocode(state["address"]).result()
    d = call_demographics(g["coords"]).result()
    s = call_sentiment(state.get("text", "")).result()
    return {"geocode": g, "demographics": d, "sentiment": s}

builder = StateGraph(EnrichState)
builder.add_node(
    "enrich",
    enrich,
    retry={"max_attempts": 3, "retry_on": [TransientApiError], "backoff": "exponential"},
)
builder.add_edge(START, "enrich")
builder.add_edge("enrich", END)
app = builder.compile(checkpointer=PostgresSaver(...))
```

If `call_demographics` 429s, the retry policy fires up to three times. `call_geocode`'s result is already recorded — it does not re-run. If `call_sentiment` returns malformed JSON, raise a non-retryable exception and surface to an interrupt.

## 5. Real-world Patterns

**E-commerce — order sagas.** Long-running food-delivery workflows persist progress so a worker crash does not double-charge or double-dispatch. "Every external call is idempotent or recorded" is the `task` wrapper pattern by another name.

> [!NOTE]
> **Cross-domain lesson:** Durable execution + structured tracing is the baseline for any workflow that crosses process restarts. The specific framework changes (Temporal, Step Functions, LangGraph); the invariant — persisted state + observable history — does not.

**Healthcare — HL7 message routing.** Hospital integration engines persist each message's journey. "Everything is replayable from persisted state" is what makes them safely operable — the same property LangGraph + LangSmith provide for LLM workflows.

## 6. Best Practices

- **Wrap non-deterministic operations in `task` boundaries.**
- **Classify failures: transient vs logical.** Retry only transient.
- **Per-node retry policies, not graph-wide.**
- **Stable `thread_id`s in LangSmith metadata.** Multi-day workflows are unreadable without thread grouping.
- **Keep node bodies short.** Five smaller nodes = five diagnosable spans.

> [!WARNING]
> **Anti-pattern: `tracing-as-replacement-for-tests`.** LangSmith traces show what happened, not what should happen. The trace can confirm the interrupt fired and the correct node ran; it cannot assert that the audit row schema is correct or that a careless thread_id change breaks multi-tenant isolation. Wire the traces; also write the tests. Both are on the Friday defense rubric.

## 7. Hands-on Exercise

Take a two-node graph (fetch URL → summarise): (1) wrap both calls in `task` boundaries; (2) compile with `PostgresSaver`; (3) retry a simulated `TransientUpstreamError` in the summariser up to three times; (4) after retries exhaust, pause via `interrupt()` for human inspection; (5) confirm the LangSmith trace shows the fetch result once, three retry attempts, and the interrupt gap.

> [!NOTE]
> **Self-check** (30 s — answer mentally before expanding)
>
> 1. You omit `@task` on `call_geocode`. The node crashes after geocode but before demographics. On resume, what happens to the geocode API call?
> 2. Your LangSmith trace shows two separate traces with an 18-hour gap under different threads. What did you forget?

<details>
<summary>Show answers</summary>

1. Without the `@task` wrapper, `call_geocode` is not recorded. On resume the enrichment node replays from the super-step boundary — which means `call_geocode` runs again. You pay for the geocode API call twice, and if the API is not idempotent (or if the result could differ — e.g., the address was updated), you get inconsistent state.
2. You forgot to pass the same `thread_id` metadata on the resume invocation. LangSmith groups traces into threads by matching that metadata key. If the initial invocation used `thread_id: "eval:4711"` but the resume omitted it or used a different value, LangSmith creates two separate threads and the 18-hour gap is invisible.

</details>

## 8. Key Takeaways

- Fault tolerance: successful branches' pending writes persist at the super-step checkpoint; only the failed branch re-runs on resume.
- `task` wrappers make non-deterministic operations replay-safe — recorded on first run, replayed from record on resume.
- Transient failures → retry with backoff. Logical failures → surface to human.
- LangSmith thread grouping via shared `thread_id` metadata makes a multi-day workflow appear as one cohesive trace.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://docs.langchain.com/oss/python/langgraph/durable-execution — retrieved 2026-05-26 — hot-tech
- https://docs.langchain.com/oss/python/langgraph/persistence — retrieved 2026-05-26 — hot-tech
- https://docs.smith.langchain.com/observability/concepts — retrieved 2026-05-26 — hot-tech
- https://docs.langchain.com/oss/python/langgraph/interrupts — retrieved 2026-05-26 — hot-tech

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

**Retry policy configuration in depth.** LangGraph's `retry` parameter on `add_node` accepts: `max_attempts` (integer), `retry_on` (list of exception types — only these trigger retries; all others propagate immediately), `backoff` (`"linear"` or `"exponential"`), `initial_interval` (seconds), and `multiplier` (for exponential). A common senior mistake is omitting `retry_on` — if not specified, LangGraph retries on *all* exceptions, including logical failures that will never succeed. Always name the exception types explicitly.

**LangSmith evaluation integration preview (D-031).** The `@traceable` decorator and LangSmith's feedback API allow attaching evaluation scores to individual runs — useful for tracking whether the consensus node's output quality degrades as proposal volume grows. W5's AIOps week builds on this: cost-per-node + quality-score-per-node together are the dual signal for auto-remediation decisions. Wire the tracing foundation today so W5 has something to instrument.

</details>

Last verified: 2026-06-06
