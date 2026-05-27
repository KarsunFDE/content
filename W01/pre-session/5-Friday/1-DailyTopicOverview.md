---
week: W01
day: Fri
title: "Pre-session — Structured outputs + production API integration"
audience: All cohort members
time_on_task_minutes: 66
last_verified: 2026-05-26
---

# W1 Fri Pre-Session — Structured outputs + production API integration

> Read after Thu's first LLM PRs land and before Fri morning. ~66 min across 7 topics. Fri is dense (explicit HITL framing — 1st of 7 programme touchpoints — and the First PR Adversarial Review). This pre-session prepares the foundation.

## 1. Structured JSON output — Pydantic and beyond (12 min)

A production LLM doesn't return strings — it returns *typed values*. Friday's PRs replace the W1 Thu Bedrock raw response with Pydantic-validated structured output.

The pattern:

```python
from pydantic import BaseModel, Field

class SolicitationDraft(BaseModel):
    title: str = Field(..., min_length=10, max_length=200)
    description: str = Field(..., min_length=50, max_length=5000)
    far_clauses_referenced: list[str] = Field(default_factory=list)
    estimated_value_usd: float | None = None
    cost_tracking: dict[str, int] = Field(..., description="input_tokens, output_tokens")

    class Config:
        extra = "forbid"   # strict mode: fail on unexpected fields
```

When you invoke Bedrock with Claude, ask for JSON in the system prompt + parse with Pydantic. If Pydantic raises, **don't return the raw output** — return a structured error, log the failure for the eval harness (W2 Fri), and either retry with a corrected prompt or escalate to HITL.

**Anti-pattern to avoid:** `try/except` around the parse with a bare-string fallback. This hides failures and makes regression invisible. Fail loud; the eval harness catches the trend.

## 2. Output validation gates — Pydantic + Bean Validation contract (10 min)

Friday's PR adds output validation at two layers:

- **Python side (`ai-orchestrator`):** Pydantic strict-mode validates the Bedrock response *before* it leaves the AI service.
- **Spring side (`solicitation-service`):** Bean Validation re-validates the response *as it arrives from the AI service*. Defence in depth — if the AI service is wrong about its contract, Spring catches it.

The two validations must agree on the schema. Codex Adversarial Review (Light strictness Fri per D-034) will flag drift between the two definitions.

## 3. Context engineering — system prompts, dynamic assembly, compression (15 min)

The prompt is an *architectural artifact*, not a string literal. Treat it as code: version-controlled, tested, ADR-worthy. This topic has three beats — the prompt's static shape, its run-time assembly, and how you keep it inside the context window.

### 3a. System prompt as architecture

Named sections, not free-form prose:

```
<persona>You are a federal contracting officer's drafting assistant.</persona>
<constraints>
- Output FAR/DFARS-compliant language only.
- Cite only sources explicitly grounded in the retrieved chunks below.
- If you can't ground a claim, say so explicitly; never invent a citation.
</constraints>
<output_format>
Return JSON matching this schema: {...}.
</output_format>
<grounding>
Retrieved chunks: {...}    ← empty W1; populated from RAG in W2
</grounding>
```

Each section has an owner, a change history, and an eval set. Drift between section ownership is where prompts rot.

### 3b. Dynamic context assembly

The W2 RAG layer will inject retrieved chunks into the `<grounding>` section at request time. Friday's PR prepares for this by:

- Separating the **static system prompt** (persona + constraints + format) from the **dynamic context** (currently empty placeholder; populated by RAG W2).
- Logging the *resolved* prompt at debug level so the cohort can replay it.

The split matters because static vs dynamic sections have different test strategies — static gets snapshot tests, dynamic gets retrieval-quality evals.

### 3c. Context compression patterns

When grounding context is large (full FAR clauses), the model hits its context window. Patterns Fri previews (deep work W2):

- **Summary buffer** — summarise the conversation history; keep last N turns verbatim.
- **Sliding window** — drop oldest turns when context window approaches limit.
- **Hierarchical compression** — summarise summaries.

W1 Fri doesn't implement these; it leaves room for them. The cohort returns here W2 Tue when RAG retrieval starts producing chunk sets that exceed a single-pass window.

## 4. Prompt evaluations + lifecycle management (8 min)

Prompts change over time. The cohort needs a discipline for managing prompt versions:

- **Prompt registry** — every prompt lives at a versioned path (`prompts/draft-solicitation/v2.md`).
- **Eval harness** — every prompt change runs against a held-out QA set (W2 Fri builds this).
- **A/B comparison** — when changing a prompt, run new + old against the same eval set; compare faithfulness, latency, cost.

Production tracing arrives W5 (LangSmith deep-dive per D-031). Until then, the eval harness lives in flat files in the repo.

## 5. Explicit HITL framing — Friday's named topic (8 min)

Friday surfaces Human-in-the-Loop (HITL) as an explicit programme thread for the first time (1st of 7 touchpoints per D-043+D-044).

The question: **when does an LLM output need a human gate before action?**

The federal-acquisitions answer:
- **Reversibility** — if the action is undoable (search, lookup, draft a paragraph), no gate needed.
- **Blast radius** — if the action affects external parties (award a contract, send a letter, publish a notice), gate it.
- **Audit demands** — if the federal context demands a human-decision audit row, gate it.

Friday's exercise: each pair commits **one HITL decision in their Phase 1 plan-spec by EOD**. Which solicitation-drafter output requires human approval before it leaves the platform? Why?

This sets up:
- W2 Thu's HITL-as-RAG-fallback pattern.
- W3 Mon Plan Day's HITL interrupt-node ADR.
- W3 Wed's HITL between multi-agent handoffs.
- W3 Thu's full LangGraph HITL deep-dive (technical anchor).
- W4 Wed's HITL re-asserted under OWASP LLM06 (Excessive Agency).
- W5 Wed's HITL authority boundaries for AI-SRE auto-remediation.

## 6. Async FastAPI integration with Spring Boot (8 min)

The `ai-orchestrator` currently has a synchronous endpoint. Streaming + retries push it toward async patterns:

- `async def` endpoints with `httpx` for outbound calls.
- Background tasks via `BackgroundTasks` for fire-and-forget audit log writes.
- Lifecycle management — `lifespan` context manager for Bedrock client + Mongo client init/cleanup.

Spring side integrates via `WebClient` calling the async FastAPI endpoint; the `solicitation-service` blocks on the response (deliberate W1 simplification — async-all-the-way arrives W3).

## 7. First PR Adversarial Review (Light) — what to expect Fri (5 min)

Friday afternoon: each pair's Day-1 plan-spec PR gets the first Codex Adversarial Review of the cohort.

At Light strictness:
- Codex flags missing ADR rationale, unclear success criteria, undefined HITL boundary.
- Findings appear as PR comments, not blocking gates.
- Pair responds with either a fix or an explicit deferral with reasoning.
- W2 Mon Plan Day opens with reviewing Friday's findings as part of the §0 retro pattern (which formally starts W3 Mon).

Don't be defensive. Treat the findings as a second-FDE pair-of-eyes. The calibration ramp (W1 Light → W4 Full per D-034) means findings will get more pointed; W1 is the warm-up.

## What you'll do W1 Fri

- Morning war-room: cost + reliability for LLM endpoints.
- Afternoon: structured output + validation + context engineering + HITL + First PR Adversarial Review + Pair Project Phase 1 Day-1 commit.
- EOD: Phase 1 plan-spec with HITL decision committed; Light MCQ submitted; first Live Defense **deferred** to Fri W2 (no scenarios with ADRs yet on Fri W1).

Last verified: 2026-05-26
