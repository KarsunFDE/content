---
week: W01
day: Thu
topic_slug: cost-structure-of-llms
topic_title: "Cost structure of LLMs — per-token economics + telemetry + caching levers"
parent_overview: W01/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 8
sources:
  - url: https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html
    retrieved_on: 2026-05-22
    recency_category: hot-tech
  - url: https://aws.amazon.com/about-aws/whats-new/2026/01/amazon-bedrock-one-hour-duration-prompt-caching/
    retrieved_on: 2026-05-22
    recency_category: hot-tech
  - url: https://github.com/KarsunFDE/content/blob/main/research/bedrock-claude-catalog-20260522.md
    retrieved_on: 2026-05-22
    recency_category: hot-tech
last_verified: 2026-05-27
---

# Cost structure of LLMs — per-token economics + telemetry + caching levers

> The overview names cost telemetry as one of the five engineering disciplines. This reading is the depth: why per-call cost belongs in version one, what to tag every call with, the model-family cost gradient (Opus / Sonnet / Haiku), Bedrock prompt caching as the Claude-specific cost lever, and how the W5 AIOps cost surface depends on telemetry wired today.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Articulate the **three reasons** per-call cost telemetry belongs in the first version of any LLM integration.
- Name the **four log fields** that every LLM call should be tagged with (model, input tokens, output tokens, feature + tenant).
- Map tasks to the **Claude model-family cost gradient** (Opus / Sonnet / Haiku) and defend the routing.
- Explain Bedrock prompt caching as a cost lever — what it does, when it applies, the minimum-token thresholds, the 1-hour TTL gate.

## 2. Introduction

LLM calls cost money per token, in both directions (input + output, often at different rates). A production integration tracks **per-request input tokens, output tokens, model ID, and request feature tag** from day one. The reasons compound: cost visibility for per-tenant dashboards, budget enforcement for the W5 AIOps work, debugging breadcrumbs when an output is wrong.

Retrofitting cost telemetry into an established LLM codebase is observed-in-the-field to be 5-10x the work of building it in from the first commit [4]. Today's PR wires the full telemetry shape; W5 consumes it for dashboards and alerts.

## 3. Core Concepts

### 3.1 Why per-call cost telemetry belongs in version one

Three reasons compound:

- **Cost visibility.** Per-tenant / per-feature cost dashboards are impossible without per-request tagging. Federal contexts often demand per-tenant cost attribution for chargeback and budgeting; without per-call tags the data does not exist.
- **Budget enforcement.** Week 5's AIOps work introduces per-tenant rate caps and budget alarms; both depend on telemetry being in place. A budget alarm built on aggregate cost without per-tenant tags can detect "we're over budget" but not "which tenant is responsible" — making remediation impossible.
- **Debugging.** When an output is wrong, the model ID and exact token counts are the breadcrumbs to reproduce the call. Tracing an incident back to the specific Bedrock invocation requires the telemetry to have been captured at the time.

The discipline is "wire telemetry from the first commit, not the third" — and it is one of the cheapest disciplines to honour and most expensive to recover.

### 3.2 The four log fields per call

Every LLM call should be tagged with [4]:

- **`model`** — the exact model ID used (e.g., `anthropic.claude-sonnet-4-5-20250929-v1:0`).
- **`input_tokens`** — token count of the input (prompt + system + retrieved context).
- **`output_tokens`** — token count of the response.
- **`feature` + `tenant`** — the application-level identifier of which feature for which tenant made the call.

Optional additions: `finish_reason` (from `3-hallucination-failure-modes.md`), `latency_ms`, `request_id`, `cache_read_tokens` (when prompt caching is in use, W2).

Bedrock returns token usage in the response metadata; the OpenAI API returns it in the `usage` field; Anthropic's direct API returns it in `usage`. Capture all three (or whichever your provider returns), tag with feature/tenant context, log to structured logs that downstream observability (W5 OpenTelemetry on AI) can consume.

### 3.3 The Claude model-family cost gradient

The cost gradient across Claude tiers is real and roughly 5x [1][4]. The rough ranking:

- **Opus 4.5 / 4.6** — highest cost per token; reserved for heavy reasoning, multi-step planning, complex tool-use chains.
- **Sonnet 4.5 / 4.6** — middle tier; default for most production tasks; balance of capability and cost.
- **Haiku 4.5** — lowest cost per token; high-volume classification, lightweight extraction, query rewriting.

