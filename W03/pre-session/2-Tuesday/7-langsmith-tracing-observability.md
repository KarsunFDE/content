---
week: W03
day: Tue
topic_slug: langsmith-tracing-observability
topic_title: "LangSmith tracing — observability for tool-using agents"
parent_overview: W03/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://docs.langchain.com/langsmith/observability-quickstart
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.langchain.com/langsmith/observability
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://blog.langchain.com/debugging-deep-agents-with-langsmith/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/langsmith/evaluation
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.ai.cc/blogs/how-to-use-langsmith-2026-complete-guide/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-06-06
---

# LangSmith tracing — observability for tool-using agents

> [!NOTE]
> **From earlier:** Topics 2–6 built the loop, the tool schemas, the idempotency contracts, the compaction policy, and the RAG patterns. None of that is debuggable at scale without a trace. This topic wires the observability layer that makes the rest inspectable.

## 1. Learning Objectives

- Describe what a LangSmith trace contains and how the run-tree structure maps to a multi-agent topology
- Configure tracing with the three required env vars and identify when the fourth (regional endpoint) is required
- Choose between env-var auto-tracing, `@traceable` decorator, and trace context manager
- Identify three practices (per-call metadata, prompt versioning, evaluator suite) that turn raw traces into debuggable history

## 2. Introduction

LangSmith captures a structured trace of every step an LLM application takes — every model call, every tool invocation, every nested graph node — in a queryable, replayable, taggable interface. Without this, debugging a tool-using agent is guesswork: when the model picks the wrong tool, you have no record of what it saw at decision time.

Per **D-031**, W3 Tue is LangSmith's first real programme appearance — not stubs, real traces in the cohort workspace. Today's single-agent traces become Wed's baseline for debugging the multi-agent supervisor. If traces are not appearing by 10:55, fix the `.env` before continuing — Wed and Thu observability depend on it.

LangSmith is a **preview and integration point** here only. Deeper evaluation workflows — dataset management, evaluator suites, regression testing — are W5 material. Today: wire it, verify it, attach metadata.

## 3. Core Concepts

### 3.1 What a trace contains

A LangSmith trace is a tree of **runs**. The root run is the top-level invocation; children are every model call, tool call, nested graph node, and branch evaluation inside it. Each run captures: inputs (exact prompt or args), outputs (exact response or return value), metadata (tokens, latency, model, cost, error), tags, and evaluator references. Walking this tree shows exactly what the system did and saw at each step.

### 3.2 The three (or four) environment variables

```bash
LANGSMITH_API_KEY=ls-...                         # workspace API key
LANGSMITH_TRACING=true                           # turns the auto-tracer on
LANGSMITH_PROJECT=acquire-gov-w3                 # buckets today's traces
LANGSMITH_ENDPOINT=https://eu.api.smith.langchain.com  # ONLY for non-US regions
```

The regional endpoint is easy to miss. An EU-region workspace silently fails authentication against the default US endpoint — the API key is not recognised. Set `LANGSMITH_ENDPOINT` explicitly for any non-US region.

> [!IMPORTANT]
> **Load env vars before the LangChain import.** LangChain's auto-tracer reads them at import time. Setting them after import is silently a no-op. This is the most common "why aren't my traces appearing?" failure mode.

### 3.3 Three tracing mechanisms

**Mechanism A — Environment-variable auto-tracing.** Any code path through LangChain or LangGraph is auto-traced when the env vars are set. No code changes. This is the default.

**Mechanism B — `@traceable` decorator.** Wrap any Python function outside LangChain/LangGraph — utility functions, custom integrations — to get it into the trace tree alongside the agent's model calls.

**Mechanism C — Trace context manager.** Ad-hoc instrumentation for third-party SDK calls you cannot decorate.

Choose A by default, layer B for non-LangChain business logic, reach for C only when neither fits.

### 3.4 Tracing as the substrate for multi-agent debugging

