---
week: W05
day: Mon
topic_slug: aiops-platform-walkthrough
topic_title: "AIOps Platform Walkthrough — Datadog hands-on + Dynatrace/New Relic/Coralogix compare-set"
parent_overview: W05/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 13
sources:
  - url: https://docs.datadoghq.com/llm_observability/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.datadoghq.com/blog/llm-otel-semantic-convention/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.dynatrace.com/news/blog/advancing-aiops-preventive-operations-powered-by-davis-ai/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://newrelic.com/blog/news/scaling-ai-agents-ai-observability
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://coralogix.com/coralogix-vs-new-relic/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# AIOps Platform Walkthrough — Datadog hands-on + Dynatrace/New Relic/Coralogix compare-set

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name four representative AIOps platforms in the 2026 compare-set (Datadog, Dynatrace, New Relic, Coralogix) and the architectural premise that distinguishes each one. The broader category also includes Splunk/Cisco, PagerDuty, BigPanda, ScienceLogic, and BMC Helix — covered in the Thu compare-matrix workshop; the four-platform cut here is illustrative, not exhaustive.
- Describe what an "AI-on-AI" observability surface is — i.e., what an AIOps platform does for AI-app telemetry beyond what it does for traditional app telemetry.
- Articulate the trade-offs across the four platforms on three axes that matter at scale: data-pipeline model, root-cause engine, and pricing/lock-in shape.
- Justify the choice of any one of the four platforms for a given engagement shape, with a defendable pivot condition.

## 2. Introduction

AIOps platforms — Artificial Intelligence for IT Operations — apply machine learning and rule-based analysis to observability telemetry (metrics, logs, traces, events) to surface anomalies, cluster alerts, suggest root causes, and increasingly trigger remediations. The category existed before the LLM era, but the rise of production LLM apps has reshaped what "operations" means: a modern AIOps platform now also has to monitor *the AI itself* — token usage, model latency, hallucination signals, drift, and per-tenant cost — not just the host metrics underneath.

By 2026 four platforms — Datadog, Dynatrace, New Relic, and Coralogix — show up often in federal-modernization and enterprise compare-sets and span the architectural-premise space cleanly enough to make a useful walkthrough quartet. They share the same headline feature list (LLM tracing, anomaly detection, AI-assisted incident triage, FedRAMP options) but differ sharply in architectural premise. The broader AIOps category includes Splunk/Cisco, PagerDuty, BigPanda, ScienceLogic, BMC Helix, Grafana Cloud, and others — the Thu compare-matrix workshop reckons with that wider field; this reading focuses on the four-platform cut because each one anchors a distinct premise (broad SaaS, causal topology, model-comparison, in-stream pipeline). A team that picks a platform without understanding *which* premise underlies *which* product ends up either fighting the tool or paying for capability it never uses.

This reading walks through the four representative platforms generically. The day's overview pins down which one the cohort picks for hands-on this week and why; here we look at the broader landscape so the choice is defensible.

## 3. Core Concepts

### 3.1 What's distinctive about AI-on-AI observability

