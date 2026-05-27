---
week: W01
day: Thu
topic_slug: model-selection-criteria
topic_title: "Model selection criteria — Bedrock Claude for the federal anchor"
parent_overview: W01/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.aws.amazon.com/bedrock/latest/userguide/claude-messages-supported-models.html
    retrieved_on: 2026-05-22
    recency_category: hot-tech
  - url: https://aws.amazon.com/blogs/publicsector/karsun-solutions-builds-modernization-platform-using-amazon-bedrock/
    retrieved_on: 2026-05-22
    recency_category: federal-regulatory
  - url: https://github.com/KarsunFDE/content/blob/main/research/bedrock-claude-catalog-20260522.md
    retrieved_on: 2026-05-22
    recency_category: hot-tech
last_verified: 2026-05-27
---

# Model selection criteria — Bedrock Claude for the federal anchor

> The overview names Bedrock+Claude as the programme anchor. This reading is the depth: why Bedrock specifically (FedRAMP, model breadth, IAM surface), why Claude specifically (Bedrock-hosted FedRAMP inheritance, model-family economics), which Claude model for which task (Opus / Sonnet / Haiku), and how to defend the choice on Friday's "Bedrock vs OpenAI direct" scenario-alternatives prompt.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Articulate the **three load-bearing reasons** Bedrock is the federal anchor and the parent considerations behind each (FedRAMP boundary, model breadth on one IAM surface, GovCloud availability).
- Match each **current Claude model** (Opus 4.5/4.6, Sonnet 4.5/4.6, Haiku 4.5) to the task class it fits best (high-stakes reasoning, default workhorse, cost-sensitive high-volume).
- Identify the **known-bad-pattern blocklist** items in this space — deprecated model IDs that tutorials still cite.
- Defend "Bedrock + Claude" against the most common Friday-scenario-alternatives challenges ("why not OpenAI direct", "why not Gemini", "why not self-hosted").

## 2. Introduction

Model selection in a federal-acquisitions context is a compliance decision before it is a capability decision. The capability ceiling of the top frontier models is comparable within a generation; the compliance posture differs substantially. Bedrock with Claude is the programme anchor because it is the path with the strongest FedRAMP posture and the broadest model breadth on a single IAM surface [1][4].

The /web-research-sourced research brief at `research/bedrock-claude-catalog-20260522.md` is the cohort's canonical reference for current model IDs, invoke signatures, prompt caching, and known-bad patterns [4]. This reading abstracts the load-bearing decisions; the brief has the implementation-level detail.

## 3. Core Concepts

### 3.1 The three load-bearing reasons for Bedrock

**FedRAMP boundary.** Bedrock has FedRAMP High in GovCloud regions [1]. Direct provider APIs (OpenAI, Anthropic direct, Gemini) largely do not. For federal acquisitions this is decisive — data residency, audit boundary, and ATO inheritance all flow from FedRAMP. A pilot that uses Anthropic direct in dev cannot easily promote to prod without a FedRAMP-gated re-architecture; a pilot that uses Bedrock from day one inherits the boundary by default.

**Model breadth on one IAM surface.** `bedrock-runtime` routes to Anthropic, Amazon Nova, Meta Llama, Mistral via one IAM-controlled endpoint [1]. No per-provider credential sprawl, no per-provider IAM-role design. When model selection later changes (a new generation arrives, a regional outage forces a fallback), the IAM surface does not change. This is the "model-ID configuration over hard-coding" discipline from §2 of `2-llms-as-engineering-systems.md` — applied at the platform level.

**GovCloud availability.** Bedrock is available in `us-gov-east-1` and `us-gov-west-1` GovCloud regions [1]. Direct provider APIs are not. For workloads that touch CUI (Controlled Unclassified Information) or higher, GovCloud is non-negotiable.

### 3.2 Current Claude model catalog on Bedrock

Per the canonical research brief [4], the current GA model IDs as of 2026-05-22 are:

