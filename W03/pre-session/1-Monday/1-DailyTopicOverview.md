---
template: pre-session-reading
week: W03
day: Mon
phase: Agentic
topic: "Agentic systems framing + HITL #3 interrupt-node boundaries ADR (Plan Day)"
estimated_total_minutes: 55
last_verified: 2026-05-26
fde_situations: [3, 4, 5, 7, 11]
tech: [LangGraph, LangChain v1.0, Bedrock InvokeModel, FAR 15.206, FAR 15.308]
sources_research_briefs:
  - research/langchain-v1-20260522.md
  - research/bedrock-claude-catalog-20260522.md
author: instructor
---

# Pre-session reading — W3 Mon · Agentic systems framing + HITL #3 ADR (Plan Day)

Week 3, Day Mon (Plan Day). Estimated total time on task: ~55 minutes. Last verified: 2026-05-26.

> **Plan Day + Gate-week kickoff.** W3 is the last week of Phase 1 — Mid-Programme Gate lands Fri (4 days from now). Three of the seven programme HITL touchpoints land this week: **#3 today (ADR), #4 Wed (soft interrupt), #5 Thu (hard interrupt + FAR audit row).** Distinct shapes. Don't pre-collapse them.

## 1. Why this matters (~80 words)

W2 ended with a working RAG layer over the FAR/DFARS corpus. W3 turns that retrieval into something the platform can *act on* — the Contracting Officer wants a multi-agent intake-triage flow shipped by Friday. 22 proposals landed in a 90-minute TEP-week window last cycle and manual triage missed two amendments. The CO's framing is the HITL anchor for the whole week: *"I — and only I — fire the irreversible actions."* Today is topology + boundaries before any code. Mon–Tue you build single-agent ReAct; Wed–Fri evolves to evaluator → consensus → SSA.

## 2. Stakeholder demand framing — the CO and the irreversibility principle (8 min)

Re-read the CO's framing from the W3 Mon war-room brief before 09:00. The single sentence — *"I — and only I — fire the irreversible actions"* — is load-bearing for the rest of the week. Translate it into a rule the platform can enforce: **HITL gates exist where actions are *irreversible*, not where actions are *risky*.** Routing a proposal to evaluator A vs evaluator B is reversible (re-route). Escalating a proposal to the CO's queue is irreversible (audit row exists; queue position is visible). Publishing an amendment to vendors is irreversible (vendor notifications fire). Awarding a contract is irreversible (FAR 5.705 publication).

The W1 Fri framing (HITL for AI Engineering) and W2 Thu framing (RAG fallback when retrieval underperforms) were the first two times the cohort saw HITL as a concept. Today it becomes a *named ADR* in the plan-spec — Decision, FAR citation, implementation primitive. Gating every reversible action creates alert fatigue and erodes the gate's value when an actual irreversible decision arrives.

## 3. Agents are state machines with HITL nodes (10 min)

A useful working definition: an **agent is a state machine** whose transitions are decided by an LLM (typically via tool calls), with persistent state across steps and explicit interrupt points where a human can resume the graph. LangGraph models this directly — nodes are functions, edges are conditional transitions, the graph state is checkpointed, and interrupts pause execution until human input arrives.

The W2 RAG layer becomes a *tool* an agent can call. The Spring services from W1 become *tools* an agent can call. The agent's job is to decide *which* tool to call, *with what arguments*, *in what order*. Three shapes you'll meet across the week:

- **Single-agent ReAct loop (Tue)** — one agent, Reason + Act + Observe in a loop. The intake-triage warm-up.
- **Supervisor-worker (Wed)** — one supervisor agent delegates to N worker agents and collects results. The evaluator → consensus flow.
- **Hard-interrupt LangGraph state machine (Thu)** — explicit interrupts at FAR-anchored gates, checkpointed across the 18-hour SSA gap.

