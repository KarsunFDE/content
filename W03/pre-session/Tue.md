---
template: pre-session-reading
week: W03
day: Tue
phase: Agentic
topic: "ReAct in LangGraph + single-agent intake-triage + LangSmith tracing"
estimated_total_minutes: 55
last_verified: 2026-05-23
fde_situations: [3, 4, 7, 11, 12]
tech: [LangGraph, LangChain v1.0, Bedrock InvokeModel, LangSmith, FastAPI]
sources_research_briefs: [research/langgraph-react-tool-calling.md, research/langsmith-tracing-setup.md]
author: instructor
---

# Pre-session reading — ReAct in LangGraph + single-agent intake-triage + LangSmith

Week 3, Day Tue. Estimated total time on task: ~55 minutes. Last verified: 2026-05-23.

## 1. Why this matters

Tuesday is the first day real LangGraph code lands in `ai-orchestrator/`. The intake-triage flow (`POST /agent/intake-triage`) gets built today as a *single-agent* ReAct loop — the lighter warm-up before Wed's multi-agent shape (per D-060). This is also the first day **LangSmith tracing** is live in the dev environment (per D-031 — W3 Tue is the first programme appearance). The discipline you build today (tool-calling schemas, idempotency, tracing) becomes load-bearing Wed when there are five agents instead of one.

## 2. Core concept in 5 minutes — ReAct is the simplest useful agent

ReAct = *Reason + Act*. The agent loops: read state, reason via LLM, optionally call a tool, observe the tool result, loop again. The loop ends when the LLM emits a "done" signal (in modern Anthropic tool-calling, that's `stop_reason: "end_turn"`).

For the intake-triage flow, the loop looks like:

1. **Read** the incoming proposal payload from `POST /agent/intake-triage`.
2. **Reason** (Bedrock invoke): "what's missing or anomalous about this proposal?"
3. **Act** (tool call): `get_solicitation(solicitation_id)` or `get_amendments(solicitation_id)` or `score_completeness(proposal_id)`.
4. **Observe** the tool result, fold back into context.
5. Loop until: triage complete → `route_to_evaluators(...)` OR anomaly found → `escalate_to_co(...)`.

Two tools mutate state (`route_to_evaluators`, `escalate_to_co`); three are read-only (`get_solicitation`, `get_amendments`, `score_completeness`). The mutating ones must be **idempotent** — calling them twice with the same arguments must not double-route or double-escalate. LangGraph's checkpointing makes this critical: if the graph crashes mid-handoff and you resume, you don't want a duplicate `route_to_evaluators` write.

## 3. What to read or watch tonight

- [LangGraph — Agent (ReAct) implementation](https://langchain-ai.github.io/langgraph/agents/agents/) (~15 min read), published 2026-01-22 (verified 2026-05-23 via /web-research). Focus on the `create_react_agent` factory and the tool-schema definitions. This is the shape Tuesday's afternoon practical implements.
- [AWS — Use Bedrock InvokeModel with Claude tool use](https://docs.aws.amazon.com/bedrock/latest/userguide/tool-use-inference-call.html) (~10 min read), retrieved 2026-05-23 via /web-research. Focus on the `tools` request parameter, the `tool_use` response content block, and how to thread tool results back via a `tool_result` user message. This is the Bedrock-direct (not Agents-for-Bedrock) shape; the cohort uses InvokeModel only this week (D-060).
- [LangSmith — Tracing setup](https://docs.smith.langchain.com/observability) (~10 min read), retrieved 2026-05-23 via /web-research. Focus on the `LANGSMITH_API_KEY` + `LANGSMITH_TRACING=true` env vars. Your instructor has wired the cohort workspace; today you'll see your agent runs appear in the LangSmith UI.
- *(optional)* [Anthropic — Tool use with Claude](https://docs.claude.com/en/docs/agents-and-tools/tool-use/overview) (~12 min read), published 2025-09-29 (verified 2026-05-23 via /web-research). Background framing on tool-use prompting; useful if you've never written a `tools` schema before.

## 4. Two questions to come in with tomorrow morning

1. The vendor uploaded a proposal to a solicitation that was *amended* yesterday — the proposal references the pre-amendment SOW. Which tool does the triage agent call first to detect the staleness? What's the schema of the tool's response?
2. Your `route_to_evaluators` tool gets called twice with the same `(proposal_id, evaluator_ids)` because the graph crashed mid-handoff and the checkpoint resumed. What's the idempotency strategy — request-deduplication key in the DB, conditional insert, or something else?

## 5. Glossary refresh (terms you'll hear today)

- **ReAct loop** — Reason → Act → Observe pattern; the simplest agent shape.
- **Tool call** — LLM emits a structured request to invoke a tool (Bedrock format: `tool_use` content block with `name`, `input`).
- **Tool result** — the response fed back into the LLM as a `tool_result` content block on a `user` message.
- **Idempotency** — same input, same outcome on retry. Critical for state-mutating tools when the graph can crash + resume.
- **Self-querying retrieval** — agent rewrites the user's query before retrieving (e.g., extracts filters: NAICS, set-aside, agency_id). Tuesday introduces this; Wed's multi-agent extends it.
- **Multi-query retrieval** — agent issues N variants of a query in parallel + fuses results. Brief intro Tue; deeper Wed.
- **RAG Fusion / CRAG** — Corrective RAG. The agent retrieves, *evaluates* the retrieval quality (LLM-judge), and re-retrieves with adjusted query if quality is low.
- **LangSmith** — LangChain's observability platform. Per D-031, first real programme appearance is today (not stubs).

## 6. What you'll ship today

- `POST /agent/intake-triage` working end-to-end as a single-agent ReAct loop in `ai-orchestrator`.
- Five tool schemas defined with strict input/output validation (Pydantic).
- LangSmith traces appearing in the cohort workspace for every triage invocation.
- Pytest covering: (a) happy-path triage → route, (b) anomaly → escalate, (c) idempotent `route_to_evaluators` (same input twice → one write).
- Codex Adversarial Review (Near-full per D-034) on the PR — expect findings on tool schemas, idempotency boundary, and tracing instrumentation gaps.

## 7. Anti-patterns to avoid today

- **Skipping the tool schemas.** "We'll just pass dicts around" — no. Bedrock's tool-use API expects JSON Schema; LangGraph state typing expects TypedDicts. Codex flags this hard at Near-full strictness.
- **Catching tool errors silently.** A failed `get_solicitation` call must propagate as a structured error to the LLM (so it can reason about a fallback) — not return `None` and let the agent hallucinate.
- **Treating the tracing as cosmetic.** Wed's multi-agent debugging *requires* the LangSmith trace history. Build the tracing discipline today.
- **Wiring `langchain.agents.AgentExecutor` from a v0.x example.** That's deprecated. The v1.0 shape is `create_react_agent` or hand-rolled with `langgraph.graph.StateGraph` — see `skills/tech-research/references/known-bad-patterns.yml` entries `langchain-chain-class`, `langchain-pre-v1-advocacy`.

## 8. Optional deep-dive (not required to participate)

- [LangGraph — Tool calling](https://langchain-ai.github.io/langgraph/agents/tools/) (~8 min read), published 2026-02-04 (verified 2026-05-23 via /web-research). Useful background on `ToolNode` + `tools_condition` for tomorrow's supervisor-worker pattern.