| Model | Bedrock ID | Best for |
|-------|-----------|---------|
| Claude Opus 4.5 | `anthropic.claude-opus-4-5-20251101-v1:0` | High-stakes reasoning, multi-step planning, complex tool-use chains |
| Claude Opus 4.6 | `anthropic.claude-opus-4-6-v1` | Same as 4.5; newer training; 1M-token context (4.7 family) |
| Claude Sonnet 4.5 | `anthropic.claude-sonnet-4-5-20250929-v1:0` | **Default workhorse** — most production tasks; balance of capability and cost |
| Claude Sonnet 4.6 | `anthropic.claude-sonnet-4-6` | Same role as 4.5; later generation |
| Claude Haiku 4.5 | `anthropic.claude-haiku-4-5-20251001-v1:0` | Cost-sensitive high-volume tasks; classification; lightweight extraction |

**Default for the programme: Sonnet 4.5 or 4.6.** Today's `ai-orchestrator` PR pins one of these via `LLM_MODEL_ID`. Opus is reserved for the agentic-systems work in W3 where multi-step planning depth matters. Haiku appears in W2 for high-volume retrieval-time tasks (chunk classification, query rewriting).

### 3.3 Known-bad pattern blocklist

The /web-research-sourced known-bad-patterns blocklist flags [4]:

- `anthropic.claude-v2` — **deprecated**, was the Claude 2 era ID, retired from the catalog. Tutorials older than 12 months still cite it.
- `anthropic.claude-instant-v1` — **deprecated**, was the Claude Instant era ID. Replaced by Haiku 4.5 economically.
- `anthropic.claude-3-sonnet-20240229-v1:0` — **superseded**, still GA in some regions, but newer Sonnet generations are the default for new code [1].

Codex Adversarial Review (Light, starting today per D-034) will flag any of these IDs in PR diffs. The fix is to update to current generation IDs and verify region availability before merge.

### 3.4 The Friday scenario-alternatives challenges

Friday's scenario-alternatives prompt will pose: *"defend Bedrock+Claude against (a) OpenAI direct, (b) Gemini via Vertex AI, (c) self-hosted Llama"*. The defences:

- **vs OpenAI direct.** OpenAI direct does not have FedRAMP High in GovCloud as of Q1 2026 [1]. Capability comparable; compliance posture not comparable. For a federal-acquisitions deployment, the FedRAMP gap alone is decisive.

- **vs Gemini via Vertex AI.** Vertex AI has FedRAMP Moderate; some GCP regions have ATOs. The breadth-on-one-surface argument is weaker (Vertex routes to Google models; Bedrock routes to multiple providers). The Karsun-context tie-breaker is that ReDuX is Bedrock-anchored — alignment with the Karsun production posture is a programme constraint, not a free choice.

- **vs self-hosted Llama.** Self-hosting wins on data-residency for the most stringent contexts (no provider-hosted inference at all). It loses on operational burden (the team owns model serving, scaling, security patching). The cohort revisits this in W5 as a "when does self-host make sense" decision; today's PR does not.

## 4. Generic Implementation

The Bedrock-runtime client setup, config-driven model ID, no provider credentials in code:

```python
import os, boto3

bedrock = boto3.client(
    "bedrock-runtime",
    region_name=os.environ.get("AWS_REGION", "us-east-1"),
    # Credentials come from the IAM role attached to the runtime, not from env.
)

MODEL_ID = os.environ["LLM_MODEL_ID"]   # e.g. anthropic.claude-sonnet-4-5-20250929-v1:0
```

In production: the IAM role attached to the FastAPI service grants `bedrock:InvokeModel` and `bedrock:InvokeModelWithResponseStream` on a resource ARN that names the specific allowed model IDs — wildcard model access is a federal-context audit finding [4]. Today's PR uses the runtime-attached role; the resource ARN narrowing arrives W4 as part of AI Security Engineering.

## 5. Real-world Patterns

**Karsun ReDuX — Bedrock-anchored federal modernization.** The AWS Public Sector Blog documents Karsun ReDuX as a Bedrock-anchored mainframe-modernization platform [3]. The architecture choice is the same one this programme makes: Bedrock for FedRAMP-bounded model access, Claude for the specific model family, IAM-bounded model-ID access. The cohort's Pair Projects align to this architecture.

