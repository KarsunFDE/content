---
week: W02
day: Wed
topic_slug: rag-pipeline-composition-and-observability
topic_title: "RAG pipeline composition + observability surface"
parent_overview: W02/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://blog.langchain.com/langchain-langgraph-1dot0/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/langsmith/trace-with-opentelemetry
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://oneuptime.com/blog/post/2026-02-06-rag-pipeline-tracing-opentelemetry/view
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://futureagi.com/blog/what-is-rag-observability-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://apxml.com/courses/large-scale-distributed-rag/chapter-5-orchestration-operationalization-large-scale-rag/monitoring-logging-alerting-distributed-rag
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# RAG pipeline composition + observability surface

## 1. Learning Objectives

By the end of this reading, the learner can:

- Compose a multi-step RAG pipeline as plain Python function calls — without depending on a v0.x `Chain` class, an LCEL `|` pipe foundation, or `.run()` entry points.
- Identify the per-stage observability primitives (correlation ID, span boundaries, structured log events) that make a RAG pipeline diagnosable in production.
- Propagate a correlation ID through every step so a single user query produces one trace.
- Choose between OpenTelemetry spans, a managed tracing platform, and structured JSON logs for a given operational maturity level.
- Read a production RAG dashboard and name the four signals (latency per stage, cost per stage, per-stage hit-rate, refusal rate) that gate a "do we ship" decision.

## 2. Introduction

The composition shape of a RAG pipeline determines what you can debug. A pipeline that hides its stage boundaries behind framework abstractions makes the framework's tracing your only window. A pipeline that exposes each stage as a normal function call lets every observability primitive your platform already supports — distributed tracing, structured logs, metrics, OpenTelemetry semantic conventions — work without ceremony.

The 2025–2026 architectural shift in LangChain v1.0 reinforces this. The `Chain` class is gone. The LCEL `|` pipe operator is no longer positioned as "the foundation"; it survives as syntactic sugar for users who want it. The official guidance is to compose pipelines as plain Python with each component invoked via `.invoke()` (or its async equivalent), assigning to variables, logging between steps, and calling normal control flow when needed. The framework provides the per-step toolkit; *your code* provides the orchestration shape.

The observability story has matured in lockstep. OpenTelemetry is now the default tracing substrate for LLM applications; LangSmith and other managed platforms support OTEL fanout so a single span emission lands in multiple backends. The current limitation — no stable OTel semantic conventions for RAG-specific span attributes — is being addressed by the GenAI SIG, but in the meantime production teams establish coherent internal conventions for span naming (`rag.retrieve.vector`, `rag.retrieve.bm25`, `rag.rerank`, `rag.generate`) and attribute keys (`rag.tenant_id`, `rag.chunks.retrieved`, `rag.faithfulness.score`).

## 3. Core Concepts

### Composition as plain Python

A production RAG pipeline composed as plain Python looks like a sequence of named function calls:

```python
def rag_query(user_query: str, tenant_id: str, correlation_id: str) -> dict:
    query_embedding = embed_query(user_query, correlation_id)
    candidates = vector_search(query_embedding, tenant_id, correlation_id)
    reranked = rerank(user_query, candidates, correlation_id)
    answer = generate(user_query, reranked, correlation_id)
    grounding = score_grounding(answer, reranked, correlation_id)
    return assemble_response(answer, reranked, grounding, correlation_id)
```

Each step:
- Accepts inputs and a correlation ID.
- Returns a value.
- Is independently testable, mockable, and replaceable.
- Logs its inputs, outputs, and timing as a structured event tagged with the correlation ID.

There is no framework binding the steps together. Control flow (try/except, conditional fallbacks, retries) uses standard Python constructs. A new step inserts as a new function call; an old step removes by deleting a line.

> [!instructor-review]
> Many existing RAG tutorials still teach composition through `LLMChain`, `RetrievalQA.from_chain_type(...)`, or LCEL `|` pipes as the central pattern. All three are pre-v1.0 framings. The blocklist entries `langchain-chain-class`, `langchain-lcel-pipe`, and `langchain-chaining-verb` capture this. The correct substitute in every case is plain Python sequential composition with `.invoke()`. If a learner's `/web-research` results land on a v0.x tutorial, the correct response is a plain-Python translation — not a "modernize this LCEL chain" exercise that keeps the `|` operator.

