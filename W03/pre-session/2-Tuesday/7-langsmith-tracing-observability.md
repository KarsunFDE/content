---
week: W03
day: Tue
topic_slug: langsmith-tracing-observability
topic_title: "LangSmith tracing — observability for tool-using agents"
parent_overview: W03/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 8
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
last_verified: 2026-05-26
---

# LangSmith tracing — observability for tool-using agents

## 1. Learning Objectives

- By the end of this reading, the learner can describe what a LangSmith trace contains (parent + nested runs, token counts, latencies, prompts, responses, tool calls and results).
- By the end of this reading, the learner can configure tracing with the three environment variables that wire it on, and identify when a fourth (regional endpoint) is required.
- By the end of this reading, the learner can articulate why tracing is a correctness substrate for multi-agent debugging — not a cosmetic add-on.
- By the end of this reading, the learner can describe the three tracing mechanisms (env-var auto-tracing for LangChain/LangGraph code, the `@traceable` decorator, the context-manager) and choose between them.
- By the end of this reading, the learner can identify three discipline practices (per-call metadata, prompt versioning, evaluator-attached evaluations) that turn raw traces into debuggable history.

## 2. Introduction

LangSmith is LangChain's observability and evaluation platform. It captures a structured trace of every step an LLM application takes — every model call, every tool invocation, every nested chain or graph node — and stores those traces in a queryable, replayable, taggable interface. Without an observability layer of some kind, debugging a tool-using agent is guesswork: when the model picks the wrong tool, gives a wrong answer, or loops on itself, you have no record of *what* it saw at decision time. With a trace, you can walk the chain backward and answer "what was the model's input when it made that decision?"

The discipline matters most when agents grow beyond a single ReAct loop. A multi-agent supervisor delegating to workers, a graph-based agent with conditional branching, a long-running session with compaction events — these systems have far too many internal state transitions to debug by reading logs. LangSmith — and equivalent platforms like LangFuse, Phoenix, Helicone, and Honeycomb's GenAI tracing — solves the same class of problem that distributed-tracing solved for microservices a decade earlier. The pattern travels: instrument the boundaries, render the trace as a tree, walk the tree to debug.

This reading is the wiring-discipline reading. Three env vars, two checks, four practices.

## 3. Core Concepts

### 3.1 What a trace contains

A LangSmith trace is a tree of **runs**. The root run is the top-level invocation (an agent call, a chain run, a graph invoke). Children are everything that happened inside — every model call, every tool call, every nested chain, every conditional branch evaluation. Each run captures:

- **Inputs** — the exact prompt or arguments passed in.
- **Outputs** — the exact response or return value.
- **Metadata** — token counts, latency, model name, cost, error status.
- **Tags** — optional labels you can attach for filtering ("user_id=42", "experiment=v3", "env=prod").
- **References** — links to evaluator results, dataset rows, and prompt versions.

Walking this tree end-to-end shows you exactly what the system did and what it saw at each step.

### 3.2 The three (or four) environment variables

LangChain and LangGraph applications wire LangSmith on by setting environment variables. With these in place, every invocation auto-traces — no code change required.

```bash
LANGSMITH_API_KEY=ls-...                        # your workspace API key
LANGSMITH_TRACING=true                          # turns the auto-tracer on
LANGSMITH_PROJECT=my-agent-project              # buckets these traces
LANGSMITH_ENDPOINT=https://eu.api.smith.langchain.com   # ONLY if your workspace is in a non-US region
```

The regional endpoint is the easy thing to miss. A workspace provisioned in the EU region will silently fail authentication with the default US endpoint — the API key isn't recognised. Set `LANGSMITH_ENDPOINT` explicitly when in any region other than US.

### 3.3 Three tracing mechanisms — pick the right one

LangSmith offers three ways to capture traces:

**Mechanism A — Environment-variable auto-tracing.** When the env vars are set, any code path that runs through LangChain or LangGraph is auto-traced. No code changes. This is the default for LangChain/LangGraph applications.

**Mechanism B — The `@traceable` decorator.** Wrap any Python function. Every call traces; the function's arguments and return value are captured. Use this for code outside the LangChain/LangGraph runtime — utility functions, custom integrations, data-prep steps that you still want in the trace.

```python
from langsmith import traceable

@traceable(run_type="tool", name="get_account_status")
def get_account_status(account_id: str) -> dict:
    return account_repo.find_one(account_id)
```

**Mechanism C — The trace context manager.** Wrap a code block. Useful for ad-hoc instrumentation when neither auto-tracing nor `@traceable` fits — for example, instrumenting a third-party SDK call that you cannot decorate.

Choose A by default, layer B for non-LangChain functions, and reach for C only when neither covers the surface.

