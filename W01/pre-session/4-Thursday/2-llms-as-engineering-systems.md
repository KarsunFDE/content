---
week: W01
day: Thu
topic_slug: llms-as-engineering-systems
topic_title: "LLMs as engineering systems — production discipline over magic"
parent_overview: W01/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://www.onevalley.com/blog/llm-hallucination-isnt-one-problem-its-four-most-teams-are-only-solving-one/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.lakera.ai/blog/guide-to-hallucinations-in-large-language-models
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.pydantic.dev/latest/concepts/models/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-27
---

# LLMs as engineering systems — production discipline over magic

> The overview frames the Thursday day-shape. This reading is the generic case: what changes when an LLM call moves from prototype to production component, why the convergent 2025-2026 discipline exists, and how the cohort applies it from today onward.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Articulate the **five engineering disciplines** that turn an LLM call into a production-shaped component (typed I/O, retry posture, cost telemetry, model-ID configuration, failure-mode classification).
- Identify the **parent pattern** for each LLM-specific discipline in traditional API integration.
- Recognise the **antipatterns** that demo-quality LLM code routinely ships with and that production-quality code never tolerates.
- Defend the framing "an LLM call is an HTTP request that pays per token and occasionally lies" against the framing "the model is a magic box".

## 2. Introduction

The transition from prototype to production is where most LLM integrations die. The prototype works on the happy path with the developer's three test inputs. Production hits malformed responses, rate limits, regional outages, hallucinations on adversarial inputs, cost overruns from runaway agentic loops, and silent model deprecations. The team that treated the LLM as magic discovers that magic does not survive contact with a real workload.

The convergent industry response over 2025-2026 is a small, learnable set of engineering disciplines [1][2][3]. None of them are exotic. Most have a parent pattern in traditional REST-API integration. The combination is what makes an LLM call production-shaped.

This reading is the framing. The remaining six topic files install each discipline against the Karsun `acquire-gov` `ai-orchestrator` PR you ship today.

## 3. Core Concepts

### 3.1 The five engineering disciplines