In a single-agent loop, raw logs suffice. Multi-agent systems destroy that affordance: three parallel workers produce three interleaved streams with no attribution. LangSmith renders this as a tree — supervisor root, three child subtrees, each with its own model and tool calls. Click into the failing subtree to see exactly what input it received. Wire tracing before building the agent.

### 3.5 Three practices that turn raw traces into debuggable history

- **Attach per-call metadata** — `user_id`, `session_id`, `feature_flag`. Filters to "all runs for user 42 where feature X was on" when reproducing a bug.
- **Pin prompt versions** — LangSmith treats prompts as versioned assets; every trace records which version ran. Regressions become attributable.
- **Attach evaluators** — trace + evaluation + dataset form a regression loop stronger than ad-hoc QA. Full evaluator workflow is W5; today wire traces and metadata.

## 4. Generic Implementation

A FastAPI service using both env-var auto-tracing (for LangChain code) and the `@traceable` decorator (for adjacent business logic):

```python
import os
from langsmith import traceable

# Load env vars BEFORE any LangChain/LangGraph import
# os.environ["LANGSMITH_API_KEY"] = "ls-..."
# os.environ["LANGSMITH_TRACING"] = "true"
# os.environ["LANGSMITH_PROJECT"] = "acquire-gov-w3"

@traceable(run_type="tool", name="resolve_agency")
def resolve_agency(agency_id: str) -> dict:
    # Custom CRM lookup — not a LangChain primitive,
    # but we want it in the trace tree alongside the agent's model calls.
    return agency_repo.find_one(agency_id)

def handle_intake_triage(agency_id: str, session_id: str, proposal_payload: dict) -> dict:
    agency = resolve_agency(agency_id)  # captured in trace via @traceable
    response = agent.invoke(
        {"messages": [{"role": "user", "content": str(proposal_payload)}]},
        config={
            "metadata": {
                "agency_id": agency_id,
                "session_id": session_id,
                "proposal_id": proposal_payload.get("proposal_id"),
            },
            "tags": ["intake-triage", "w3-tue", "prod"],
        },
    )
    return response["messages"][-1].content
```

Env vars load before import. `resolve_agency` is decorated so it appears in the trace tree alongside the agent's model calls. Metadata and tags are passed at invoke time for UI filtering.

## 5. Real-world Patterns

**Customer-support copilots (Zendesk, Intercom).** Every conversation tagged with the ticket ID. When a user reports a bad answer, engineers filter to that ticket and walk the trace. Replay tests prompt changes against the exact failing conversation.

**Code assistants (Cursor, Cognition Labs).** A single request produces dozens of nested runs across planner + worker agents. The trace tree is the only practical debug surface.

> [!NOTE]
> **Two trace audiences.** LangSmith = *technical* trace (tokens, latencies, payloads) for engineers. `AuditEvent` table = *regulatory* trace (who acted, with what authority) for the CO and OIG. Never conflate them.

**Regulated industries.** LangSmith answers "why did the agent decide X?"; AuditEvent answers "was it authorised?"

> [!TIP]
> **Replay as a first-class debug tool.** Swap model or prompt, re-run the exact same input — validate a fix against the historical failing trace before deploying.

## 6. Best Practices

- Wire env vars before the LangChain import — auto-tracing reads them at import time
- Set `LANGSMITH_PROJECT` per environment — prod, staging, dev traces stay separate
- Attach `user_id`, `session_id`, and at least one tag at every invocation
- Layer `@traceable` on non-LangChain business logic you want in the trace tree
- Be intentional about PII — traces capture full inputs/outputs; redact at the boundary or use self-hosted observability

> [!WARNING]
> **Anti-pattern: `tracing-as-afterthought`.** Adding observability after the agent is built means the first production failures have no trace history to debug from. Wire LangSmith before the first tool call, not after the first incident. In multi-agent systems, retrofitting tracing is harder — metadata contracts must be designed with the topology, not bolted on afterward.

## 7. Hands-on Exercise

