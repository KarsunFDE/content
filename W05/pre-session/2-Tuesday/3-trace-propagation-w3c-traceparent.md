---
week: W05
day: 2-Tuesday
topic_slug: trace-propagation-w3c-traceparent
topic_title: "Trace Propagation — W3C `traceparent` across services"
parent_overview: W05/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://www.w3.org/TR/trace-context/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://www.w3.org/TR/trace-context-2/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://uptrace.dev/opentelemetry/context-propagation
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://opentelemetry.io/docs/specs/otel/context/api-propagators/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://last9.io/blog/opentelemetry-context-propagation/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
last_verified: 2026-05-26
---

# Trace Propagation — W3C `traceparent` across services

## 1. Learning Objectives

By the end of this reading, the learner can:

- Decompose a `traceparent` HTTP header into its four fields (version, trace-id, parent-id, trace-flags) and state the bit-length and allowed values for each.
- Explain what each service is required to do on inbound and outbound requests to keep a trace joined, and what the spec says happens when an inbound `traceparent` is malformed.
- Identify the difference between `traceparent` (W3C standard) and earlier formats (B3 single/multi-header) and explain why OpenTelemetry defaulted to W3C.
- Reason about the `tracestate` header as the vendor-extension companion to `traceparent`, and why it exists separately.

## 2. Introduction

A distributed trace is only useful if every service touching a request agrees on a shared identifier. For the first decade of distributed tracing, every vendor invented its own header format — Zipkin used `X-B3-TraceId` / `X-B3-SpanId`, Jaeger used `uber-trace-id`, Dapper used internal Google headers, AWS X-Ray used `X-Amzn-Trace-Id`. A trace that crossed two systems with different headers was a trace that broke.

