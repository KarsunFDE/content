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
last_verified: 2026-05-26
---

# Workflow resiliency + debugging LangGraph with LangSmith

## 1. Learning Objectives

By the end of this reading, the learner can:

- Explain LangGraph's fault-tolerance model: what survives a node failure, what survives a process restart, and what re-runs on resume.
- Distinguish "transient failure" (retry-friendly) from "logical failure" (state-fixup required) and design node-level retry policies accordingly.
- Use LangSmith's project/trace/run/thread hierarchy to debug a multi-step workflow and find the failing step.
- Identify which operations must be wrapped in `task` boundaries to keep replay deterministic.
- Read a LangSmith trace that crosses an interrupt and explain what the gap between the last pre-interrupt run and the resume run means.

## 2. Introduction

Every workflow system has a story for "what happens when something goes wrong" — and the quality of that story is a big part of what separates frameworks you can deploy from frameworks you wish you could deploy. The default answer in plain Python is *bad*: a half-finished workflow that crashed leaves state somewhere, no one knows where, and you find out about it when a downstream system notices a missing record. The minimum viable better answer is durable state plus structured tracing: persisted progress so resumption is possible, and observable history so debugging is possible.

LangGraph and LangSmith together provide that minimum viable better answer for graph-shaped LLM workflows. LangGraph's persistence layer ensures fault tolerance — a node failure does not lose the work of its siblings, and a process restart does not lose the work of completed super-steps. LangSmith provides the observability — every node, every LLM call, every retrieval shows up as a structured run inside a trace, with the metadata you need to figure out what went wrong without reproducing the bug locally.

The two complement each other. Persistence keeps the workflow alive across failures; tracing makes the failures legible. Today's wiring against the real graph leans on both — the resiliency story for "Bedrock 429'd at step 4, we resumed and finished step 5" is only credible if the trace shows the retry and the resume both fired.

## 3. Core Concepts

### Fault tolerance at the super-step boundary

LangGraph's persistence layer writes a checkpoint at the end of each super-step ([source: docs.langchain.com persistence, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/persistence)). The key resilience property follows from this: **when a node fails mid-execution, LangGraph stores pending checkpoint writes from any other nodes that completed successfully at that super-step.** When you resume, the successful nodes are not re-run.

In a fan-out scenario, this is load-bearing. If you fan out to four parallel workers and worker B raises, the checkpoint persists workers A, C, and D's results. Resume re-runs only worker B. Without this, a single flaky branch would force you to redo all four — at LLM-call prices, that adds up quickly.

The resume path:

```python
# Initial invocation hit a node failure.
try:
    graph.invoke(payload, config={"configurable": {"thread_id": "ticket-42"}})
except SomeError:
    pass

# Resume from the same thread — picks up where it left off.
graph.invoke(None, config={"configurable": {"thread_id": "ticket-42"}})
```

Passing `None` as input on resume tells the runtime to read state from the checkpointer and continue.

### Determinism and consistent replay

When a workflow resumes, the code does *not* pick up at the exact line where it stopped; it resumes from the most recent appropriate checkpoint and replays from there ([source: docs.langchain.com durable-execution, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/durable-execution)). This has a critical implication: **anything inside the node that ran before the failure will run again on resume.**

If your node makes an external API call and then crashes, the API call will be re-issued on replay. For an idempotent call (a `GET`) this is fine. For a non-idempotent call (a payment capture, a model invocation that bills you, a `POST` that creates a record), it is not.

The fix is to wrap non-deterministic and side-effectful operations inside `task` boundaries. The framework records the task's result on first run; on replay it reads the recorded result instead of re-executing:

```python
from langgraph.func import task

@task
def capture_payment(intent_id: str) -> dict:
    return stripe.capture(intent_id)   # the side effect

def payment_node(state):
    result = capture_payment(state["intent_id"]).result()   # recorded on first run
    return {"capture": result}
```

The same pattern applies to LLM calls that you do not want billed twice on resume.

### Retry policies — transient vs logical failures

Two failure shapes need different handling:

- **Transient failures** — rate limits (429), brief network errors, ephemeral upstream blips. The right move is retry with backoff. LangGraph supports per-node retry configuration via `retry` policies on `add_node`, with parameters for max attempts, initial interval, multiplier, and which exception types are retryable.
- **Logical failures** — the LLM produced output that fails validation, the downstream API returned a 4xx because the input was malformed, the state is genuinely wrong. Retry will not help. The right move is to surface the failure for human inspection — possibly via an interrupt — and either fix the state and resume, or roll back to an earlier checkpoint.