Wire LangSmith into a FastAPI service with a `POST /agent/onboard` route, a LangGraph agent, and two helpers (`load_user_profile`, `check_eligibility`) outside LangChain. Tasks: (1) set the three (or four) env vars; (2) add `@traceable` to both helpers; (3) attach `user_id`, `session_id`, and a `prod` tag at invoke time; (4) write a one-paragraph runbook note for how an on-call engineer finds the trace for a specific user complaint.

> [!NOTE]
> **Self-check** (30 s — answer mentally before expanding)
>
> 1. Why must env vars be set before the LangChain import, not after?
> 2. When do you use `@traceable` rather than relying on env-var auto-tracing?

<details>
<summary>Show answers</summary>

1. LangChain's auto-tracer initialises its instrumentation at import time by reading the environment variables. If they are set after the import, the tracer has already initialised with no-op stubs and will silently produce no traces for the rest of the process lifetime.
2. Use `@traceable` for any function that runs outside the LangChain/LangGraph runtime — custom business logic, third-party SDK calls, database lookups — that you want visible in the trace tree alongside the agent's model calls. Without the decorator, these functions are invisible in the trace even though they may be doing significant work the agent depends on.

</details>

## 8. Key Takeaways

- **Trace = tree of runs** — root + nested children, each with inputs, outputs, tokens, latency, metadata, tags
- **Three env vars** (`API_KEY`, `TRACING=true`, `PROJECT`) wire it; `LANGSMITH_ENDPOINT` required for non-US regions — easy to miss
- **Substrate for multi-agent debugging** — interleaved raw logs are unparsable; the trace tree disentangles them
- **Env-var auto-tracing for LangChain/LangGraph; `@traceable` for adjacent business logic**
- **Per-call metadata + prompt versioning** turn raw traces into debuggable history — full evaluator workflow is W5

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://docs.langchain.com/langsmith/observability-quickstart — Tracing quickstart (LangChain docs) — retrieved 2026-05-26 — hot-tech
- https://www.langchain.com/langsmith/observability — LangSmith: AI Agent & LLM Observability Platform — retrieved 2026-05-26 — hot-tech
- https://blog.langchain.com/debugging-deep-agents-with-langsmith/ — Debugging Deep Agents with LangSmith (LangChain blog) — retrieved 2026-05-26 — hot-tech
- https://docs.langchain.com/langsmith/evaluation — LangSmith Evaluation (LangChain docs) — retrieved 2026-05-26 — hot-tech
- https://www.ai.cc/blogs/how-to-use-langsmith-2026-complete-guide/ — How to Use LangSmith in 2026: Complete Tracing & Debugging Guide — retrieved 2026-05-26 — hot-tech

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

The 2026 LangSmith Engine release adds an AI-assisted analysis layer that examines traces and suggests fixes for failing or expensive runs. Useful as a second pass — it can surface patterns across hundreds of traces that a human reviewer would miss (e.g., "the agent consistently calls get_solicitation twice per run when the amendment is from agency X"). Do not rely on it as a substitute for the three practices (per-call metadata, prompt versioning, evaluator suite) — those are the data that make the AI-assisted analysis useful.

For the acquire-gov regulatory-compliance story: the LangSmith trace and the `AuditEvent` table serve different audit audiences. The LangSmith trace is for the engineering team — it is the debug substrate. The `AuditEvent` table is for the contracting officer and OIG — it is the regulatory record. Never conflate the two. The `AuditEvent` schema is governed by FAR record-keeping requirements; the LangSmith trace schema is governed by engineering convenience. If you store PII or FOUO material in LangSmith, you need either a self-hosted deployment or explicit data-handling agreements with LangChain. For Cohort #1, the cohort workspace key is provisioned by the instructor — do not store actual proposal content in traces; use stub payloads.

Prompt versioning in LangSmith works via the Hub: `hub.pull("my-org/my-prompt:v3")` returns a prompt object that also records the version identifier in any traces that use it. Senior FDEs building the W5 evaluation harness should design prompt identifiers as `{org}/{name}:{semver}` from day one — retrofitting versioning after the fact requires re-tagging historical traces, which is tedious and error-prone.

</details>

Last verified: 2026-06-06
