---
week: W05
day: Tue
title: "Pre-session — OpenTelemetry on AI workloads: traceparent, gen_ai semconv, Python instrumentation, cost-as-signal, drift detection"
audience: All cohort members
time_on_task_minutes: 55
last_verified: 2026-05-26
research_recency_windows: [foundation-stable-12mo, hot-tech-3mo]
---

# W5 Tue Pre-Session — OpenTelemetry on AI workloads

> Read **before** W5 Tue morning. ~55 min total. Tue is the OTel-on-AI hands-on day. By EOD Tue your pair has Datadog Agent live in `acquire-gov` + every service emitting consistent `traceparent` + Bedrock token/cost rolling up as span attributes + drift-detection signals primed for Wed's war-room.

## 1. Why OTel on AI matters in `acquire-gov` (8 min)

W1 Tue's brownfield-debt inventory captured **Item 6: inconsistent correlation IDs** — three of the five services log a per-request identifier under different field names. Today's work converts all four services + the Angular SPA to emit W3C `traceparent`, the IETF standard for distributed-trace propagation.

The canonical scenario: a CO submits a `POST /draft-solicitation` request. Today (pre-Tue), tailing logs across services tells you 5 separate stories with no join key. **By EOD Tue**, the Datadog APM trace view shows a single trace spanning Angular RUM → api-gateway → solicitation-service → ai-orchestrator → Bedrock invocation → AuditEvent write — joined on `traceparent`.

This is **Item 6 modernised end-to-end this afternoon.** And it's the prerequisite for Wed's HITL #7 work (auto-fix-the-audit-gap requires being able to *find* an audit gap; OTel makes the gap legible).

