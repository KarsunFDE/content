---
template: research-brief
tech: AWS Bedrock Claude model catalog
version_pinned: "bedrock-2023-05-31 (anthropic_version); Claude Sonnet 4.5 / Opus 4.5 / Haiku 4.5 (current GA)"
last_verified: 2026-05-22
last_verified_via: WebSearch+WebFetch (fallback; web-research unavailable)
recency_window: hot-tech 3mo
sources_count: 4
target_weeks: [W01-Thu-Fri, W02]
candidates_deferred: []
known_bad_patterns_flagged:
  - id: bedrock-old-model-ids
    note: "Many older tutorials still cite anthropic.claude-v2 / anthropic.claude-instant. These are NOT current. Brief pins to Sonnet 4.5 / 4.6, Opus 4.5 / 4.6, Haiku 4.5."
known_bad_patterns_checked: true
author: research-subagent
---

# Tech research brief — AWS Bedrock Claude model catalog

Last verified: 2026-05-22 · Recency window applied: hot-tech 3mo · Pinned version: anthropic_version=bedrock-2023-05-31; Sonnet 4.5 / Opus 4.5 / Haiku 4.5

## 1. What it is (3–5 sentences, no jargon)

AWS Bedrock is Amazon's managed foundation-model API: a single `bedrock-runtime` endpoint that routes requests to Anthropic, Amazon Nova, Meta Llama, Mistral, and other model providers. For the Karsun FDE context, we use it as the federally-compliant path to Claude — Bedrock-hosted Claude models inherit AWS's FedRAMP/IL posture in a way the direct Anthropic API does not. The cohort touches Bedrock in W1 (first Claude call) and W2 (RAG retrieval-augmented generation with prompt caching to amortise long system prompts and retrieved-context blocks).

## 2. Current stable state (as of `last_verified`)

- **Current Claude model IDs on Bedrock (GA):**
  - Claude Opus 4.5 — `anthropic.claude-opus-4-5-20251101-v1:0`
  - Claude Opus 4.6 — `anthropic.claude-opus-4-6-v1`
  - Claude Sonnet 4.5 — `anthropic.claude-sonnet-4-5-20250929-v1:0`
  - Claude Sonnet 4.6 — `anthropic.claude-sonnet-4-6`
  - Claude Haiku 4.5 — `anthropic.claude-haiku-4-5-20251001-v1:0`
  - Older still GA: Claude Opus 4 / 4.1, Sonnet 4, 3.7 Sonnet, 3.5 Sonnet v2.