### Correlation IDs as the cross-stage primitive

A correlation ID is a single identifier (a UUID, a `traceparent` value from W3C Trace Context, a domain-specific request ID) that travels with the request through every stage and gets attached to every log entry, every metric label, every database query, every downstream service call.

Production correlation IDs typically come from one of three places:
1. The request enters the system with a trace ID already attached (an upstream gateway issued it, or the client did).
2. The system generates one at the ingress and propagates it downstream.
3. The OpenTelemetry SDK creates a span context at the entry boundary.

The discipline: every function that participates in handling the request accepts the correlation ID as a parameter (or reads it from a context variable) and includes it in every log message. The payoff is exact: a developer investigating a slow query searches for the correlation ID and gets every event in the request's lifecycle, ordered by time, across services. Without correlation IDs, debugging a distributed RAG pipeline is archeology.

### Span boundaries and OpenTelemetry

A *span* is the unit of work in distributed tracing. Each stage of the pipeline opens a span at start and closes it at end; child spans nest inside their parent. The result is a tree (a trace) that visualizes the full request lifecycle.

For RAG specifically, the standard per-query trace looks like:

```
rag.query                          (root span — full request)
├── rag.embed.query                (~50ms — embedding model)
├── rag.retrieve.vector            (~80ms — vector store)
│   └── store.read                 (~60ms — actual DB read)
├── rag.retrieve.bm25              (~30ms — sparse, when triggered)
├── rag.rerank                     (~200ms — cross-encoder)
├── rag.generate                   (~1500ms — LLM)
│   ├── llm.completion             (~1450ms — provider call)
│   └── llm.tokens                 (count, cost attributes)
└── rag.score.faithfulness         (~150ms — eval pass)
```

Each span carries attributes: `rag.chunks.retrieved=12`, `rag.tenant_id=...`, `rag.cache_hit=false`, `rag.faithfulness.score=0.91`. Production teams establish internal naming conventions and apply them consistently because the GenAI SIG's semantic conventions are still in draft.

### Observability tools on the spectrum

**Structured logs (JSON + log aggregator).** Lowest barrier: every stage emits a JSON event with the correlation ID and stage-specific attributes. Ingest into Datadog, Loki, Elasticsearch, CloudWatch, or any structured log store. Cheap, vendor-neutral, no SDK lock-in. Loses the cross-service "request as a tree" visualization but recovers it via log queries on `correlation_id`.

**OpenTelemetry spans (OTEL Collector + tracing backend).** Industry-standard distributed tracing. Spans flow to Jaeger, Honeycomb, Tempo, Datadog APM, or any OTLP backend. The application emits OTEL spans once; the Collector routes them. This is the right substrate for any pipeline with more than ~3 services.

**Managed LLM-observability platform (LangSmith, Arize Phoenix, Braintrust, Maxim, Helicone, etc.).** LLM-specific abstractions: prompt versions, evaluator outputs, dataset replays, A/B comparisons. LangSmith supports OTEL fanout — you emit OTEL once and route to both LangSmith and your general tracing backend. Production maturity level: when LLM-specific debugging (compare prompt v1 vs v2 on the same eval set, replay a regression) becomes a regular workflow.

### The "do we ship" dashboard

Four signals gate the standard release decision for a RAG pipeline change:

1. **Per-stage latency (p50, p95, p99).** Did the new code shift any stage's distribution? A regression hidden in the rerank step doesn't show up in end-to-end p95 alone.
2. **Per-stage cost.** Tokens in, tokens out, embeddings called, reranks invoked. A new pattern that improves quality at 3x the cost may not be a ship.
3. **Per-stage hit-rate.** What fraction of queries hit each cache layer? What fraction triggered each fallback path? A drop in retrieval-cache hit rate means the cache key or the corpus update logic changed.
4. **Refusal rate (and breakdown).** Total refusals; breakdown by cause (empty-vector, low-faithfulness, corpus-gap). A surge in low-faithfulness refusals is the canary for a generation regression.

