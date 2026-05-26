---
week: W05
day: 2-Tuesday
topic_slug: why-otel-on-ai-matters
topic_title: "Why OTel on AI matters"
parent_overview: W05/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://opentelemetry.io/blog/2026/genai-observability/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://openobserve.ai/blog/opentelemetry-for-llms/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://uptrace.dev/blog/opentelemetry-ai-systems
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.datadoghq.com/blog/llm-otel-semantic-convention/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://horovits.medium.com/opentelemetry-for-genai-and-the-openllmetry-project-81b9cea6a771
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Why OTel on AI matters

## 1. Learning Objectives

By the end of this reading, the learner can:

- Explain why traditional APM signals (CPU, error rate, p99 latency) are **necessary but insufficient** for an AI workload, and name three failure modes they miss.
- Describe what a **distributed trace** of an LLM-backed request looks like end-to-end (browser → gateway → backend service → model invocation → downstream effects) and why a missing trace at any hop becomes a debugging black hole.
- Distinguish between the **three telemetry signals** OpenTelemetry standardises (traces, metrics, logs) and which one each AI-specific question (cost overrun, drift, jailbreak attempt) most cleanly maps to.
- Justify why an open standard (OpenTelemetry + W3C Trace Context) matters more for AI workloads than for traditional services, in one paragraph aimed at a sceptical engineering manager.

## 2. Introduction

For two decades application performance monitoring (APM) has been built around an unspoken assumption: **the same input produces the same output, and behaviour drifts mainly when infrastructure drifts.** A `POST /checkout` that succeeded yesterday should succeed today; if it fails, the cause is usually a slow database query, a saturated thread pool, an expired certificate, or a code change. The signals you need are well-known — CPU, memory, request rate, error rate, latency percentiles, log volume.

LLM-backed services break this assumption at the root. The same prompt, the same temperature, the same model can produce a perfectly fine response one moment and a hallucinated, policy-violating, or runaway-token response the next. The infrastructure may be entirely healthy — green dashboards, sub-second latencies, zero error rate — and the system can still be *failing in the way that matters*: producing wrong, unsafe, or absurdly expensive outputs. Traditional APM cannot see this kind of failure because it is not measuring the right surface.

OpenTelemetry (OTel) is the open-source observability framework the industry has converged on as the response. It is a CNCF-graduated project that defines a vendor-neutral way to capture **traces, metrics, and logs** from a system, with SDKs for every mainstream language and an interchange protocol (OTLP) that every major observability backend now ingests. As of early 2026, OTel's **GenAI semantic conventions** — a standard schema for the attributes you attach to LLM-call spans — exited experimental for client/inference spans, and Datadog, Honeycomb, New Relic and others have native ingest paths for them ([OpenTelemetry blog, 2026](https://opentelemetry.io/blog/2026/genai-observability/) — retrieved 2026-05-26; [Datadog LLM Observability, 2026](https://www.datadoghq.com/blog/llm-otel-semantic-convention/) — retrieved 2026-05-26).

This reading frames *why* this matters before later topics dig into the *how*. The short version: LLM workloads invented new failure modes — silent drift, runaway token spend, prompt-injection-driven tool misuse, agentic loops — and only an observability stack that knows what an LLM call is can surface them. Bolting AI-specific tools on next to your existing APM produces two disconnected silos; standardising on OTel produces one trace that spans both.

## 3. Core Concepts

### 3.1 The three telemetry signals, applied to AI

OTel collects three kinds of signal. Each maps to a different class of AI question.

- **Traces** are causally-linked records of a single request flowing through every component that touches it. Each component emits a **span** — a timed record with a parent span and attributes. For an AI workload, the trace is the only place you can reconstruct *which retrieval returned which chunks, which tool the agent called next, why the second LLM hop spent 6000 output tokens.* Without trace-level capture, the same prompt that produced a hallucinated answer is irreproducible.
- **Metrics** are numeric aggregates over time (rate, sum, histogram). For AI, the load-bearing metrics are **token usage**, **cost per request**, **per-tool invocation count**, **per-agent step count**, and **eval scores** if you score in production. They are what dashboards and alerts run on. Metrics cannot tell you *why* token usage doubled — they tell you *that* it did, and you go to traces to find out why.
- **Logs** are unstructured or semi-structured events. For AI, the most important logs are **prompt and completion content** (when policy allows you to capture them — see PII concerns below) and **policy/guardrail events** (a prompt-injection detector fired, a content filter blocked a response). Logs feed eval pipelines and post-incident review ([OpenTelemetry GenAI events spec](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-events/) — retrieved 2026-05-26).