### 3.4 Tracing is the substrate for multi-agent debugging

In a single-agent ReAct loop, you could in principle debug from raw logs — there's one model, one trace of tool calls, one linear sequence. Multi-agent systems destroy that affordance. A supervisor agent dispatches three worker agents in parallel; one returns a contradictory result; the supervisor weighs and resolves; the user sees an aggregated answer. With raw logs you have three interleaved log streams and no way to tell which worker said what when.

LangSmith renders this as a tree: the supervisor's root run with three child run subtrees (one per worker), each subtree containing the worker's own model calls and tool calls, plus the supervisor's resolution step. You click into the subtree of the worker that returned a contradictory result and see exactly what it was given as input.

This is why the discipline of "wire tracing first, then build the agent" matters. Without traces, multi-agent debugging is guesswork. With traces, it's a tree walk.

### 3.5 What you do with a trace — three practices

Three practices turn raw traces into actively debuggable history:

- **Attach per-call metadata.** When you invoke the agent, pass run metadata: `user_id`, `session_id`, `feature_flag`, `experiment_arm`. This lets you filter traces in the LangSmith UI to "all the runs for user 42 where feature X was on" — vital when reproducing a user-reported bug.
- **Pin prompt versions.** LangSmith treats prompts as first-class versioned assets. When you change a prompt, you bump the version; every trace records which prompt version it ran. When a regression shows up, you can attribute it.
- **Attach evaluators.** LangSmith's evaluation harness lets you run datasets against your agent and score the runs with custom evaluators. Trace + evaluation + dataset together form a regression-testing loop that's far stronger than ad-hoc QA.

The 2026 LangSmith Engine release adds an AI-assisted layer that analyses traces and suggests fixes for failing or expensive runs — useful as a second pass, but no substitute for the three practices above.

### 3.6 Replay — testing a fix without re-running production

The replay feature lets you take an existing trace, swap the model or the prompt, and re-run the exact same input. This is the right way to validate a candidate fix to a prompt regression — you take the trace where the agent misbehaved, replay it under the new prompt, see if the new prompt produces the right behaviour on the same input. No need to re-trigger the user-flow that produced the original trace.

## 4. Generic Implementation

A generic Python service that uses both env-var auto-tracing (for the LangChain code) and the `@traceable` decorator (for adjacent business logic).

```python
import os
from langsmith import traceable

# --- Environment configuration ---
# Set these in your .env or runtime environment before imports.
# os.environ["LANGSMITH_API_KEY"] = "ls-..."
# os.environ["LANGSMITH_TRACING"] = "true"
# os.environ["LANGSMITH_PROJECT"] = "support-bot-prod"
# # EU region only: os.environ["LANGSMITH_ENDPOINT"] = "https://eu.api.smith.langchain.com"

# --- Business-logic helper, decorated for tracing ---
@traceable(run_type="tool", name="resolve_customer")
def resolve_customer(email: str) -> dict:
    # Custom lookup against the CRM — not a LangChain primitive,
    # but we want it in the trace tree so we can see what the agent saw.
    return crm.lookup_by_email(email)

# --- Agent invocation, with run metadata for traceability ---
def handle_support_message(user_id: str, session_id: str, message: str) -> str:
    customer = resolve_customer(user_id)   # captured in trace via @traceable
    response = agent.invoke(
        {"messages": [{"role": "user", "content": message}]},
        config={
            "metadata": {
                "user_id": user_id,
                "session_id": session_id,
                "customer_tier": customer["tier"],
            },
            "tags": ["support-bot", "prod"],
        },
    )
    return response["messages"][-1].content
```

Three things worth noticing:

1. **The env vars are loaded before the LangChain import.** LangChain's auto-tracer reads them at import time; setting them after import is silently a no-op.
2. **`resolve_customer` is decorated** so it appears in the trace tree alongside the agent's model calls. Without the decorator the CRM lookup is invisible.
3. **Metadata and tags are passed at invoke time.** This lets you filter LangSmith traces by `user_id` or `customer_tier` in the UI — essential for reproducing a specific user's report.

## 5. Real-world Patterns

### 5.1 Customer-support copilots — Zendesk, Intercom

Support-bot vendors lean heavily on LangSmith (or equivalent) because the failure mode "bot gave a wrong answer to user X" is impossible to debug without the original prompt context. Pattern: every conversation is tagged with the ticket ID at invoke time; when a user reports a bad answer, support engineers filter LangSmith to that ticket and walk the trace tree. Replay lets them test prompt changes against the exact failing conversation before deploying.

### 5.2 Code assistants — Cursor, Cognition Labs, Replit Agent