The four together form the dashboard the eval-driven war-room Friday inspects. Without them the question *"do we ship?"* is unanswerable.

## 4. Generic Implementation

A traced RAG pipeline in plain Python with OpenTelemetry spans. Outside the federal-acquisitions domain — imagine an internal-knowledge search at a consulting firm.

```python
from opentelemetry import trace
import logging
import time
import json

tracer = trace.get_tracer("rag.pipeline")
log = logging.getLogger("rag.pipeline")

def rag_query(user_query: str, tenant_id: str) -> dict:
    with tracer.start_as_current_span("rag.query") as root:
        root.set_attribute("rag.tenant_id", tenant_id)
        root.set_attribute("rag.query_length", len(user_query))
        correlation_id = trace.format_trace_id(root.get_span_context().trace_id)

        # Embedding
        with tracer.start_as_current_span("rag.embed.query") as span:
            t0 = time.time()
            query_vec = embed(user_query)
            elapsed_ms = (time.time() - t0) * 1000
            span.set_attribute("rag.embed.model", "text-embedding-3-small")
            span.set_attribute("rag.embed.dim", len(query_vec))
            log.info(json.dumps({
                "event": "rag.embed.query",
                "correlation_id": correlation_id,
                "tenant_id": tenant_id,
                "latency_ms": round(elapsed_ms, 1),
            }))

        # Retrieve
        with tracer.start_as_current_span("rag.retrieve.vector") as span:
            t0 = time.time()
            candidates = vector_search(query_vec, tenant_id, k=20)
            elapsed_ms = (time.time() - t0) * 1000
            span.set_attribute("rag.chunks.retrieved", len(candidates))
            log.info(json.dumps({
                "event": "rag.retrieve.vector",
                "correlation_id": correlation_id,
                "tenant_id": tenant_id,
                "chunks_retrieved": len(candidates),
                "latency_ms": round(elapsed_ms, 1),
            }))

        # Rerank, generate, score — same shape
        reranked = rerank(user_query, candidates, correlation_id)
        answer = generate(user_query, reranked, correlation_id)
        faithfulness = score_faithfulness(answer, reranked, correlation_id)

        root.set_attribute("rag.faithfulness.score", faithfulness)
        return {
            "answer": answer,
            "faithfulness": faithfulness,
            "correlation_id": correlation_id,
        }
```

Notice the absence of any framework abstraction. The OpenTelemetry SDK provides the span primitives; the logging library provides the structured event primitives; the rest is plain Python.

## 5. Real-world Patterns

**E-commerce semantic search.** A large electronics retailer described their RAG search backbone in a 2026 engineering blog: every stage emits OTEL spans into the OTLP Collector, which routes to Honeycomb for general-purpose debugging and to Arize Phoenix for LLM-specific analysis. The dual-fanout pattern (OTEL Collector → multiple backends) was a deliberate decision to avoid lock-in while still getting LLM-specific tooling. Their dashboards track p50/p95/p99 per stage; an alert on the rerank stage's p99 was the first signal in a recent regression that traced to a Cohere API performance dip.

**Fintech research assistant.** A fintech-research team described their migration from raw print-statement debugging to structured JSON logs with correlation IDs as the most impactful change in their first year of production RAG. The change was non-architectural — every existing function got a `correlation_id` parameter, every existing log line became a JSON event — and unlocked the team's ability to investigate user-reported issues by query ID rather than guessing. They added OTEL spans 9 months later as the team scaled.

**Healthcare clinical-search observability.** A health-tech startup running a clinical-literature search tool described an explicit policy that *no* user-identifying or patient-identifying tokens land in span attributes — only structured metadata (query length, tenant ID, chunk count, faithfulness score). The discipline was driven by HIPAA scope: their observability pipeline was not in-scope for PHI handling, so the data going in had to be free of PHI. They achieved this by sanitizing at the span-creation site, not after the fact.

**Gaming player-support knowledge base.** A multiplayer-game studio's customer-support RAG used LangSmith as the LLM-specific tracing backend, with OTEL fanout to Datadog for general infrastructure tracing. The LangSmith UI was the daily-driver for prompt-version comparisons and replaying problematic queries against a new model; Datadog was the daily-driver for infrastructure regressions and on-call response. Same OTEL emission feeding both.