**Federal-pilot Bedrock-vs-direct decisions, 2024-2025.** Multiple federal AI pilots reported choosing Bedrock specifically to avoid the FedRAMP re-architecture problem when promoting from pilot to production [1][3]. The pattern: pilot starts on Bedrock, stays on Bedrock through ATO; pilot starts on direct provider, hits the FedRAMP gate, gets re-architected, loses 3-6 months.

**Commercial — cost-routing across Claude tiers.** Production commercial Claude deployments often route across Opus / Sonnet / Haiku at runtime per request — heavyweight reasoning to Opus, default tasks to Sonnet, high-volume classification to Haiku. The cost dispersion across tiers is roughly 5x; the routing logic pays for itself in week one [1]. The cohort sees this in W5 as an AIOps cost-management pattern.

## 6. Best Practices

- **Pin the specific Claude model ID via configuration, never via literal.** The same code should run against Sonnet 4.5 today and Sonnet 4.6 next quarter with a config change [4].
- **Use Sonnet as the default; reach for Opus only when the task demands its depth.** Capability and cost gradient is real; Sonnet covers most tasks at substantially lower cost [4].
- **Verify region availability before merging a model-ID change.** Not every model is GA in every region; GovCloud especially lags commercial regions [1].
- **Treat "self-hosted Llama" as a W5+ conversation, not a W1 one.** The operational burden is significant; the programme is structured to land Bedrock first and earn the right to revisit self-host with data.
- **Always cite the FedRAMP boundary as the load-bearing reason for Bedrock.** In federal-context design reviews, "we picked Bedrock for capabilities" is a weaker argument than "we picked Bedrock for the FedRAMP boundary" [1].

## 7. Hands-on Exercise

**(10 minutes, paper.)** For each of the following hypothetical Karsun engagements, choose Bedrock + Claude model and defend the selection in three sentences max:

- (a) A grants-management portal that receives 50k FOIA requests/month and needs each classified into one of 12 categories before routing.
- (b) An OIG audit-response assistant that drafts long-form responses to audit findings, citing FAR Subpart 33.2.
- (c) A WAWF post-award helpdesk that answers contracting-officer questions in real time with conversational latency requirements (< 2s first token).

**What good looks like.** (a) Haiku 4.5 — classification at scale, cost-dominated decision. (b) Opus 4.5 or 4.6 — high-stakes reasoning, FAR-cited drafting, accuracy-dominated decision. (c) Sonnet 4.5 or 4.6 with streaming enabled — balance of capability and latency. All three answers reference FedRAMP-via-Bedrock as the compliance floor.

## 8. Key Takeaways

- Can you articulate the three load-bearing reasons for Bedrock as the federal anchor and rank them by federal-context relevance?
- Can you match each current Claude model to its task class and defend "Sonnet as default workhorse" against pushback to default to Opus?
- Can you spot the three known-bad-pattern model IDs in a code review and explain why they fail (deprecation, supersession, GA-region drift)?
- Can you defend Bedrock+Claude against the three Friday scenario-alternatives (OpenAI direct, Gemini via Vertex, self-hosted Llama) with the FedRAMP boundary as the load-bearing argument?

## Sources

1. [Supported foundation models in Amazon Bedrock — AWS Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html) — retrieved 2026-05-26 via /web-research
2. [Supported Claude models on Bedrock — AWS](https://docs.aws.amazon.com/bedrock/latest/userguide/claude-messages-supported-models.html) — retrieved 2026-05-22 via WebFetch (fallback)
3. [AWS Public Sector Blog — Karsun ReDuX](https://aws.amazon.com/blogs/publicsector/karsun-solutions-builds-modernization-platform-using-amazon-bedrock/) — retrieved 2026-05-22 via WebFetch (fallback)
4. [Bedrock Claude catalog research brief (`fde-10-week/research/bedrock-claude-catalog-20260522.md`)](https://github.com/KarsunFDE/content/blob/main/research/bedrock-claude-catalog-20260522.md) — last_verified 2026-05-22

Last verified: 2026-05-27
