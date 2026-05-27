---
week: W01
day: Thu
title: "Pre-session — LLM Engineering Essentials (LLMs as engineering systems + Bedrock invocation + retry + cost)"
audience: All cohort members
time_on_task_minutes: 68
last_verified: 2026-05-27
cohort1_compression: "Thursday is Day-3 of the 4-day W1. Morning war-room (D4) is the first Incident-mode scenario — a contracting officer asks the team to draft a solicitation against ai-orchestrator. PM practical lands the production-quality LLM PR. This pre-session installs the engineering substrate."
---

# W1 Thu Pre-Session — LLM Engineering Essentials

> Read **Wednesday evening / before Thursday 08:30**. ~68 min across 7 topics. Thursday is where the cohort treats LLM calls as production-system components — same engineering discipline as any other API integration, with LLM-specific contours. The PM PR lands `services/ai-orchestrator/`'s first production-shaped Bedrock invocation: system prompt + model selection + streaming + retry + cost tracking. Friday's pre-session (in `pre-session/5-Friday/`) then layers structured-output validation, context engineering, prompt evaluations, and the 1st of 7 HITL touchpoints on top of this substrate.

## 1. Thu at a glance — LLM Engineering Essentials (5 min)

Thursday has three blocks:

| Time | Block | Output |
|------|-------|--------|
| 09:00–12:00 | War-room D4 (Incident mode) — contracting-officer email demands a solicitation draft from `ai-orchestrator`'s current (broken) implementation | Pair triages the broken Bedrock call; documents the seven gaps to close in the PM PR |
| 13:00–16:00 | Practical — Pair Project + `acquire-gov` `ai-orchestrator` both land the production-quality LLM integration PR | PR opened; Codex Adversarial Review (Light) per D-034 starts running on every Thursday PR |
| 16:00–17:00 | Conceptual closeout — daily deliverable submission + Day-3 retro + Wed-evening read for Friday | Day-3 deliverable closed; Fri pre-session begins |

**The Thursday discipline:** every LLM call is a production-system component. That means: a typed contract for input + output, retry policy that distinguishes retryable from non-retryable errors, per-call cost telemetry tagged with feature + tenant from day one, streaming for any user-facing endpoint, and explicit model-ID configuration (never hard-coded). Tuesday + Wednesday were onboarding + framing; Thursday is when production discipline starts.

**Codex Adversarial Review (Light) starts Thursday on real PRs.** Per D-034, calibration is Light for W1 — Codex flags findings; most are framed as coaching, not blocking. P0 findings still block; P1 architectural-drift items appear as conversation starters rather than gates. Codex will appear in your PR threads from today onward. Don't be alarmed. Read the findings; treat them as a second pair of eyes.

## 2. LLMs as engineering systems, not magic (8 min)

The transition from "LLM as a clever demo" to "LLM as a production system component" is the substrate Thursday installs. The pattern is unforgiving in the specifics: an LLM call is an HTTP request that pays per token, returns text that may be ill-formed, occasionally fails for transient reasons, occasionally fails for permanent reasons that look transient, and routinely produces output that is grammatically perfect and factually wrong.