[W3C Trace Context recommendation, retrieved 2026-05-23 via /web-research, https://www.w3.org/TR/trace-context/]

## 2. Trace Propagation — W3C `traceparent` across 5 acquire-gov components (10 min)

`traceparent` is a single HTTP header carrying `<version>-<trace-id>-<parent-id>-<flags>`. Example:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

- `version` — `00` (current).
- `trace-id` — 16-byte (128-bit) hex; identifies the trace.
- `parent-id` — 8-byte (64-bit) hex; identifies the parent span.
- `flags` — sampling decision.

Every service in `acquire-gov` either propagates an existing `traceparent` or generates a new one if missing. The OTel SDKs do this automatically once instrumented across the 5 components:

- **Angular SPA** — Datadog RUM SDK auto-injects `traceparent` on `fetch`/`XHR` outbound requests (gated by `allowedTracingUrls`).
- **api-gateway, solicitation-service, evaluation-service** (Spring Boot 2.7.18) — OTel Java auto-agent (`-javaagent:opentelemetry-javaagent.jar`) auto-instruments Spring MVC + HttpClient + JDBC.
- **ai-orchestrator** (Python FastAPI) — `opentelemetry-instrumentation-fastapi` + `opentelemetry-instrumentation-requests` for outbound Bedrock calls.

The cohort's afternoon work: install the agents, verify the trace joins, and audit the **AuditEvent** table records the `traceparent` so the Audit Log Search admin view (`/admin/audit`) can correlate audit rows back to traces. This is the AIOps observability surface for Wed's HITL #7 work.

[OpenTelemetry Java instrumentation, retrieved 2026-05-23 via /web-research, https://opentelemetry.io/docs/zero-code/java/agent/]
[OpenTelemetry Python FastAPI instrumentation, retrieved 2026-05-23 via /web-research, https://opentelemetry.io/docs/zero-code/python/instrumentation/]

## 3. Span Attributes — `gen_ai.*` semconv + tenant tags (10 min)

OTel's GenAI semantic convention defines span attributes for LLM operations. The `gen_ai.*` namespace is still moving (hot-tech 3-month window — instructor re-verifies pre-cohort), but stable enough to ground W5 work. The cohort wires the ai-orchestrator's Bedrock `InvokeModel` calls to emit:

| Attribute | Source | Purpose |
|-----------|--------|---------|
| `gen_ai.system` | constant `"aws.bedrock"` | platform identifier |
| `gen_ai.request.model` | `anthropic.claude-3-5-sonnet-20240620-v1:0` | model identifier |
| `gen_ai.request.temperature` | request param | sampling temperature |
| `gen_ai.request.max_tokens` | request param | output cap |
| `gen_ai.usage.input_tokens` | Bedrock response | input token count |
| `gen_ai.usage.output_tokens` | Bedrock response | output token count |
| `gen_ai.response.finish_reasons` | Bedrock response | stop reason |
| `acquire_gov.agency_id` | tenant context | tenant routing |
| `acquire_gov.endpoint` | route name | per-endpoint cost roll-up |

The two `gen_ai.usage.*` attributes plus `acquire_gov.agency_id` give the Datadog dashboard *cost per tenant per endpoint per day* — which is topic 5 (cost-as-signal) surface. Span attributes (unlike log lines) are queryable in APM filters and Watchdog AI can correlate by them — that's why tenant identity lives here, not in a log statement.

[OpenTelemetry GenAI semantic conventions, retrieved 2026-05-23 via /web-research, https://opentelemetry.io/docs/specs/semconv/gen-ai/]

## 4. OTel Instrumentation in the Python AI Service — auto-instrumentation + manual Bedrock decorator (10 min)

The ai-orchestrator is the only Python service in `acquire-gov`, and it's also the service that calls Bedrock. So it carries two instrumentation paths:

**Path A — Auto-instrumentation (FastAPI + outbound HTTP):**

```python
# services/ai-orchestrator/app/observability/otel_init.py
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

FastAPIInstrumentor.instrument_app(app)
RequestsInstrumentor().instrument()
```

This gives you HTTP-server spans on every FastAPI route + HTTP-client spans on every outbound `requests`/`urllib3` call. Zero changes to your business code. The OTLP exporter ships them to `datadog-agent:4317` (gRPC).

**Path B — Manual decorator on Bedrock InvokeModel:**

Auto-instrumentation doesn't know about Bedrock's request/response shape — it sees an HTTPS call to `bedrock-runtime.us-east-1.amazonaws.com` and that's it. The `gen_ai.*` semconv attributes from topic 3 require a manual decorator that reads the Bedrock response body and writes the token counts as span attributes:

```python
# services/ai-orchestrator/app/llm/bedrock_invoke.py
@trace_bedrock_call  # custom decorator — sets gen_ai.* attrs from response
def invoke_claude(prompt: str, tenant_id: str, endpoint: str) -> str:
    response = bedrock_client.invoke_model(...)
    return response['body'].read()
```

The decorator reads `response['usage']['input_tokens']` + `response['usage']['output_tokens']` from the Bedrock response and sets them on the current span, plus the `acquire_gov.*` tenant attrs from FastAPI request context.

**Why both paths matter:** auto-instrumentation is what makes `traceparent` propagation work end-to-end (topic 2); the manual decorator is what makes cost-as-signal (topic 5) and drift detection (topic 6) work. Skip the decorator and Datadog LLM Observability dashboards stay empty even though traces show up.

[OpenTelemetry Python FastAPI instrumentation, retrieved 2026-05-23 via /web-research, https://opentelemetry.io/docs/zero-code/python/instrumentation/]

## 5. Cost as an AIOps Signal — Watchdog AI on token-cost-per-tenant (8 min)

Traditional AIOps treats cost as a *report*. AI-native AIOps treats cost as a *signal* — a sudden cost spike on `/draft-solicitation` for a single tenant is operationally interesting (could be: amplification attack, prompt regression triggering longer outputs, retry storm, malicious prompt-injection trying to exhaust the budget — OWASP LLM10 Unbounded Consumption is the category Wed surfaces).

Datadog's Watchdog AI surfaces these as anomalies if cost-per-request is instrumented as a metric. The cohort's Tue afternoon work wires `gen_ai.usage.input_tokens` + `gen_ai.usage.output_tokens` into a Datadog metric with tags `acquire_gov.agency_id` + `acquire_gov.endpoint`, then sets a Watchdog AI watch on the metric.

The Wed Audit-log Activity report uses this same data source: AuditEvents joined to span cost-attributes give the OIG a "what did the AI cost the agency on this contract" view. **Cost-as-signal feeds Wed's AIOps Incident Drill** — the auto-fix-the-audit-gap case Wed names depends on this being instrumented today.

[Datadog Watchdog AI overview, retrieved 2026-05-23 via /web-research, https://docs.datadoghq.com/watchdog/]

## 6. Drift Detection on Agentic Systems — Tue instrumentation enables Wed war-room (9 min)

Today's instrumentation also lights up **drift detection** as a Wed topic. The W3 multi-agent flow (`POST /agent/intake-triage`) emits per-agent step spans. A concrete drift example: yesterday, on an inbound RFI from Agency-A, the intake-triage flow fired `RoutingAgent → ClauseExtractionAgent → ComplianceAgent`. Today, on a near-identical RFI from the same agency, it fires `RoutingAgent → ClauseExtractionAgent → ComplianceAgent → EscalationAgent → ComplianceAgent` (one extra hop, with a loop). That shape-change is drift — detectable as a Watchdog AI anomaly *only because* the per-agent step spans are emitted consistently and the agent-name lives on the span as an attribute.

What makes it drift (vs. legitimate variance): the input distribution didn't change meaningfully, but the agent-firing pattern did. Possible causes Wed will discuss — model update, prompt regression, retrieval drift in the underlying KB, or genuine new edge case. The cohort doesn't decide root cause Tue — Tue is *making the signal visible*. Wed war-room reasons over it.

You won't build drift detection Tue — that's Wed war-room. But Tue's instrumentation is the prerequisite. If the cohort skips a service's OTel install Tue, Wed's drift detection has a blind spot.

[`skills/aiops-curriculum/references/ai-sre-patterns.md` §3 root-cause analysis, in-repo]

## Closing checklist — what you'll commit by EOD Tue

- `acquire-gov/infra/docker/datadog-agent/` — Datadog Agent docker-compose service + config.
- 3 Java services × OTel Java auto-agent JVM args + `OTEL_PROPAGATORS=tracecontext,baggage`.
- ai-orchestrator OTel Python SDK init + `gen_ai.*` attribute decorator on Bedrock calls (both paths from topic 4).
- Angular `app.module.ts` Datadog RUM init with `mask-user-input` (LLM02 mitigation).
- `AuditEvent` schema migration: add `traceparent VARCHAR(64)` column.
- Smoke test: `POST /draft-solicitation` produces a single Datadog APM trace spanning 5 services + Bedrock call; AuditEvent row has matching `traceparent`.

---

## Sources

- W3C Trace Context — https://www.w3.org/TR/trace-context/ (foundation-stable 12-month window; W3C REC 2020, current). Retrieved 2026-05-23 via /web-research.
- OpenTelemetry Java zero-code instrumentation — https://opentelemetry.io/docs/zero-code/java/agent/ (foundation-stable 12-month window). Retrieved 2026-05-23 via /web-research.
- OpenTelemetry Python instrumentation — https://opentelemetry.io/docs/zero-code/python/instrumentation/ (foundation-stable 12-month window). Retrieved 2026-05-23 via /web-research.
- OpenTelemetry GenAI semantic conventions — https://opentelemetry.io/docs/specs/semconv/gen-ai/ (hot-tech 3-month window — re-verify pre-cohort; semconv is moving). Retrieved 2026-05-23 via /web-research.
- Datadog Watchdog AI — https://docs.datadoghq.com/watchdog/ (hot-tech 3-month window). Retrieved 2026-05-23 via /web-research.
- `skills/aiops-curriculum/references/ai-sre-patterns.md` (in-repo).
- `training-project/feature-inventory-target.md` for Item 6 + Item 2 surface area (in-repo).

Last verified: 2026-05-26