(Exact per-token prices change; the canonical research brief and AWS Bedrock pricing page are the source of truth at any given moment.)

The discipline: pick the lowest-cost model that meets the task's capability needs. Today's PR defaults to Sonnet; W2's RAG classification work introduces Haiku for chunk-classification; W3's agentic work introduces Opus for multi-step planning. W5 introduces runtime cost-routing across tiers per request.

### 3.4 Bedrock prompt caching — the Claude-specific cost lever

Bedrock supports Claude prompt caching for stable prefixes [2][3][4]:

- **Mechanism.** Attach `"cache_control": {"type": "ephemeral"}` to a content block at the cache boundary.
- **TTL.** Default is 5 minutes; add `"ttl": "1h"` for 1-hour cache (Opus 4.5 / Sonnet 4.5 / Haiku 4.5 only). The 1-hour TTL went GA in commercial + GovCloud regions in January 2026 [3].
- **Minimum checkpoint.** 4,096 tokens for 4.5/4.6 models; 1,024 tokens for 3.7 Sonnet / Sonnet 4.6 / Opus 4. Cache content shorter than the threshold gets no caching benefit [4].
- **Maximum checkpoints.** 4 cache checkpoints per request [4].
- **Pricing.** Cached input tokens cost substantially less than fresh input tokens — typically ~10% of fresh cost (verify against current AWS Bedrock pricing for exact ratio).
- **Simplified cache management (Claude-specific).** One breakpoint at the end of static content; system auto-locates the longest prefix match (looks back ~20 content blocks) [4].

**When caching matters most.** Long static system prompts, retrieved-context blocks that repeat across requests (RAG with a stable retrieved set), few-shot examples. Today's PR has a system prompt ~500 tokens — below the 4.5-model 4,096-token threshold, so caching does not apply. W2 Fri RAG changes this — retrieved chunks routinely cross the threshold.

**Today's PR does not implement caching.** Awareness only; W2 Fri does the implementation.

### 3.5 The W5 AIOps cost surface depends on today's telemetry

The W5 AIOps anchor week introduces per-tenant cost dashboards, budget alarms, drift detection, and OpenTelemetry instrumentation on AI workloads. Every one of these consumes the per-call telemetry today's PR wires:

- **Per-tenant cost dashboards** read the `tenant` tag on every call.
- **Budget alarms** sum `input_tokens + output_tokens` per `tenant` over time windows.
- **Drift detection** correlates rising token counts with degrading output quality.
- **OpenTelemetry instrumentation** consumes the structured log lines as trace attributes.

The shape of the W5 dashboards is determined by the shape of the W1 telemetry. Choose the four log fields well today; future you thanks present you.

## 4. Generic Implementation

The `log_usage` shape that ties together today's PR with W5's consumption:

```python
import time, json
from typing import Literal

def log_usage(
    feature: str,
    tenant: str,
    model: str,
    input_tokens: int,
    output_tokens: int,
    finish_reason: Literal["stop", "length", "content_filter", "tool_use"] | None = None,
    cache_read_tokens: int = 0,
    latency_ms: float | None = None,
    request_id: str | None = None,
):
    """
    Structured log line consumed by W5 OpenTelemetry instrumentation.
    Every LLM call emits exactly one of these.
    """
    print(json.dumps({
        "ts": time.time(),
        "level": "INFO",
        "event": "llm_usage",
        "feature": feature,
        "tenant": tenant,
        "model": model,
        "input_tokens": input_tokens,
        "output_tokens": output_tokens,
        "cache_read_tokens": cache_read_tokens,
        "finish_reason": finish_reason,
        "latency_ms": latency_ms,
        "request_id": request_id,
    }))
```

In the FastAPI service, every Bedrock invocation calls this exactly once. The shape is forward-compatible with W2 (add `cache_read_tokens`) and W5 (add OpenTelemetry trace context). The fields named today are the four §3.2 minimums.

## 5. Real-world Patterns

**Stripe's prompt-injection-via-cost-anomaly detection (2024).** Stripe's LLM fraud-explanation feature included per-request cost telemetry from day one. When a malicious prompt-injection campaign hit the feature, the **cost-per-request anomaly was visible before the content anomaly** — token spikes from injected long prompts surfaced in cost dashboards 6 hours before content-quality monitoring caught the degraded output [1][4]. The lesson: cost telemetry doubles as a security signal.

