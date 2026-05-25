---
template: pre-session-reading
week: W03
day: Mon
phase: Agentic
topic: "Agentic systems framing + HITL interrupt-node boundaries (HITL #3)"
estimated_total_minutes: 50
last_verified: 2026-05-23
fde_situations: [3, 4, 5, 7, 11]
tech: [LangGraph, LangChain v1.0, Bedrock InvokeModel, FAR 15.206, FAR 15.308]
sources_research_briefs: [research/langgraph-1.0-state-machine.md, research/bedrock-direct-invokemodel.md]
author: instructor
---

# Pre-session reading — Agentic systems framing + HITL interrupt-node boundaries

Week 3, Day Mon (Plan Day). Estimated total time on task: ~50 minutes. Last verified: 2026-05-23.

## 1. Why this matters (this is a Plan Day + a Gate-week kickoff)

W3 is the last week of Phase 1. Friday is the Mid-Programme Gate — the cohort defends *everything* it built since W1 Thu: LLM essentials, RAG, and now agentic patterns. Monday is the Plan Day that frames the whole week, and the §0 retro on the W2 RAG plan is the second time you've retrospected your own planning (per D-036). This week also lands three of the seven programme HITL touchpoints — **#3 today**, #4 Wed, #5 Thu. By Friday you need to defend not just *that* you built an agent, but *which agent transitions you put a human gate on, and why*.

## 2. Core concept in 5 minutes — agents are state machines with HITL nodes

A useful working definition: an agent is a **state machine** whose transitions are decided by an LLM (often via tool calls), with persistent state across steps and explicit interrupt points where a human can resume the graph. That's it. The rest is implementation detail.

LangGraph (the framework you'll use this week) models this directly: nodes are functions, edges are conditional transitions, and `interrupt_before=[...]` lets you pause the graph at a node and wait for human input before continuing. The graph state is checkpointed — you can pause at the consensus-complete node Wednesday afternoon and resume Thursday morning when the SSA shows up to approve.

The W2 RAG layer you built last week becomes a *tool* an agent can call. The Spring services from W1 become *tools* an agent can call. Today's plan-spec decides which of those agent transitions need a human gate before they fire. That's HITL #3.

Reference the W2 plan-spec retro pattern: this Plan Day's §0 retro is your second one (W2 Mon was the first). You're getting practice; the discipline matters for the Friday gate.

## 3. What to read or watch tonight

Each link below is dated, primary-source where possible, and time-estimated.

- [LangGraph — Human-in-the-loop overview](https://langchain-ai.github.io/langgraph/concepts/human_in_the_loop/) (~12 min read), published 2025-12-15 (verified 2026-05-23 via /web-research). Focus on the four HITL patterns: approve/reject, edit graph state, review tool calls, and validate human input. We'll exercise all four this week.
- [FAR 15.206 — Amending the solicitation](https://www.acquisition.gov/far/15.206) (~8 min read), retrieved 2026-05-23 via /web-research. Focus on §15.206(b) — *"When, either before or after receipt of proposals, the Government changes its requirements or terms and conditions, the contracting officer shall amend the solicitation."* This is the rule that makes the `POST /draft-amendment` flow a *CO-must-approve-before-publish* gate, not an auto-publish flow.
- [FAR 15.308 — Source selection decision](https://www.acquisition.gov/far/15.308) (~6 min read), retrieved 2026-05-23 via /web-research. Focus on the rule that the **SSA decision authority cannot be delegated**. This is why the `record_award` LangGraph node is a *hard* interrupt, not a soft one.
- *(optional)* [Anthropic — Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) (~15 min read), published 2024-12-19 (verified 2026-05-23 via /web-research). *Skip if you've already read it.* The single best framing on when to reach for an agent vs a workflow — useful background for today's ADR work.

## 4. Two questions to come in with tomorrow morning

1. Which of your pair-project's existing endpoints become *tools* an agent can call this week, and which need a human-gate wrapper before they're agent-callable?
2. For each HITL gate you commit in today's ADR — is it a **soft** interrupt (suggest a handoff, allow override) or a **hard** interrupt (block until human input)? What FAR/regulatory citation justifies that choice?

## 5. Glossary refresh (terms you'll hear today)

- **Agent** — a state machine whose transitions are LLM-decided (typically via tool calls), with persistent state across steps.
- **Tool** — a callable function the agent can invoke (Bedrock, a Spring endpoint, the W2 RAG layer, a database query). Tools have schemas — what they accept, what they return.
- **Interrupt node** — a LangGraph node that pauses the graph and waits for human input before continuing. Soft = suggestive; hard = blocking.
- **Checkpoint** — the persisted graph state. In dev: `MemorySaver`. In production-shape: `PostgresSaver`. Lets you resume an agent run that paused yesterday.
- **ReAct loop** — *Reason + Act* pattern: the agent reasons about what to do, calls a tool, observes the result, reasons again, calls another tool, until done. This week's single-agent baseline.
- **Supervisor-worker** — multi-agent pattern where one agent (supervisor) delegates to worker agents and collects results. Wed's anchor.
- **HITL** — Human-in-the-Loop. The cohort has now seen this three times before today (W1 Fri framing, W2 Thu RAG fallback, last week's plan-spec). Today it becomes a *named ADR* in your plan-spec.
- **§0 retro** — first section of every Plan Day's plan-spec from W2 onward (per D-036). Walks last week's plan against what actually happened.

## 6. Plan Day shape (today)

- **09:00–10:00 — Gate-boundary 1:1s.** 20 min per pair with the instructor. Coaching + role-rotation honesty check + gate-readiness signal. Not a graded check — a calibration moment before the gate week starts.
- **10:00–12:00 — War-room (`war-room/Mon.md`).** Stakeholder demand framing + agent-topology whiteboarding for the intake-triage flow.
- **13:00–17:00 — Plan-spec authoring.** §0 retro on W2 plan + three required ADRs (single vs multi-agent, LangGraph as framework, HITL #3 interrupt-node boundaries). Same `templates/week-plan-spec.md` shape as W2.
- **17:00 — Plan-spec integrity check.** Instructor signs off each pair's plan before Tuesday morning. If §3 ADRs are hedges instead of decisions, the pair revises before Tuesday.

## 7. Optional deep-dive (not required to participate)

- [LangGraph — Time travel and checkpointing](https://langchain-ai.github.io/langgraph/concepts/persistence/) (~10 min read), published 2025-11-08 (verified 2026-05-23 via /web-research). Useful framing for Thu's deep-dive on `MemorySaver` vs `PostgresSaver`.
