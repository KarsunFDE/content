---
week: W01
day: Thu
topic_slug: retry-logic-and-strategies
topic_title: "Retry logic and strategies — backoff, jitter, circuit-break, never-retry"
parent_overview: W01/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 8
sources:
  - url: https://callsphere.ai/blog/retry-strategies-llm-api-calls-exponential-backoff-jitter-tenacity
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/
    retrieved_on: 2026-05-22
    recency_category: foundation-stable
last_verified: 2026-05-27
---

# Retry logic and strategies — backoff, jitter, circuit-break, never-retry

> The overview names retry as one of the five engineering disciplines. This reading is the depth: which HTTP status codes are retryable and which never are, why jitter alone is worth a 60-80% reduction in retry storms, the bounded-attempt + circuit-breaker shape, and the LLM-specific contour on top of the traditional HTTP retry pattern.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Classify HTTP status codes into **retryable / never-retryable** and defend each classification with one sentence.
- Implement **exponential backoff with jitter** correctly — the jitter math matters; constant-delay retry is the antipattern.
- Bound retry attempts and **circuit-break** on persistent failure rather than retry indefinitely.
- Identify the **LLM-specific retry contours** on top of the traditional HTTP retry pattern.

## 2. Introduction

LLM APIs fail at provider level [1] at a non-trivial rate — typical commercial APIs report 1-5% transient failure plus higher rates during regional incidents. A production-shaped retry policy follows traditional HTTP-API-integration rules with LLM-specific tuning. The traditional discipline is well-understood; the LLM-specific contours are about content-filter rejections, model-ID errors, and shared quota considerations that don't appear in non-LLM APIs.

AWS's own published research on distributed-system retries [3] reports 60-80% reduction in retry storms from jitter alone — a foundation-stable result that applies directly to Bedrock retry posture.

## 3. Core Concepts

### 3.1 Retryable vs never-retryable status codes

The classification is decisive — retrying a never-retryable error wastes quota, slows incident recovery, and produces misleading logs.

| Status | Class | Action | Rationale |
|--------|-------|--------|-----------|
| 200 | Success | Don't retry | Result is in hand |
| 400 | Bad Request | **Never retry** | Malformed request; retrying produces identical failure |
| 401 / 403 | Auth | **Never retry** | Credential or permission problem; retry will not fix it |
| 404 | Not Found | **Never retry** | Resource not found; retry produces identical result |
| 422 | Content filter | **Never retry** (LLM-specific) | Bedrock content filter rejected; retry produces identical rejection |
| 429 | Rate-limited | **Retry** with backoff + `Retry-After` honored | Transient; provider explicitly signaling retry |
| 500 / 502 / 503 / 504 | Server transient | **Retry** with backoff | Transient server-side issue |
| Connection timeout / reset | Network | **Retry** with backoff | Transient network issue |

The classification is HTTP-standard; the LLM-specific addition is 422 (Bedrock content filter). Retrying a content-filter rejection wastes a request and an audit log entry.

### 3.2 Exponential backoff with jitter — the math

The schedule for retrying a transient failure [1][3]:

- **Base delay:** 1 second.
- **Exponential growth:** delay = `2 ** attempt` (1s, 2s, 4s, 8s, 16s).
- **Jitter:** add ±25% random variation to break synchronised retry storms.
- **Honor `Retry-After`:** when the provider sends this header, use it as the floor.

The Python idiom:

```python
import random, time

def jittered_delay(attempt: int) -> float:
    base = 2 ** attempt                           # 1, 2, 4, 8, 16
    jitter = random.uniform(-0.25, 0.25) * base   # ±25%
    return max(0.1, base + jitter)

for attempt in range(5):
    try:
        return bedrock.invoke_model(...)
    except APIStatusError as e:
        if e.status_code in {400, 401, 403, 404, 422}:
            raise                                  # never retry
        retry_after = float(e.response.headers.get("Retry-After", 0))
        delay = max(jittered_delay(attempt), retry_after)
        time.sleep(delay)
raise RuntimeError("LLM call failed after 5 attempts")
```