The Anthropic *Building Effective Agents* essay frames the agent-vs-workflow choice: workflows are LLMs orchestrated against pre-defined patterns; agents dynamically direct their own processes. Reach for the workflow shape unless you need dynamic direction. Today's plan-spec decides which shape each transition is.

[!instructor-review] **Static `interrupt_before=[...]` vs dynamic `interrupt()` — canonical story for W3.** Both primitives are valid in LangGraph v1.x. The programme's canonical wiring Mon→Tue→Wed→Thu is **static `interrupt_before`** for the FAR-anchored gates — simpler, inspectable at compile time, matches what the war-room walkthroughs ship against, and what production codebases everywhere still use. The LangGraph docs (retrieved 2026-05-26) describe **dynamic `interrupt()`** as the docs-canonical alternative when the *pause condition itself* is runtime-decided (e.g., "interrupt only if refund > $500"). Both shapes get surfaced explicitly across the week — Mon ADR (today), Wed soft-interrupt deep-dive (file 5), Thu hard-interrupt deep-dive (file 6) — with consistent framing. Don't let a cohort member read the LangChain blog and conclude the curriculum is stale.

## 4. Memory patterns — thread vs cross-thread vs shared (8 min)

The plan-spec has to make a memory choice today. Three shapes the cohort will exercise this week:

- **Per-thread (single-agent state, Tue)** — `EvaluationState` TypedDict lives inside one LangGraph thread keyed by `evaluation_id`. State dies when the thread ends. `MemorySaver` is fine for dev.
- **Supervisor-curated (multi-agent, Wed)** — supervisor decides what each worker agent sees. Workers don't share scratchpads; they get a filtered view. Anti-pattern to avoid: shared scratchpad across all agents (writes balloon, traceability collapses).
- **Checkpointed cross-restart (Thu)** — `PostgresSaver` writes the full state to `checkpoints` + `checkpoint_blobs` + `checkpoint_writes` tables. Thread survives the 18-hour SSA gap. `thread_id` namespace = `f"{agency_id}:{evaluation_id}"` (Item 10 multi-tenant fix).

The plan-spec doesn't have to pick the production memory shape today — but it must name the choice and defer it explicitly. *"We use `MemorySaver` Tue–Wed; switch to `PostgresSaver` Thu when the hard-interrupt gates land"* is a defensible plan-spec entry. *"We'll figure out memory later"* is not.

## 5. Eval-driven agent development — every transition has a measurable signal (7 min)

The cohort met evaluations W1 Thu (prompt evals + lifecycle) and W2 Thu (RAGAS — faithfulness, context recall, context precision, answer relevance). W3 lifts the same principle to *agent* scope: **every agent transition should have an eval signal you can measure offline before turning it on in production.**

Concretely: when the triage agent calls `score_completeness(proposal_id)`, what signal tells you the score is right? When the agent calls `route_to_evaluators(...)`, what signal tells you the routing matched what a senior CO would have chosen? These signals become the offline eval set you replay every time the agent's prompt changes — so a prompt edit that improves the LLM-judge faithfulness on intake-triage doesn't silently regress on completeness scoring.

This is the foundation for W6's eval-report deliverable (Phase 2 Gate). Today's plan-spec doesn't have to *implement* the eval set — but each ADR should name the eval signal the decision implies. *"This soft-gate is right when human override rate < 15% on a 50-proposal replay"* is the shape.

## 6. Iterative spec-driven dev + the §0 retro discipline (5 min)

Today is the **second §0 retro** of the programme (per D-036). The W2 Mon Plan Day was the first; the discipline is *living*, not novel. Iterative spec-driven dev means: every Plan Day's first move is to retrospect the previous week's plan-spec against what actually happened, then carry the lessons forward.

