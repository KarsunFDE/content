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
last_verified: 2026-05-26
---

# Cost + latency management — per-node instrumentation

## 1. Learning Objectives

By the end of this reading, the learner can:

- Explain why per-node instrumentation (not per-flow) is the right granularity for cost and latency observability in a multi-step LLM workflow.
- Apply the OpenTelemetry GenAI Semantic Conventions to attach standard token-usage and operation-duration metrics to each node.
- Pick reasonable alert thresholds for per-evaluation cost and per-node p95 latency, and justify the choices.
- Explain why fan-out workflows amplify cost by N and how to size budgets accordingly.
- Identify which cost signals indicate a prompt problem versus an architectural problem (and what to do about each).

## 2. Introduction

It is easy to ship an LLM workflow whose unit economics nobody knows. The first month's bill is the surprise; the second month's bill, if traffic has scaled, is the crisis. The fix is not "watch the bill more carefully" — by the time the bill shows up, the spending has already happened, and the aggregation is at the wrong granularity to act on. The fix is per-step instrumentation: every model call, every node, every flow exposes its cost and latency as a first-class observable, so you can answer questions like "which node is responsible for half the bill" and "which step's p95 latency tripled last week" without exporting CSVs out of a billing console.

This problem space is mature in non-LLM observability — distributed traces, span-level latency histograms, RED metrics (Rate, Errors, Duration). The novel piece for LLM workflows is **cost as a first-class signal alongside latency**, because LLM calls have variable cost-per-call driven by token counts. The OpenTelemetry community has standardised the attribute names for this through the GenAI Semantic Conventions ([source: opentelemetry.io semconv gen-ai, retrieved 2026-05-26](https://opentelemetry.io/docs/specs/semconv/gen-ai/)); using those names — rather than inventing your own — buys you tool interoperability and avoids the worst-case of "we have observability but the dashboards do not stitch together."

This reading is the generic foundation: what to instrument, what to alert on, and what the signals tell you when they fire.

## 3. Core Concepts

### Why per-node, not per-flow

A single workflow invocation may make 4N+2 LLM calls — N parallel evaluators, two aggregators, two router decisions. The total per-flow cost is the sum of those, but the *cost driver* is rarely uniform. Usually one node dominates: the consensus aggregator that re-reads every input, the router that uses a very long system prompt, the evaluator that processes large documents. Aggregating only at the flow level hides which step you should be optimising.

The same shape applies to latency. A flow's p95 latency is the sum of step latencies on the slowest path; a single 3-second step can dominate. Without per-node breakdowns, the right optimisation target is invisible.

### OpenTelemetry GenAI semantic conventions

The OpenTelemetry GenAI semantic conventions ([source: opentelemetry.io semconv gen-ai, retrieved 2026-05-26](https://opentelemetry.io/docs/specs/semconv/gen-ai/)) define the namespace and attribute names for LLM observability. The headline metrics ([source: opentelemetry.io gen-ai-metrics, retrieved 2026-05-26](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-metrics/)):

- **`gen_ai.client.token.usage`** — histogram of input + output tokens per operation. Recommended when token counts are available.
- **`gen_ai.client.operation.duration`** — wall-clock duration of the operation.
- **`gen_ai.client.operation.time_to_first_chunk`** — for streaming responses, the time to the first chunk.
- **`gen_ai.client.operation.time_per_output_chunk`** — for streaming, the inter-chunk interval.

The headline attribute names you attach to spans and metrics:

- **`gen_ai.operation.name`** — what kind of operation (`chat`, `generate_content`, `text_completion`).
- **`gen_ai.provider.name`** — which provider (`openai`, `aws.bedrock`, `gcp.vertex_ai`).
- **`gen_ai.request.model`** — the model identifier requested.
- **`gen_ai.token.type`** — `input` or `output` (so the same metric can be split by type).

If you are already in LangSmith, you get most of this for free — runs already capture token counts and durations. The point of the OTel naming is portability: if you later route metrics into Prometheus or Datadog or Honeycomb, the attribute names are the same and the dashboards keep working.

### Token-usage histograms over averages

Tokens-per-call follow a long-tailed distribution — most calls are small, occasional calls are very large (a user pasted a 100k-token document). Averages hide the tail. The OTel spec specifies an explicit-bucket histogram for `gen_ai.client.token.usage` with buckets at `[1, 4, 16, 64, 256, 1024, 4096, 16384, 65536, 262144, 1048576, ...]` precisely because the distribution is logarithmic.

The operational rule: **alert on p95 / p99 of token usage, not on the mean.** The mean usually looks fine right up until the moment the bill arrives.

### Two alert categories: cost ceilings and latency p95s

The two operational alerts worth defining up front:

1. **Per-flow cost ceiling.** If a single invocation's projected cost exceeds $X, halt and escalate to a human before continuing. The ceiling reflects business risk tolerance — a runaway loop on a $50-per-flow budget is a meaningfully different problem from one on a $0.50 budget.
2. **Per-node p95 latency threshold.** If a node's p95 latency exceeds a threshold, surface it. A consensus-aggregator p95 > 60s is almost always a prompt-size or context-engineering problem, not infrastructure.

> [!instructor-review]
> *Bedrock per-token pricing is hot-tech category (3-month recency). The cohort-#1 cost ceiling band needs re-verification against current Bedrock pricing during the Sunday-before pass — pricing has shifted multiple times in the last 6 months across the Claude family on Bedrock. Verify via /web-research against the AWS Bedrock pricing page before the Thu session and supply the pair-decision band at that point. Do not cite a specific dollar figure from this reading.*

### Cost as a debugging signal

A node whose cost-per-invocation is increasing over time is telling you something. Usually one of three things:

- **Prompt drift** — the system prompt or template grew over time as features were added; the token cost grew with it. The fix is prompt-engineering: trim what is unused.
- **Context bloat** — the node receives more context than it needs (the whole conversation when only the last turn matters). The fix is context-engineering: shrink the input.
- **Architectural** — the node is doing work that should be split into two nodes (e.g., a single mega-aggregator that should be a tree of pairwise aggregators). The fix is graph restructuring.

The signal is the same; the right intervention differs. The discipline is to *look* at the cost trajectory per node, not to settle for "the model is expensive."

### Latency p95s as a context-window signal

For LLM calls, latency is roughly proportional to input + output tokens. A node whose p95 latency creeps up usually has a corresponding token-count creep. Plotting per-node p95 latency alongside per-node p95 input tokens makes the correlation visible. When they move together, it is a token-count problem. When latency moves but tokens do not, it is a provider-side throttling or queueing problem — different fix.

### Fan-out cost multiplication

The cost surprise that bites teams switching from single-agent to multi-agent designs is the fan-out multiplier. A flow that was 2 calls/invocation becomes 4N + 2 calls/invocation. For N = 5, that is 22 calls — 11x more. The unit economics need to be re-evaluated when the architecture changes; an evaluator-flow that is profitable at 2 calls may not be at 22.

The mitigation patterns:

- **Smaller models on workers, larger model on aggregator.** Workers do the bulk of the calls; aggregator does the synthesis. Spending the cheap-model budget on the workers and the expensive-model budget on the aggregator usually wins on cost-per-flow.
- **Caching of repeated context.** If every worker sees the same 4k-token system prompt, providers' prompt-caching features can collapse that to a fraction.
- **Sampling.** Not every flow needs full fan-out. Use the wide pattern only when a quick-and-narrow path is uncertain.

## 4. Generic Implementation

A non-Karsun example: a customer-support triage flow that uses a small model to classify the incoming ticket, fans out to N parallel sub-agents (one per relevant knowledge base), and aggregates to a single response. The example shows how to instrument each node with OTel GenAI attribute names so the spans are usable in any backend.

```python
from opentelemetry import trace, metrics
from opentelemetry.trace import SpanKind
from langgraph.graph import StateGraph, START, END
from typing import Annotated, TypedDict
from operator import add

tracer = trace.get_tracer(__name__)
meter = metrics.get_meter(__name__)

# Standard OTel GenAI histograms.
token_usage_histogram = meter.create_histogram(
    name="gen_ai.client.token.usage",
    unit="{token}",
    description="Tokens used per LLM call",
)
operation_duration_histogram = meter.create_histogram(
    name="gen_ai.client.operation.duration",
    unit="s",
    description="Wall-clock duration of LLM operation",
)

class TriageState(TypedDict):
    ticket_id: str
    text: str
    category: str | None
    kb_results: Annotated[list, add]
    response: str | None

def call_llm_with_instrumentation(operation_name: str, model_id: str, prompt: str) -> dict:
    with tracer.start_as_current_span(
        operation_name,
        kind=SpanKind.CLIENT,
        attributes={
            "gen_ai.operation.name": "chat",
            "gen_ai.provider.name": "anthropic",
            "gen_ai.request.model": model_id,
        },
    ) as span:
        start = time.monotonic()
        result = provider.chat(model=model_id, prompt=prompt)
        duration = time.monotonic() - start

        # Attach usage attributes to the span.
        span.set_attribute("gen_ai.usage.input_tokens", result["usage"]["input_tokens"])
        span.set_attribute("gen_ai.usage.output_tokens", result["usage"]["output_tokens"])

        # Emit histograms for cross-backend dashboards.
        token_usage_histogram.record(
            result["usage"]["input_tokens"],
            attributes={"gen_ai.operation.name": "chat", "gen_ai.request.model": model_id,
                        "gen_ai.token.type": "input"},
        )
        token_usage_histogram.record(
            result["usage"]["output_tokens"],
            attributes={"gen_ai.operation.name": "chat", "gen_ai.request.model": model_id,
                        "gen_ai.token.type": "output"},
        )
        operation_duration_histogram.record(duration,
            attributes={"gen_ai.operation.name": "chat", "gen_ai.request.model": model_id})

        return result

def classify_ticket(state: TriageState) -> dict:
    # Cheaper model for the classifier — high call rate, simple decision.
    out = call_llm_with_instrumentation("classify", "small-fast-model", state["text"])
    return {"category": out["text"]}

# (fan-out, worker, aggregator nodes follow the same instrumentation pattern,
# using larger models only where the work justifies it)
```

Three things to notice in the wiring:

- The attribute names match the OTel spec exactly. Any compliant backend can render dashboards from these spans without custom configuration.
- Both spans (for span-level inspection) and metrics (for aggregated dashboards) are emitted. They serve different queries.
- The classifier uses a small model deliberately. Selecting model-by-node-purpose is one of the cheapest interventions on flow cost.

The same pattern scales to any multi-step LLM workflow regardless of domain.

## 5. Real-world Patterns

**Fintech — Plaid's per-endpoint cost dashboards.** Plaid publishes per-endpoint latency and error-rate dashboards to customers as a contractual artifact. Internal versions split by upstream bank and by model version. The pattern that maps to LLM workflows: customers care about the *user-facing* operation's latency, but engineers debug at the *step* level. Both views need to exist.

**E-commerce — Shopify's per-shop checkout latency dashboards.** Shopify monitors checkout latency at multiple percentiles, split by shop tier and region. A regression at p99 in a single region is detected faster than a regression in the global mean — the lesson is the same one OTel pushes for token-usage histograms. Averages hide everything.

**Gaming — Riot Games' match-server cost analytics.** Riot's match-making and match-server fleet exposes per-region cost-per-match metrics. When a code change increases the cost per match by 5%, the change is detected in days. The discipline that transfers: every unit of work has a cost-per-unit metric, every metric has an owner, and changes that move the metric trigger investigation. Translate "match" to "workflow invocation" and the rule is identical.

**Healthcare — radiology AI vendors' cost-per-study tracking.** Vendors that resell radiology AI as a managed service track cost-per-study (model inference + storage + transmission) and price-per-study to maintain a margin. When a model upgrade increases inference cost by 20%, the price either rises or the company eats the margin loss. The relevant pattern is the maturity of "we know what every unit costs" — most LLM workflow shops are not there yet, and a bad surprise per quarter is the consequence.

## 6. Best Practices

- **Use the OpenTelemetry GenAI attribute names from the start.** Renaming attributes later is painful; tooling already understands the spec names.
- **Emit both spans and metrics.** Spans support trace-level inspection; metrics support aggregated dashboards. They serve different queries.
- **Alert on p95 and p99, not on means.** Long-tailed distributions need percentile alerts.
- **Track cost per node, not per flow.** Per-flow numbers hide which step you should optimise.
- **Pick model-per-node by job.** Workers can use smaller models; aggregators use the bigger one. Mixing is cheaper than uniform.
- **Plot latency p95 against input-token p95 per node.** When they move together it is a token problem; when they diverge it is a provider problem.
- **Set the per-flow cost ceiling deliberately and revisit it on every architecture change.** A multi-agent rewrite that 10x's the call count without a budget update is a quiet bill bomb.

## 7. Hands-on Exercise

**Code task (15 min).** Add OTel-spec instrumentation to a three-node graph (classify → enrich → respond). For each LLM call:

1. Open a span named after the operation with `gen_ai.operation.name`, `gen_ai.provider.name`, `gen_ai.request.model` attributes.
2. After the call, attach `gen_ai.usage.input_tokens` and `gen_ai.usage.output_tokens` to the span.
3. Emit the same usage values to a `gen_ai.client.token.usage` histogram with `gen_ai.token.type` distinguishing input from output.
4. Wire the OTel SDK to log spans + metrics to stdout (the console exporter is fine for this exercise).
5. Run the graph once and confirm both spans and metrics appear in the console output.

**What good looks like.** Each node's span shows up with the full set of GenAI attributes. The token-usage histogram records two values per LLM call (one input, one output) with the `gen_ai.token.type` attribute distinguishing them. A weak answer either invents attribute names ("input_tokens" without the namespace) or emits only spans without metrics — both work locally but neither composes with downstream tooling.

## 8. Key Takeaways

- Why is per-node instrumentation the right granularity, and what does per-flow aggregation hide? (LO1)
- Which OTel GenAI attribute names should I use, and what does each tell me? (LO2)
- What are reasonable shapes for a per-flow cost ceiling and a per-node p95 latency threshold, and why are means insufficient? (LO3)
- How does fan-out amplify cost, and which mitigation patterns reduce the multiplier? (LO4)
- When per-node cost climbs over time, which interventions does each cause (prompt drift / context bloat / architecture) suggest? (LO5)

## Sources

1. [OpenTelemetry Semantic Conventions for Generative AI (overview)](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — retrieved 2026-05-26
2. [OpenTelemetry Semantic Conventions for Generative AI metrics](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-metrics/) — retrieved 2026-05-26
3. [LangSmith Observability concepts — runs, traces, metadata](https://docs.smith.langchain.com/observability/concepts) — retrieved 2026-05-26
4. [LangGraph Persistence — super-step boundaries, checkpoint emission points](https://docs.langchain.com/oss/python/langgraph/persistence) — retrieved 2026-05-26

Last verified: 2026-05-26
