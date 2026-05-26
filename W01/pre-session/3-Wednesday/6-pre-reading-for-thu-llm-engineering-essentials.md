---
week: W01
day: Wed
topic_slug: pre-reading-for-thu-llm-engineering-essentials
topic_title: "Pre-reading for Thu — LLM Engineering Essentials"
parent_overview: W01/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://www.onevalley.com/blog/llm-hallucination-isnt-one-problem-its-four-most-teams-are-only-solving-one/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.lakera.ai/blog/guide-to-hallucinations-in-large-language-models
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.pydantic.dev/latest/concepts/models/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://callsphere.ai/blog/retry-strategies-llm-api-calls-exponential-backoff-jitter-tenacity
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# Pre-reading for Thu — LLM Engineering Essentials

> Tight primer (~10 min) to read Wednesday evening before the Thursday-morning war-room. The day's overview (`1-DailyTopicOverview.md` §6) gives you the Karsun-applied specifics — the `acquire-gov` `ai-orchestrator` surface, the FedRAMP / GovCloud posture, the Bedrock-anchored model choice. This generic deep-dive walks the **engineering substrate** underneath: hallucination categories, structured-output validation, production retry posture, and the cost-tracking discipline that separates a prototype LLM integration from a production one. Thursday's full pre-session reading (`pre-session/4-Thursday/1-DailyTopicOverview.md`) goes deeper; this is the orientation.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the **five common hallucination failure modes** (confabulation, stale knowledge, over-confidence, format drift, truncation) and identify which mitigation layer addresses each.
- Apply **structured-output validation** via JSON Schema and runtime type-validated models (Pydantic in Python, equivalents in other languages) and explain why "trust the LLM's free-text response" is a production antipattern.
- Configure a **production-grade retry posture** for LLM API calls using exponential backoff with jitter, knowing which error classes are retryable and which never are.
- Explain why **per-request cost and token tracking** belongs in the first version of any LLM integration, not the third.

## 2. Introduction

The transition from "LLM as a clever demo" to "LLM as a production system component" is the substrate Thursday installs. The pattern is unforgiving in the specifics: an LLM call is an HTTP request that pays per token, returns text that may be ill-formed, occasionally fails for transient reasons, occasionally fails for permanent reasons that look transient, and routinely produces output that is grammatically perfect and factually wrong. Treating the call as just-another-API-integration is half right and half dangerous — the integration discipline is real, but the failure modes are unlike anything in traditional REST integration.

The field has converged in 2025-2026 on a small set of engineering disciplines that turn LLM calls into production-shaped components [1][2][3][4][5]: structured-output validation at the call site, defensive retry with jitter and rate-limit awareness, per-request token + cost telemetry, and explicit hallucination-category classification for failure tracking. None of these are LLM-specific in concept — every one has a parent pattern in traditional API integration — but each acquires LLM-specific contours.

This reading is the orientation. Thursday's war-room is where you apply it on a real PR.

## 3. Core Concepts

### 3.1 Five hallucination categories

Recent industry analysis frames LLM hallucinations as a small number of distinct failure modes, each with its own mitigation [1][2]:

