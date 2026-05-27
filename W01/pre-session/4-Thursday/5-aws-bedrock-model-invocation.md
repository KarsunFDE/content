---
week: W01
day: Thu
topic_slug: aws-bedrock-model-invocation
topic_title: "AWS Bedrock model invocation — InvokeModel signature, system prompt, configuration"
parent_overview: W01/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModel.html
    retrieved_on: 2026-05-22
    recency_category: hot-tech
  - url: https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html
    retrieved_on: 2026-05-22
    recency_category: hot-tech
  - url: https://github.com/KarsunFDE/content/blob/main/research/bedrock-claude-catalog-20260522.md
    retrieved_on: 2026-05-22
    recency_category: hot-tech
last_verified: 2026-05-27
---

# AWS Bedrock model invocation — InvokeModel signature, system prompt, configuration

> The overview names today's PR scope. This reading is the implementation depth: the exact Bedrock `InvokeModel` signature for Claude, how the system prompt is structured as an engineering artifact (not a string literal), the configuration discipline that makes model rotation a config change rather than a code change, and where `services/ai-orchestrator/app/main.py` currently breaks.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Write a correctly-shaped Bedrock `InvokeModel` call for Claude — `anthropic_version`, `messages`, `max_tokens`, `system`, `temperature` — from memory.
- Structure a **system prompt as a named-section artifact** rather than a free-form prose string.
- Configure model ID, region, and inference parameters from environment (or config file) — never from code literals.
- Identify the deliberate gaps in `services/ai-orchestrator/app/main.py`'s starting state and the minimum diff to close each one.

## 2. Introduction

The Bedrock `InvokeModel` API for Claude is small. Five fields handle 90% of production cases. The discipline is not in the API surface — it is in what you wrap around the call: a system prompt structured as engineering content, configuration-driven model ID, telemetry from day one. Today's PR establishes all three around the smallest correct invocation.

The /web-research-sourced research brief at `research/bedrock-claude-catalog-20260522.md` is the canonical reference for the invocation signature, prompt-caching specifics, and known-bad-pattern flags [4]. This reading focuses on the application layer that calls into that signature.

## 3. Core Concepts

### 3.1 The Bedrock InvokeModel signature for Claude

Per the canonical research brief [4] and the AWS API reference [2], the Claude InvokeModel body is JSON with these fields:

```python
body = {
    "anthropic_version": "bedrock-2023-05-31",   # required; pinned by Bedrock
    "messages": [
        {"role": "user", "content": "..."},
        # alternating user/assistant; multi-turn supported
    ],
    "max_tokens": 1024,                          # output cap; required
    "system": "...",                             # optional; system prompt
    "temperature": 0.2,                          # optional; 0.0 deterministic, 1.0 high variance
    "top_p": 0.9,                                # optional; nucleus sampling
    "top_k": 40,                                 # optional; vocabulary cap
    "stop_sequences": ["</response>"],           # optional; early termination
}
```

The call:

```python
resp = bedrock_runtime.invoke_model(
    modelId=os.environ["LLM_MODEL_ID"],
    body=json.dumps(body),
)
output = json.loads(resp["body"].read())
text = output["content"][0]["text"]
input_tokens = output["usage"]["input_tokens"]
output_tokens = output["usage"]["output_tokens"]
```

Today's PR uses exactly this shape. Streaming (`InvokeModelWithResponseStream`) is `6-streaming-responses.md`. Structured outputs (tool use) arrive Friday.

### 3.2 The system prompt as engineering artifact

The system prompt is not a string literal — it is a versioned, named-section, ADR-worthy artifact. The shape for today's PR:

```python
SYSTEM_PROMPT = """
<persona>
You are a federal contracting officer's drafting assistant.
</persona>
<constraints>
- Output FAR/DFARS-compliant language only.
- Never invent a FAR citation. If you cannot ground a citation, say so explicitly.
- Output between 200 and 800 words per draft section.
</constraints>
<output_format>
Plain text for W1. Friday's PR converts to structured JSON via tool use.
</output_format>
<grounding>
No retrieved chunks for W1. W2's RAG layer populates this section.
</grounding>
"""
```

