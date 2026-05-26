---
template: pre-session-reading
week: W03
day: Tue
phase: Agentic
topic: "ReAct in LangGraph — single-agent intake-triage, tool-calling discipline, agentic-RAG patterns, LangSmith goes live"
estimated_total_minutes: 55
last_verified: 2026-05-26
fde_situations: [3, 4, 7, 11, 12]
tech: [LangGraph, LangChain v1.0, Bedrock InvokeModel, LangSmith, FastAPI]
sources_research_briefs:
  - research/langchain-v1-20260522.md
  - research/bedrock-claude-catalog-20260522.md
author: instructor
---

# Pre-session reading — ReAct in LangGraph + single-agent intake-triage + LangSmith

Week 3, Day Tue (Phase 1 Gate week). Estimated time on task: ~55 min. Last verified: 2026-05-26.

> **Calendar reality.** Checkpoint 1 exam runs in-person 09:00–10:30 (90 min, both tiers). War-room begins 10:30 post-exam and runs to 12:00 — half-day shape. The pre-session reading below is sized to that compression: read it tonight (Mon evening) so 10:30 starts at full speed.

## 1. Why this matters (5 min)

Today is when Mon's topology becomes Tuesday's code. The intake-triage flow (`POST /agent/intake-triage` in `services/ai-orchestrator`) goes from blank FastAPI endpoint to a working **single-agent ReAct loop** — the lighter warm-up before Wed's multi-agent pivot to evaluator → consensus → SSA-review (per D-060). LangSmith tracing comes alive in the cohort workspace for the first time (D-031). And every tool the agent calls — read or write — gets a Pydantic schema and an idempotency story before it lands in `main`. Wed's multi-agent debugging requires the trace history you build today; Wed's audit-fan-out anti-pattern presupposes you nailed idempotency today. Get the discipline right at one agent before there are five.

## 2. ReAct loop = Reason + Act + Observe (12 min)

ReAct is the simplest useful agent shape: read state, **reason** via the LLM, **act** by calling a tool (or finishing), **observe** the tool result, loop again. The loop terminates when the model emits a "done" signal — for Claude on Bedrock, that's `stop_reason: "end_turn"` with no further `tool_use` content block.

For the acquire-gov intake-triage flow, today's loop reads:

1. **Read** the incoming proposal payload from `POST /agent/intake-triage`.
2. **Reason** (Bedrock InvokeModel): *"is anything stale, missing, or anomalous about this submission?"*
3. **Act** (tool call): `get_solicitation(solicitation_id)` or `get_amendments(solicitation_id)` or `score_completeness(proposal_id)`.
4. **Observe** the tool result; fold it back into the conversation context.
5. **Loop** until: triage complete → `route_to_evaluators(...)` OR anomaly found → `escalate_to_co(...)`.

This is the **agent vs workflow** distinction the Anthropic "Building effective agents" framing draws: a workflow follows a pre-specified path; an agent decides at each step which tool to call (or whether to stop). Intake-triage is on the agent side of that line — the CO didn't pre-specify "always check amendments first"; the model decides based on what it sees in the payload.

LangChain v1.0's `create_agent` factory is one ergonomic shape for this loop; LangGraph's `StateGraph` + `ToolNode` is the more general primitive the cohort will live in from Wed onward. Today: see both, prefer `create_agent` for the warm-up endpoint, then we move to hand-wired `StateGraph` on Wed.

