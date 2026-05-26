---
week: W05
day: 2-Tuesday
topic_slug: otel-instrumentation-python-ai-service
topic_title: "OTel Instrumentation in the Python AI Service — auto-instrumentation + manual decorator"
parent_overview: W05/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://opentelemetry.io/docs/zero-code/python/instrumentation/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://opentelemetry-python-contrib.readthedocs.io/en/latest/instrumentation/fastapi/fastapi.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://opentelemetry.io/docs/instrumentation/python/manual/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://signoz.io/blog/opentelemetry-fastapi/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://opentelemetry.io/docs/languages/python/cookbook/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
last_verified: 2026-05-26
---

# OTel Instrumentation in the Python AI Service — auto-instrumentation + manual decorator

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish between **zero-code (auto)** instrumentation and **manual** instrumentation in Python OpenTelemetry, and decide which to apply to each layer of a service.
- Install and enable `opentelemetry-instrumentation-fastapi` for inbound HTTP and `opentelemetry-instrumentation-requests` (or `httpx`) for outbound calls without modifying business code.
- Write a manual decorator or context-manager span that wraps an LLM client call and emits the GenAI semconv attributes — token usage, model identity, finish reasons — on the inner span.
- Choose between the **decorator** form (`@tracer.start_as_current_span(...)`) and the **context-manager** form (`with tracer.start_as_current_span(...) as span:`) based on whether you need access to the span inside the function body.

## 2. Introduction

Python is the typical home for the LLM-orchestration layer in a polyglot service stack — pandas-friendly, the vendor SDKs land there first, FastAPI is a natural fit for the HTTP front. That means the Python AI service is usually the place where the cross-service trace meets the LLM call, and getting the instrumentation right at this seam is the single highest-leverage observability move in an AI workload.

OpenTelemetry's Python ecosystem offers two complementary paths. **Auto-instrumentation** wraps common libraries (the HTTP server, the HTTP client, the database driver, the message broker) with no code changes — you add a startup wrapper or a one-line `instrument_app()` call and every request becomes a span. **Manual instrumentation** is for the parts auto-instrumentation cannot see: your business logic, your LLM-vendor calls, your custom retry loops, your agentic orchestration steps. The two paths share one tracer and one trace context, so a manual span nests cleanly inside an auto-generated parent span ([OpenTelemetry Python — Auto-Instrumentation](https://opentelemetry.io/docs/zero-code/python/instrumentation/) — retrieved 2026-05-26).

This reading walks the two paths in sequence — auto first, then manual — and shows how they compose into a single instrumented LLM service.

## 3. Core Concepts

### 3.1 Auto-instrumentation for FastAPI