The field has converged in 2025-2026 on a small set of engineering disciplines that turn LLM calls into production-shaped components [1][2][3][4][5]: structured-output validation at the call site (Friday's depth), defensive retry with jitter and rate-limit awareness, per-request token + cost telemetry, explicit hallucination-category classification for failure tracking, and a discipline of model-ID configuration over hard-coding. None of these are LLM-specific in concept — every one has a parent pattern in traditional API integration — but each acquires LLM-specific contours.

Deep dive: [`2-llms-as-engineering-systems.md`](2-llms-as-engineering-systems.md) (~10 min).

## 3. Hallucination failure modes (10 min)

Five distinct failure modes, each with its own mitigation [1][2]:

- **Confabulation** — model invents a fact (fabricated FAR citation, non-existent function, wrong API parameter). Mitigation: retrieval-grounding (RAG, W2) or constrained tool calls.
- **Stale knowledge** — training data older than the current state (regulation amended after cut-off, deprecated library version). Mitigation: explicit cut-off awareness, retrieval-grounding for time-sensitive facts.
- **Over-confidence in plausible outputs** — confidently phrased answer contradicts source material. Mitigation: faithfulness evaluation (W2 Fri), structured citations back to source.
- **Format drift** — returns prose when asked for JSON, or JSON with wrong shape. Mitigation: structured-output APIs (Friday's depth), schema validation at call site.
- **Truncation** — hits context window or output cap mid-response. Mitigation: explicit output-length budgets, streaming-aware consumers, defensive parsing.

The Stanford AI Index 2026 reports hallucination rates across 26 evaluated LLMs in 2025-2026 ranging from 22% to 94% depending on task and evaluation methodology [2]. **The variance is the lesson:** there is no single "hallucination rate" you can plan against — task-specific evaluation, per W2, is the only honest way to characterise risk.

Deep dive: [`3-hallucination-failure-modes.md`](3-hallucination-failure-modes.md) (~10 min).

## 4. Model selection criteria — Bedrock + Claude for the federal anchor (10 min)

Programme anchor is AWS Bedrock with Claude. The defence-of-choice (you'll be asked Friday in the "Bedrock vs OpenAI direct" scenario-alternatives prompt):

- **FedRAMP posture.** Bedrock has FedRAMP High in GovCloud regions; direct provider APIs largely do not [6]. For federal acquisitions this is decisive — data residency, audit boundary, and ATO inheritance all flow from FedRAMP.
- **Model breadth on one endpoint.** `bedrock-runtime` routes to Anthropic, Amazon Nova, Meta Llama, Mistral via one IAM-controlled surface — no per-provider credential sprawl [6].
- **Current GA model IDs** (as of 2026-05-22 per /web-research [6]):
  - Opus 4.5 — `anthropic.claude-opus-4-5-20251101-v1:0`
  - Opus 4.6 — `anthropic.claude-opus-4-6-v1`
  - Sonnet 4.5 — `anthropic.claude-sonnet-4-5-20250929-v1:0`
  - Sonnet 4.6 — `anthropic.claude-sonnet-4-6`
  - Haiku 4.5 — `anthropic.claude-haiku-4-5-20251001-v1:0`
- **Known-bad-pattern blocklist flag:** any code using `anthropic.claude-v2` / `anthropic.claude-instant` IDs is stale [6]. Substitute current generation and verify region availability before merge.

Read `training-project/docs/adrs/0002-bedrock-as-llm-anchor.md` for the full ADR rationale.

Deep dive: [`4-model-selection-criteria.md`](4-model-selection-criteria.md) (~10 min).

## 5. AWS Bedrock model invocation (10 min)

Today's PR moves `services/ai-orchestrator/app/main.py`'s `POST /draft-solicitation` from broken-on-arrival to production-shaped. Starting state has **deliberate gaps**: no system prompt, no streaming, no retry, no cost tracking, no structured-output schema.

The Bedrock InvokeModel signature for Claude [6]:

```python
body = {
    "anthropic_version": "bedrock-2023-05-31",
    "messages": [{"role": "user", "content": "..."}],
    "max_tokens": 1024,
    "system": "<persona + constraints + format>",
    "temperature": 0.2,
}
resp = bedrock_runtime.invoke_model(
    modelId=os.environ["LLM_MODEL_ID"],   # never hard-coded
    body=json.dumps(body),
)
```

Today's PR adds: system-prompt architecture, model-ID from environment, basic invoke wired through FastAPI. Streaming + retry + cost are §6 + §7 + §8 below.

Deep dive: [`5-aws-bedrock-model-invocation.md`](5-aws-bedrock-model-invocation.md) (~10 min).

## 6. Streaming responses (8 min)

A 30-second blocking POST to Bedrock is a 30-second hung Angular UI. Today's PR converts the endpoint to streaming via `InvokeModelWithResponseStream` on the Bedrock side and Server-Sent Events (SSE) through FastAPI to the Angular client.

The pattern: each Bedrock chunk arrives as a separate `chunk` event with `bytes` payload; FastAPI `StreamingResponse` forwards them as SSE `data:` frames; Angular consumes via `EventSource`. Critical detail: **streaming and structured outputs do not compose cleanly** — Friday's structured-output PR has to choose between streaming-with-partial-JSON parsing (complex) or non-streaming for the structured path (simple). W1 stays simple; W3 revisits.

Deep dive: [`6-streaming-responses.md`](6-streaming-responses.md) (~8 min).

## 7. Retry logic and strategies (8 min)

Bedrock APIs fail 1-5% of the time at provider level [5], plus higher transient failure rates during regional incidents. Production retry posture:

- **Retry on:** 429 (rate-limited), 500/502/503/504 (server transient), connection timeouts.
- **Never retry on:** 400 (malformed request), 401/403 (auth), other 4xx (content-filter rejections, invalid model IDs).
- **Honor `Retry-After`** as a floor.
- **Exponential backoff with jitter** — schedule 1s, 2s, 4s, 8s with ±25% jitter to prevent thundering-herd retries [5].
- **Bounded attempt count** — 3-5 attempts user-facing, 5-7 background; beyond that, circuit-break.

Deep dive: [`7-retry-logic-and-strategies.md`](7-retry-logic-and-strategies.md) (~8 min).

## 8. Cost structure of LLMs (8 min)

LLM calls cost money per token, in both directions (input + output, often at different rates). A production integration tracks **per-request input tokens, output tokens, model ID, and request feature tag** from day one. Today's PR wires this telemetry from the first commit.

Bedrock prompt caching [6] is the Claude-specific cost lever the cohort touches W2: `cache_control: {type: "ephemeral", ttl: "1h"}` on a content block at the cache boundary. Min checkpoint 4,096 tokens for 4.5/4.6 models. Max 4 checkpoints. Today's PR does not use caching; W2 Fri does.

Deep dive: [`8-cost-structure-of-llms.md`](8-cost-structure-of-llms.md) (~8 min).

## What you'll do W1 Thu

Morning war-room (`war-room/D4.md`) — first Incident-mode scenario in `acquire-gov`'s solicitation flow. Afternoon practical — Pair Project + `acquire-gov` both land production-quality LLM integration into the Spring Boot endpoint that calls `ai-orchestrator`. Conceptual closeout — Day-3 deliverable + Codex review starts running on your PR.

> [!instructor-review]
> Wed-evening install / verify checklist before Thursday morning:
> - `aws bedrock list-foundation-models --region us-east-1` returns Claude model IDs without errors.
> - `python -c "import boto3; print(boto3.client('bedrock-runtime', region_name='us-east-1'))"` returns a client without errors.
> - `http://localhost:8000/health` returns the (lying) `ok` response after `docker-compose up`.
> - If any of these fail, flag at Thursday's 09:00 standup before the practical block starts.

> [!instructor-review]
> Bedrock model IDs change frequently. The exact IDs in today's `ai-orchestrator` PR must be verified against the **current** Bedrock console at PR time, not pasted from this reading. Known-bad-pattern blocklist flags any code using the deprecated `anthropic.claude-v2` / `anthropic.claude-instant` IDs — substitute the current generation and verify region availability before merge.

## Sources

1. [LLM Hallucination Isn't One Problem. It's Four. — OneValley](https://www.onevalley.com/blog/llm-hallucination-isnt-one-problem-its-four-most-teams-are-only-solving-one/) — retrieved 2026-05-26 via /web-research, recency hot-tech.
2. [LLM Hallucinations in 2026: How to Understand and Tackle AI's Most Persistent Quirk — Lakera](https://www.lakera.ai/blog/guide-to-hallucinations-in-large-language-models) — retrieved 2026-05-26 via /web-research, recency hot-tech.
3. [Models (Pydantic docs)](https://docs.pydantic.dev/latest/concepts/models/) — retrieved 2026-05-26 via /web-research, recency foundation-stable.
4. [Supported foundation models in Amazon Bedrock — AWS Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html) — retrieved 2026-05-26 via /web-research, recency hot-tech.
5. [Retry Strategies for LLM API Calls: Exponential Backoff with Jitter — CallSphere](https://callsphere.ai/blog/retry-strategies-llm-api-calls-exponential-backoff-jitter-tenacity) — retrieved 2026-05-26 via /web-research, recency hot-tech.
6. [AWS Bedrock Claude model catalog research brief (`fde-10-week/research/bedrock-claude-catalog-20260522.md`)](https://github.com/KarsunFDE/content/blob/main/research/bedrock-claude-catalog-20260522.md) — last_verified 2026-05-22 via WebSearch+WebFetch (research-protocol-permitted fallback when /web-research unavailable that day); covers model IDs, invoke_model signature, prompt caching, known-bad-pattern flags.

Recency categories: **hot-tech (3-month window)** — Bedrock Claude model availability, current model IDs, prompt caching specifics, hallucination industry framing. **Foundation-stable (12-month window)** — Pydantic, HTTP retry classification, jitter math.

Last verified: 2026-05-27