Three things the §0 retro on W2 surfaces (specific to your pair — instructor will name them in Mon's gate-boundary 1:1):

1. **What did the W2 RAG plan underestimate?** (For most pairs: multi-tenant filtering on the retrieval boundary — Item 10. That lesson shapes the W3 agent-handoff multi-tenant work.)
2. **What did the W2 plan over-engineer?** (Coaching candidate: hybrid retrieval with reranker on Day 1 when basic kNN would have surfaced the problem.)
3. **Which ADR from W2 turned out to be load-bearing?** (Carry the framing into W3.)

The §0 retro runs **before** the three required ADRs land in the afternoon plan-spec block — lessons feed the new plan, not the other way around. Iterative spec-driven dev is *how you keep the plan honest*; W4 Wed's deep-dive on the technique is on top of this practiced foundation, not a fresh introduction.

## 7. HITL #3 — interrupt-node boundaries ADR (10 min) [HITL touchpoint #3]

**This is the Plan Day's anchor work.** For each transition in your pair's intake-triage topology, decide: **no gate / soft gate / hard gate**, with FAR citation where applicable. The ADR is the deliverable that lands in each pair-repo by 17:00. Three required components:

1. **Decision** — per transition: no gate, soft gate (suggestive default the human can approve/edit/reject), or hard gate (blocks until human input arrives — no auto-resume timeout).
2. **FAR citation OR reversibility argument** — name the regulation that anchors the gate (e.g., **FAR 15.206** for amendment publication; **FAR 5.705** for award publication). If no FAR clause applies, the justification is the reversibility argument from §2.
3. **Implementation primitive** — `interrupt_before=[node_name]` for hard gates (LangGraph's static-interrupt primitive; what the cohort wires Thu); a tool-layer approval token for soft gates that cross service boundaries.

**FAR 15.308 (SSA decision authority) is NOT today's ADR.** That clause anchors HITL #5 on Thu (`record_award` and `ssa_review_ssdd` nodes). The trap to avoid is importing tomorrow's gate into today's intake-triage topology. Today is intake-triage; FAR 15.308 fires when the evaluator → consensus → SSA flow lands Wed–Thu.

**Three distinct HITL shapes across the week — keep them named separately:**

| Touchpoint | Day | Shape | Anchor |
|------------|-----|-------|--------|
| **#3 (today)** | Mon | **ADR-level boundary design** — *which transitions get gates and why* | FAR 15.206 (amendment), reversibility argument |
| #4 | Wed | Soft-interrupt **implementation** — supervisor → worker handoff visibility | Multi-agent decision quality, no FAR clause |
| #5 | Thu | Hard-interrupt **implementation** + audit row + FAR citation field | FAR 15.308 (SSA non-delegation), FAR 5.705 (award publication) |

**A pair with one hard gate (`escalate_to_co`) + one soft gate (`route_to_evaluators`) is defensible.** A pair with six HITL gates is over-engineering — coach down. A pair with zero gates hasn't heard the CO's framing — re-read §2.

---

## Plan Day shape (today — what the day looks like)

| Time | Block |
|------|-------|
| 09:00–10:00 | **Gate-boundary 1:1s** — 20 min per pair with instructor. Coaching + role-rotation honesty check + gate-readiness signal. First of three (W3 Mon, W5 Mon, W6 Mon). Not a graded check — calibration before gate week starts. |
| 10:00–12:00 | **War-room** (`war-room/D1.md`) — stakeholder demand framing, whiteboard topology, draft HITL #3 ADR. |
| 13:00–17:00 | **Plan-spec authoring.** Order: (1) §0 retro on W2 plan-spec (per D-036), (2) §1–§7 of `templates/week-plan-spec.md`, (3) three required ADRs — single-vs-multi-agent · LangGraph as framework · HITL #3 interrupt-node boundaries. |
| 17:00 | **Plan-spec integrity check.** Instructor signs off each pair's plan before Tuesday morning. If §3 ADRs are hedges instead of decisions, the pair revises before Tuesday. |

**Tomorrow morning:** Checkpoint 1 exam in-person 09:00–10:30 (both tiers, 90 min). War-room runs 10:30–12:00 (half-day shape). LangSmith goes live Tue per D-031 — wire `LANGSMITH_API_KEY` + `LANGSMITH_TRACING=true` in `ai-orchestrator/.env` before the war-room block opens.

## Two questions to come in with tomorrow morning

1. Which of your pair-project's existing endpoints become *tools* an agent can call this week, and which need a human-gate wrapper before they're agent-callable? (Tomorrow's tool-calling discipline beat builds on this.)
2. For each HITL gate you commit in today's ADR — soft or hard, and what FAR citation OR reversibility argument justifies that choice? (Tomorrow's idempotency-for-state-mutating-tools topic depends on a clean answer here.)

## Glossary refresh (terms you'll hear today)

- **Agent** — a state machine whose transitions are LLM-decided (typically via tool calls), with persistent state across steps.
- **Tool** — a callable function the agent can invoke (Bedrock InvokeModel, a Spring endpoint, the W2 RAG layer, a database query). Tools have schemas — what they accept, what they return.
- **Interrupt** — a LangGraph mechanism that pauses the graph and waits for human input. **Static** = `interrupt_before=[...]` at compile time; **dynamic** = `interrupt(...)` called inside a node. Soft = suggestive; hard = blocking.
- **Checkpoint** — the persisted graph state. In dev: `MemorySaver`. In prod-shape: `PostgresSaver`. Lets you resume an agent run that paused yesterday.
- **ReAct loop** — *Reason + Act + Observe* pattern; the agent reasons, calls a tool, observes the result, reasons again. Tue's single-agent baseline.
- **Supervisor-worker** — multi-agent pattern where one agent delegates to worker agents and collects results. Wed's anchor.
- **HITL** — Human-in-the-Loop. Programme thread: 7 touchpoints total; **3 of 7 land this week** (#3 Mon ADR, #4 Wed soft interrupt, #5 Thu hard interrupt + FAR audit).
- **§0 retro** — first section of every Plan Day's plan-spec from W2 onward (per D-036). Walks the previous week's plan against what actually happened.
- **Irreversibility principle** — HITL gates exist where actions are *irreversible*, not where actions are *risky*. The W1 Fri framing the cohort revisits today.

## Optional deep-dive (not required to participate)

- [LangGraph — Persistence + time travel](https://docs.langchain.com/oss/python/langgraph/persistence) (~10 min read), retrieved 2026-05-26 via /web-research. Useful framing for Thu's `MemorySaver` vs `PostgresSaver` deep-dive — see the checkpoint state-history API (`graph.get_state_history(thread_id)`) which becomes the OIG audit-replay surface Thu.
- [Anthropic — Building Effective Agents (full essay)](https://www.anthropic.com/research/building-effective-agents) (~15 min read), published 2024-12-19, retrieved 2026-05-26 via /web-research. *Skip if you read it for W1 Thu.* The agent-vs-workflow framing the §3 topic above leans on.

---

## Sources

All citations retrieved 2026-05-26 via /web-research per `pipeline/RESEARCH-PROTOCOL.md`. Recency windows applied: hot-tech 3-month (LangGraph + LangChain v1.0), federal-regulatory 6-month (FAR 15.206 + 15.308 — FAC 2026-01 effective 2026-03-13 confirmed), foundation-stable 12-month (state-machine / ReAct framing).

- LangChain v1.0 + LangGraph framing pulls from cached research brief `research/langchain-v1-20260522.md` (verified 2026-05-22, well inside 3-month hot-tech window).
- Bedrock InvokeModel posture pulls from `research/bedrock-claude-catalog-20260522.md` (verified 2026-05-22, inside window).
- FAR 15.206 + FAR 15.308 confirmed at FAC 2026-01 (effective 2026-03-13) via acquisition.gov retrieval 2026-05-26.
- LangGraph interrupt-pattern current-state confirmation: `docs.langchain.com/oss/python/langgraph/interrupts` retrieved 2026-05-26. The static-vs-dynamic-interrupt nuance is surfaced as `[!instructor-review]` in §3.

Last verified: 2026-05-26