> Source: [LangChain v1 overview](https://docs.langchain.com/oss/python/releases/langchain-v1) — `create_agent` framing. Retrieved 2026-05-22 via `/web-research`; see `research/langchain-v1-20260522.md`.

## 3. Tool-calling discipline — schemas and the Bedrock tool-use API (10 min)

A "tool call" in this stack means: the model emits a structured request to invoke one of your named functions, you execute the function, you thread the result back as a `tool_result` content block on a `user` message, the model continues reasoning. On Bedrock InvokeModel with Claude, the shape is:

- **Request:** include a `tools` parameter — a JSON-Schema-flavoured list of `{name, description, input_schema}` objects.
- **Response:** the model returns a `content` array; one of the blocks is `{type: "tool_use", id, name, input}`.
- **Thread-back:** your next request carries the prior assistant turn verbatim, plus a `user` turn whose content includes `{type: "tool_result", tool_use_id, content}`.

Pydantic v2 is the load-bearing piece here. Each tool's `input_schema` is a Pydantic `BaseModel.model_json_schema()`; tool inputs validate at the boundary before they touch your DB. The five intake-triage tools, with their schemas:

| Tool | Type | Schema (signature shape) |
|------|------|---------------------------|
| `get_solicitation` | read | `(solicitation_id: UUID) → Solicitation` |
| `get_amendments` | read | `(solicitation_id: UUID, agency_id: UUID) → list[Amendment]` |
| `score_completeness` | read | `(proposal_id: UUID) → CompletenessScore` |
| `route_to_evaluators` | **write** | `(proposal_id: UUID, evaluator_ids: list[UUID], idempotency_key: str) → RouteResult` |
| `escalate_to_co` | **write** | `(proposal_id: UUID, reason_code: str, idempotency_key: str) → EscalationResult` |

Three read, two write. The two writes are the focus of §4. Note `get_amendments` carries `agency_id` explicitly — the agent does **not** get to forget the multi-tenant boundary, even when the prompt seems to want to. Item 10 (multi-tenant filter) surfaces here.

> Source: [AWS Bedrock — Use InvokeModel with Claude tool use](https://docs.aws.amazon.com/bedrock/latest/userguide/tool-use-inference-call.html) — request/response shape. Retrieved 2026-05-22 via `/web-research`; see `research/bedrock-claude-catalog-20260522.md`.

## 4. Idempotency for state-mutating tools (8 min)

The CO's framing yesterday — *"I — and only I — fire the irreversible actions"* — applies to the agent's writes, not just to the HITL ADR. **`route_to_evaluators` and `escalate_to_co` must be idempotent.** Calling either twice with the same inputs must produce one effect — one routing row, one CO escalation — not two.

Why this matters today: LangGraph checkpointing (Thu's deep-dive) means if the graph crashes mid-handoff and resumes, the tool you were calling **may execute again**. Without idempotency, that's a duplicate `AuditEvent` row, a duplicate CO notification, a duplicate evaluator assignment. With idempotency, the second call is a no-op.

Three viable shapes — pick one per tool and document it in the pair-repo:

1. **Idempotency-key column.** Tool accepts `idempotency_key` parameter (e.g., `f"route:{proposal_id}:{evaluator_set_hash}"`). DB has a unique constraint on `(idempotency_key)`. Second call hits the constraint, returns the first call's result.
2. **Conditional insert.** `INSERT ... WHERE NOT EXISTS (SELECT 1 FROM routes WHERE proposal_id = $1 AND evaluator_set_hash = $2)`. Slightly less expressive than (1) but no extra column.
3. **Request-deduplication via Redis.** `SETNX idempotency:{key} 1 EX 86400`. Useful when the write spans multiple DB tables but you can't wrap them in a single SQL constraint.

**This is the answer to last night's "two questions" Q2.** The intake-triage flow uses shape (1) — explicit `idempotency_key` parameter on both write tools, unique constraint in Postgres on `audit_events.idempotency_key`. Codex Adversarial Review at Near-full strictness will flag any write tool that lacks one, hard.

**Acquire-gov framing — federal audit-trail integrity.** Idempotency here isn't a hygiene preference; it's an OIG-audit defense. A duplicate `route_to_evaluators` row makes the audit trail say two evaluator assignments happened. Per FAR/agency record-keeping discipline, that's the audit trail lying about what occurred — and that's the bigger violation than the original double-write.

## 5. In-context memory and compaction for long proposals (5 min)

Federal solicitation responses are long. A 50-page Section L plus a 30-page Section M plus referenced cost templates puts the agent well past a comfortable single-turn context window even on Claude Sonnet 4.5's 200K tokens. Two patterns the cohort exercises today:

- **Summarization-on-overflow.** When the running message history nears budget (track tokens explicitly per turn), summarize the older turns into a single system-message preamble: *"Earlier in this triage run, the agent confirmed solicitation X exists, found amendment Y posted at 16:30 yesterday, and scored proposal Z at 0.82 completeness."* The summary preserves the **observations** (durable facts) while shedding the **reasoning** (model-internal scratchwork that's already produced its output).
- **Selective retention.** Not all tool results are equal. The `get_amendments` result rows are durable; the model's prior reasoning *about* those rows can be dropped. Retention policy: keep all `tool_result` blocks, summarize `assistant` thinking blocks beyond N turns.

This is the same primitive W1 Thu's "Context compression patterns" introduced — same idea, agent scope rather than chat scope. LangGraph state will carry the compaction policy as a node next week; today it's hand-rolled inside the `create_agent` middleware slot.

## 6. Agentic-RAG patterns — self-querying, multi-query, fusion, CRAG (15 min)

> Density-grouping topic. This is one cohesive body — "patterns the agent uses to retrieve from the W2 RAG layer" — that the PDF splits into five items. Read it as one.

The W2 RAG layer (`POST /rag/clause-search`) is callable as an agent tool today. **First time the W2 layer is consumed by an *agent*, not a human.** Four agentic-retrieval patterns, framed as a family the cohort will choose between this week (today + Wed):

- **Self-querying retrieval.** Agent rewrites the user's query before retrieving — specifically, it extracts **filters** from natural-language intent. Example: *"clauses for small-business set-asides on cloud RFPs"* → agent emits `query: "small-business set-aside clauses"` + `filters: {NAICS: "541512", set_aside_code: "SBA"}`. The filters narrow the vector search to the relevant tenant + NAICS slice. **The intake-triage flow uses this shape today** when calling `clause-search` to retrieve the amendment's clause text for the response narrative.
- **Multi-query retrieval.** Agent issues **N parallel variants** of the query (e.g., 3 paraphrases), retrieves for each, fuses results. Defends against single-phrasing brittleness. Brief intro today; Wed's supervisor pattern dispatches the variants in parallel.
- **RAG Fusion.** Reciprocal Rank Fusion across the N variants' result sets — items that rank in multiple variants float to the top. The fusion is deterministic (ranks-based), not LLM-judged.
- **CRAG (Corrective RAG).** Agent retrieves, *evaluates the retrieval quality* via an LLM-judge step (confident / uncertain / wrong), and either uses the retrieval, re-retrieves with adjusted query, or falls back to a different source. CRAG is the most expensive of the four — extra LLM call per retrieval — but the strongest answer when retrieval brittleness is a known failure mode.

**Acquire-gov framing.** The 17-entity acquisitions schema (Vendor ↔ Proposal ↔ Evaluation ↔ Award ↔ ContractModification ↔ Cpar — W3 Wed's KG/CG topic) is the structured side; the RAG layer is the unstructured side (clause text, OIG advisories, FAR/DFARS prose). Agentic-RAG is **how the agent navigates the unstructured side using filters derived from the structured side.** Self-querying is the daily pattern; multi-query + fusion + CRAG are escalation tools when the simple retrieval comes back wrong.

> Source: pattern names taken from the LangChain agentic-RAG canon (self-query retriever, multi-query retriever, RAG Fusion, CRAG); validated against current docs retrieved 2026-05-22 via `/web-research` (`research/langchain-v1-20260522.md`).

## 7. LangSmith tracing — the first programme appearance (8 min)

Per D-031, **W3 Tue is the first real programme appearance of LangSmith** — not stubs, not screenshots, real traces in the cohort workspace. Two env vars wire it on:

```bash
LANGSMITH_API_KEY=ls-...           # cohort workspace key — instructor distributed
LANGSMITH_TRACING=true             # turns on the auto-tracer
LANGSMITH_PROJECT=acquire-gov-w3   # bucket today's traces
```

Once wired, every LangChain or LangGraph invocation in `ai-orchestrator` emits a trace tree: parent run (the `create_agent` invocation), nested child runs for each Bedrock call + each tool call, with token counts, latency-per-call, and the full prompt + response payload captured.

**Why today's discipline matters Wed.** Wed's multi-agent debugging *requires* the trace history. When the supervisor agent makes a bad delegation decision, the only way to see *why* is to walk the trace: what context did the supervisor see when it decided? LangSmith's "Replay" button replays the model call with the exact prompt; without it, debugging multi-agent is guesswork. **If LangSmith isn't capturing your traces by 10:55 Tue, stop coding and fix the `.env` — Wed-Thu observability depends on it.**

**Acquire-gov framing — tracing as audit substrate.** LangSmith captures the *technical* trace (tokens, latencies, payloads). The acquire-gov `AuditEvent` table captures the *regulatory* trace (who did what, when, with what authority — FAR/DFARS-citable). They're complementary: LangSmith answers "why did the agent decide X?"; AuditEvent answers "was the decision properly authorised?". Today, both go live; both are non-negotiable.

> Source: [LangSmith — Tracing setup](https://docs.smith.langchain.com/observability) — `LANGSMITH_TRACING` / `LANGSMITH_API_KEY` env vars. Retrieved 2026-05-23 via `/web-research` (existing pipeline citation preserved).

## What you'll ship today

- **Working `POST /agent/intake-triage`** as a single-agent ReAct loop, detecting Acme's stale proposal on the three deterministic signals (amendment posted-after timestamp, acknowledgement-required flag, no ack row) — *not* by LLM-parsing the cover letter.
- **Five tool schemas** with strict Pydantic input/output validation (three read, two write).
- **Idempotency keys** on `route_to_evaluators` and `escalate_to_co`; documented in the pair-repo.
- **LangSmith traces live** for every triage invocation; one screenshot in `docs/W3-D2-langsmith-trace.png`.
- **Pytest:** happy-path · staleness-detected-no-ack-escalate · idempotent-escalation (second call = no-op).
- **ADR:** *"Staleness response policy — re-route vs reject vs flag"* citing FAR 15.206(c).
- **Codex Adversarial Review** at Near-full strictness on the PR (D-034). Expect findings on schema strictness, idempotency edge cases, and tracing instrumentation completeness.

## Anti-patterns to avoid today *(reinforcement — these are wiring-discipline reminders, not new concepts)*

- **Skipping the tool schemas.** *"We'll just pass dicts around"* — no. Bedrock's tool-use API expects JSON Schema; LangGraph state typing expects TypedDicts. Codex flags this hard at Near-full strictness.
- **Catching tool errors silently.** A failed `get_solicitation` call must propagate as a structured error to the LLM (so it can reason about a fallback), not return `None` and let the agent hallucinate around the gap.
- **Treating tracing as cosmetic.** Wed's multi-agent debugging requires the LangSmith history. If your traces aren't appearing by 10:55, fix the `.env` before writing more code.
- **Wiring `langchain.agents.AgentExecutor` or `chain.run()` from a v0.x example.** Both removed/moved to `langchain-classic` at v1.0 GA (Oct 2025). The current shape is `create_agent` (LangChain v1.0) or hand-rolled `langgraph.graph.StateGraph` (Wed). The verb is **composing** or **sequential composition**, not "chaining". See `skills/tech-research/references/known-bad-patterns.yml` entries `langchain-chain-class`, `langchain-chaining-verb`, `langchain-lcel-pipe`.
- **LLM-parsing the cover letter to detect staleness.** Expensive, slow, fragile. Three deterministic signals are in the schema. Use them. LLM for the *response narrative*, not for the *detection*.
- **Forgetting the multi-tenant filter on `get_amendments`.** `agency_id` is in the request context. Auto-attach it to the query — don't trust the prompt to remember. Item 10 surface.

## Two questions to come in with tomorrow morning

1. Your `supervisor` agent (Wed) is about to delegate to `evaluator_worker_3`. You want a *suggested* default action the human can approve, reject, or edit — not a hard block. Which LangGraph primitive shape is that (`interrupt_before` vs `interrupt_after` vs middleware), and what does the audit row look like for a soft-resume?
2. Three pairs each picked a different graph backend for the W3 Wed scenario alternative (Neo4j vs Postgres recursive CTE vs NetworkX in-process). Without running benchmarks, which one collapses first when CPARs queries traverse 4+ hops and *why*? Which one would FedRAMP-Moderate boundary considerations push toward, and why?

## Glossary refresh

- **ReAct loop** — Reason → Act → Observe. The simplest useful agent shape.
- **Tool call / tool_use** — Bedrock-format content block: `{type: "tool_use", id, name, input}`. The structured request for the host to execute a named function.
- **Tool result / tool_result** — Response threaded back as `{type: "tool_result", tool_use_id, content}` on a `user` turn.
- **Idempotency** — Same input, same outcome on retry. Critical for state-mutating tools when LangGraph can checkpoint + resume.
- **Self-querying retrieval** — Agent rewrites the query to extract filters before vector search.
- **Multi-query retrieval** — Agent issues N variants in parallel; fuses results.
- **RAG Fusion** — Reciprocal Rank Fusion across multi-query variants (deterministic, ranks-based).
- **CRAG** — Corrective RAG. LLM-judge step evaluates retrieval quality and re-retrieves if low.
- **LangSmith** — LangChain's observability platform. Per D-031, first real programme appearance is today.
- **Sequential composition** — Plain-Python `result = step_b(step_a(x))`. Not "chaining"; not LCEL pipes.

## What today actually looks like (logistics)

- **09:00–10:30** — Checkpoint 1 exam, in-person, both tiers, 90 min, no laptops. Covers W1 Thu–Fri + W2.
- **10:30–10:35** — Break.
- **10:35–12:00** — War-room: stale-proposal incident; single-agent build; LangSmith goes live.
- **13:00–17:00** — Practical block: harden the agent (full schemas, idempotency keys, LangSmith metadata per call), ADR on staleness policy, Codex Adversarial Review on the Tue PR.
- **17:00** — EOD ship.
- **Async this week** — Your Checkpoint 1 audit interview is a 30-min Zoom on your free time, scheduled by the auditors. Doesn't bite into war-room or practical blocks.
- **Tomorrow** — Pivot day. Single-agent intake-triage hands off into multi-agent evaluator → consensus → SSA-review. HITL #4 lands. Read `pre-session/3-Wednesday/1-DailyTopicOverview.md` tonight.

## Optional deep-dive (not required to participate)

- [LangGraph — Tool calling](https://langchain-ai.github.io/langgraph/agents/tools/) — `ToolNode` + `tools_condition` background; useful prep for Wed's supervisor-worker pattern. Retrieved 2026-05-23 via `/web-research` (existing pipeline citation preserved).
- [Anthropic — Tool use with Claude](https://docs.claude.com/en/docs/agents-and-tools/tool-use/overview) — background framing on tool-use prompting if you've never written a `tools` schema before. Retrieved 2026-05-23 via `/web-research` (existing pipeline citation preserved).
- [Anthropic — Building effective agents](https://www.anthropic.com/research/building-effective-agents) — the agent-vs-workflow framing referenced in §2.

## Sources

- `research/langchain-v1-20260522.md` — LangChain v1.0 (v1.3.0 latest, 2026-05-12). Pinned `create_agent` framing; `Chain` class removed to `langchain-classic`; "chaining" verb deprecated. Retrieved 2026-05-22 via `/web-research`.
- `research/bedrock-claude-catalog-20260522.md` — Bedrock Claude model catalog + InvokeModel tool-use API shape. Retrieved 2026-05-22 via `/web-research`.
- [LangSmith — Tracing setup](https://docs.smith.langchain.com/observability) — retrieved 2026-05-23 via `/web-research` (preserved from prior authoring pass).
- [LangGraph — Agent (ReAct) implementation](https://langchain-ai.github.io/langgraph/agents/agents/) — retrieved 2026-05-23 via `/web-research` (preserved from prior authoring pass).
- [AWS Bedrock — InvokeModel + Claude tool use](https://docs.aws.amazon.com/bedrock/latest/userguide/tool-use-inference-call.html) — retrieved via `/web-research` per the Bedrock brief.

Last verified: 2026-05-26
