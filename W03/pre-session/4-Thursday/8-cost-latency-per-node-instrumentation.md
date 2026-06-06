---
week: W03
day: Thu
topic_slug: cost-latency-per-node-instrumentation
topic_title: "Cost + latency management — per-node instrumentation"
parent_overview: W03/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://opentelemetry.io/docs/specs/semconv/gen-ai/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-metrics/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.smith.langchain.com/observability/concepts
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/oss/python/langgraph/persistence
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-06-06
---

# Cost + latency management — per-node instrumentation

> [!NOTE]
> **From earlier:** Wed's multi-agent topology introduced fan-out at the architecture level. Today fan-out becomes a cost multiplier — what was 2 Bedrock calls per evaluation is now 4N+2. Wire per-node instrumentation before the call count surprises you on the bill.

## 1. Learning Objectives

By the end of this reading, you can:

- Explain why per-node instrumentation (not per-flow) is the right granularity for cost and latency observability.
- Apply OpenTelemetry GenAI Semantic Convention attribute names to each node's LLM calls.
- Set defensible alert thresholds for per-flow cost and per-node p95 latency.
- Explain how fan-out multiplies cost and which mitigation patterns reduce the multiplier.
- Identify which cost-climbing signals indicate a prompt problem versus an architecture problem.

## 2. Introduction

It is easy to ship an LLM workflow whose unit economics nobody knows. The first month's bill is the surprise; the second month's at scale is the crisis. The fix is per-step instrumentation: every model call exposes cost and latency as first-class observables. A monthly bill arrives at the wrong granularity to act on.