**Why named sections.** Three reasons. (a) Each section has a clear owner — persona is product, constraints are compliance, output format is engineering, grounding is data engineering. (b) Each section has an evaluation strategy — persona gets snapshot tests, constraints get rule-based checks, grounding gets retrieval-quality evals. (c) Drift between section ownership is where prompts rot; the named-section shape makes drift visible.

**Anti-pattern this displaces.** A free-form prose prompt where persona, constraints, format, and grounding all live in one paragraph. Edit one and the others bend; eval one and the others go untested.

### 3.3 Configuration-driven model ID

The discipline:

- `LLM_MODEL_ID` env var holds the model ID — e.g. `anthropic.claude-sonnet-4-5-20250929-v1:0` [4].
- `AWS_REGION` env var holds the region — e.g. `us-east-1` for commercial, `us-gov-east-1` for GovCloud.
- `LLM_MAX_TOKENS` env var holds the output cap default.
- `LLM_TEMPERATURE` env var holds the default temperature.

No code change should be required to:
- Roll forward from Sonnet 4.5 to Sonnet 4.6 when 4.6 is approved.
- Switch between commercial and GovCloud regions for a deployment.
- Tune the output cap or temperature in response to evaluation results.

Codex Adversarial Review (Light, starting today) will flag any literal model ID in code [4].

### 3.4 The deliberate gaps in the starting state

The starting state of `services/ai-orchestrator/app/main.py` has a `POST /draft-solicitation` endpoint with **deliberate gaps** per the training-project README's brownfield-debt inventory:

| Gap | Today's PR closes via |
|-----|----------------------|
| No system prompt | §3.2 named-section system prompt |
| No model-ID configuration | §3.3 env var + `os.environ["LLM_MODEL_ID"]` |
| No streaming | `6-streaming-responses.md` — `InvokeModelWithResponseStream` |
| No retry | `7-retry-logic-and-strategies.md` — retry-with-jitter |
| No cost tracking | `8-cost-structure-of-llms.md` — `log_usage(...)` per call |
| No structured-output schema | **Friday's PR** — tool use + Pydantic strict mode |
| No `finish_reason` logging | `3-hallucination-failure-modes.md` truncation mitigation — add to log shape |

The structured-output gap is deliberately deferred to Friday — today's PR is "make the call work production-shaped"; Friday's PR is "constrain the output shape". The split keeps PR sizes reviewable and matches the PDF's day-shape (Thu = invocation discipline; Fri = output discipline).

### 3.5 Prompt caching — what to know walking in

Bedrock supports Claude prompt caching for stable prefixes [3][4]:

- Attach `"cache_control": {"type": "ephemeral"}` to a content block at the cache boundary.
- Add `"ttl": "1h"` for 1-hour cache (Opus 4.5 / Sonnet 4.5 / Haiku 4.5 only); default is 5 minutes.
- Min checkpoint = 4,096 tokens for 4.5/4.6 models, 1,024 tokens for 3.7 Sonnet / Sonnet 4.6 / Opus 4 [4].
- Max 4 cache checkpoints per request.
- Bedrock also offers Claude-specific "simplified cache management" — one breakpoint at end of static content, system auto-locates the longest prefix match.

**Today's PR does not use caching.** The retrieved-context surface that benefits from caching arrives in W2 with RAG. The note here is awareness; W2 Fri does the implementation.

## 4. Generic Implementation

A minimum production-shaped Bedrock InvokeModel — system prompt as artifact, config-driven model ID, usage logging. Streaming, retry, structured output deferred to their own topic files:

```python
import os, json, boto3
from app.logging import log_usage

bedrock = boto3.client("bedrock-runtime", region_name=os.environ["AWS_REGION"])
MODEL_ID = os.environ["LLM_MODEL_ID"]

SYSTEM_PROMPT = open("prompts/draft-solicitation/v1.md").read()

def draft_solicitation(user_request: str, feature: str, tenant: str) -> dict:
    body = {
        "anthropic_version": "bedrock-2023-05-31",
        "system": SYSTEM_PROMPT,
        "messages": [{"role": "user", "content": user_request}],
        "max_tokens": int(os.environ.get("LLM_MAX_TOKENS", 1024)),
        "temperature": float(os.environ.get("LLM_TEMPERATURE", 0.2)),
    }
    resp = bedrock.invoke_model(modelId=MODEL_ID, body=json.dumps(body))
    out = json.loads(resp["body"].read())
    log_usage(
        feature=feature, tenant=tenant, model=MODEL_ID,
        input_tokens=out["usage"]["input_tokens"],
        output_tokens=out["usage"]["output_tokens"],
        finish_reason=out.get("stop_reason"),
    )
    return {"text": out["content"][0]["text"], "stop_reason": out.get("stop_reason")}
```