Classifying a failure into one bucket or the other is the engineer's job. The framework gives you the levers; the discipline is to not retry-loop your way through what is actually a state bug.

### LangSmith's hierarchy: project / trace / thread / run

LangSmith organises observability into four levels ([source: docs.smith.langchain.com observability concepts, retrieved 2026-05-26](https://docs.smith.langchain.com/observability/concepts)):

- **Project** — a container for all traces from a single application or service.
- **Trace** — the sequence of runs for a single operation (one workflow invocation; one user request).
- **Thread** — multiple traces grouped by a shared `session_id`/`thread_id`/`conversation_id` metadata key. Used to view a multi-turn or multi-day workflow as a single sequence.
- **Run** — a single unit of work inside a trace. One LLM call is a run; one node is a run; one tool call is a run. If you are familiar with OpenTelemetry, a run is a span.

The thread level is what makes a multi-day workflow legible. A single workflow that fires once, interrupts for 18 hours, then resumes will show up as two traces in the same thread — the gap between them is visible in the timeline. Without the thread grouping, the two traces look unrelated.

### Tracing a workflow that crosses an interrupt

When a graph hits an interrupt, the trace pauses at the interrupt node ([source: docs.langchain.com interrupts, retrieved 2026-05-26](https://docs.langchain.com/oss/python/langgraph/interrupts)). The trace itself stays open in LangSmith but no new runs accumulate until the resume fires. When the resume happens via `Command(resume=...)`, a new run appears in the trace timeline with the time gap visible.

This visual is the single most useful artifact in a defense or review: it shows that the system *did* pause for the human and *did* wait, in real time, for as long as the human took. A reviewer who asks "did the system really wait, or did it auto-approve?" can be answered by pointing at the timeline.

To make this work end-to-end, ensure that the resume invocation reuses the same `thread_id` *and* attaches the same metadata as the initial invocation. The `session_id`/`thread_id`/`conversation_id` metadata key is what makes LangSmith group the two traces into one thread.

### What to instrument on every node

Treat each node as a span and attach the metadata that makes the trace useful in retrospect:

- **Inputs and outputs** — what the node read and wrote. LangSmith captures this automatically when you use the `@traceable` decorator or the LangChain/LangGraph instrumentation.
- **Latency** — wall-clock time. Already captured by LangSmith.
- **Token counts** — for LLM calls. Use the OpenTelemetry GenAI semantic-convention attribute names where possible (`gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`).
- **Correlation identifier** — a stable ID that threads through every cross-service hop, so the workflow's trace can be joined to the upstream HTTP request and downstream service calls.

The minimum bar for production: any failure should be diagnosable from the trace alone, without re-running the workflow locally.

## 4. Generic Implementation

A non-Karsun example: a generic data-enrichment pipeline that calls three external APIs in sequence (geocoding, demographics, sentiment), with retries on transient failures and tracing throughout.

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

# Wrap each external call in a task so a resume does not re-issue it.
@task
def call_geocode(address: str) -> dict:
    return geocode_api.lookup(address)   # external API; non-deterministic

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

# Each invocation is one trace; thread_id groups them.
config = {"configurable": {"thread_id": f"enrich:{record_id}"}}
app.invoke({"record_id": record_id, "address": "..."}, config=config)
```

What this gives you:

- **If `call_demographics` 429s** — the retry policy on the node retries up to three times with exponential backoff. The earlier `call_geocode` task's result is already recorded; it does not re-run.
- **If the process crashes mid-pipeline** — the next invocation with the same `thread_id` resumes from the last checkpoint. The recorded tasks are read from state; only un-run tasks execute.
- **If `call_sentiment` returns malformed JSON** — the retry policy probably should not fire (this is a logical failure, not transient). The cleaner shape is to raise a non-retryable exception type and let the graph surface to an interrupt for human inspection.
- **In LangSmith** — each invocation produces a trace; multiple invocations on the same `thread_id` show up as a thread; each task is a separate run inside its parent node.

The single most important property: when this pipeline misbehaves in production, a developer can open LangSmith, find the trace, and see exactly which task failed and what it returned, without re-running the pipeline against possibly-changed external state.

## 5. Real-world Patterns

**E-commerce — order-processing sagas (DoorDash, Uber Eats).** Long-running order workflows persist their progress so a worker crash mid-order does not double-charge or double-dispatch. The pattern is structurally identical to LangGraph's super-step + checkpoint: each step is recorded, retries are bounded, and observability captures every transition. Engineering blog posts from the food-delivery space repeatedly emphasise that "every external call is idempotent or recorded" — exactly the `task` wrapper pattern.

**Fintech — Plaid's transaction-fetch retries.** Plaid maintains long-lived per-account workflows that periodically fetch transactions from upstream banks. When an upstream bank's API is flaky, Plaid's workers retry within a bounded budget and surface persistent failures as a status the consumer can subscribe to. The observable trace of "what we tried, what the bank returned, when we gave up" is a contractually visible artifact — clients use it to debug their own integrations.

**Logistics — shipment status reconciliation.** Multi-carrier shipping platforms run reconciliation workflows that poll each carrier's tracking API on a schedule, persist the polled state, and reconcile against the merchant's shipping records. The retries-with-backoff and structured-trace patterns are universal; the failure modes that bite teams are non-idempotent retries (issuing a duplicate label) and traces that span too many invocations to navigate (where the thread-id grouping pays off).

**Healthcare — HL7 message-routing engines.** Hospital integration engines route HL7 v2 messages between EHRs, labs, and pharmacies. Each message's journey is persisted and observable: which queue it sat in, which transformations it went through, which downstream system acknowledged. The "everything is replayable from persisted state" property is what makes message-routing engines safely operable; the same property is what LangGraph's checkpointer + LangSmith trace give to LLM workflows.

## 6. Best Practices

- **Wrap non-deterministic and side-effectful operations in `task` boundaries.** Replay-safety is the difference between a workflow that survives crashes cleanly and one that double-bills.
- **Classify failures explicitly: transient vs logical.** Retry only the first; surface the second to a human or to an interrupt.
- **Set per-node retry policies, not a single graph-wide retry.** Different nodes have different failure profiles.
- **Use stable `thread_id`s and attach them to LangSmith metadata.** A multi-day workflow is unreadable without thread grouping.
- **Annotate every node with `@traceable` or rely on the framework's auto-instrumentation.** Implicit traces are still traces; explicit ones are easier to filter.
- **Capture token-usage metadata using the OpenTelemetry GenAI attribute names.** Standard names make traces portable across observability backends.
- **Keep node bodies short.** A node that does five things is one trace span with five hidden operations; five smaller nodes are five spans with five visible operations.

## 7. Hands-on Exercise

**Code task (15 min).** Take a small two-node graph that fetches a URL and then summarises the response. Modify it so that:

1. The fetch and the summarise calls are each inside a `task`.
2. The graph compiles with a `PostgresSaver` (or `SqliteSaver` if Postgres is not available locally).
3. A simulated failure in the summariser raises a custom `TransientUpstreamError`, retried up to three times via the node's retry policy.
4. After the retries exhaust, the graph pauses (use `interrupt()`) and surfaces the last error for human inspection.
5. The whole run produces a LangSmith trace where you can see the fetch task's recorded result, the summariser retry attempts, and the interrupt.

**What good looks like.** The fetch result appears once in the trace, not three times (because the `task` boundary recorded it). The summariser runs three times. The interrupt is the final run in the trace. On resume with `Command(resume={"ack": True})`, the workflow completes without re-fetching. A weak answer omits the `task` wrappers, so the fetch re-runs on every replay; the trace becomes unreadably noisy.

## 8. Key Takeaways

- When a node in a parallel branch fails, what survives in the checkpoint and what re-runs on resume? (LO1)
- What is the difference between a transient and a logical failure, and how do I configure each? (LO2)
- How do project / trace / thread / run nest in LangSmith, and which level groups a multi-day workflow into one view? (LO3)
- Why does an external API call need to be inside a `task` boundary to be replay-safe? (LO4)
- What does the gap between the last pre-interrupt run and the resume run in a LangSmith trace tell me? (LO5)

## Sources

1. [LangGraph Durable Execution — replay, determinism, task boundaries (v1.x docs)](https://docs.langchain.com/oss/python/langgraph/durable-execution) — retrieved 2026-05-26
2. [LangGraph Persistence — fault tolerance, pending writes, checkpoint boundaries](https://docs.langchain.com/oss/python/langgraph/persistence) — retrieved 2026-05-26
3. [LangSmith Observability concepts — projects, traces, threads, runs](https://docs.smith.langchain.com/observability/concepts) — retrieved 2026-05-26
4. [LangGraph Interrupts — interrupt model, resume contract](https://docs.langchain.com/oss/python/langgraph/interrupts) — retrieved 2026-05-26

Last verified: 2026-05-26