The novel piece for LLM workflows is **cost as a first-class signal alongside latency**. The OpenTelemetry GenAI Semantic Conventions ([source: opentelemetry.io semconv gen-ai, retrieved 2026-05-26](https://opentelemetry.io/docs/specs/semconv/gen-ai/)) standardise the attribute names; using them buys tool interoperability. Wire this today — W5 AIOps builds cost-as-signal into auto-remediation.

## 3. Core Concepts

### 3.1 Why per-node, not per-flow

A single invocation may make 4N+2 LLM calls. The total per-flow cost is the sum, but the *cost driver* is rarely uniform — usually one node dominates: the consensus aggregator re-reading every input, a router with a long system prompt, an evaluator on large documents. Per-flow aggregation hides which step to optimise. Same for latency: one 3-second node dominates the p95 without per-node breakdown.

### 3.2 OpenTelemetry GenAI semantic conventions

The OTel GenAI conventions ([source: opentelemetry.io gen-ai-metrics, retrieved 2026-05-26](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-metrics/)) define:

| Metric / Attribute | What it captures |
|--------------------|-----------------|
| `gen_ai.client.token.usage` | Histogram of input + output tokens per operation |
| `gen_ai.client.operation.duration` | Wall-clock duration of the LLM operation |
| `gen_ai.operation.name` | Operation type (`chat`, `generate_content`) |
| `gen_ai.provider.name` | Provider (`aws.bedrock`, `openai`) |
| `gen_ai.request.model` | Model identifier requested |
| `gen_ai.token.type` | `input` or `output` — splits the usage histogram |
| `gen_ai.usage.input_tokens` | Input token count (span attribute) |
| `gen_ai.usage.output_tokens` | Output token count (span attribute) |

In LangSmith, token counts and durations are captured automatically. The OTel naming matters for portability: route metrics to Prometheus or Datadog later and dashboards keep working.

> [!TIP]
> **Use OTel GenAI attribute names from the start.** Renaming later is painful — tooling already understands the spec names and dashboards built on standard names survive provider migrations.

### 3.3 Token-usage histograms over averages

Token counts follow a long-tailed distribution. The OTel spec mandates explicit-bucket histograms for `gen_ai.client.token.usage` because the distribution is logarithmic — averages hide the tail.

> [!IMPORTANT]
> **Alert on p95/p99, not the mean.** A single evaluator processing a 50-page proposal has a low mean but a p99 that is 20× the median — the mean gives no warning until the bill arrives.

### 3.4 Fan-out cost multiplication and cost-as-signal

Fan-out turns 2 calls/invocation into 4N+2 — 11× at N=5. Mitigations: smaller models on workers + larger on aggregator; Bedrock prompt-caching on repeated system prompts; sampling for high-confidence cases.

Rising per-node cost signals: **prompt drift** (system prompt grew); **context bloat** (node receives more than it needs); **architectural** (mega-aggregator should be a tree of pairwise nodes). Plot per-node cost trajectory; "the model is expensive" is not actionable.

## 4. Generic Implementation

Customer-support triage flow with OTel-spec instrumentation:

```python
from opentelemetry import trace, metrics
from opentelemetry.trace import SpanKind

tracer = trace.get_tracer(__name__)
meter = metrics.get_meter(__name__)

token_usage = meter.create_histogram(
    name="gen_ai.client.token.usage",
    unit="{token}",
    description="Tokens used per LLM call",
)
op_duration = meter.create_histogram(
    name="gen_ai.client.operation.duration",
    unit="s",
    description="Wall-clock duration of LLM operation",
)

def call_llm_instrumented(op_name: str, model_id: str, prompt: str) -> dict:
    with tracer.start_as_current_span(
        op_name,
        kind=SpanKind.CLIENT,
        attributes={
            "gen_ai.operation.name": "chat",
            "gen_ai.provider.name": "aws.bedrock",
            "gen_ai.request.model": model_id,
        },
    ) as span:
        import time
        t0 = time.monotonic()
        result = bedrock_client.invoke_model(model_id=model_id, prompt=prompt)
        elapsed = time.monotonic() - t0
        span.set_attribute("gen_ai.usage.input_tokens", result["usage"]["input_tokens"])
        span.set_attribute("gen_ai.usage.output_tokens", result["usage"]["output_tokens"])
        token_usage.record(result["usage"]["input_tokens"],
            attributes={"gen_ai.request.model": model_id, "gen_ai.token.type": "input"})
        token_usage.record(result["usage"]["output_tokens"],
            attributes={"gen_ai.request.model": model_id, "gen_ai.token.type": "output"})
        op_duration.record(elapsed, attributes={"gen_ai.request.model": model_id})
        return result
```

Attribute names match the OTel spec exactly; both spans and metrics are emitted; the classifier uses a small model. Selecting model-by-node-purpose is one of the cheapest cost interventions available.

## 5. Real-world Patterns

**Fintech — Plaid per-endpoint dashboards.** Plaid publishes per-endpoint latency and error-rate dashboards as a contractual artifact. Customers care about user-facing latency; engineers debug at step level. Both views must exist.

> [!NOTE]
> **Cross-domain lesson:** Per-unit cost metrics with an owner — cost-per-endpoint, cost-per-invocation — make a 5% regression detectable in days rather than months. The pattern scales from gaming match servers to fintech APIs to LLM workflows.

**E-commerce — Shopify checkout percentiles.** Shopify monitors checkout at p50/p95/p99 by shop tier and region. A p99 regression in one region is detected faster than a global-mean regression — the same lesson OTel applies to token-usage histograms.

## 6. Best Practices

- **Use OTel GenAI attribute names from the start.**
- **Emit both spans and metrics.** Spans for trace-level inspection; metrics for aggregated dashboards.
- **Alert on p95/p99, not means.**
- **Track cost per node, not per flow.**
- **Pick model-per-node by job.** Workers use smaller models; the aggregator uses the bigger one.

> [!WARNING]
> **Anti-pattern: `cost-tracking-as-afterthought`.** Teams defer instrumentation until "after the feature ships." By then the architecture is locked, call counts are in production, and the first bill has already arrived. Wiring OTel GenAI attributes is a 20-line addition per node — do it before the PR merges. Bedrock pricing is hot-tech (3-month recency window); verify current per-token pricing via `/web-research` before the Sunday-before-W3 pass. Do not hardcode a dollar figure.

## 7. Hands-on Exercise

Add OTel-spec instrumentation to a three-node graph (classify → enrich → respond): open a span per LLM call with `gen_ai.operation.name`, `gen_ai.provider.name`, `gen_ai.request.model`; attach `gen_ai.usage.input_tokens` and `gen_ai.usage.output_tokens`; emit to a `gen_ai.client.token.usage` histogram with `gen_ai.token.type` splitting input/output; wire OTel SDK to stdout; confirm spans and metrics appear.

> [!NOTE]
> **Self-check** (30 s — answer mentally before expanding)
>
> 1. The consensus aggregator's p95 latency triples over two weeks; its p95 input-token count also triples. What is the most likely cause and fix?
> 2. Your flow goes from 2 to 6 parallel evaluators. How does per-flow cost change, and which mitigation reduces the multiplier most cheaply?

<details>
<summary>Show answers</summary>

1. The latency and token count move together — this is a token-count problem, not a provider-side issue. The most likely cause is context bloat: the aggregator is receiving more input per invocation (accumulating conversation history or more proposals). The fix: trim the aggregator's input to the minimum — per-evaluator summaries instead of full transcripts.
2. From 2 calls to 6× calls + fixed overhead = roughly 3× the evaluator-call cost. The cheapest mitigation is prompt-caching on the repeated system prompt every evaluator sees. If every worker sees the same 4 k-token system prompt, caching collapses repeated input-token cost to near zero. Second cheapest: smaller model on workers, full model on the aggregator only.

</details>

## 8. Key Takeaways

- Per-node instrumentation reveals the cost driver; per-flow aggregation hides it.
- OTel GenAI attribute names are the standard — use them for portability.
- Alert on p95/p99 of token usage and latency; LLM cost distributions are long-tailed.
- Fan-out multiplies cost by N; mitigate with smaller models on workers, prompt-caching, and sampling.
- W5 AIOps builds cost-as-signal into auto-remediation — wire the foundation today.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://opentelemetry.io/docs/specs/semconv/gen-ai/ — retrieved 2026-05-26 — hot-tech
- https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-metrics/ — retrieved 2026-05-26 — hot-tech
- https://docs.smith.langchain.com/observability/concepts — retrieved 2026-05-26 — hot-tech
- https://docs.langchain.com/oss/python/langgraph/persistence — retrieved 2026-05-26 — hot-tech

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

**OTel histogram bucket selection for token usage.** The OTel GenAI spec recommends explicit buckets at `[1, 4, 16, 64, 256, 1024, 4096, 16384, 65536, 262144, 1048576, ...]` — logarithmic spacing because token counts span six orders of magnitude. If you configure a linear histogram instead, the 50-page-proposal tail cases all fall into the last bucket and p99 is unresolvable. Configure the buckets explicitly when initialising the OTel SDK's metric exporter; do not rely on defaults.

**Cost ceiling as a HITL gate.** A per-flow cost ceiling can be wired as a dynamic `interrupt()` inside the supervisor node: if projected cost for the current fan-out exceeds `$X`, interrupt before dispatching evaluators and surface the estimated cost to a human for approval. This is the same interrupt primitive from files 5–6, applied to a cost-control policy rather than a regulatory one. W5 AIOps formalises this as "cost as a signal for auto-remediation with escalation."

</details>

Last verified: 2026-06-06