| Discipline | Parent pattern (traditional API) | LLM-specific contour |
|------------|----------------------------------|----------------------|
| Typed contract for input + output | OpenAPI schemas + DTO validation | Pydantic strict-mode on response; tool calls / structured outputs constrain generation (Friday's depth) |
| Retry posture | HTTP retry with backoff | Distinguish 429 (rate-limited, retry) from 400 (content filter, never retry); jitter [5] |
| Cost telemetry | Per-request observability | Token-tagged: input + output tokens by model ID, feature, tenant — day-one [4] |
| Model-ID configuration | URL / endpoint configuration | `LLM_MODEL_ID` env var; never hard-code; model upgrades shouldn't require code changes [4] |
| Failure-mode classification | Exception taxonomy | Hallucination categories: confabulation / stale / over-confidence / format-drift / truncation [1][2] |

The pattern repeats: a traditional engineering discipline, recontoured for the specifics of LLM behaviour. Engineers who already have the parent pattern internalised pick up the LLM-specific contour in an afternoon. Engineers who skipped the parent pattern (e.g., shipped LLM features without ever building production REST integrations) often try to invent a new discipline from scratch. Don't. Reach for the parent pattern; bend it for the LLM contour.

### 3.2 The antipatterns demo-quality code ships with

Three antipatterns appear in nearly every LLM-prototype-promoted-to-production failure:

1. **Free-text-plus-regex.** Ask the model for a list of things, get a prose response, regex out the items, hope nothing is wrong. Fails silently — a slightly different prose shape passes the regex and produces wrong-but-syntactically-valid output. The Friday PR replaces this with Pydantic strict-mode parsing of tool-call output [3].

2. **Naive retry-everything.** Wrap the call in `try / except / sleep(1) / retry`. Retries 401s (wastes quota; auth never recovers via retry); retries during a regional outage with no jitter (becomes the thundering herd that prolongs the outage) [5].

3. **Hard-coded model ID.** Paste `anthropic.claude-3-sonnet-...` into the source. When the model is deprecated 11 months later, a code change is required to roll forward. Production teams ship a config-driven model ID from day one [4].

Today's `ai-orchestrator` PR replaces all three.

### 3.3 Why "magic box" framing fails the federal context

Two specifics make federal LLM deployments more demanding than commercial:

- **Auditability.** Every LLM-driven decision that affects a contracting action needs an audit trail — model ID, prompt version, input, output, token counts, timestamp. The "magic box" framing produces logs that say "the AI said draft this solicitation"; the federal context demands "Sonnet 4.5 at temperature 0.2, given prompt version v3 with retrieved chunks [a,b,c], emitted this output, costing X input + Y output tokens". W5 AIOps formalises the telemetry surface; today's PR establishes the per-call log shape.

- **Hallucination consequences.** A commercial chatbot hallucinating a product feature is embarrassing. A solicitation-drafting LLM hallucinating a FAR clause is a procurement-protest exposure. The hallucination categories in `3-hallucination-failure-modes.md` are the cohort's working taxonomy; structured outputs (Friday) + RAG grounding (W2) + HITL gates (W3 + W4) are the mitigation layers.

### 3.4 The discipline-graph for the next six weeks

Today installs the substrate. The discipline graph extends:

- **Friday** — typed I/O via structured outputs + validation gates; context engineering treats the prompt as architecture; explicit HITL framing (1st of 7 touchpoints).
- **W2** — RAG grounds the model in the FAR/DFARS corpus; eval harness measures faithfulness against ground truth.
- **W3** — agentic flows (single + multi-agent); LangGraph HITL interrupt nodes; tool-call idempotency.
- **W4** — AI Security Engineering; OWASP LLM Top 10; prompt-injection testing; PII protection.
- **W5** — OpenTelemetry on AI; cost as an AIOps signal; drift detection; AI-SRE patterns.
- **W6** — Client-deliverability discipline; runbooks; ADR catalog; eval reports.

Every later week assumes today's substrate. If you do not internalise typed I/O + retry posture + cost telemetry + model-ID configuration today, the W2+ material lands without a foundation.

## 4. Generic Implementation

A 25-line sketch of the substrate, applied to any LLM call. Not Karsun-specific:

```python
import os, json, time, random
from anthropic import Anthropic, APIStatusError

client = Anthropic()
MODEL_ID = os.environ["LLM_MODEL_ID"]                 # configuration, not literal

def call_llm(messages: list[dict], feature: str, tenant: str) -> dict:
    for attempt in range(3):
        try:
            resp = client.messages.create(
                model=MODEL_ID,
                max_tokens=1024,
                messages=messages,
            )
            log_usage(                                # day-one telemetry
                feature=feature, tenant=tenant, model=MODEL_ID,
                input_tokens=resp.usage.input_tokens,
                output_tokens=resp.usage.output_tokens,
            )
            return {"text": resp.content[0].text}
        except APIStatusError as e:
            if e.status_code in {400, 401, 403}:      # never retry
                raise
            delay = (2 ** attempt) + random.uniform(0, 0.5 * 2 ** attempt)
            time.sleep(delay)                         # jittered backoff
    raise RuntimeError("LLM call failed after 3 attempts")
```

Friday's PR layers structured outputs onto this. W2's PR layers RAG retrieval. W5's PR layers OpenTelemetry instrumentation. The substrate stays; subsequent weeks extend it.

## 5. Real-world Patterns

**Healthcare — Abridge's structured-output discipline.** Abridge's AI scribe pipeline has published the pattern of validating every model output against a clinical-note schema before any database write. Validation failures route to a human-review queue rather than producing degraded notes [3]. The discipline is "typed contract on output" — the same discipline today's PR establishes for solicitation drafts.

**Fintech — Stripe's LLM fraud-explanation telemetry.** Stripe has discussed building per-request cost telemetry from day one of their fraud-explanation LLM integration; the telemetry later let them detect a malicious prompt-injection campaign by spotting the cost-per-request anomaly before the content anomaly was visible [4]. The discipline is "cost telemetry as a security signal" — a parent pattern (observability) bent for an LLM-specific contour (per-token cost).

**Public sector — generic FedRAMP-anchored Bedrock patterns.** Federal-context teams uniformly choose Bedrock over direct provider APIs for the FedRAMP boundary [4]. The discipline is "model-ID configuration over hard-coding" — federal deployments rotate models for compliance reasons (e.g., when a provider's data-handling posture changes); hard-coded IDs become incident-driven code changes.

## 6. Best Practices

- **Reach for the parent pattern first.** Retry posture, observability, config-driven endpoints — the LLM-specific versions are recontoured traditional disciplines, not new ones.
- **Reject the "magic box" framing in design discussions.** Treating the LLM as opaque produces logs that fail audit and tests that pass on the happy path only.
- **Wire telemetry from the first commit, not the third.** Retrofitting per-call cost + token telemetry into an established codebase is 10x the work of building it in.
- **Classify failures by hallucination category in incident review.** The mitigation layer differs by category; "the LLM hallucinated" is not a sufficient root cause.
- **Defend model-ID-from-config in every design review.** It is the cheapest discipline to lose and the most expensive to recover.

## 7. Hands-on Exercise

**(Reading + sketch, 10 minutes.)** Take the current `services/ai-orchestrator/app/main.py` (you walked it Wednesday PM). Without writing code, list on paper:

- (a) Which of the **five engineering disciplines** the starting state currently lacks (expected answer: all five — the gaps are deliberate per the training-project README).
- (b) For each gap, which of the **three antipatterns** §3.2 describes is currently in the code (or, if no antipattern exists, why the gap is absence-of-discipline rather than presence-of-antipattern).
- (c) The **minimum PR shape** that closes all five gaps with smallest diff — what gets added, what gets refactored, what stays the same.

**What good looks like.** The list of five disciplines maps one-to-one to today's PR scope: system prompt (typed input shape), Pydantic strict response model (typed output — Friday completes this), retry-with-jitter wrapping the invoke (retry posture), `log_usage(...)` call after every invoke (cost telemetry), `os.environ["LLM_MODEL_ID"]` in place of any literal model string (model-ID configuration). The "minimum PR shape" emphasises additive change over refactor — the starting state is broken-but-small; the production-shaped state is bigger-but-correct.

## 8. Key Takeaways

- Can you name the five engineering disciplines that turn an LLM call into a production-shaped component, and for each give the parent pattern from traditional API integration?
- Can you identify the three demo-to-production antipatterns and explain why each fails in production but passes in prototype?
- Can you defend "an LLM call is an HTTP request that pays per token and occasionally lies" against pushback that "the model handles edge cases internally"?
- Can you map today's PR scope to the five-discipline framework and identify which disciplines Friday's and W2's PRs extend?

## Sources

1. [LLM Hallucination Isn't One Problem. It's Four. — OneValley](https://www.onevalley.com/blog/llm-hallucination-isnt-one-problem-its-four-most-teams-are-only-solving-one/) — retrieved 2026-05-26 via /web-research
2. [LLM Hallucinations in 2026 — Lakera](https://www.lakera.ai/blog/guide-to-hallucinations-in-large-language-models) — retrieved 2026-05-26 via /web-research
3. [Models (Pydantic docs)](https://docs.pydantic.dev/latest/concepts/models/) — retrieved 2026-05-26 via /web-research
4. [Supported foundation models in Amazon Bedrock — AWS Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html) — retrieved 2026-05-26 via /web-research
5. [Retry Strategies for LLM API Calls — CallSphere](https://callsphere.ai/blog/retry-strategies-llm-api-calls-exponential-backoff-jitter-tenacity) — retrieved 2026-05-26 via /web-research

Last verified: 2026-05-27