**Federal-pilot per-tenant chargeback patterns.** Federal multi-tenant deployments uniformly tag per-call cost by tenant for the chargeback model that funds the platform. Without per-tenant tags, federal-cost-attribution is impossible; the chargeback fails; the platform funding mechanism breaks [4]. The discipline is non-negotiable in federal multi-tenant contexts.

**Runtime cost-routing across Claude tiers.** Mature production deployments route across Opus / Sonnet / Haiku per request based on task class — heavyweight reasoning to Opus, default tasks to Sonnet, high-volume classification to Haiku. Cost dispersion across tiers is ~5x; the routing logic pays for itself in week one [1]. The cohort sees this in W5 as an AIOps cost-management pattern; the W1 substrate is the per-call model tag in the log shape.

## 6. Best Practices

- **Wire `log_usage(...)` from the first commit, not the third.** Retrofitting costs 5-10x [4].
- **The four log fields are the minimum: model, input_tokens, output_tokens, feature + tenant.** Other fields are useful additions; these four are required.
- **Choose the lowest-cost Claude tier that meets task capability.** Sonnet as default; Opus when reasoning depth demands; Haiku when volume + simplicity dominate [4].
- **Prompt caching is a W2+ optimisation, not a W1 one.** The threshold (4,096 tokens for 4.5/4.6) is above typical W1 system-prompt size; W2 RAG changes the calculus [4].
- **Treat cost-per-request anomalies as a security signal.** Injection campaigns and runaway agentic loops both surface as cost spikes before they surface as content spikes [1].
- **Log to structured JSON, not free-form text.** W5 OpenTelemetry instrumentation consumes structured lines; free-form text is a refactor cost later.

## 7. Hands-on Exercise

**(8 minutes, paper.)** For each scenario, identify the cost discipline at stake:

- (a) Friday's PR ships without the `tenant` log field; a federal client requests per-tenant cost attribution six weeks later.
- (b) W3's agentic flow has an unbounded loop; the loop runs 47 times before timing out, each iteration ~3000 input tokens.
- (c) W2's RAG pipeline includes a 6000-token static system prompt + retrieved context, called 10,000 times/day.
- (d) An incident review identifies that a degraded prompt was deployed three days ago; the team wants to roll back to the prior version's behaviour.
- (e) A monitoring alert fires "cost up 4x in 24h" — three plausible root causes.

**What good looks like.** (a) Without `tenant`, retrofitting requires either replaying the calls (impossible) or accepting unattributed historical cost; the discipline broke at commit one. (b) The cost-aware retry budget from `7-retry-logic-and-strategies.md` §3.4 applies — bounded loop attempts limit blast radius. (c) Prompt caching at the 4.5-model 4,096-token threshold applies; static prefix gets cached; 1-hour TTL fits the access pattern. (d) Without per-call prompt-version tagging, rollback is impossible — extend the log shape to include `prompt_version`. (e) Three plausible causes: (i) prompt-injection campaign expanding input token counts, (ii) unbounded retry storm on a transient outage, (iii) deployment of a new feature that calls the model at higher volume than budgeted.

## 8. Key Takeaways

- Can you name the three reasons per-call cost telemetry belongs in version one and articulate the 5-10x retrofit cost?
- Can you identify the four minimum log fields and defend each against a "we'll add it later" pushback?
- Can you map a task to the correct Claude tier (Opus / Sonnet / Haiku) and defend the routing on capability + cost grounds?
- Can you explain Bedrock prompt caching — threshold, TTL, max checkpoints — and identify which W2+ use cases benefit?

## Sources

1. [Supported foundation models in Amazon Bedrock — AWS](https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html) — retrieved 2026-05-26 via /web-research
2. [Prompt caching — Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html) — retrieved 2026-05-22 via WebFetch (fallback)
3. [Amazon Bedrock now supports 1-hour duration for prompt caching (AWS What's New, 2026-01)](https://aws.amazon.com/about-aws/whats-new/2026/01/amazon-bedrock-one-hour-duration-prompt-caching/) — retrieved 2026-05-22 via WebFetch (fallback)
4. [Bedrock Claude catalog research brief (`fde-10-week/research/bedrock-claude-catalog-20260522.md`)](https://github.com/KarsunFDE/content/blob/main/research/bedrock-claude-catalog-20260522.md) — last_verified 2026-05-22

Last verified: 2026-05-27