The discipline OTel imposes is that all three signals share a **trace ID and span ID**. A cost spike (metric) on a span links to the prompt content (log) and the parent trace (trace) — three signals, one story.

### 3.2 Failure modes traditional APM cannot see

Three concrete failure classes drive the AI-observability investment.

- **Silent quality drift.** A model provider pushes a minor version. Your prompt's output quality drops 8% on a benchmark you don't have. Latency is fine. Error rate is zero. Users complain on social media before your dashboards do. Detecting this requires *eval-in-production*, which requires capturing inputs and outputs on the span — APM alone is blind.
- **Cost runaways.** A retrieval bug causes the LLM to be invoked twice per request instead of once. A regex change causes prompts to be 2× longer. An agent loops three times instead of one. Daily spend triples. Your CFO sees it before your engineer does ([Vantage, 2026 — AI Cost Observability](https://www.vantage.sh/blog/finops-for-ai-token-costs) — retrieved 2026-05-26). Detection requires **per-call token attributes on the span**, rolled up into a cost metric you can alert on.
- **Agentic mis-orchestration.** A multi-agent system suddenly fires an extra agent on a class of inputs it didn't before. Or it skips a guardrail-checking agent. The user-visible output may look fine; the *shape of the trace* is the only place this is visible. This is the drift case agentic-system instrumentation is built to surface ([Confident AI, 2026 — best observability platforms](https://www.confident-ai.com/knowledge-base/compare/best-ai-observability-platforms-to-monitor-response-drift-2026) — retrieved 2026-05-26).

Each of these failure modes has the same root signature: **looks fine in CPU/latency/error metrics; visible only in the AI-specific layer.** That layer is what OTel-on-AI is for.

### 3.3 Why an open standard, specifically

A team could build all of this on a proprietary AI-observability vendor — and many do. The argument for OTel as the foundation is three-part.

- **Vendor neutrality.** Your application emits OTLP. Your backend can be Datadog today, Grafana Tempo tomorrow, Honeycomb the day after, an internal OpenSearch cluster on a FedRAMP-bounded environment the day after that. The instrumentation does not change.
- **End-to-end traces across mixed stacks.** A multi-language system (Angular front-end, Java services, Python AI service) can only produce one joined trace if every component speaks the same context-propagation protocol. OTel adopted **W3C Trace Context** as that protocol — the next topic in this day's reading covers it directly.
- **AI-specific schema convergence.** The GenAI semantic conventions are a single attribute namespace (`gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, etc.) every vendor ingests. Without that, every backend has its own field names and dashboards do not port across tools ([Uptrace, 2026 — OpenTelemetry for AI Systems](https://uptrace.dev/blog/opentelemetry-ai-systems) — retrieved 2026-05-26).

The pragmatic effect: OTel-instrumented code is portable. You can swap observability vendors without rewriting your instrumentation. For an LLM application — where the right backend keeps shifting as the AI-observability market matures — this matters more than it does for a CRUD service.

## 4. Generic Implementation

A minimal mental model for an OTel-on-AI deployment, drawn generically (no acquire-gov terms):

```
[ Browser / RUM SDK ]
        |  traceparent (W3C)
        v
[  API Gateway  ]   <-- OTel Java agent (auto-instrumentation)
        |  traceparent
        v
[ Business Service ] <-- OTel Java agent
        |  traceparent
        v
[ AI Service (Py) ] <-- OTel Python SDK (auto-instr for HTTP server,
        |               manual decorator for LLM client)
        |  traceparent
        v
[ Model Provider / Inference Backend ]
        ^
        |  response body: usage.input_tokens, usage.output_tokens
        |
        +-- decorator reads usage and writes to current span as
            gen_ai.usage.input_tokens / gen_ai.usage.output_tokens
            gen_ai.request.model
            gen_ai.response.finish_reasons
```

Two things to notice in the generic shape:

- **Most spans are free.** Auto-instrumentation libraries cover HTTP server, HTTP client, database client, message broker, etc. — for every common language. You add `-javaagent:` or `opentelemetry-instrument` to the startup line and you get the full transport-layer trace.
- **LLM spans are not free.** The auto-instrumentation knows there is an HTTPS call out to `api.openai.com` or `bedrock-runtime.us-east-1.amazonaws.com`; it does **not** know to read the response body and extract token counts, finish reasons, or model identifiers. Those require either a vendor-specific instrumentation library (OpenLLMetry, Traceloop SDK, etc.) or a small manual decorator your team writes ([OpenLLMetry / Traceloop, 2026](https://github.com/traceloop/openllmetry) — retrieved 2026-05-26).

The right call for a new project: install the auto-instrumentation agent first to get the cross-service trace working; add manual LLM-call instrumentation second to populate `gen_ai.*` attributes.

## 5. Real-world Patterns

A few outside-federal-acquisitions examples of organisations applying this pattern in production.

- **Fintech — fraud-triage copilot.** A US neobank built a customer-support copilot that drafts replies to fraud-claim emails. Initial APM showed sub-second latency and zero error rate, but the support team flagged that 6% of replies referenced policies the bank had deprecated a year prior. Adding OTel-with-`gen_ai.*` spans plus prompt/completion logs let them correlate the bad replies to a single embedding-store partition that had not been re-indexed after a policy update. The fix was a data-pipeline issue, but the observability gap was the AI-specific layer ([Vantage, 2026](https://www.vantage.sh/blog/finops-for-ai-token-costs) — retrieved 2026-05-26).
- **E-commerce — generative product descriptions.** A mid-size retailer used an LLM to auto-generate product descriptions at scale. The team's first month showed an average cost-per-description of $0.012. Two weeks after a prompt-template change, this jumped to $0.041 with no corresponding error-rate or latency change. Per-request token attributes captured on the span surfaced that the new template encouraged 3× longer outputs; the team rolled back, saving roughly $80k/month ([Finout, 2026 — FinOps for AI](https://www.finout.io/blog/finops-in-the-age-of-ai-a-cpos-guide-to-llm-workflows-rag-ai-agents-and-agentic-systems) — retrieved 2026-05-26).
- **Healthcare — clinical-note summarisation.** A hospital network running an internal note-summarisation tool sat on three observability vendors over 18 months as the AI-observability market matured. Because they had standardised on OTel-OTLP emission, the switch each time was a backend-config change, not a re-instrumentation of every clinical-app service. The same `gen_ai.*` attributes lit up dashboards in all three vendors.
- **Gaming — anti-cheat reasoning service.** An online-game studio runs a reasoning model to triage suspected cheat reports. They surfaced an agentic-drift incident: the triage agent began invoking a "request-additional-evidence" sub-agent 4× more often than baseline. APM was green; the agent-shape change in the trace was the only signal. The cause was a prompt drift from a vendor model update ([Confident AI, 2026](https://www.confident-ai.com/knowledge-base/compare/best-ai-observability-platforms-to-monitor-response-drift-2026) — retrieved 2026-05-26).

In each case, traditional APM was not *wrong*. It was *insufficient*. The AI-specific failure modes lived in a layer that traditional APM doesn't cover.

## 6. Best Practices

- **Instrument the cross-service trace first, the LLM call second.** Get auto-instrumentation working end-to-end and verify a single trace ID propagates across every hop before adding `gen_ai.*` attributes.
- **Capture token usage on every LLM span as a first-class attribute, not just a log line.** Span attributes are queryable in APM filters and drive metric rollups; log-only token counts strand the data.
- **Treat prompt/completion content as logs, not span attributes, and gate them on PII policy.** Span attributes are often indexed; logs are easier to sample, redact, and retain on a different schedule ([maketocreate, 2026 — Tracing AI Agents Without Leaking PII](https://maketocreate.com/opentelemetry-genai-tracing-ai-agents-without-leaking-pii/) — retrieved 2026-05-26).
- **Pin a known OTel SDK version and re-verify GenAI semconv stability every release.** The semantic conventions are exiting experimental in stages; stale instrumentation drifts off the latest attribute names.
- **Wire trace ID into every downstream record** (audit log, message-broker payload, asynchronous job context) — anything that touches an LLM-affected record should carry the trace ID so future incidents can join across them.
- **Resist the temptation to build a parallel AI-observability silo.** The whole point of OTel is one trace across both worlds; running a separate AI tool that doesn't speak OTLP creates two stories and a join you do not want to maintain.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** Take an imagined three-service application from a domain you know — fintech, healthcare, e-commerce, gaming, logistics, anything *not* federal acquisitions:

- A web/mobile client
- A backend API service in your language of choice
- A new AI assistant service that calls an external LLM provider on behalf of the user

Sketch the trace for a single user action that flows through all three. Label every span boundary and every place a `traceparent` would be propagated. Then list:

- Three traditional APM metrics that *would* catch a failure in this trace.
- Three failure modes that none of those metrics would catch — only AI-specific attributes (token usage, agent shape, eval score) would.
- The one place in the trace where you would attach the `gen_ai.*` attributes, and exactly which attributes you would prioritise.

**What good looks like.** The diagram has a single trace ID flowing from the client through every service, with a clean parent-child span relationship. The AI-specific failure modes list at minimum: silent quality drift, cost runaway from longer outputs, and a guardrail bypass or tool-misuse case. The candidate identifies that `gen_ai.*` attributes belong on the *innermost* LLM-call span (the one that owns the model invocation), not on the outer business-logic span.

## 8. Key Takeaways

- Why does traditional APM miss AI-specific failures? *Because LLM workloads produce variable outputs from identical infrastructure, and APM measures the infrastructure layer.*
- What are the three telemetry signals OTel standardises, and which AI question does each best answer? *Traces — reproducibility; metrics — cost and rate alerts; logs — prompt/completion review and policy events.*
- Why does the open standard (OTel + W3C Trace Context) matter more for AI workloads than for traditional services? *Because the AI-observability vendor market is still maturing; portable instrumentation lets you change backends without re-instrumenting.*
- Where on a trace do `gen_ai.*` attributes belong? *On the innermost LLM-call span — the span that owns the model invocation — not on the outer business-logic span.*

## Sources

1. [Inside the LLM Call: GenAI Observability with OpenTelemetry — OpenTelemetry blog (2026)](https://opentelemetry.io/blog/2026/genai-observability/) — retrieved 2026-05-26
2. [OpenTelemetry for LLMs: Complete SRE Guide for 2026 — OpenObserve](https://openobserve.ai/blog/opentelemetry-for-llms/) — retrieved 2026-05-26
3. [OpenTelemetry for AI Systems: LLM and Agent Observability (2026) — Uptrace](https://uptrace.dev/blog/opentelemetry-ai-systems) — retrieved 2026-05-26
4. [Datadog LLM Observability natively supports OpenTelemetry GenAI Semantic Conventions — Datadog (2026)](https://www.datadoghq.com/blog/llm-otel-semantic-convention/) — retrieved 2026-05-26
5. [OpenTelemetry for GenAI and the OpenLLMetry project — Horovits / Medium (2026)](https://horovits.medium.com/opentelemetry-for-genai-and-the-openllmetry-project-81b9cea6a771) — retrieved 2026-05-26
6. [Best AI Cost Observability Tools in 2026 — Vantage](https://www.vantage.sh/blog/finops-for-ai-token-costs) — retrieved 2026-05-26
7. [5 Best AI Observability Platforms to Monitor Response Drift in 2026 — Confident AI](https://www.confident-ai.com/knowledge-base/compare/best-ai-observability-platforms-to-monitor-response-drift-2026) — retrieved 2026-05-26

Last verified: 2026-05-26