The W3C Trace Context recommendation (first published in 2020, with Level 2 currently in Candidate Recommendation status) standardised the format ([W3C Trace Context Level 1](https://www.w3.org/TR/trace-context/) and [Level 2](https://www.w3.org/TR/trace-context-2/) — retrieved 2026-05-26). It defines two HTTP headers — `traceparent` and `tracestate` — and a strict mutation protocol. OpenTelemetry adopted W3C Trace Context as its default propagation format and ships W3C-aware injectors and extractors in every language SDK.

For AI workloads, propagation matters more than it does for a monolithic service. The path of a single user action — browser front-end → API gateway → multiple back-end services → LLM-orchestration service → model invocation → audit-log write — often crosses five or six processes and at least two languages. A propagation gap at any hop turns the trace into a stub and an outage into a guessing game.

## 3. Core Concepts

### 3.1 The `traceparent` header, field by field

A `traceparent` value is a single ASCII string with four dash-separated fields. The W3C spec gives this canonical example ([W3C Trace Context Level 1](https://www.w3.org/TR/trace-context/) — retrieved 2026-05-26):

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
              ^  ^                                ^                ^
              |  trace-id (16 bytes / 32 hex)    |                |
              version                            parent-id        trace-flags
                                                 (8 bytes /       (1 byte)
                                                 16 hex)
```

- **`version`** — two hex characters (1 byte). Current value is `00`. The value `ff` is reserved as invalid. When a receiving service sees a higher version, it must downgrade to the current version rather than reject the header.
- **`trace-id`** — 32 hex characters (16 bytes / 128 bits). Globally unique to the trace. **All-zeros is invalid** and means "no trace". This field is **constant** across every hop in the trace.
- **`parent-id`** — 16 hex characters (8 bytes / 64 bits). Identifies the **span that produced the outbound request**. **All-zeros is invalid.** This field **changes at every hop** — each service replaces it with its own current span ID before propagating onward.
- **`trace-flags`** — 2 hex characters (1 byte). Currently only the lowest bit is defined: `01` means *sampled*, `00` means *not sampled*. Sampling is a hint, not a contract; downstream systems may sample differently.

The spec is strict on validation. If any field is malformed — wrong length, non-hex characters, all-zeros where forbidden — vendors **must ignore the entire header** and start a new trace ([W3C Trace Context Level 1](https://www.w3.org/TR/trace-context/) — retrieved 2026-05-26). The accompanying `tracestate` is also discarded in this case. This is deliberately unforgiving: a partly-broken header would silently produce broken traces.

### 3.2 What each service has to do

Every service in a propagation chain runs the same two-step ritual:

**On inbound (extract):**
1. Read the `traceparent` header (and optionally `tracestate`).
2. If present and valid, use the `trace-id` as the current trace and the `parent-id` as the parent of the new span this service creates.
3. If absent or invalid, generate a new `trace-id` and treat this request as the root of a new trace.

**On outbound (inject):**
1. Before making a downstream HTTP call, build a new `traceparent` from `(version=00, trace-id=<unchanged>, parent-id=<current-span-id>, trace-flags=<sampling>)`.
2. Inject it as a header on the outbound request.
3. Forward `tracestate` unchanged unless the service has a registered vendor entry to update.

OpenTelemetry's auto-instrumentation libraries do both steps for you on the common HTTP-server and HTTP-client paths. You can verify by curling a service with no headers and one with a deliberately-malformed `traceparent` and watching the resulting trace IDs — the auto-instrumentation should produce a fresh trace in both cases ([Last9 — OpenTelemetry Context Propagation](https://last9.io/blog/opentelemetry-context-propagation/) — retrieved 2026-05-26).

### 3.3 `tracestate` — the vendor extension

`tracestate` is a separate header that carries **vendor-specific** key-value pairs. Format:

```
tracestate: vendor1=value1,vendor2=value2
```

The W3C spec limits this to 32 list-members and 512 characters per member ([W3C Trace Context Level 1](https://www.w3.org/TR/trace-context/) — retrieved 2026-05-26). Each vendor key is unique to a tracing system — `dd=...` for Datadog, `congo=...` for Congo Inc., etc. The reason `tracestate` exists separately from `traceparent`: a vendor needs to ship its own span ID forward without breaking the W3C-standard fields. The protocol says only **your own** entry may be modified; entries from other vendors must be forwarded verbatim.

For AI workloads, `tracestate` rarely needs custom writes — the standard `traceparent` plus span attributes carries everything. The header still exists in the chain because some backend vendors use it to ship sampling decisions and root-span hints.

### 3.4 W3C vs B3 — and why OTel chose W3C

Before W3C, the most widely-used format was **B3**, originally from Zipkin / Twitter. B3 has two encodings:

- **B3 multi-header.** `X-B3-TraceId`, `X-B3-SpanId`, `X-B3-ParentSpanId`, `X-B3-Sampled`, `X-B3-Flags` — five separate headers.
- **B3 single-header.** `b3: <trace-id>-<span-id>-<sampled>-<parent-span-id>`.

B3 supports a debug flag and lets both sides of an RPC share a span ID — semantics OpenTelemetry's model does not have ([OpenTelemetry Propagators API spec](https://opentelemetry.io/docs/specs/otel/context/api-propagators/) — retrieved 2026-05-26).

OpenTelemetry shipped W3C as the default propagator because:

- It is a W3C Recommendation — multi-vendor, slow-moving, with a formal change process.
- It has a single canonical header (`traceparent`) rather than five.
- It has a dedicated `tracestate` for vendor extensions — vendors can add metadata *without* extending the spec.
- It is mandatory for any browser-side instrumentation that calls into the OTel ecosystem (e.g. Datadog RUM, Honeycomb's web SDK).

OpenTelemetry SDKs ship both W3C and B3 propagators and let you configure a chain (extract from both formats, inject in W3C). For a brownfield system where some upstream service still emits B3, the chain configuration is the bridge.

## 4. Generic Implementation

A worked example: three services (Service A → Service B → Service C), each in a different language. The trace ID propagates because all three speak W3C.

```
# 1. External caller hits Service A (Java / Spring Boot, OTel Java agent attached).
#    No inbound traceparent. The agent generates a new trace-id.
GET /widget/42 HTTP/1.1
Host: service-a.internal

# 2. Service A's OTel agent generated:
#    trace-id    = 4bf92f3577b34da6a3ce929d0e0e4736
#    span-id     = 00f067aa0ba902b7  (Service A's current span)
#    Outbound to Service B carries:
GET /pricing HTTP/1.1
Host: service-b.internal
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01

# 3. Service B (Python FastAPI, opentelemetry-instrumentation-fastapi):
#    Extracts inbound traceparent. Reuses trace-id, creates its own span.
#    Its current span-id is 7a6f8b9c2d1e0f33. Outbound to Service C carries:
GET /availability HTTP/1.1
Host: service-c.internal
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-7a6f8b9c2d1e0f33-01
                ^                                ^
                trace-id (same)                  Service B's span-id, not Service A's

# 4. Service C (Node.js, @opentelemetry/instrumentation-http):
#    Same extract-and-replace ritual. Its span-id will appear as parent-id
#    if it ever calls another service.
```

The only field that changes hop-to-hop is **`parent-id`**. The `trace-id` stays constant — that constancy is what lets the observability backend join all three spans into a single trace.

**One concrete gotcha to call out at this point**: if Service B reads the inbound header but a middleware *strips* `traceparent` before the OTel instrumentation runs (some API gateways do this with custom header allow-lists), Service B will silently start a new trace. The fix is gateway config, not application code.

## 5. Real-world Patterns

- **E-commerce — Black Friday hot-path.** A large online retailer's checkout traverses storefront → cart → inventory → payments → fulfilment. After consolidating on W3C `traceparent`, their on-call team reduced mean-time-to-isolation for checkout failures from 40 minutes to 6 — the bottleneck had been five different header formats across services with five different ownership teams. The standardisation, not new tooling, was the lever ([Uptrace — OpenTelemetry Context Propagation](https://uptrace.dev/opentelemetry/context-propagation) — retrieved 2026-05-26).
- **Healthcare — referral system across hospital networks.** A US health-tech company runs a referral-routing service that spans three hospital networks running heterogeneous EMR vendors. Each network's EMR emitted a different proprietary correlation ID. Wrapping the referral service in an OTel gateway that re-emits W3C `traceparent` outbound gave the operations team a joined trace for the first time — they did not need to change any EMR-vendor code.
- **Gaming — matchmaking pipeline.** A multiplayer-game studio's matchmaking flow hits 7 services in 2 seconds. They debugged a regional-latency issue by inspecting the gap *between* `parent-id` and the next service's first span — the gap was a TLS handshake hiding inside a service-mesh sidecar. Without consistent `parent-id` propagation, the gap would have been invisible.
- **Logistics — package-tracking event chain.** A global parcel-delivery service runs an event-driven backbone where most propagation is via Kafka messages, not HTTP. They serialise `traceparent` into a Kafka message header at produce time and extract on consume — same propagation contract, different transport. This pattern (W3C string format, attached to whichever transport you have) is the spec's escape hatch for non-HTTP messaging.

## 6. Best Practices

- **Standardise on W3C across the whole estate before anything else.** Bridging via the OTel propagator chain is an interim step; the long-term contract is W3C everywhere.
- **Verify propagation with a curl-and-grep smoke test on every new service** — send a request with a known `traceparent`, grep the receiving service's logs for the same `trace-id`, fail the deploy if it does not match.
- **Audit every API gateway and service mesh for header allow-lists.** A surprisingly common cause of broken traces is a gateway that strips unfamiliar headers as a "security" default.
- **Carry `traceparent` into non-HTTP transports** — Kafka headers, SQS message attributes, gRPC metadata. The W3C format is transport-agnostic; the propagation contract is yours to maintain.
- **Treat malformed inbound `traceparent` as a fresh trace, never as an error.** The spec mandates it; instrumentation libraries do it for free; do not write custom logic that rejects requests with bad headers.
- **Reserve `tracestate` for vendor-supplied metadata.** Do not invent your own `tracestate` key for application data — span attributes are the right home for that.
- **Sample at the edge, propagate the sampling decision.** Once a trace's `trace-flags` is set to sampled (`01`), respect it downstream. Resampling per-service produces broken traces.

## 7. Hands-on Exercise

**Small code task (10–15 min).** Given two HTTP services in your language of choice (call them `service-up` and `service-down`), instrument both with OTel auto-instrumentation, then write a curl-based smoke test that proves propagation works.

Acceptance criteria:

- `curl -H 'traceparent: 00-aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa-bbbbbbbbbbbbbbbb-01' http://service-up/echo` produces a downstream span in `service-down` whose log line contains the same `trace-id` (`aaa...`) and a *different* `parent-id` (specifically, `service-up`'s outbound span ID, not `bbbb...`).
- A curl with no `traceparent` produces a new trace, and the new `trace-id` is visible in both services' logs and is the same in both.
- A curl with a malformed `traceparent` (e.g. `00-zzz-bbbb-01`) produces a fresh trace, not an error.

**What good looks like.** The candidate's solution depends only on auto-instrumentation — they do not write custom header-extraction code. They verify with grep, not by trusting a UI. They notice that the second hop's `parent-id` differs from what was supplied — that is the spec working correctly. They sample with `trace-flags=01` and confirm the downstream span is recorded; with `00` it would be dropped by sampling.

## 8. Key Takeaways

- What are the four fields of `traceparent` and how long is each? *`version` (1B), `trace-id` (16B), `parent-id` (8B), `trace-flags` (1B).*
- Which field changes hop-to-hop and which stays constant? *`parent-id` changes at every hop; `trace-id` stays constant.*
- What happens when an inbound `traceparent` is malformed? *The receiving service ignores it and starts a fresh trace; it does not raise an error.*
- Why did OpenTelemetry choose W3C over B3 as its default? *Single standardised header, formal change process, separate `tracestate` slot for vendor extensions, and direct support for browser-side instrumentation.*

## Sources

1. [W3C Trace Context — W3C Recommendation (Level 1, 2020)](https://www.w3.org/TR/trace-context/) — retrieved 2026-05-26
2. [W3C Trace Context Level 2 — W3C Candidate Recommendation](https://www.w3.org/TR/trace-context-2/) — retrieved 2026-05-26
3. [OpenTelemetry Context Propagation: W3C TraceContext & Troubleshooting Guide — Uptrace](https://uptrace.dev/opentelemetry/context-propagation) — retrieved 2026-05-26
4. [Propagators API — OpenTelemetry specification](https://opentelemetry.io/docs/specs/otel/context/api-propagators/) — retrieved 2026-05-26
5. [OpenTelemetry Context Propagation for Better Tracing — Last9](https://last9.io/blog/opentelemetry-context-propagation/) — retrieved 2026-05-26

Last verified: 2026-05-26
