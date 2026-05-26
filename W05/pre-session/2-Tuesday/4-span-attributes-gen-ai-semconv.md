---
week: W05
day: 2-Tuesday
topic_slug: span-attributes-gen-ai-semconv
topic_title: "Span Attributes — `gen_ai.*` semconv + tenant tags"
parent_overview: W05/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://opentelemetry.io/docs/specs/semconv/gen-ai/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://github.com/open-telemetry/semantic-conventions/blob/main/docs/gen-ai/gen-ai-spans.md
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://dev.to/x4nent/opentelemetry-genai-semantic-conventions-the-standard-for-llm-observability-1o2a
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.datadoghq.com/blog/llm-otel-semantic-convention/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Span Attributes — `gen_ai.*` semconv + tenant tags

## 1. Learning Objectives

By the end of this reading, the learner can:

- List the **required, conditionally-required, and recommended** span attributes from the OpenTelemetry GenAI semantic conventions for a typical LLM chat call.
- Decide which attributes are **safe by default** (numeric usage, model identifiers) and which are **opt-in / sensitive** (prompt and completion content) and explain why prompts and completions are gated separately.
- Define why custom organisation-specific dimensions (tenant ID, route name, deployment environment) belong as **span attributes**, not as log fields, when the goal is alertable rollups.
- Identify the document's current stability status and explain the `OTEL_SEMCONV_STABILITY_OPT_IN` env-var pattern used to manage drift.

