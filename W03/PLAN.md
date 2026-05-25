---
week: W03
title: "Agentic Systems — single agent → multi-agent → LangGraph HITL"
phase: "Phase 1 — AI Adoption into Brownfield (concludes Fri W3 Mid-Programme Gate)"
gate: "Phase 1 Presentation & Defense + Mid-Program Retro — Fri 12 Jun 2026"
density_target: "5–8 topics/day (W3 trends toward 8 due to multi-agent + KG/CG density)"
hitl_touchpoints_landing_this_week: ["#3 Mon Plan Day ADR (interrupt-node boundaries)", "#4 Wed multi-agent handoffs", "#5 Thu LangGraph deep-dive"]
phase_1_anchor_features:
  - "POST /agent/intake-triage (Mon-Tue warm-up flow — single→multi-agent)"
  - "Evaluator → consensus → SSA-review handoff (Wed-Fri — HITL #5 anchor)"
  - "POST /draft-amendment + FAR 15.206 CO approval gate"
  - "POST /eval/ssdd-draft + FAR 15.308 SSA decision authority (cannot delegate)"
  - "17-entity KG: Vendor ↔ Proposal ↔ Evaluation ↔ Award ↔ ContractModification ↔ Cpar"
  - "Debt item 6 — correlation IDs across agent handoffs"
d_060_resolution: "Both flows run sequentially this week (Mon-Tue intake-triage, Wed-Fri evaluator→consensus→SSA). Bedrock direct InvokeModel only — Agents-for-Bedrock deferred to W5 per D-050."
last_verified: 2026-05-23
---

# W03 PLAN — Mon–Fri at a glance

