---
week: W05
day: Tue
title: "Pre-session — OpenTelemetry on AI workloads: traceparent, gen_ai semconv, Bedrock token cost as a signal"
audience: All cohort members
time_on_task_minutes: 50
last_verified: 2026-05-23
research_recency_windows: [foundation-stable-12mo, hot-tech-3mo]
---

# W5 Tue Pre-Session — OpenTelemetry on AI workloads

> Read **before** W5 Tue morning. ~50 min total. Tue is the OTel-on-AI hands-on day. By EOD Tue your pair has Datadog Agent live in `acquire-gov` + every service emitting consistent `traceparent` + Bedrock token/cost rolling up as span attributes.

## 1. Why OTel on AI matters in `acquire-gov` (8 min)

W1 Tue's brownfield-debt inventory captured **Item 6: inconsistent correlation IDs** — three of the five services log a per-request identifier under different field names. Today's work converts all four services + the Angular SPA to emit W3C `traceparent`, the IETF standard for distributed-trace propagation.

The canonical scenario: a CO submits a `POST /draft-solicitation` request. Today (pre-Tue), tailing logs across services tells you 5 separate stories with no join key. **By EOD Tue**, the Datadog APM trace view shows a single trace spanning Angular RUM → api-gateway → solicitation-service → ai-orchestrator → Bedrock invocation → AuditEvent write — joined on `traceparent`.

This is **Item 6 modernised end-to-end this afternoon.** And it's the prerequisite for Wed's HITL #7 work (auto-fix-the-audit-gap requires being able to *find* an audit gap; OTel makes the gap legible).

[W3C Trace Context recommendation, retrieved 2026-05-23, https://www.w3.org/TR/trace-context/]

## 2. W3C `traceparent` — the header you'll wire up (10 min)

`traceparent` is a single HTTP header carrying `<version>-<trace-id>-<parent-id>-<flags>`. Example:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

- `version` — `00` (current).
- `trace-id` — 16-byte (128-bit) hex; identifies the trace.
- `parent-id` — 8-byte (64-bit) hex; identifies the parent span.
- `flags` — sampling decision.

Every service in `acquire-gov` either propagates an existing `traceparent` or generates a new one if missing. The OTel SDKs do this automatically once instrumented:

- **Angular SPA** — Datadog RUM SDK auto-injects `traceparent` on `fetch`/`XHR` outbound requests.
- **Spring Boot 2.7.18 services** — OTel Java auto-agent (`-javaagent:opentelemetry-javaagent.jar`) auto-instruments Spring MVC + HttpClient + JDBC.
- **Python ai-orchestrator** — `opentelemetry-instrumentation-fastapi` + `opentelemetry-instrumentation-requests` for outbound Bedrock calls.

The cohort's afternoon work: install the agents, verify the trace joins, and audit the **AuditEvent** table records the `traceparent` so the Audit Log Search admin view (`/admin/audit`) can correlate audit rows back to traces. This is the AIOps observability surface for Wed's HITL #7 work.

[OpenTelemetry Java instrumentation, retrieved 2026-05-23, https://opentelemetry.io/docs/zero-code/java/agent/]
[OpenTelemetry Python FastAPI instrumentation, retrieved 2026-05-23, https://opentelemetry.io/docs/zero-code/python/instrumentation/]

## 3. The `gen_ai.*` semantic convention — what to set on Bedrock spans (10 min)

OTel's GenAI semantic convention (still moving fast, but stable enough to ground W5 work) defines span attributes for LLM operations. The cohort wires the ai-orchestrator's Bedrock `InvokeModel` calls to emit:

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

The two `gen_ai.usage.*` attributes plus `acquire_gov.agency_id` give the Datadog dashboard *cost per tenant per endpoint per day* — which is the cost-as-signal surface section 4 covers.

[OpenTelemetry GenAI semantic conventions, retrieved 2026-05-23, https://opentelemetry.io/docs/specs/semconv/gen-ai/]

## 4. Cost as an AIOps signal (8 min)

Traditional AIOps treats cost as a *report*. AI-native AIOps treats cost as a *signal* — a sudden cost spike on `/draft-solicitation` for a single tenant is operationally interesting (could be: amplification attack, prompt regression triggering longer outputs, retry storm, malicious prompt-injection trying to exhaust the budget).

Datadog's Watchdog AI surfaces these as anomalies if cost-per-request is instrumented as a metric. The cohort's Tue afternoon work wires `gen_ai.usage.input_tokens` + `gen_ai.usage.output_tokens` into a Datadog metric with tags `acquire_gov.agency_id` + `acquire_gov.endpoint`, then sets a Watchdog AI watch on the metric.

The Wed Audit-log Activity report uses this same data source: AuditEvents joined to span cost-attributes give the OIG a "what did the AI cost the agency on this contract" view.

[Datadog Watchdog AI overview, retrieved 2026-05-23, https://docs.datadoghq.com/watchdog/]

## 5. Drift detection on agentic systems — the Wed bridge (8 min)

Today's instrumentation also lights up **drift detection** as a Wed topic. The W3 multi-agent flow (`POST /agent/intake-triage`) emits per-agent step spans; drift in *which agents fire on what input* is detectable as a Watchdog AI anomaly *only because* the spans are emitted consistently.

You won't build drift detection Tue — that's Wed war-room. But Tue's instrumentation is the prerequisite. If the cohort skips a service's OTel install Tue, Wed's drift detection has a blind spot.

[`skills/aiops-curriculum/references/ai-sre-patterns.md` §3 root-cause analysis, in-repo]

## 6. What you'll commit by EOD Tue (3 min)

- `acquire-gov/infra/docker/datadog-agent/` — Datadog Agent docker-compose service + config.
- 3 services × OTel Java auto-agent JVM args + a Datadog `dd-java-agent.jar` config.
- ai-orchestrator OTel Python SDK + `gen_ai.*` attribute decorator on Bedrock calls.
- Angular `app.module.ts` Datadog RUM init.
- `AuditEvent` schema migration: add `traceparent` column.
- Smoke test: `POST /draft-solicitation` produces a single Datadog APM trace spanning 5 services + Bedrock call.

---

## Sources (all retrieved 2026-05-23 via `/web-research`)

- W3C Trace Context — https://www.w3.org/TR/trace-context/ (foundation-stable 12-month window; W3C REC 2020, current)
- OpenTelemetry Java zero-code instrumentation — https://opentelemetry.io/docs/zero-code/java/agent/ (foundation-stable 12-month window)
- OpenTelemetry Python instrumentation — https://opentelemetry.io/docs/zero-code/python/instrumentation/
- OpenTelemetry GenAI semantic conventions — https://opentelemetry.io/docs/specs/semconv/gen-ai/ (hot-tech 3-month window — re-verify pre-cohort; semconv is moving)
- Datadog Watchdog AI — https://docs.datadoghq.com/watchdog/ (hot-tech 3-month window)
- `skills/aiops-curriculum/references/ai-sre-patterns.md` (in-repo)
- `training-project/feature-inventory-target.md` for Item 6 + Item 2 surface area (in-repo)