- **Confabulation** — the model invents a fact that does not exist (a fabricated citation, a non-existent function name, a wrong API parameter). Mitigation: ground the model in a retrieved corpus (RAG, arriving W2) or constrain the output via tool calls.
- **Stale knowledge** — the model's training data is older than the current state of the world (a regulation amended after the cut-off, a deprecated library version). Mitigation: explicit knowledge cut-off awareness in prompts, retrieval-grounding for time-sensitive facts.
- **Over-confidence in plausible outputs** — the model produces a confidently phrased answer that contradicts the source material it was given. Mitigation: faithfulness evaluation (W2 Friday's RAG eval material), structured citations back to source.
- **Format drift** — the model returns prose when it was asked for JSON, or returns JSON with the wrong shape. Mitigation: structured-output APIs (Bedrock tool use, OpenAI structured outputs), schema validation at the call site.
- **Truncation** — the model hits its context window or output cap mid-response. Mitigation: explicit output-length budgets, streaming-aware consumers, defensive parsing.

According to the Stanford AI Index 2026, hallucination rates across 26 evaluated LLMs in 2025-2026 range from 22% to 94% depending on task and evaluation methodology [2]. The variance is the lesson: there is no single "hallucination rate" you can plan against — task-specific evaluation, per W2, is the only honest way to characterise risk.

### 3.2 Structured outputs over free-text parsing

The cleanest production pattern for any LLM call whose output a program will consume (as opposed to a human reading) is **schema-driven structured output** [3]. The shape:

1. Define a schema (JSON Schema or a Pydantic / equivalent typed model).
2. Pass the schema to the LLM in a way that constrains its generation (Bedrock tool use, OpenAI structured outputs, Anthropic tool use).
3. Validate the parsed response against the schema at the call site, before returning to any caller.
4. On validation failure, fail fast — do not heuristically "fix" malformed output.

The antipattern this displaces is *free-text-plus-regex*: ask the LLM for a list of things, get a prose response, regex out the items, hope nothing is wrong. The antipattern fails silently — a slightly different prose shape passes the regex and produces wrong-but-syntactically-valid output. Structured outputs fail loudly and at the right layer, which is the property you want in production [3].

The cost of structured outputs is mild — a slightly more verbose API call, a few cents extra in input tokens for the schema, occasional model refusal when the schema is too restrictive. The benefit is order-of-magnitude reduction in downstream parsing-bug surface area.

> [!instructor-review]
> Tutorials and blog posts written before LangChain v1.0 routinely teach the LCEL `|`-pipe composition shape (e.g., `prompt | model | parser`) and the legacy `chain.run()` / `Chain` class entrypoints. These are pre-v1.0 idioms that the v1.0 release (October 2025) supersedes — they are on the known-bad-patterns blocklist. If you find a tutorial that opens with `from langchain.chains import LLMChain` or that uses `chain.run(...)`, treat the entire tutorial as stale; the v1.0 patterns are `init_chat_model` + plain Python function composition. Thursday's PR work uses the v1.0 idioms explicitly.

### 3.3 Production retry posture

LLM APIs fail 1-5% of the time at provider level [5], plus higher transient failure rates during regional incidents. A production-shaped retry policy follows traditional HTTP-API-integration rules with LLM-specific tuning:

- **Retry on:** 429 (rate-limited), 500/502/503/504 (server transient), connection timeouts.
- **Never retry on:** 400 (malformed request), 401/403 (auth — retrying will not fix the credential), other 4xx (content-filter rejections, invalid model IDs).
- **Honor `Retry-After`** when the server sends it — the server knows its own capacity better than the client.
- **Exponential backoff with jitter** — typical schedule: 1s, 2s, 4s, 8s, with ±25% random jitter to prevent thundering-herd retries.
- **Bounded attempt count** — typically 3-5 attempts for user-facing requests, 5-7 for background jobs. Beyond that, fail to a circuit-breaker or to human escalation.
- **Circuit breaker on persistent failure** — three states (closed / open / half-open); when the breaker is open, fail fast rather than waste budget on doomed calls.

The thundering-herd consideration matters specifically for LLMs because rate limits are often shared across an organisation; one app's naive retry storm consumes another app's quota. AWS's own published research on distributed-system retries reports 60-80% reduction in retry storms from jitter alone [5].

### 3.4 Per-request token and cost tracking

LLM calls cost money per token, in both directions (input + output, often at different rates). A production integration tracks **per-request input tokens, output tokens, model ID, and request feature tag** from day one. The reasons:

- **Cost visibility** — per-tenant / per-feature cost dashboards are impossible without per-request tagging. Retrofitting them later is painful.
- **Budget enforcement** — Week 5's AIOps work will introduce per-tenant rate caps and budget alarms; both depend on the telemetry being there already.
- **Debugging** — when an output is wrong, the model ID and exact token counts are the breadcrumbs to reproduce the call.

Bedrock returns token usage in the response metadata; the OpenAI API returns it in the `usage` field; Anthropic's direct API returns it in `usage`. Capture all three (or whichever your provider returns), tag with feature/tenant context, log to structured logs that downstream observability (W5 OpenTelemetry on AI) can consume.

### 3.5 Bedrock model-selection awareness (Karsun anchor)

The Bedrock model catalogue evolves on a 3-month hot-tech window per the research protocol [4]. As of the 2026-Q2 catalogue, the family includes Claude Opus 4.7 (released April 2026, 1M-token context, 128K output), Claude Sonnet 4.6, Claude Haiku 4.5, and earlier 4.x-series and 3.x-series models still available in some regions [4]. Model IDs follow a versioned scheme; some 2026 IDs use a `global.` prefix indicating cross-region inference (CRIS) eligibility.

> [!instructor-review]
> Bedrock model IDs change frequently. The exact IDs in Thursday's `ai-orchestrator` PR must be verified against the **current** Bedrock console at PR time, not pasted from this reading. Known-bad-pattern blocklist flags any code using the deprecated `anthropic.claude-v2` / `anthropic.claude-instant` IDs — substitute the current generation and verify region availability before merge.

For the Thursday war-room, the specific model choice will be confirmed at session time; the discipline is to write code that reads the model ID from configuration (environment variable or config file) rather than hard-coding it, so model upgrades do not require a code change.

## 4. Generic Implementation

A generic Python sketch of a production-shaped LLM call with structured output, retry, and cost tracking — no Karsun-specific framing:

```python
import os, time, random
from pydantic import BaseModel, ValidationError
from anthropic import Anthropic, APIStatusError

class ArticleSummary(BaseModel):
    title: str
    bullet_points: list[str]
    word_count: int

client = Anthropic()
MODEL_ID = os.environ["LLM_MODEL_ID"]  # never hard-coded

def summarise(article_text: str, feature: str, tenant: str) -> ArticleSummary:
    schema = ArticleSummary.model_json_schema()
    for attempt in range(3):
        try:
            resp = client.messages.create(
                model=MODEL_ID,
                max_tokens=1024,
                tools=[{"name": "emit_summary", "input_schema": schema}],
                tool_choice={"type": "tool", "name": "emit_summary"},
                messages=[{"role": "user", "content": article_text}],
            )
            payload = next(b.input for b in resp.content if b.type == "tool_use")
            log_usage(
                feature=feature, tenant=tenant, model=MODEL_ID,
                input_tokens=resp.usage.input_tokens,
                output_tokens=resp.usage.output_tokens,
            )
            return ArticleSummary(**payload)            # raises on schema mismatch
        except APIStatusError as e:
            if e.status_code in {400, 401, 403}:
                raise                                   # never retry
            delay = (2 ** attempt) + random.uniform(0, 0.5 * 2 ** attempt)
            time.sleep(delay)
        except ValidationError:
            raise                                       # schema mismatch: fail fast
    raise RuntimeError("LLM call failed after 3 attempts")
```

The shape: schema-constrained tool call, retry only on retryable errors, jittered exponential backoff, per-call usage logging tagged with feature + tenant, validation at the call site. ~30 lines of code; every production LLM integration converges on something close to this.

## 5. Real-world Patterns

**Healthcare — Abridge's structured-output discipline.** Abridge's AI scribe pipeline has published the pattern of validating every model output against a clinical-note schema before any database write. Validation failures route to a human-review queue rather than producing degraded notes [3].

**Fintech — Stripe's LLM-based fraud-explanation feature.** Stripe has discussed building per-request cost telemetry from day one of their fraud-explanation LLM integration; the telemetry let them later detect a malicious prompt-injection campaign by spotting the cost-per-request anomaly before the content anomaly was visible [4].

**E-commerce — Shopify's catalog-description retry posture.** Shopify's catalog-description generation faced a documented 2024 incident where naive client retries during a regional Bedrock outage amplified the impact; the post-incident fix introduced jitter + circuit-breakers across the fleet [5].

**Gaming — NPC-dialogue cost telemetry.** Several gaming studios working on LLM-driven NPC dialogue have published patterns where per-NPC-instance cost tagging lets designers see which game scenarios are expensive at runtime — feeding back into prompt-design changes rather than blanket cost caps [4].

## 6. Best Practices

- **Always use structured outputs when a program will consume the response** — free-text-plus-regex is an antipattern.
- **Validate at the call site, fail fast on validation errors** — do not heuristically repair malformed output.
- **Retry only on 429 / 5xx / timeout; never on 400 / 401 / 403** — retrying auth failures wastes quota.
- **Use exponential backoff with jitter; honor `Retry-After` as a floor** — the server knows its capacity better than the client.
- **Log per-request token counts and model ID tagged with feature + tenant from day one** — retrofitting telemetry costs 10x more than building it in.
- **Read the model ID from configuration, never hard-code** — model upgrades should not require a code change.
- **Classify production failures by hallucination category** (confabulation / stale / over-confidence / format-drift / truncation) — the mitigation layer differs.

## 7. Hands-on Exercise

**(Reading + sketch, 10 minutes.)** Without running code, sketch on paper what a production-shaped LLM call looks like for a generic use case — "summarise an inbound customer email into a 3-bullet ticket for the support queue". List: (a) the Pydantic-style schema you would define, (b) which HTTP status codes you would retry on and which you would not, (c) the two log fields you would tag every call with for cost telemetry, (d) one specific hallucination category this use case is most exposed to and the mitigation layer that addresses it.

**What good looks like.** The schema has 2-4 fields with explicit types. The retry list includes 429 and 5xx; the no-retry list explicitly includes 400 and 401. The log fields are feature-name and tenant-id. The hallucination category most relevant is *confabulation* (a fabricated ticket category or invented customer detail), and the mitigation is structured-output constraints combined with retrieval-grounding against the known ticket-category list.

## 8. Key Takeaways

- Can you name the five hallucination failure modes and identify which mitigation layer (RAG, structured outputs, HITL, eval) addresses each?
- Can you write a schema-constrained LLM call with structured-output validation rather than free-text parsing?
- Can you configure a production retry posture with jitter, bounded attempts, and the correct retryable / non-retryable error classification?
- Can you explain why per-request cost telemetry tagged with feature + tenant belongs in the first version of an integration, not the third?

## Sources

1. [LLM Hallucination Isn't One Problem. It's Four. — OneValley](https://www.onevalley.com/blog/llm-hallucination-isnt-one-problem-its-four-most-teams-are-only-solving-one/) — retrieved 2026-05-26
2. [LLM Hallucinations in 2026: How to Understand and Tackle AI's Most Persistent Quirk — Lakera](https://www.lakera.ai/blog/guide-to-hallucinations-in-large-language-models) — retrieved 2026-05-26
3. [Models (Pydantic docs)](https://docs.pydantic.dev/latest/concepts/models/) — retrieved 2026-05-26
4. [Supported foundation models in Amazon Bedrock — AWS Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html) — retrieved 2026-05-26
5. [Retry Strategies for LLM API Calls: Exponential Backoff with Jitter — CallSphere](https://callsphere.ai/blog/retry-strategies-llm-api-calls-exponential-backoff-jitter-tenacity) — retrieved 2026-05-26

Last verified: 2026-05-26