**Why jitter alone is worth the 60-80% reduction.** Without jitter, multiple clients failing at the same moment all retry at the same delays (1s, 2s, 4s ...). The aggregate retry traffic is a synchronised spike at each backoff boundary — the thundering herd. Jitter breaks the synchronisation; the aggregate retry traffic becomes a smooth curve. AWS's published research [3] documents this with quantified production numbers.

### 3.3 Bounded attempts + circuit breaker

Unbounded retry is the antipattern. The bound:

- **User-facing requests:** 3-5 attempts, then fail to the caller with a clear error.
- **Background jobs:** 5-7 attempts, then fail to dead-letter for human triage.
- **Persistent failure across multiple requests:** circuit-break.

The circuit breaker [3] is a three-state state machine wrapping the call:

- **Closed** (normal) — calls pass through; failures increment a counter.
- **Open** (failing) — calls fail fast without hitting the API; the counter triggered the open after threshold (e.g., 5 failures in 30s).
- **Half-open** (probing) — after a cool-down period (e.g., 30s), one call is allowed through; success closes the circuit, failure reopens it.

Today's PR does not implement a full circuit breaker — the bounded-attempt loop is sufficient for W1. W5 AIOps adds the circuit-breaker layer as part of production observability.

### 3.4 LLM-specific retry contours

Three places the traditional HTTP retry pattern bends for LLMs:

- **Content filter (422).** Bedrock content filters reject prompts that violate the model's safety policy. Retrying produces identical rejection. The downstream code should surface this as a distinct error class — not a generic failure — so the calling code can react appropriately (e.g., escalate to HITL, return a sanitised error to the user).
- **Shared quota.** Rate limits are often shared across an organisation; one app's naive retry storm consumes another app's quota. Jitter matters even more here than in non-LLM APIs [1].
- **Token-cost-aware retry budget.** Retrying a long-prompt failure consumes input tokens each time. For very long prompts (RAG with extensive retrieved context in W2), a `max_retries=5` budget can mean 5x the input-token cost. W5 AIOps adds cost-aware retry decisioning; W1 keeps retry budget tight (3 attempts user-facing).

### 3.5 The thundering-herd consideration

AWS's research [3]: when a regional Bedrock outage briefly affects all clients, the post-outage recovery period is when client-side retries can do real damage. Clients that hammer the recovering service with no jitter slow recovery; clients with jitter let the service recover. The math is published: 60-80% reduction in retry traffic from jitter alone, with no other change.

In federal contexts, the considerations are sharper:
- Shared GovCloud quota across agency tenants.
- Audit trail volume from retry attempts.
- Cost attribution when retries consume input tokens.

## 4. Generic Implementation

The retry wrapper composed with the rest of today's PR. Combines never-retry classification, jittered backoff, bounded attempts, and `Retry-After` honoring:

```python
import os, json, random, time, boto3
from botocore.exceptions import ClientError

bedrock = boto3.client("bedrock-runtime", region_name=os.environ["AWS_REGION"])
MODEL_ID = os.environ["LLM_MODEL_ID"]

NEVER_RETRY = {400, 401, 403, 404, 422}
MAX_ATTEMPTS = 3

def call_with_retry(body: dict) -> dict:
    for attempt in range(MAX_ATTEMPTS):
        try:
            resp = bedrock.invoke_model(modelId=MODEL_ID, body=json.dumps(body))
            return json.loads(resp["body"].read())
        except ClientError as e:
            status = e.response["ResponseMetadata"]["HTTPStatusCode"]
            if status in NEVER_RETRY:
                raise
            if attempt == MAX_ATTEMPTS - 1:
                raise
            retry_after = float(e.response.get("Retry-After", 0))
            base = 2 ** attempt
            jitter = random.uniform(-0.25, 0.25) * base
            delay = max(0.1, base + jitter, retry_after)
            time.sleep(delay)
    raise RuntimeError("unreachable")
```