> Master plan for Week 3 per `RevaturePro_FDE-Karsun_v2.pdf` + D-060 (both multi-agent flows sequentially). W3 is the **last week of Phase 1** — the Mid-Programme Gate lands Fri 12 Jun 2026 at end of day. Phase 1 Defense rubric is cumulative across W1–W3 (LLM essentials + RAG + agentic). This week is also the densest HITL week of the programme: **three of the seven programme HITL touchpoints land here** (#3 Mon Plan Day, #4 Wed multi-agent handoffs, #5 Thu LangGraph deep-dive). Topic density runs 7–8/day; pre-session readings are scoped tight to honour the cap.

## Mon — Plan Day · Agentic Systems Framing + §0 Retro on W2 · **HITL #3 lands here** *(7 topics)*

*First Plan Day of W3 + first **gate-boundary 1:1** (Mon 09:00–10:00, 20 min × 3 pairs per `PIPELINE.md` §4 — coaching + gate readiness check). §0 retro on W2 plan-spec is mandatory (per D-036) — second time the cohort retros their own plan.*

**Morning (gate-boundary 1:1s then war-room — `war-room/Mon.md`)** — 09:00–10:00 Trainer 1:1s (3 × 20 min). 10:00 war-room shifts to Plan Day mode: stakeholder demand framing — the CO needs a multi-agent flow that triages incoming proposals during TEP-week peak load *without* losing audit rows on agent handoffs (Item 2 surface). Cohort sketches the agent topology on the whiteboard before committing to an ADR.

**Afternoon (practical — plan-spec authoring)** — Each pair authors a **week-plan-spec** (`templates/week-plan-spec.md`) covering Tue–Fri. **Three required ADRs:** (a) single-agent vs multi-agent split for `POST /agent/intake-triage`; (b) LangGraph as the state-machine framework (vs CrewAI / hand-rolled — defended against the evaluator→consensus→SSA flow shape); (c) **HITL #3 named ADR — interrupt-node boundaries** (which agent transitions require human gates, with FAR citation: 15.206 for amendments, 15.308 for SSA decision authority). §0 retro on W2 plan-spec runs **before** these three ADRs land — the lessons from W2 RAG planning shape the W3 agentic plan.

**Conceptual (`pre-session/Mon.md`)** — Agent Architecture Patterns + Memory Patterns + Eval-Driven Agent Development principle. Pre-reads Tue's ReAct + tool-calling work.

## Tue — ReAct Workflows in LangGraph + Single-Agent Intake-Triage *(8 topics, at cap)*

*Build day. The intake-triage flow goes from blank-FastAPI-endpoint to working single-agent ReAct loop calling Bedrock + the solicitation-service. First real LangGraph code lands in the repo today. LangSmith tracing introduced (per D-031 — first real appearance).*

**Morning (war-room — `war-room/Tue.md`)** — Incident mode: vendor uploads a proposal to a solicitation that was amended yesterday; the proposal references the *pre-amendment* SOW. Cohort builds the intake-triage agent that detects the staleness + routes back to vendor with the amendment delta. Today is **tool-calling discipline day** — when is a Bedrock invocation a *tool call* vs a *direct prompt*?

**Afternoon (practical, dense)** — Implement `POST /agent/intake-triage` as a single-agent ReAct loop in `ai-orchestrator`: tools are `get_solicitation`, `get_amendments`, `score_completeness`, `route_to_evaluators`, `escalate_to_co`. **Idempotency for state-mutating tools** (every `route_to_evaluators` call must be idempotent — same proposal_id + evaluator_id = same outcome; LangGraph thread checkpoint is the persistence point). **In-context memory + compaction** for long proposals. **LangSmith tracing** wired into the dev environment (real LangSmith, not stubs, per D-031). Agentic RAG hand-off: when triage needs clause context, it invokes `POST /rag/clause-search` from W2 — first time the W2 RAG layer is consumed by an *agent*, not a human.

**Conceptual (`pre-session/Tue.md`)** — Self-querying retrieval + multi-query retrieval + RAG Fusion / CRAG framing for Wed's multi-agent introduction.

## Wed — Multi-Agent Design Patterns + KG/CG Thinking + Scenario Research · **HITL #4 lands here** *(8 topics, at cap; dedicated scenario-research slot per D-040)*

*Pivot day. Mon-Tue's single-agent intake-triage handoffs into the harder Wed-Fri flow: **evaluator-agent → consensus-agent → SSA-review-agent**. Multi-agent anti-patterns named explicitly (chatty handoffs, audit fan-out, lost context). Knowledge Graph + Context Graph thinking is the substantial mid-W3 placement per D-030. **Dedicated scenario-research slot** runs in the practical block.*

**Morning (war-room — `war-room/Wed.md`)** — Supervisor-worker pattern + hierarchical orchestration on the evaluator→consensus→SSA flow. Multi-agent anti-patterns (chatty agents, infinite handoff loops, audit fan-out where every agent writes its own AuditEvent and Item 2's race becomes catastrophic). **HITL between multi-agent handoffs (4th of 7 touchpoints)** — when does the supervisor agent need a human gate before delegating to the next worker? Anchor case: consensus-agent → SSA-review-agent transition (FAR 15.308 says the SSA decision authority cannot delegate — so the transition into the SSA-review-agent is a *human approval gate*, not a programmatic handoff).

**Afternoon (dedicated scenario-research slot per D-040, 2-3 hr)** — Cohort splits into pairs and researches the 3 scenario-alternatives prompts for W3 (see `scenarios/`). **Parallel conceptual block — Knowledge Graph + Context Graph thinking on the 17-entity acquire-gov schema:** Vendor ↔ Proposal ↔ Evaluation ↔ Award ↔ ContractModification ↔ Cpar. CO query example: *"show me every contract this vendor has won with red CPARs"* — joins Vendor → Award → Contract → Cpar with rating filter. Cohort sketches the KG retrieval on the whiteboard; instructor frames Neo4j vs Postgres recursive CTE vs NetworkX in-process as Wed's third scenario.

**Conceptual (`pre-session/Wed.md`)** — Advanced memory patterns for multi-agent (shared scratchpad vs per-agent thread vs supervisor-curated context). Pre-reads Thu's state-machine work.

## Thu — State Machines + LangGraph HITL Deep-Dive · **HITL #5 lands here** *(8 topics, at cap — technical anchor day)*

*The technical anchor day. The full evaluator→consensus→SSA LangGraph implementation lands today. LangGraph `interrupt_before` is exercised at every HITL gate. Soft vs hard interrupts named explicitly. Checkpointing + persistence required — the SSA may not approve the SSDD draft until the next morning, so the graph state has to survive the gap.*

**Morning (war-room — `war-room/Thu.md`)** — State schema design for the evaluator→consensus→SSA flow. Cohort defines the `EvaluationState` TypedDict (proposal_ids, evaluator_scores, consensus_complete, ssdd_draft, ssa_approval_status, audit_correlation_id). **Parallel execution in LangGraph** — multiple evaluator-agents scoring different proposals in parallel before fan-in to consensus. **Checkpointing + persistence** (`MemorySaver` for dev, `PostgresSaver` for the production-shape; threads keyed by `evaluation_id`).

**Afternoon (practical, dense — HITL #5 deep-dive)** — Implement the LangGraph state machine: nodes for `evaluator_score_proposal`, `consensus_aggregate`, `ssa_review_ssdd`, `record_award`. **`interrupt_before=["ssa_review_ssdd", "record_award"]`** — the two hard-gate transitions (FAR 15.308 SSA approval cannot delegate; FAR 5.705 award publication is irreversible). **Soft vs hard interrupts:** soft interrupts (HITL #4 supervisor approval) allow the graph to *suggest* a handoff; hard interrupts (HITL #5 SSA approval) *block* until human input is captured. **HITL audit trail** — every interrupt resume writes a row to `AuditEvent` with `actor_id = human reviewer`, `before/after` state snapshots, threaded `correlation_id` (Item 6 fix surfaces here). Workflow resiliency: what happens when Bedrock 429s mid-graph? Checkpointing means resume-from-last-good-node, not restart. Debugging LangGraph via LangSmith traces (W3 first real consumption per D-031). Cost + latency budget: each evaluator-agent invocation is a Bedrock call; cohort instruments token-per-node + latency-per-node.

**Conceptual (`pre-session/Thu.md`)** — Phase 1 Defense rubric preview + framing notes. Cohort comes Friday with their cumulative W1-W3 defense narrative drafted.

## Fri — **Phase 1 Presentation & Defense + Mid-Program Retro** *(GATE DAY — no standard war-room)*

*Mid-Programme Gate. Phase 1 (W1 Thu – W3 Fri) concludes. Pairs demo + defend their pair-project as built across Phase 1. Defense is **cumulative**: W1 LLM essentials + W2 RAG + W3 agentic patterns. Per D-039, pairs defend both **what was adopted** AND **what was discovered** — the Phase-1 discoveries become the input to W4 Mon's §0 retro that drives the Phase 2 modernization plan.*

**Morning (Phase 1 Defense, 09:00–12:00)** — Each pair gets a **45-minute slot** (15-min demo + 10-min Agency CIO Q&A + 10-min OIG Q&A + 10-min per-individual architecture defense for gate rigor, calibrated to tier per CLAUDE.md `entry-tier-applied-recognition`). Instructor + one external observer (Karsun delivery lead or eng lead) run the rubric (`assessments/W03-Phase1-Defense-rubric.md`).

**Afternoon (Mid-Program Retro, 13:00–15:00)** — Cohort + instructor walk Phase 1 end-to-end. Three retro artifacts authored: (a) per-pair retro (`retros/W03-pair-N-retro.md`, 3 files), (b) cohort-wide retro (`retros/W03-cohort-retro.md`) naming the modernization discoveries that drive W4 Mon §0 retro. **Phase 1 plan-spec freeze tag** lands on each pair's repo (`phase1-freeze-pair-N-20260612` per D-056). HITL #6 (W4 Wed) is pre-framed in the cohort retro so cohort goes into the weekend with the security framing teed up.

**EOD Fri (15:00–16:30)** — Light weekly MCQ (rotation per `PIPELINE.md` §13 — W3 is MCQ + Live Defense + **Gate**); Phase 1 cumulative score recorded; W4 Mon Plan Day briefing distributed (the **acquire-gov shared brownfield** comes back into scope for W4 modernization, alongside each pair's project).

## Special notes for W3

- **Both multi-agent flows run sequentially this week (D-060 resolution).** Mon-Tue = intake-triage warm-up (lighter); Wed-Fri = evaluator→consensus→SSA + HITL #5 LangGraph deep-dive (harder). Cohort gets both shapes — not one or the other.
- **Bedrock direct InvokeModel only.** No Agents-for-Bedrock, no Bedrock Knowledge Bases this week — those are W5 (per D-050 + D-060 exception scope). The cohort wires LangGraph + LangChain v1.0 + plain `boto3.client("bedrock-runtime").invoke_model()`.
- **LangSmith tracing is real this week.** Per D-031, W3 Tue is the first programme appearance — instructor's LangSmith workspace is wired into `ai-orchestrator/.env` before Tue morning (cohort-instantiation checklist item).
- **Three HITL touchpoints this week — that's nearly half the programme's HITL thread (3 of 7).** Pre-session readings name them by number consistently; instructor reinforces the count out loud Mon morning so cohort tracks the thread.
- **Density risk Wed + Thu:** both days run at 8 topics (at cap). If Wed afternoon scenario research overruns, KG/CG conceptual block defers to Thu pre-session (acceptable carry; KG/CG is still inside W3). If Thu LangGraph deep-dive overruns, the cost + latency instrumentation block defers to Fri morning's first 60 minutes (last opportunity inside Phase 1).
- **§0 plan-retro on W2 Mon spec is mandatory.** Second time the cohort runs the retro; the W2 RAG plan-spec lessons are concrete inputs to the W3 agentic plan (e.g., if the W2 plan underestimated multi-tenant filtering work — Item 10 — that lesson shapes the W3 agent-handoff multi-tenant boundary work).
- **Phase 1 Defense is tier-aware.** Senior FDE candidates defend production-shape decisions (cost budgets, on-call shape, FedRAMP boundary, observability completeness); Entry FDE candidates defend applied-recognition (per CLAUDE.md memory `entry-tier-applied-recognition.md`) — they show they can *recognize and apply* the patterns, not necessarily defend them at production scale.
- **No new tech introduced after Wed.** Thu is implementation + HITL deep-dive against tech the cohort has already met (LangGraph from Tue, KG/CG from Wed). This is deliberate — gate week shouldn't be cramming new concepts on Day 4.

## Assets in this folder

- `PLAN.md` — this file.
- `pre-session/Mon.md` through `Fri.md` — pre-session reading material (one per day).
- `war-room/Mon.md` through `Thu.md` — morning war-room scenario. (`Fri.md` is the Phase 1 Defense + Mid-Program Retro, authored under `assessments/` + `retros/`.)
- `scenarios/W03-SA-1.md` through `W03-SA-3.md` — scenario-alternatives prompts released Wed morning, ADR due EOD Thu W4 (after the gate), Live Defense scheduled inside W4.
- `assessments/W03-Phase1-Defense-rubric.md` — Phase 1 Gate defense rubric (cumulative W1–W3 scope, tier-aware).
- `assessments/W03-MCQ.md` — weekly MCQ (mid-tier, post-defense).
- `codex-reviews/` — placeholder for Codex Adversarial Review outputs on W3 PRs (Near-full strictness per D-034).
- `retros/W03-pair-N-retro.md` × 3 + `W03-cohort-retro.md` — Mid-Program Retros authored Fri afternoon.