## 6. Best Practices

- Compose pipelines as plain Python with each component called via `.invoke()` (or async equivalent). The framework provides the per-step toolkit; your code is the orchestration shape.
- Propagate a correlation ID through every stage. Make it a function parameter or a context variable; do not rely on implicit propagation through globals.
- Open one span per logical stage; nest spans for sub-operations. Use a consistent naming convention (`rag.retrieve.vector`, `rag.rerank`, `rag.generate`) before the GenAI SIG settles the standard.
- Structure log events as JSON with `correlation_id`, `tenant_id`, `event`, `latency_ms`, and stage-specific attributes. Hand-formatted strings lose the cross-event query capability.
- Sanitize span attributes for sensitive data at the emission site, not in post-processing. Observability pipelines are not always in-scope for the same compliance regime as the application.
- Emit OTEL once and fan out via the Collector to multiple backends rather than coupling the application to any single vendor.
- Build the four-signal dashboard (per-stage latency, cost, hit-rate, refusal rate) before you build features that depend on it being there. Friday's war-room expects it.

## 7. Hands-on Exercise

**Code task (15 min).** Refactor a small RAG pipeline (use the example from this reading or write one) to:

1. Accept or generate a correlation ID at the entry function.
2. Pass the correlation ID through every stage.
3. Emit a structured JSON log event at the start and end of each stage with `event`, `correlation_id`, `latency_ms`, and at least one stage-specific attribute.
4. Compose all stages as plain Python function calls — no `Chain` class, no LCEL `|` pipe operator, no `.run()` entry point.

**What good looks like.** Every function signature includes `correlation_id`. Every log entry is a single JSON object with consistent keys across stages. Latency is measured per stage. The composition is a sequence of variable assignments with normal control flow (try/except for fallbacks). A grep on `correlation_id=<value>` recovers the full request trace from the logs. No v0.x LangChain primitives appear anywhere in the orchestration code.

## 8. Key Takeaways

- *What is the v1.0-correct way to compose a RAG pipeline?* — Plain Python function calls with each component invoked via `.invoke()`. No `Chain` class, no LCEL pipe as foundation, no `.run()` entry points.
- *Why is the correlation ID load-bearing?* — Because it is the only primitive that ties all events from a single request together across services and stages. Without it, debugging a distributed pipeline is archeology.
- *What is the standard observability stack in 2026?* — Structured JSON logs as the baseline, OpenTelemetry spans for distributed tracing, optionally a managed LLM-observability platform for LLM-specific workflows. OTEL Collector fanouts to multiple backends.
- *What four signals gate a "do we ship" decision?* — Per-stage latency (p50/p95/p99), per-stage cost, per-stage hit-rate, refusal rate with cause breakdown.
- *How do you handle PHI/PII in observability?* — Sanitize at the span-creation site, not after the fact. Observability pipelines may be outside the application's compliance scope.

## Sources

1. [LangChain and LangGraph Agent Frameworks Reach v1.0 Milestones (LangChain Blog)](https://blog.langchain.com/langchain-langgraph-1dot0/) — retrieved 2026-05-26
2. [Trace with OpenTelemetry (LangSmith Docs)](https://docs.langchain.com/langsmith/trace-with-opentelemetry) — retrieved 2026-05-26
3. [How to Implement RAG Pipeline Tracing with OpenTelemetry (OneUptime, Feb 2026)](https://oneuptime.com/blog/post/2026-02-06-rag-pipeline-tracing-opentelemetry/view) — retrieved 2026-05-26
4. [What is RAG Observability? Tracing Retrieval in 2026 (FutureAGI)](https://futureagi.com/blog/what-is-rag-observability-2026) — retrieved 2026-05-26
5. [RAG Monitoring & Alerting | Distributed Systems (APX ML)](https://apxml.com/courses/large-scale-distributed-rag/chapter-5-orchestration-operationalization-large-scale-rag/monitoring-logging-alerting-distributed-rag) — retrieved 2026-05-26

Last verified: 2026-05-26