Traditional APM watches a service and asks: is it up, fast, and correct? LLM observability adds three concerns ([Datadog LLM Observability docs, 2026-05-26](https://docs.datadoghq.com/llm_observability/)):

- **Token + cost tracing.** Each request emits a trace of LLM calls, tool invocations, retrieval steps, and agent decisions. Inputs, outputs, latency, token usage, and cost are captured at every step.
- **Quality + safety.** Built-in evaluators for hallucination, drift, sensitive-data exposure, prompt-injection. These don't replace human evals, but they catch the long tail at runtime.
- **Agentic workflow monitoring.** End-to-end visibility across multi-step agent flows — the hop count, the tool-call pattern, the human-in-the-loop handoff. This is the surface that didn't exist three years ago.

The OpenTelemetry GenAI semantic conventions (the `gen_ai.*` span attributes) are the de facto standard for emitting this telemetry in a vendor-neutral way ([Datadog OTel semconv post, 2026-05-26](https://www.datadoghq.com/blog/llm-otel-semantic-convention/)). All four platforms in the compare-set ingest OTel; the question is what each one does with it.

### 3.2 Datadog — broad SaaS, statistical anomaly detection

Datadog's architectural premise is breadth: one SaaS platform spanning metrics, logs, traces, RUM, security, and now LLM observability, with a unified UI and a strong agent (the Datadog Agent) deployed close to the workload. Watchdog AI is the anomaly-detection engine — statistical, processes billions of data points, identifies anomalies, and points at probable root causes ([Datadog LLM Observability, 2026-05-26](https://docs.datadoghq.com/llm_observability/)). Bits AI is the conversational layer that lets responders ask natural-language questions inside an incident.

The strengths: rich UI, mature integrations, a FedRAMP High option via GovCloud. The weaknesses: at large telemetry volumes the indexed-everything pricing model dominates the bill, and Watchdog's statistical model is less explainable than Dynatrace's causal graph.

### 3.3 Dynatrace — Smartscape topology + Davis causal AI

Dynatrace's premise is opposite: causation, not correlation. Davis AI is a reasoning engine that traverses the Smartscape topology graph to establish *causality* — finding the root cause based on interdependencies and event timelines in a deterministic manner ([Dynatrace causation engine, 2026-05-26](https://www.dynatrace.com/news/blog/next-generation-dynatrace-davis-ai-becomes-the-default-causation-engine/)). 2026's Davis AI release pushes toward *preventive* operations — predicting and preventing incidents before they occur, not just speeding up response after they do ([Dynatrace preventive operations, 2026-05-26](https://www.dynatrace.com/news/blog/advancing-aiops-preventive-operations-powered-by-davis-ai/)).

The strengths: deterministic root-cause output that engineers can trust, less alert noise, strong story for highly-interconnected microservices. The weaknesses: heavy OneAgent footprint, steeper licensing model, and a UI culture that takes longer to onboard than Datadog's.

### 3.4 New Relic — agent-monitoring framing, model comparison workflows

New Relic's 2026 focus has been agent-monitoring as a first-class workflow. Their AI Agent Monitoring posture is shaped around the use case of swapping models — e.g., moving from GPT-4 to Claude — and comparing impact on response times, token usage, and budget alignment ([New Relic AI Observability, 2026-05-26](https://newrelic.com/blog/news/scaling-ai-agents-ai-observability)). The platform leans on apdex-driven framing (legacy strength) plus an assistant UX.

Strengths: usage-based pricing can be cheaper for spiky workloads; model-comparison workflows are well-built. Weaknesses: less topology-aware than Dynatrace, less broad than Datadog's surface.

### 3.5 Coralogix — Streama in-stream processing, per-GB ingest pricing

Coralogix's premise is data-pipeline-first: the Streama engine processes telemetry *in-stream* before indexing, classifying logs into "instant analysis" vs "archive" tiers via a TCO optimizer ([Coralogix vs New Relic, 2026-05-26](https://coralogix.com/coralogix-vs-new-relic/)). The claim is monitoring 4× more data at lower cost by analysing before indexing.

Strengths: predictable per-GB ingest pricing, strong fit for high-volume logging workloads, modern Kubernetes integration. Weaknesses: smaller integration catalog than Datadog or Dynatrace; the architectural shift to streaming requires teams to model their data pipelines explicitly rather than rely on indexed-everything defaults.

### 3.6 The three axes that actually distinguish them at scale

| Axis | Datadog | Dynatrace | New Relic | Coralogix |
|------|---------|-----------|-----------|-----------|
| Data-pipeline model | Index-on-ingest | Index-on-ingest + Grail lakehouse | Index-on-ingest, usage-billed | In-stream Streama, tiered |
| Root-cause engine | Watchdog (statistical) | Davis (causal, topology-aware) | Assistant + apdex | Streama rules + ML |
| Pricing/lock-in posture | Per-host + per-feature | Per-DPS host-units | Usage (data + user) | Per-GB ingest |
| FedRAMP availability | High (GovCloud) | High | Moderate | Moderate |
| LLM-observability native | Yes | Yes (via Davis) | Yes (AIM) | Yes |
| OTel GenAI semconv | Native | Native | Native | Native |

The trade-off teams keep underestimating is the *data-pipeline* axis: a platform whose model is "index everything" will dominate the bill at any non-trivial telemetry volume regardless of the per-host sticker price.

## 4. Generic Implementation

A generic example of emitting AI-app telemetry in a vendor-neutral way using OpenTelemetry GenAI semantic conventions. All four platforms ingest this without code change.

```python
from opentelemetry import trace
from opentelemetry.trace import Status, StatusCode

tracer = trace.get_tracer("checkout-recommender")

def get_recommendation(user_id: str, cart_items: list[str]) -> dict:
    # Generic e-commerce example: recommend a checkout add-on.
    with tracer.start_as_current_span("llm.recommend") as span:
        # OpenTelemetry GenAI semantic conventions: gen_ai.* attributes.
        # Note: `gen_ai.system` (older name) was renamed to `gen_ai.provider.name` (current).
        # See W05 Tue file 4 (`4-span-attributes-gen-ai-semconv.md`) for semconv-currency details.
        span.set_attribute("gen_ai.provider.name", "anthropic")
        span.set_attribute("gen_ai.request.model", "claude-sonnet-current")
        span.set_attribute("gen_ai.operation.name", "chat")
        span.set_attribute("gen_ai.user.id", user_id)  # subject to PII policy
        try:
            response = call_llm(user_id, cart_items)
            # Capture token counts and cost-relevant fields.
            span.set_attribute("gen_ai.usage.input_tokens", response.input_tokens)
            span.set_attribute("gen_ai.usage.output_tokens", response.output_tokens)
            span.set_attribute("gen_ai.response.finish_reason", response.finish_reason)
            span.set_status(Status(StatusCode.OK))
            return response.payload
        except Exception as exc:
            span.set_status(Status(StatusCode.ERROR, str(exc)))
            raise
```

Each AIOps platform reads the same `gen_ai.*` attributes; the dashboards, anomaly detectors, and cost-attribution surfaces are platform-specific. If a team writes its instrumentation against OTel GenAI semconv rather than a vendor SDK, swapping platforms later becomes a configuration change, not a rewrite.

> [!instructor-review]
> Bedrock model IDs evolve quickly. Avoid pinning specific Claude model identifiers in shipped curriculum code samples — use `claude-sonnet-current` or similar placeholder and direct learners to verify the current ID via the AWS Bedrock console within the last 3 months per the known-bad-patterns blocklist (`bedrock-old-model-ids`).

> [!instructor-review]
> OTel GenAI semconv is in active evolution. `gen_ai.system` (older name) was renamed to `gen_ai.provider.name` (current). The code sample above uses the current spelling; learners who encounter `gen_ai.system` in older docs or vendor blog posts should treat it as the legacy alias. See W05 Tue file 4 (`4-span-attributes-gen-ai-semconv.md`) for the semconv-currency details. Sunday-before `/web-research` re-verify per the hot-tech 3-month recency window — semconv field names have churned in this category before and may again.

## 5. Real-world Patterns

**Fintech — trading-platform incident triage**. A trading platform running on a deeply interconnected microservices estate adopted Dynatrace specifically for the Smartscape + Davis causation pairing. The team had been drowning in correlated alerts on Datadog and could not consistently distinguish symptom from cause at incident time. After the migration, deterministic root-cause output collapsed their mean-time-to-resolution because responders trusted the engine's conclusion enough to skip ad-hoc topology spelunking ([Dynatrace causal AI overview, 2026-05-26](https://www.dynatrace.com/news/blog/next-generation-dynatrace-davis-ai-becomes-the-default-causation-engine/)).

**Healthcare imaging — model swap workflow**. A healthcare-imaging SaaS that periodically swaps between general-purpose LLMs for radiology-report summarisation chose New Relic AI Monitoring expressly for the model-comparison workflow. The team's recurring decision — "is the new model worth the higher token cost?" — maps onto the New Relic UX of side-by-side latency-vs-cost comparison ([New Relic AI Observability, 2026-05-26](https://newrelic.com/blog/news/scaling-ai-agents-ai-observability)).

**Gaming — high-volume logs at predictable cost**. A free-to-play gaming studio emitting tens of terabytes of player-event logs per day moved to Coralogix to pin observability cost to a defensible per-GB rate. The Streama engine's "analyse before index" model lets the studio keep behavioural anomaly detection on high-volume streams while archiving the bulk of raw logs for forensics — a workload shape where index-on-ingest pricing would have been ruinous ([Coralogix vs New Relic, 2026-05-26](https://coralogix.com/coralogix-vs-new-relic/)).

**E-commerce — broad SaaS for breadth-of-stack**. A multi-brand e-commerce platform standardised on Datadog because the requirement was breadth: Angular RUM for the storefront, Java APM for the order service, Python LLM observability for the recommender, and security signals across all of it under one pane. Watchdog AI's statistical anomaly detection is less explainable than causal RCA, but the integration breadth and UI familiarity earned its keep across a heterogeneous team ([Datadog observability in the AI age, 2026-05-26](https://www.datadoghq.com/blog/datadog-ai-innovation/)).

## 6. Best Practices

- **Instrument against OpenTelemetry GenAI semantic conventions, not a vendor SDK.** Keep the vendor swap reversible — write code that emits `gen_ai.*` span attributes and let the platform consume them.
- **Pick the platform on data-pipeline economics before features.** At any non-trivial telemetry volume the data-pipeline model dominates the bill; sticker-price-per-host is a distraction.
- **Don't conflate root-cause hint with root cause.** Statistical anomaly detectors (Watchdog, Streama) suggest probabilities. Causal engines (Davis) make determinable claims. Match the engine to the trust level your incident process expects.
- **Onboard one model layer at a time.** Stand up host + APM first, then add LLM observability, then add security. Standing up all surfaces at once obscures which dashboards earn their keep.
- **Treat FedRAMP boundary as a hard yes/no, not a negotiation.** Federal-modernization engagements either need it or don't; if they do, the platform shortlist collapses fast.
- **Audit per-tenant cost attribution before committing.** AI-app cost-per-tenant is a load-bearing capability for any multi-tenant deployment; verify the platform actually surfaces it cleanly before signing.
- **Plan for an alternative-vendor compare-matrix even if you've picked your default.** A defensible choice names the conditions under which you'd pivot — engagement shape, telemetry volume, root-cause trust level, lock-in tolerance.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** Choose a hypothetical service you've worked on (or invent one: a multi-tenant SaaS with one Angular SPA, three Java services, one Python LLM orchestrator). For each of the four platforms, write a one-paragraph answer to:

1. What's the headline reason to pick this platform for this service?
2. What's the one capability that would make you immediately pivot away?
3. What does the per-tenant cost-attribution story look like on this platform?

**What good looks like.** A finished page has four short paragraphs — one per platform — each grounded in the platform's actual architectural premise (in-stream vs index-on-ingest, causal vs statistical, broad-SaaS vs data-pipeline-first). If the answers come out indistinguishable across platforms, the underlying framing has collapsed into marketing-speak; rewrite until each paragraph would make sense only for that specific platform.

## 8. Key Takeaways

- *What distinguishes AI-on-AI observability from traditional APM?* Token + cost tracing, hallucination/drift evaluators, and agentic-workflow visibility — all converging on OpenTelemetry GenAI semantic conventions.
- *What's the architectural premise of each platform in the compare-set?* Datadog: broad SaaS, statistical anomaly. Dynatrace: causal topology + preventive ops. New Relic: model-swap workflow, usage-billed. Coralogix: in-stream Streama, per-GB ingest.
- *Which axis decides the platform at scale?* Data-pipeline economics — index-on-ingest vs in-stream — dominates the bill at non-trivial telemetry volume regardless of per-host sticker price.
- *How do you keep the platform choice reversible?* Instrument against OpenTelemetry GenAI semantic conventions, not a vendor SDK; the swap then becomes config, not rewrite.

## Sources

1. [Datadog LLM Observability — official docs](https://docs.datadoghq.com/llm_observability/) — retrieved 2026-05-26
2. [Datadog LLM Observability natively supports OpenTelemetry GenAI Semantic Conventions](https://www.datadoghq.com/blog/llm-otel-semantic-convention/) — retrieved 2026-05-26
3. [Advancing AIOps: Preventive operations powered by Davis AI (Dynatrace)](https://www.dynatrace.com/news/blog/advancing-aiops-preventive-operations-powered-by-davis-ai/) — retrieved 2026-05-26
4. [Scaling AI Agents With AI Observability (New Relic)](https://newrelic.com/blog/news/scaling-ai-agents-ai-observability) — retrieved 2026-05-26
5. [Coralogix vs New Relic — Streama architecture](https://coralogix.com/coralogix-vs-new-relic/) — retrieved 2026-05-26

Last verified: 2026-05-26