The wrapper composes with streaming and cost-telemetry per their own files. Streaming + retry composition: retry wraps the entire streaming call, not individual chunks — mid-stream failures abort the stream and retry from the start.

## 5. Real-world Patterns

**Shopify catalog-description retry incident (2024).** Shopify's catalog-description generation faced a documented incident where naive client retries during a regional Bedrock outage amplified the impact; the post-incident fix introduced jitter + circuit-breakers across the fleet [1]. The pattern: naive retry without jitter accelerates outages rather than mitigating them.

**AWS Builders' Library — the canonical retry+jitter reference.** AWS's own engineering organisation publishes the canonical guidance on retry + backoff + jitter [3] — required reading for any production AWS-touching code. The 60-80% retry-storm reduction number comes from this source; it is the basis for jitter being mandatory rather than optional.

**Stripe's content-filter-rejection class.** Stripe's LLM fraud-explanation feature surfaces content-filter rejections as a distinct downstream class rather than a generic 4xx, enabling targeted incident response — content-filter spikes get a different runbook than rate-limit spikes [1]. The pattern: surface LLM-specific error classes distinctly rather than collapsing into "the LLM failed".

## 6. Best Practices

- **Always classify status codes; never `retry on any exception`.** Generic retry is the antipattern; quota waste and misleading logs are the cost.
- **Use ±25% jitter on exponential backoff.** The math is well-established; constant-delay retry produces synchronised retry storms [3].
- **Honor `Retry-After` as a floor, not a ceiling.** The server knows its own capacity; clients should respect the signal.
- **Bound attempts at 3-5 user-facing, 5-7 background.** Beyond that, fail to caller or dead-letter.
- **Surface LLM-specific error classes (content filter, model-ID error) distinctly from generic 4xx.** Downstream code can react appropriately; runbooks can target the right failure mode.
- **Wrap the entire streaming call in retry, not individual chunks.** Mid-stream failures abort and restart; the alternative requires partial-output recovery that is its own engineering surface.

## 7. Hands-on Exercise

**(8 minutes, paper.)** For each of these failure scenarios, write the correct retry response:

- (a) Bedrock returns 429 with `Retry-After: 5` header on attempt 1.
- (b) Bedrock returns 401 (the IAM role lost permission overnight).
- (c) Bedrock returns 503 on attempts 1, 2, 3 with `MAX_ATTEMPTS=3`.
- (d) Bedrock content filter returns 422 on attempt 1 ("prompt contains potentially sensitive content").
- (e) The Python `boto3` call raises a connection timeout on attempt 1.

**What good looks like.** (a) Wait `max(5s, jittered_delay(0))` = 5s, retry. (b) Raise immediately; never retry auth. (c) After attempt 3 fails, raise to caller with clear error. (d) Raise immediately as a `ContentFilterRejection`-class error; never retry. (e) Retry with `jittered_delay(0)`; connection timeouts are transient.

## 8. Key Takeaways

- Can you classify the eight common HTTP status codes (200/400/401/403/404/422/429/5xx) into retryable vs never-retryable and defend each?
- Can you implement exponential backoff with jitter correctly, including honoring `Retry-After`?
- Can you bound retry attempts appropriately for user-facing vs background contexts?
- Can you identify the three LLM-specific retry contours (content filter, shared quota, token-cost-aware budgeting) and defend each?

## Sources

1. [Retry Strategies for LLM API Calls — CallSphere](https://callsphere.ai/blog/retry-strategies-llm-api-calls-exponential-backoff-jitter-tenacity) — retrieved 2026-05-26 via /web-research
2. [Supported foundation models in Amazon Bedrock — AWS](https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html) — retrieved 2026-05-26 via /web-research
3. [Timeouts, Retries, and Backoff with Jitter — AWS Builders' Library](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) — retrieved 2026-05-22 via WebFetch (fallback)

Last verified: 2026-05-27