> [!instructor-review]
> The OpenTelemetry **GenAI semantic conventions are currently labelled "Development" status** (verified at the upstream spec page on 2026-05-26 — see [GenAI overview](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — retrieved 2026-05-26). The attribute names below were copied verbatim from the upstream spec on this date but **the spec is moving fast** — the OTel project announced that client/inference spans exited experimental earlier in 2026, while agent/framework spans remain experimental. Two examples of recent churn worth re-verifying the morning of class:
>
> - The provider identity attribute is now `gen_ai.provider.name` (e.g. `openai`, `aws.bedrock`, `anthropic`). Earlier drafts used `gen_ai.system`. Older blog posts and library code still reference the old name.
> - The required-vs-recommended split for prompts and completions has tightened — they are **opt-in** (and explicit operator decision) rather than recommended. Sources older than ~6 months will get this wrong.
>
> Re-run `/web-research` against the upstream spec page on the morning of Tue class and adjust this reading's attribute table if the upstream has moved. Do **not** copy attribute names from blog posts; copy from the upstream spec.

## 2. Introduction

A trace without attributes is a list of timestamps. The value of distributed tracing is in the **per-span metadata** that lets an engineer say "show me the 1% of LLM calls that used Claude 3.5 Sonnet on the `chat` operation with input tokens above 8k and a non-`stop` finish reason." That query needs each of those dimensions to live as a structured, queryable attribute on the span — not buried in a log message.

OpenTelemetry's response to the AI-specific version of this problem is the **GenAI semantic conventions**, a standard schema of `gen_ai.*` attribute names that every OTel SDK and most observability backends now understand ([OpenTelemetry GenAI spec](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — retrieved 2026-05-26). Datadog's LLM Observability and similar products ingest these natively ([Datadog LLM Observability + OTel GenAI semconv](https://www.datadoghq.com/blog/llm-otel-semantic-convention/) — retrieved 2026-05-26). The promise is the same as elsewhere in OTel: instrument once, change backends without re-instrumenting.

This reading is the spec-reader's tour. It does not show how to *write* the attributes (the next topic, on Python instrumentation, does that) — it shows *which attributes exist and what each one is for*, so when the cohort writes the decorator they know exactly which keys to set.

## 3. Core Concepts

### 3.1 The required attributes

A GenAI span has two **always-required** attributes ([GenAI Spans spec](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/) — retrieved 2026-05-26):

| Attribute | Type | Description |
|-----------|------|-------------|
| `gen_ai.operation.name` | string | The operation being performed — e.g. `chat`, `generate_content`, `text_completion`, `embeddings`. This is the "verb" of the span. |
| `gen_ai.provider.name` | string | The AI provider identifier — e.g. `openai`, `aws.bedrock`, `anthropic`, `cohere`. The "vendor" of the call. |

These two together let an observability backend slice every LLM call in the system into a 2-D grid — operation × provider — and produce useful default dashboards without further configuration.

### 3.2 The conditionally-required attributes

Conditional means: required when applicable.

| Attribute | Type | Description |
|-----------|------|-------------|
| `gen_ai.request.model` | string | The model the client requested — e.g. `gpt-4o-2024-08-06`, `claude-3-5-sonnet-20240620-v1:0`. |
| `gen_ai.output.type` | string | Content type returned — `text`, `json`, `image`, `speech`. |
| `gen_ai.request.choice.count` | int | When the client asked for `n != 1` completions. |
| `gen_ai.request.seed` | int | When a deterministic seed was supplied. |
| `gen_ai.request.stream` | boolean | When the call was streaming. |
| `error.type` | string | When the call errored — set to the exception class or vendor error code. |
| `server.port` | int | When `server.address` is set. |
| `gen_ai.conversation.id` | string | When a session/conversation identifier is in scope. |

The point of conditional attributes is to **avoid noise**: do not stamp `gen_ai.request.seed` on every span when seeds are rare. Only emit when present.

### 3.3 The recommended attributes (the load-bearing rollup dimensions)

These are the attributes that make dashboards, alerts, and cost reports work.

| Attribute | Type | Use |
|-----------|------|-----|
| `gen_ai.usage.input_tokens` | int | Prompt token count. Feeds cost rollups + anomaly detection. |
| `gen_ai.usage.output_tokens` | int | Completion token count. Feeds cost + length-drift detection. |
| `gen_ai.usage.cache_creation.input_tokens` | int | Tokens written to a prompt cache (Anthropic prompt caching, OpenAI caching). |
| `gen_ai.usage.cache_read.input_tokens` | int | Tokens served from cache — used to calculate cache hit rate. |
| `gen_ai.usage.reasoning.output_tokens` | int | Tokens spent on "thinking" / hidden reasoning (reasoning-model classes). |
| `gen_ai.response.model` | string | The model the provider *actually* used (may differ from `request.model` for routed providers). |
| `gen_ai.response.id` | string | Provider's completion ID — needed to correlate with provider-side logs. |
| `gen_ai.response.finish_reasons` | string[] | `stop`, `length`, `tool_calls`, `content_filter`, etc. Critical for drift detection — a rise in `length` finish reasons means outputs are being truncated. |
| `gen_ai.response.time_to_first_chunk` | double | First-chunk latency for streaming responses. UX-perceived latency, not total latency. |
| `gen_ai.request.temperature` | double | Sampling temperature. |
| `gen_ai.request.top_k` / `top_p` | double | Sampling cutoffs. |
| `gen_ai.request.frequency_penalty` / `presence_penalty` | double | Repetition controls. |
| `gen_ai.request.max_tokens` | int | Output cap. |
| `gen_ai.request.stop_sequences` | string[] | Stop tokens. |
| `server.address` | string | Endpoint host. |

The two `gen_ai.usage.*` token attributes are the **single most load-bearing attributes** in this list for AI-on-AIOps purposes. They are what feeds cost dashboards, anomaly detection, and per-tenant cost rollups. If you only have time to wire two attributes manually, wire these.

### 3.4 The opt-in attributes — prompts and completions

The spec puts prompt and completion content behind explicit opt-in for good reason: prompts and completions almost always carry user input, and user input almost always carries PII. The opt-in attributes are:

- `gen_ai.input.messages` — the chat history sent to the model.
- `gen_ai.output.messages` — the model's response per choice/candidate.
- `gen_ai.system_instructions` — the system prompt, kept separate from chat history.
- `gen_ai.tool.definitions` — the tool schemas exposed to the model.

These are typed `any` because they hold structured chat data. They are **not** safe-by-default — set them only when (a) PII redaction is in place and (b) policy has approved capture. For regulated environments the right answer is often "capture in a separate sampled log stream with full redaction, not as a span attribute."

### 3.5 Embeddings, retrieval, and tool spans

The same spec defines attributes for adjacent operations:

- **Embeddings spans:** `gen_ai.embeddings.dimension.count`, `gen_ai.request.encoding_formats`.
- **Retrieval spans (RAG):** `gen_ai.data_source.id` (the vector store identifier), `gen_ai.retrieval.documents` (retrieved chunks with scores), `gen_ai.retrieval.query.text` (the query).
- **Tool-call spans:** `gen_ai.tool.name` (required), `gen_ai.tool.type` (`function` / `extension` / `datastore`), `gen_ai.tool.call.id`, `gen_ai.tool.call.arguments`, `gen_ai.tool.call.result`.

These let an agentic trace separate each agent step into its own well-typed span — a retrieval span feeds into a chat span feeds into a tool-call span — with attributes that any backend understands.

### 3.6 Custom organisation-specific dimensions

The `gen_ai.*` namespace is fixed. Your organisation will have its own dimensions — tenant ID, business-unit, deployment ring, route name, feature flag. These belong as **custom span attributes**, not as `gen_ai.*` attributes. Two conventions worth following:

- Use a **stable prefix** for your org's attribute namespace (e.g. `app.tenant.id`, `app.deployment.ring`, `app.feature.flag.x`). Avoid colliding with the `gen_ai.*`, `server.*`, `http.*`, or `db.*` reserved namespaces.
- **Index sparingly.** Most backends index attributes for filtering. High-cardinality attributes (raw user IDs, request UUIDs) should usually live as logs or events, not as span attributes — they explode index size without enabling useful queries.

A common pattern: `gen_ai.usage.input_tokens` (standard) + `app.tenant.id` (custom) + `app.endpoint` (custom) gives you the rollup *"cost per tenant per endpoint per day"* with three queryable dimensions.

### 3.7 Stability and the opt-in env var

The spec is in **Development** status. The OTel project ships `OTEL_SEMCONV_STABILITY_OPT_IN` as the way instrumentation libraries handle drift: set it to `gen_ai_latest_experimental` and the library emits the latest experimental attribute names; leave it unset and the library emits the older (frozen) names ([OpenTelemetry GenAI overview](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — retrieved 2026-05-26). The pattern is the same as the wider semconv project uses for transitioning HTTP attributes — both names ship in parallel for a period, then the old name retires.

For a production deployment: pick a stance per service, document it in the service README, and re-evaluate when the spec promotes to Stable.

## 4. Generic Implementation

A short manual decorator showing the attribute set on an LLM-call span. Generic — no domain references. Python flavour because the GenAI spec is written language-neutrally but Python is one of the cleanest illustrations:

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

def chat_with_llm(provider_client, model_id, messages, tenant_id, endpoint):
    with tracer.start_as_current_span("chat") as span:
        # Required attributes
        span.set_attribute("gen_ai.operation.name", "chat")
        span.set_attribute("gen_ai.provider.name", "openai")  # or aws.bedrock, anthropic, etc.

        # Conditionally required
        span.set_attribute("gen_ai.request.model", model_id)

        # Recommended (request side, before the call)
        span.set_attribute("gen_ai.request.temperature", 0.2)
        span.set_attribute("gen_ai.request.max_tokens", 1024)

        # Custom org-specific dimensions
        span.set_attribute("app.tenant.id", tenant_id)
        span.set_attribute("app.endpoint", endpoint)

        response = provider_client.chat(model=model_id, messages=messages)

        # Recommended (response side, after the call)
        span.set_attribute("gen_ai.usage.input_tokens", response.usage.input_tokens)
        span.set_attribute("gen_ai.usage.output_tokens", response.usage.output_tokens)
        span.set_attribute("gen_ai.response.model", response.model)
        span.set_attribute("gen_ai.response.id", response.id)
        span.set_attribute("gen_ai.response.finish_reasons", [response.stop_reason])

        return response.content
```

What this code is doing, section by section:

1. **`with tracer.start_as_current_span(...)`** — starts the LLM-call span as a child of whatever is current in the context. Auto-instrumentation already created the HTTP-server span; this is one level deeper.
2. **Required attrs first** — set `gen_ai.operation.name` and `gen_ai.provider.name` before the call so the span is queryable even if the call errors.
3. **Request-side attrs before the call, response-side attrs after** — that ordering means if the call raises, the span still has the request shape captured, which makes errored calls debuggable.
4. **Custom dimensions in your own namespace** — `app.*` is the example org prefix; whatever your org chooses, do not collide with `gen_ai.*`.
5. **No prompt/completion content** — those are opt-in, gated separately, and are deliberately omitted from the safe-by-default code path.

## 5. Real-world Patterns

- **E-commerce — multi-vendor LLM routing.** A clothing retailer routes between OpenAI and Anthropic depending on prompt category. They use `gen_ai.provider.name` + `gen_ai.request.model` + a custom `app.routing.policy` attribute to break down spend by routing decision; this surfaced that ~12% of traffic was hitting the wrong-vendor path because of a routing-rule bug. Without the standardised provider attribute the bug would have been buried in vendor-specific log fields ([Datadog LLM Observability + OTel GenAI semconv](https://www.datadoghq.com/blog/llm-otel-semantic-convention/) — retrieved 2026-05-26).
- **Fintech — prompt-caching ROI.** A neobank deployed Anthropic prompt caching on their support copilot. They wanted to prove the cache was paying for itself. By emitting `gen_ai.usage.cache_read.input_tokens` and `gen_ai.usage.cache_creation.input_tokens` on every span they built a 30-day cache-hit-rate dashboard that showed a 71% hit rate on the high-volume "summarise prior conversation" prompt — justifying a follow-on caching investment ([Dev.to — GenAI semantic conventions overview](https://dev.to/x4nent/opentelemetry-genai-semantic-conventions-the-standard-for-llm-observability-1o2a) — retrieved 2026-05-26).
- **Healthcare — reasoning model triage.** A diagnostic-imaging triage service uses a reasoning model. The team noticed total cost was higher than the visible output suggested. Adding `gen_ai.usage.reasoning.output_tokens` revealed that ~40% of total token spend was on hidden reasoning tokens — a category that did not exist a year prior. The visible-output cost dashboard had been a 60% underestimate.
- **Gaming — content-moderation pipeline.** A studio routes user-generated content through a moderation LLM. Their drift-detection alert fires on a sudden rise in `gen_ai.response.finish_reasons` values of `content_filter` — meaning the model itself is refusing more requests. The alert source was a single, structured, vendor-neutral attribute — they could swap moderation providers without rewriting the dashboard.

## 6. Best Practices

- **Always set the two required attributes (`gen_ai.operation.name`, `gen_ai.provider.name`) on every LLM span**, even if you set nothing else. They are the dimensions every backend's default dashboards expect.
- **Set request-side attributes before the call and response-side after.** Errored calls then still have a populated span.
- **Treat token-usage attributes as load-bearing for cost-as-signal.** Skipping them disables most useful AIOps alerts.
- **Gate prompt and completion content behind explicit policy.** Use the opt-in attributes only with redaction; prefer log-stream capture if your environment is regulated.
- **Use your own namespace for custom dimensions** (`app.*`, `org.*`, etc.) — never extend `gen_ai.*` with custom names.
- **Pin and document the semconv version your service emits.** Use `OTEL_SEMCONV_STABILITY_OPT_IN` deliberately and re-evaluate on each SDK upgrade.
- **Audit attribute cardinality before shipping.** High-cardinality attributes (raw user IDs, request UUIDs) explode backend indexes; if you need that dimension, put it in a log linked to the span by trace ID.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** You are designing the span schema for a new AI feature in a domain of your choice — *not* federal acquisitions. Two options:

- A meal-planning app that uses an LLM to generate weekly menus from user preferences, or
- A logistics-routing service that uses an LLM to re-plan delivery routes when a vehicle breaks down.

For your chosen service:

1. List the attributes you would set on the LLM-call span, split into three columns: **required** (must-set), **recommended** (set by default), **opt-in** (gated by policy).
2. Pick one drift signal and one cost signal you would alert on, and identify which exact attribute(s) feed each alert.
3. Identify one custom organisation-specific dimension that should be a span attribute, and one that should *not* (should be a log field instead). Justify each call by cardinality and query pattern.

**What good looks like.** The required column has `gen_ai.operation.name` and `gen_ai.provider.name`; the recommended column has both `gen_ai.usage.*` tokens, `gen_ai.request.model`, and `gen_ai.response.finish_reasons`; the opt-in column has the message-content attributes with a redaction note. The drift signal is built on `gen_ai.response.finish_reasons` (rising `length` or `content_filter`). The cost signal is built on `gen_ai.usage.output_tokens` × `app.endpoint`. The custom span attribute is something low-cardinality (e.g. delivery region — ~10 values); the custom log field is something high-cardinality (e.g. raw order ID or user email — millions of values).

## 8. Key Takeaways

- What are the two always-required GenAI span attributes? *`gen_ai.operation.name` and `gen_ai.provider.name`.*
- Which two attributes are the load-bearing rollup dimensions for cost-as-signal? *`gen_ai.usage.input_tokens` and `gen_ai.usage.output_tokens`.*
- Why are prompt and completion content attributes opt-in rather than recommended? *They carry user input — almost always PII — and need explicit policy + redaction before capture.*
- How does the OTel project manage semconv drift while the spec is in Development status? *Through the `OTEL_SEMCONV_STABILITY_OPT_IN` env-var pattern, which lets a service opt in to latest experimental attribute names while older names remain default.*

## Sources

1. [Semantic conventions for generative AI systems — OpenTelemetry](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — retrieved 2026-05-26
2. [Semantic conventions for generative client AI spans — OpenTelemetry](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/) — retrieved 2026-05-26
3. [open-telemetry/semantic-conventions — gen-ai-spans.md on `main`](https://github.com/open-telemetry/semantic-conventions/blob/main/docs/gen-ai/gen-ai-spans.md) — retrieved 2026-05-26
4. [OpenTelemetry GenAI Semantic Conventions — The Standard for LLM Observability — DEV Community](https://dev.to/x4nent/opentelemetry-genai-semantic-conventions-the-standard-for-llm-observability-1o2a) — retrieved 2026-05-26
5. [Datadog LLM Observability natively supports OpenTelemetry GenAI Semantic Conventions — Datadog](https://www.datadoghq.com/blog/llm-otel-semantic-convention/) — retrieved 2026-05-26

Last verified: 2026-05-26