`opentelemetry-instrumentation-fastapi` is the single library that gets you a span on every inbound HTTP request. Two equivalent ways to enable it ([Python contrib FastAPI docs](https://opentelemetry-python-contrib.readthedocs.io/en/latest/instrumentation/fastapi/fastapi.html) — retrieved 2026-05-26):

**Static (recommended):**

```python
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
FastAPIInstrumentor.instrument_app(app)
```

**Class-based (when you need finer config):**

```python
FastAPIInstrumentor().instrument(...)
```

The instrumentor automatically **extracts `traceparent` and `tracestate` from incoming headers**, so inbound requests join the upstream trace without any custom code. It also exposes three hooks for adding custom attributes — `server_request_hook` (ASGI scope after span creation), `client_request_hook` (on receive), and `client_response_hook` (on send) — all of which check `span.is_recording()` before setting attributes.

For services that need to **exclude noisy paths** (`/health`, `/metrics`, `/livez`), pass `excluded_urls` either via the env var (`OTEL_PYTHON_FASTAPI_EXCLUDED_URLS="health,metrics"`) or via the call parameter. Excluding the health-check endpoint is almost always correct — otherwise it dominates span volume and obscures real traffic.

### 3.2 Auto-instrumentation for outbound HTTP

The Python AI service calls an LLM provider over HTTPS. To get a child span for that outbound call automatically, instrument the HTTP client:

```python
from opentelemetry.instrumentation.requests import RequestsInstrumentor
RequestsInstrumentor().instrument()
```

For services using `httpx` (the modern async-friendly HTTP client most LLM SDKs now use under the hood), the equivalent is `opentelemetry-instrumentation-httpx`. The SDK-of-the-day will choose for you — install the matching instrumentor.

Auto-instrumentation knows how to:

- Open a CLIENT span around each outbound HTTP request.
- Set `http.method`, `http.url`, `http.status_code`, `server.address`, `server.port`.
- Inject `traceparent` into outbound headers so the next service joins the trace.

What auto-instrumentation **does not** know:

- That the outbound call to `api.openai.com` or `bedrock-runtime.us-east-1.amazonaws.com` is an LLM call, not a generic HTTP call.
- That the response body has a `usage.input_tokens` / `usage.output_tokens` payload.
- That the request has a model identifier worth recording.

That gap is exactly what the manual decorator fills.

### 3.3 Manual instrumentation — two equivalent forms

OpenTelemetry's Python API offers two equivalent ways to start a span. Both wrap the same `start_as_current_span` primitive ([OpenTelemetry Python manual instrumentation](https://opentelemetry.io/docs/instrumentation/python/manual/) — retrieved 2026-05-26).

**Context-manager form** — preferred when you need to access the span inside the function body:

```python
from opentelemetry import trace
tracer = trace.get_tracer(__name__)

def do_work():
    with tracer.start_as_current_span("do_work") as span:
        span.set_attribute("custom.attr", "value")
        # work happens here
```

**Decorator form** — concise when the span boundary equals the function boundary:

```python
@tracer.start_as_current_span("do_work")
def do_work():
    # work happens here; the span is implicit
    pass
```

Both produce identical traces. The decorator form is cleaner when you do not need to set attributes from inside the function; the context-manager form is more flexible when you do — which is most of the time for an LLM call, because the attributes depend on the response body.

A common idiom is to combine both: a small custom decorator that wraps your LLM client method, opens the span via the context manager internally, sets request-side attributes before the call and response-side attributes after.

### 3.4 The full layered picture

A request landing in the Python AI service produces a span stack like this:

```
[ FastAPI auto-span ]                    <- inbound HTTP, parent extracted from traceparent
   |
   +-- [ business-logic span (optional, manual) ]
        |
        +-- [ LLM-call span (manual, gen_ai.* attrs) ]
             |
             +-- [ outbound HTTP auto-span ]    <- HTTPS to LLM provider
```

The trace shows four levels of detail without any single layer being too fat. The outermost span is the user-perceived request; the innermost is the network call to the provider. Token attributes live on the middle (LLM-call) span — not on the outbound HTTP span — because the LLM-call span owns the *semantic* operation, while the HTTP span owns the *transport*.

### 3.5 Configuring the exporter

None of this works until the SDK knows where to send the spans. The standard configuration uses environment variables read at process start:

```
OTEL_SERVICE_NAME=ai-service
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
OTEL_PROPAGATORS=tracecontext,baggage
```

The `OTEL_PROPAGATORS=tracecontext,baggage` line is what selects W3C `traceparent` as the propagation format — it is the default in modern SDKs, but pinning it explicitly is good practice. For a mixed estate that still has some B3 emitters upstream, the value becomes `tracecontext,baggage,b3multi` and the SDK accepts either format on extract while still emitting W3C on inject ([SigNoz — OpenTelemetry FastAPI guide](https://signoz.io/blog/opentelemetry-fastapi/) — retrieved 2026-05-26).

## 4. Generic Implementation

Putting it together — a generic AI-orchestration service. Domain-neutral naming throughout.

```python
# observability/otel_init.py
import os
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

def init_otel(app):
    provider = TracerProvider()
    provider.add_span_processor(
        BatchSpanProcessor(OTLPSpanExporter(
            endpoint=os.environ["OTEL_EXPORTER_OTLP_ENDPOINT"]
        ))
    )
    trace.set_tracer_provider(provider)

    # Auto-instrument inbound FastAPI requests
    FastAPIInstrumentor.instrument_app(
        app,
        excluded_urls="health,ready,metrics",
    )
    # Auto-instrument outbound HTTP calls
    RequestsInstrumentor().instrument()
```

```python
# llm/llm_call.py — the manual decorator that wraps the LLM client
from functools import wraps
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

def trace_llm_call(operation_name: str, provider_name: str):
    """Wrap an LLM-client method so each call emits a gen_ai.* span."""
    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            with tracer.start_as_current_span("chat") as span:
                # Required attributes set before the call
                span.set_attribute("gen_ai.operation.name", operation_name)
                span.set_attribute("gen_ai.provider.name", provider_name)

                # Recommended request-side attrs
                model = kwargs.get("model") or args[0]
                span.set_attribute("gen_ai.request.model", model)
                if "temperature" in kwargs:
                    span.set_attribute("gen_ai.request.temperature",
                                       kwargs["temperature"])
                if "max_tokens" in kwargs:
                    span.set_attribute("gen_ai.request.max_tokens",
                                       kwargs["max_tokens"])

                try:
                    response = fn(*args, **kwargs)
                except Exception as exc:
                    span.set_attribute("error.type", type(exc).__name__)
                    span.record_exception(exc)
                    raise

                # Recommended response-side attrs (read the response body)
                usage = getattr(response, "usage", None)
                if usage is not None:
                    span.set_attribute("gen_ai.usage.input_tokens",
                                       usage.input_tokens)
                    span.set_attribute("gen_ai.usage.output_tokens",
                                       usage.output_tokens)
                if hasattr(response, "model"):
                    span.set_attribute("gen_ai.response.model", response.model)
                if hasattr(response, "stop_reason"):
                    span.set_attribute(
                        "gen_ai.response.finish_reasons",
                        [response.stop_reason])
                return response
        return wrapper
    return decorator


# Usage
@trace_llm_call(operation_name="chat", provider_name="openai")
def invoke_chat(model, messages, temperature=0.2, max_tokens=1024):
    return openai_client.chat.completions.create(
        model=model, messages=messages,
        temperature=temperature, max_tokens=max_tokens,
    )
```

What each section does:

1. **`init_otel(app)`** sets up the SDK and turns on both auto-instrumentation paths. Called once at FastAPI startup.
2. **`trace_llm_call`** is a parameterised decorator. It takes the operation and provider names because those are required attributes and the wrapper cannot infer them.
3. **Request-side attrs set before the call** — if `fn()` raises, the span still has `gen_ai.operation.name`, `gen_ai.provider.name`, and `gen_ai.request.model` recorded.
4. **Try/except records the exception on the span** with `error.type` and `record_exception` — this is how OTel signals errors on a span without losing the rest of the attributes.
5. **Response-side attrs set after the call returns** — the token counts and finish reasons come from the response body.
6. **The decorator nests inside the FastAPI auto-span** automatically, because `start_as_current_span` reads the current context and parents the new span under it.

> [!instructor-review]
> This decorator example is deliberately written against a *generic* OpenAI-style client surface. If your vendor's SDK (Anthropic, AWS Bedrock, etc.) shapes the response differently — `response['body'].read()` for Bedrock invoke vs. a typed Pydantic model for OpenAI — the response-side attribute extraction code needs to match. Several published instrumentation libraries (OpenLLMetry / Traceloop SDK, Logfire, the OpenAI/Anthropic OTel instrumentation packages) ship vendor-specific decorators that handle this. Prefer using one of those for production rather than hand-rolling; the hand-rolled version above is shown for learning purposes only.

## 5. Real-world Patterns

- **Healthcare — clinical-note summariser.** A US health-tech provider ships a FastAPI service that calls OpenAI to summarise consult notes. Their initial instrumentation used auto-only and produced traces with sub-second total latency and no LLM-specific signal. Adding the manual decorator surfaced that 8% of summaries were hitting `finish_reason=length` (truncated) — a UX failure their dashboards missed because total latency looked fine ([SigNoz — OpenTelemetry FastAPI guide](https://signoz.io/blog/opentelemetry-fastapi/) — retrieved 2026-05-26).
- **E-commerce — product-Q&A bot.** A retailer's product-Q&A FastAPI service runs in three regions and switches between two LLM providers depending on regional cost. The team scoped each region's bot under a separate `OTEL_SERVICE_NAME`. The auto-FastAPI span plus the manual `gen_ai.*` decorator together produced a single per-region dashboard with model-version, finish-reason, and per-region token-cost columns — driven entirely by span attributes, no custom log parsing.
- **Logistics — driver-routing optimisation.** A delivery network uses an LLM to re-plan routes when a vehicle goes off-route. The FastAPI endpoint had two LLM calls (planner + verifier) in sequence. The decorator pattern, applied to each, surfaced that the verifier was consuming 3× the input tokens of the planner — a prompt-template bug the team hadn't seen because end-to-end latency was within SLO.
- **Gaming — narrative-content service.** A studio's narrative-generation FastAPI service experimented with `gen_ai.request.temperature` as a span attribute and discovered drift in their own production tooling: a config-management bug had ramped temperature from 0.2 to 1.4 on 4% of requests. The drift showed up as a sudden bimodal distribution on a single span attribute — invisible to traditional APM.

## 6. Best Practices

- **Initialise the SDK once, at process start, before any business code runs.** Late initialisation produces missing spans on early requests.
- **Always exclude noise endpoints (`/health`, `/metrics`, `/livez`) from auto-instrumentation.** They dominate volume and obscure real traffic.
- **Prefer published vendor-specific instrumentation libraries over hand-rolled decorators in production.** OpenLLMetry, Logfire, and the per-vendor OTel packages handle response-shape differences correctly across SDK versions.
- **Set required attributes before the call so errored calls remain debuggable.** Use `record_exception` + `error.type` for the exception path.
- **Pin `OTEL_PROPAGATORS=tracecontext,baggage` explicitly.** The default may shift across SDK versions; pinning makes the propagation contract a code artefact, not an implicit one.
- **Set `OTEL_SERVICE_NAME` per logical service**, not per process. Three regional copies of the same service can share a service name and disambiguate by `deployment.environment.name` or a custom region attribute.
- **Keep manual spans shallow.** One span per logical operation (e.g. one per LLM call, one per retrieval) — not one per function. Deep span trees explode storage cost and visualisation noise.

## 7. Hands-on Exercise

**Small code task (10–15 min).** Take a minimal FastAPI service in any domain *other* than federal acquisitions — say a recipe-suggestion API. Add OTel auto-instrumentation for inbound and outbound, plus a single manual decorator around the call to whichever LLM SDK you have at hand (you can stub the actual LLM call with a fake response object that has `usage.input_tokens`, `usage.output_tokens`, and `stop_reason`).

Acceptance criteria:

- Starting the service produces a configured tracer that exports to either a console exporter (for the exercise) or an OTLP endpoint.
- Issuing a `curl` request to a recipe endpoint produces a trace with **at least three spans** in the right parent-child order: FastAPI inbound → LLM-call (manual) → outbound HTTP (auto).
- The middle (LLM-call) span carries `gen_ai.operation.name`, `gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, and `gen_ai.response.finish_reasons`.
- Triggering an error path (raise an exception from the stub) records the exception on the LLM-call span and the span has `error.type` set.

**What good looks like.** The candidate uses `FastAPIInstrumentor.instrument_app(app)` and `RequestsInstrumentor().instrument()` for the two auto paths. The manual decorator uses the context-manager form (`with tracer.start_as_current_span(...) as span:`) so it can read response attributes after the call. Required attributes are set *before* the call, recommended attributes *after*. The error path uses `record_exception` rather than swallowing the exception silently.

## 8. Key Takeaways

- What does Python OTel auto-instrumentation cover for free in a FastAPI AI service? *Inbound HTTP spans (FastAPI), outbound HTTP spans (requests / httpx), and W3C `traceparent` extract+inject — without any code changes.*
- What does auto-instrumentation *not* cover for an LLM call, and how do you fill the gap? *Token usage, model identity, finish reasons. Filled by a manual decorator or context-manager span that sets `gen_ai.*` attributes from the LLM response body.*
- When do you use the decorator form vs. the context-manager form? *Decorator when the span boundary equals the function boundary and you do not need to set attributes from inside; context-manager when you do.*
- Where in the span stack do `gen_ai.*` attributes belong? *On the manual LLM-call span, not on the outer FastAPI span and not on the inner outbound HTTP span — that middle span owns the semantic LLM operation.*

## Sources

1. [Auto-instrumentation Example — OpenTelemetry Python](https://opentelemetry.io/docs/zero-code/python/instrumentation/) — retrieved 2026-05-26
2. [OpenTelemetry FastAPI Instrumentation — opentelemetry-python-contrib](https://opentelemetry-python-contrib.readthedocs.io/en/latest/instrumentation/fastapi/fastapi.html) — retrieved 2026-05-26
3. [Instrumentation — OpenTelemetry Python manual](https://opentelemetry.io/docs/instrumentation/python/manual/) — retrieved 2026-05-26
4. [Implementing OpenTelemetry in FastAPI — SigNoz](https://signoz.io/blog/opentelemetry-fastapi/) — retrieved 2026-05-26
5. [Cookbook — OpenTelemetry Python](https://opentelemetry.io/docs/languages/python/cookbook/) — retrieved 2026-05-26

Last verified: 2026-05-26