- **invoke_model signature (Claude):** body is JSON with `anthropic_version: "bedrock-2023-05-31"`, `messages: [...]`, `max_tokens`, optional `system`, `temperature`, `top_p`, `top_k`, `stop_sequences`. Call as `bedrock_runtime.invoke_model(modelId=..., body=json.dumps(body))`.
- **Prompt caching:** GA. In InvokeModel for Claude, attach `"cache_control": {"type": "ephemeral"}` to a content block at the cache boundary. Add `"ttl": "1h"` for 1-hour cache (Opus 4.5 / Sonnet 4.5 / Haiku 4.5 only); default is 5 minutes. Min checkpoint = 4,096 tokens for 4.5/4.6 models, 1,024 tokens for 3.7 Sonnet / Sonnet 4.6 / Opus 4. Max 4 cache checkpoints per request. Bedrock also offers Claude-specific "simplified cache management" — one breakpoint at end of static content, system auto-locates the longest prefix match (looks back ~20 content blocks).
- Breaking changes in last 3 months: 1-hour prompt-cache TTL went GA in commercial + GovCloud regions in Jan 2026 (AWS What's New, 2026-01). Opus 4.6 + Sonnet 4.6 added to the catalog (model IDs as above).
- Known-bad-pattern check: **flagged** — `anthropic.claude-v2` / `anthropic.claude-instant` are not the current IDs. Brief pins to 4.5/4.6 series.

## 3. What we teach (and what we deliberately don't)

- **In scope (W1–W2):** `bedrock-runtime` client via boto3; `invoke_model` body shape for Claude; selecting Sonnet 4.5 as the default cohort model (balance of cost/latency/capability); enabling prompt caching with `cache_control: ephemeral` + 1h TTL for long system prompts and W2 RAG retrieved-context blocks; IAM policy shape for `bedrock:InvokeModel` on specific model ARNs.
- **Out of scope:** Bedrock Agents (the managed-agent product) — out of scope for W1–W2; we build agents with LangChain `create_agent`. Bedrock Knowledge Bases — out of scope; we build RAG against MongoDB Atlas Vector Search directly. Converse API — mentioned as the higher-level alternative to InvokeModel but we teach InvokeModel for clarity on the wire format.
- **Misconceptions to pre-empt:** (a) "Claude on Bedrock is the same model ID as on Anthropic's direct API." — false; IDs differ. (b) "Prompt caching is automatic." — false for Claude (must mark `cache_control`); only Nova auto-caches. (c) "1-hour TTL is the default." — false; default is 5 min, opt in via `"ttl": "1h"`.

## 4. Recommended primary sources

- [Supported Claude models on Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/claude-messages-supported-models.html) — accessed 2026-05-22. Canonical model list.
- [Prompt caching for faster model inference](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html) — accessed 2026-05-22. Authoritative on `cache_control`, model IDs with TTL support, min token thresholds, simplified cache management.
- [Amazon Bedrock now supports 1-hour duration for prompt caching (AWS What's New)](https://aws.amazon.com/about-aws/whats-new/2026/01/amazon-bedrock-one-hour-duration-prompt-caching/) — published 2026-01, accessed 2026-05-22. Authoritative GA date for the 1h TTL.
- [Claude Code × Amazon Bedrock Backend Setup Guide 2026](https://claudelab.net/en/articles/claude-code/claude-code-bedrock-backend-setup-2026) — accessed 2026-05-22. Useful IAM ARN shape for the model wildcards.

## 5. Code reference snippets (idiomatic, current API)

```python
import boto3, json

br = boto3.client("bedrock-runtime", region_name="us-east-1")

body = {
    "anthropic_version": "bedrock-2023-05-31",
    "system": "You are a federal-modernization SME. Reply concisely.",
    "messages": [
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "What is FedRAMP High?"},
                {
                    "type": "text",
                    "text": LONG_REFERENCE_DOC,  # >= 4096 tokens for 4.5-series cache
                    "cache_control": {"type": "ephemeral", "ttl": "1h"}
                }
            ]
        }
    ],
    "max_tokens": 2048,
    "temperature": 0.2,
}

resp = br.invoke_model(
    modelId="anthropic.claude-sonnet-4-5-20250929-v1:0",
    body=json.dumps(body),
)
out = json.loads(resp["body"].read())
```

## 6. Risks and watch-items

- Bedrock model catalog moves quarterly. Re-verify model IDs on the Sunday before W1 and again before W2.
- Prompt-cache token minimums are model-specific (1,024 vs 4,096). If the cohort can't hit 4,096 tokens of static content, they won't get a cache hit on 4.5-series models — pick the example carefully.
- Regional availability is uneven. Opus 4.6 is not everywhere; W1 demos should target `us-east-1` for safety.

## 7. Alternatives the cohort should be aware of

- Direct Anthropic API — used in adversarial-comparison discussion (no FedRAMP inheritance, different model IDs, same prompt-caching semantics but different SDK).
- AWS Converse API — higher-level alternative to InvokeModel; preferred if you don't need wire-level control.

## 8. Brief sign-off

- Drafted by: research-subagent (2026-05-22)
- Reviewed against `known-bad-patterns.yml`: **flagged** (bedrock-old-model-ids); brief counter-teaches.
- 1-month-release check: not triggered (1h TTL GA was Jan 2026, model IDs are Oct–Nov 2025).
- Approved for downstream artifact authoring: 2026-05-22

## Sources

- AWS Bedrock Supported Claude models, docs.aws.amazon.com, retrieved 2026-05-22 via WebFetch. <https://docs.aws.amazon.com/bedrock/latest/userguide/claude-messages-supported-models.html>
- AWS Bedrock Prompt caching, docs.aws.amazon.com, retrieved 2026-05-22 via WebFetch. <https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html>
- AWS What's New: 1-hour prompt caching, aws.amazon.com, retrieved 2026-05-22 via WebSearch. <https://aws.amazon.com/about-aws/whats-new/2026/01/amazon-bedrock-one-hour-duration-prompt-caching/>
- Claude Code × Bedrock setup 2026, claudelab.net, retrieved 2026-05-22 via WebSearch. <https://claudelab.net/en/articles/claude-code/claude-code-bedrock-backend-setup-2026>