Code assistants run multi-agent pipelines where a planner agent dispatches to specialised workers (file-edit, search, test-run, lint-fix). The trace tree is the only practical debug surface — a single user request produces dozens of nested runs across multiple workers. Cognition Labs and Cursor have both publicly described how tracing is non-negotiable for shipping multi-agent code-modification systems.

### 5.3 Healthcare clinical-decision-support tools

Clinical-decision-support tools have a compliance need that makes tracing load-bearing: every model-driven recommendation must be reconstructible after the fact for medical-record integrity. The trace becomes the evidentiary substrate — what the model saw, what it recommended, what tools it called. The same shape applies in regulated industries generally (legal, finance, aviation).

### 5.4 Search-and-research products — Perplexity, You.com

Multi-step research assistants run query planning, parallel retrieval, source ranking, and answer synthesis as nested runs. The trace tree exposes the full provenance chain — which sources informed which claims. The provenance trace becomes the user-facing "show your work" feature, not just an internal debug tool.

## 6. Best Practices

- **Wire the env vars before the LangChain import.** Auto-tracing reads them at import time.
- **Set `LANGSMITH_PROJECT` per environment.** Prod, staging, and dev traces should land in separate projects to prevent dataset pollution.
- **Attach `user_id`, `session_id`, and at least one experimental-arm tag at every invocation.** Without these, filtering to reproduce a specific bug is unmanageable.
- **Pin prompt versions to traces.** When prompts are versioned, regressions become attributable.
- **Layer `@traceable` on non-LangChain business logic.** Anything you would want to see in the trace tree should be in the trace tree.
- **Build a small evaluator suite early.** Even five hand-curated regression cases run nightly is better than zero.
- **Treat replay as a first-class debug tool.** Test prompt and model changes against historical traces before deploying.
- **Be intentional about PII.** Traces capture full inputs and outputs — if those contain sensitive data, either redact at the boundary or use a self-hosted observability stack.

## 7. Hands-on Exercise

**Code exercise (15 min).** You are wiring LangSmith into an existing FastAPI service that hosts a single-agent customer-onboarding workflow. The service has:

- An invoke route `POST /agent/onboard` that accepts `{user_id, message}`.
- A LangGraph agent assembled at startup.
- Two business-logic helpers (`load_user_profile`, `check_eligibility`) currently not in LangChain.

Tasks:

1. Add the three (or four — note the region question) environment variables to the service config.
2. Add a `@traceable` decorator to both business-logic helpers.
3. Modify the invoke route to attach `user_id` and `session_id` metadata plus a `prod` tag to the agent invocation.
4. Write a one-paragraph note for the runbook explaining how an on-call engineer would find the trace for a specific user complaint.

**What good looks like.** A solution sets env vars in the service config (not in code), decorates the two helpers with `@traceable(run_type="tool", name=...)`, attaches metadata in the `config=` argument of `agent.invoke`, and the runbook note explains the LangSmith filter syntax for finding traces by `user_id`. Bonus: the writer flags that `load_user_profile` may include PII and proposes either a redaction step or a self-hosted alternative.

## 8. Key Takeaways

- **What does a LangSmith trace contain?** (A tree of runs — root + nested children — each with inputs, outputs, tokens, latency, and optional metadata/tags.)
- **What four env vars matter, and which one is easy to miss?** (`LANGSMITH_API_KEY`, `LANGSMITH_TRACING`, `LANGSMITH_PROJECT`; plus `LANGSMITH_ENDPOINT` for non-US regions — the easy miss.)
- **Why is tracing the substrate for multi-agent debugging?** (Multiple interleaved agent traces are unparsable from raw logs; the trace tree disentangles them.)
- **When do you use `@traceable` vs env-var auto-tracing?** (Env-var for LangChain/LangGraph code; `@traceable` for business logic you want in the trace tree alongside it.)
- **What three practices turn raw traces into actively debuggable history?** (Per-call metadata; prompt versioning; attached evaluators with a small regression suite.)

## Sources

1. [Tracing quickstart — LangChain docs](https://docs.langchain.com/langsmith/observability-quickstart) — retrieved 2026-05-26
2. [LangSmith: AI Agent & LLM Observability Platform](https://www.langchain.com/langsmith/observability) — retrieved 2026-05-26
3. [Debugging Deep Agents with LangSmith (LangChain blog)](https://blog.langchain.com/debugging-deep-agents-with-langsmith/) — retrieved 2026-05-26
4. [LangSmith Evaluation — LangChain docs](https://docs.langchain.com/langsmith/evaluation) — retrieved 2026-05-26
5. [How to Use LangSmith in 2026: Complete Tracing & Debugging Guide](https://www.ai.cc/blogs/how-to-use-langsmith-2026-complete-guide/) — retrieved 2026-05-26

Last verified: 2026-05-26