~25 lines including the prompt file load. Today's PR adds retry-with-jitter around the `invoke_model` line (see `7-retry-logic-and-strategies.md`) and streaming behind a feature flag (see `6-streaming-responses.md`).

## 5. Real-world Patterns

**Karsun ReDuX Bedrock invocation shape.** Public-source documentation on Karsun ReDuX [3, via canonical brief] indicates the Bedrock invocation shape closely tracks the AWS-documented pattern: config-driven model selection, IAM-role-bounded model ARN, telemetry per call. Today's PR pattern aligns to this.

**Multi-tenant per-call tagging.** Production deployments uniformly tag every Bedrock invocation with tenant ID, feature ID, and request ID for downstream cost attribution and audit [4]. The tag set is established at the first commit; retrofitting it later is observed-in-the-field to be 5-10x the work.

**Prompt files in version control, not in code.** Mature LLM-integration codebases keep prompts in dedicated files (`prompts/<feature>/<version>.md`) rather than as Python strings. This enables: independent prompt versioning, prompt diffing in PR review, prompt-specific evaluation against a held-out set, A/B comparison when changing prompts [4]. Today's PR establishes this convention.

## 6. Best Practices

- **Pin `anthropic_version: "bedrock-2023-05-31"` — it is the only supported value.** Newer values may appear; the brief is the canonical source for the current pinned value [4].
- **Use `temperature: 0.2` as default for production drafting tasks.** Lower temperature, more deterministic output, easier evaluation. Reach higher only when variance is the desired property.
- **`max_tokens` should be sized to the longest expected output plus headroom.** Under-sizing causes silent truncation; over-sizing costs nothing per request but enables expensive runaway agentic outputs in W3+.
- **Load prompts from files, not Python literals.** A prompt diff in `git log` is more legible and reviewable than a multi-line string change in source.
- **`finish_reason` (also called `stop_reason`) belongs in every log line.** It is the only signal that distinguishes natural completion from `max_tokens` truncation from content-filter rejection.

## 7. Hands-on Exercise

**(10 minutes, paper.)** Walk through the starting `services/ai-orchestrator/app/main.py` (from Wednesday PM's walkthrough). For each of the seven gaps in §3.4, sketch on paper:

- (a) The specific line(s) of code that close the gap.
- (b) Which other gaps the closure depends on (e.g., cost telemetry depends on the invoke call returning usage info).
- (c) Which gaps are explicitly deferred to Friday or later weeks.

**What good looks like.** All seven gaps mapped one-to-one to PR additions. Dependency graph identifies that streaming + retry + cost telemetry all wrap the invoke call (order matters: retry inside-streaming or streaming-inside-retry has different semantics). Friday's structured-output gap is explicitly out of scope for today. The PR scope ends at "the call works production-shaped"; the PR does not attempt structured outputs.

## 8. Key Takeaways

- Can you write a correctly-shaped Claude Bedrock `InvokeModel` body from memory, with all required fields and the `anthropic_version` pinned to `bedrock-2023-05-31`?
- Can you defend a system prompt structured as named sections against pushback that "it's just a string"?
- Can you identify all literal model IDs in a code review and explain why each is a configuration-discipline violation?
- Can you walk a teammate through the seven deliberate gaps in the starting `ai-orchestrator` state and articulate which today's PR closes vs which Friday closes?

## Sources

1. [Supported foundation models in Amazon Bedrock — AWS](https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html) — retrieved 2026-05-26 via /web-research
2. [InvokeModel — Amazon Bedrock Runtime API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModel.html) — retrieved 2026-05-22 via WebFetch (fallback)
3. [Prompt caching — Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html) — retrieved 2026-05-22 via WebFetch (fallback)
4. [Bedrock Claude catalog research brief (`fde-10-week/research/bedrock-claude-catalog-20260522.md`)](https://github.com/KarsunFDE/content/blob/main/research/bedrock-claude-catalog-20260522.md) — last_verified 2026-05-22

Last verified: 2026-05-27
