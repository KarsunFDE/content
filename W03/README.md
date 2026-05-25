# W03 · Agentic Systems

*Phase:* Phase 1 — AI Adoption into Brownfield (**concludes Fri W3 12 Jun 2026 at Mid-Programme Gate**)
*Gate:* **Phase 1 Presentation & Defense + Mid-Program Retro** (W3 Fri)
*Project phase:* Pair Project Phase 1 → Phase 1 Gate (cumulative W1–W3 defense)
*Theme:* Single agent → multi-agent → LangGraph HITL + Knowledge/Causal Graphs
*Weekly assessments:* MCQ (mid-tier, Fri post-defense) + **Phase 1 Defense Rubric** (Fri AM) + Codex on PRs (Near-full per D-034)
*HITL touchpoints landing this week:* **#3** Mon Plan Day ADR (interrupt-node boundaries) · **#4** Wed multi-agent handoffs · **#5** Thu LangGraph deep-dive

## D-060 hard answers shaping W3

- **Both multi-agent flows run sequentially.** Mon-Tue intake-triage (lighter warm-up); Wed-Fri evaluator → consensus → SSA-review (HITL #5 LangGraph deep-dive anchor). Cohort meets both shapes — not one or the other.
- **Bedrock direct InvokeModel only.** No Agents-for-Bedrock, no Bedrock Knowledge Bases (those land W5 per D-050 + D-060 exception scope).
- **LangSmith is real this week.** First programme appearance per D-031 — wired into `ai-orchestrator/.env` Tue morning.

## v2 day-by-day (per `RevaturePro_FDE-Karsun_v2.pdf` + D-060 + this PLAN.md)

| Day | Headline | Anchor topics (5–8/day cap) |
|-----|----------|------------------------------|
| **Mon Plan Day** | Agentic Systems Framing + §0 Retro on W2 + **HITL #3** | Gate-boundary 1:1s (09:00–10:00); Stakeholder demand framing; Agent Architecture Patterns; AI backlog generation; Memory Patterns; Eval-Driven Agent Development principle; **HITL interrupt-node boundaries committed as ADR** (FAR 15.206 amendments + FAR 15.308 SSA decision authority) |
| Tue | ReAct in LangGraph + Single-Agent Intake-Triage | Tool Calling; Idempotency for state-mutating tools; In-context memory & compaction; **LangSmith tracing** (per D-031, first real appearance); Agentic RAG; Self-querying retrieval; RAG Fusion / CRAG; Single-agent `POST /agent/intake-triage` implementation |
| Wed | Multi-Agent Patterns + KG/CG + Scenario Research + **HITL #4** | Supervisor-worker pattern; Hierarchical orchestration; Parallel fan-out; Multi-agent anti-patterns; **HITL between multi-agent handoffs** (consensus→SSA gate); Advanced memory patterns for multi-agent; **Knowledge Graph thinking** (D-030); **Context Graph thinking**; scaffolded orchestrator with KG+CG; dedicated scenario-research slot (per D-040) |
| Thu | State Machines + LangGraph HITL Deep-Dive + **HITL #5** | State schema design (`EvaluationState` TypedDict); Parallel execution in LangGraph; Checkpointing & persistence (`MemorySaver` dev / `PostgresSaver` prod-shape); **HITL with `interrupt_before` nodes** (technical anchor day); Soft vs hard interrupts; HITL audit trail (Item 6 correlation-id fix surfaces here); Workflow resiliency; Debugging LangGraph via LangSmith; Cost & latency management |
| **Fri Gate Day** | **Phase 1 Presentation & Defense + Mid-Program Retro** | 09:00–12:00 paired Demo & Defense (45 min/pair: 15-min demo + 10-min Agency CIO Q&A + 10-min OIG Q&A + 10-min per-individual architecture defense, tier-aware per CLAUDE.md `entry-tier-applied-recognition`). 13:00–15:00 Mid-Program Retro feeds W4 Mon §0 plan retrospective. EOD: weekly MCQ + Phase 1 freeze tag + W4 Mon briefing distribution. |

## Phase 1 anchor features touched this week (per `training-project/week-dependency-map.md` §W3)

- **`POST /agent/intake-triage`** — single-agent ReAct loop Tue → migrated to multi-agent shape on the evaluator→consensus→SSA flow Wed-Fri (D-060).
- **Evaluator → Consensus → SSA-review handoff** — LangGraph state machine with `interrupt_before` at `ssa_review_ssdd` and `record_award` (FAR 15.308 + FAR 5.705).
- **`POST /draft-amendment`** — CO approval gate before publish (FAR 15.206 amendment transition). Mon ADR; Wed multi-agent handoff implementation.
- **`POST /eval/ssdd-draft`** — SSA reviews + signs SSDD tradeoff narrative (FAR 15.308 SSA decision authority **cannot delegate**). Thu LangGraph hard-interrupt anchor.
- **17-entity KG: Vendor ↔ Proposal ↔ Evaluation ↔ Award ↔ ContractModification ↔ Cpar** — Wed afternoon KG/CG thinking exercise.
- **Debt Item 6 (correlation IDs)** — surfaces as a *load-bearing* constraint when agent handoffs become cross-service write boundaries. Thu HITL audit-trail work names the fix (deferred to W5 W3C `traceparent` rollout).

## How this folder gets filled

When the instructor instantiates a cohort, this folder is the destination for all
week-W03 content artifacts. Each subfolder corresponds to an artifact type from
`pipeline/PIPELINE.md` §5.

| Subfolder | Artifact type | Skill that authors it |
|-----------|---------------|------------------------|
| `pre-session/Mon..Fri.md` | Pre-session reading (one per day, capped at 5–8 topics/day for W3 per CLAUDE.md) | `pre-session-author` |
| `war-room/Mon..Thu.md` | Morning war-room scenario (Fri = Phase 1 Defense, not a standard war-room) | `war-room-scenario` |
| `scenarios/W03-SA-#.md` | Alternative-tech scenario (2–3 per week per D-040) | `scenario-alternatives` |
| `assessments/W03-Phase1-Defense-rubric.md` | **Phase 1 Gate** defense rubric (cumulative W1–W3, tier-aware) | `live-defense-rubric` (Gate variant) |
| `assessments/W03-MCQ.md` | Mid-tier MCQ for W3, run post-defense Fri | `mcq-generator` |
| `codex-reviews/` | Codex Adversarial Review on Karsun PRs (Near-full per D-034) | `codex-adversarial-review` |
| `retros/W03-pair-N-retro.md` + `W03-cohort-retro.md` | Mid-Program Retros (3 pair + 1 cohort), feeds W4 Mon §0 retro | `weekly-retro` |
| `PLAN.md` | One-page week plan (Mon–Fri grid) | Instructor (hand-authored, reviewed weekly) |

## Special notes for Week 3

- **Last week of Phase 1.** The Mid-Programme Gate concludes the AI Adoption phase. W4 Mon's §0 retro names the modernization needs Phase 1 surfaced — the cohort goes into the weekend already knowing the Phase 2 framing.
- **Three HITL touchpoints land this week — nearly half the programme's HITL thread.** Pre-session readings name them by number consistently (#3 Mon, #4 Wed, #5 Thu).
- **§0 retro on W2 plan-spec is mandatory** (per D-036). Second time the cohort runs the iterative spec-driven retro; the discipline is now expected, not novel.
- **Codex Adversarial Review hits Near-full strictness** (W3 per D-034 calibration ramp). P0 + most P1 findings block; only P2 architectural-drift framed as coaching.
- **First gate-boundary 1:1s** (Mon 09:00–10:00, 20 min × 3 pairs per `PIPELINE.md` §4). Coaching + role-rotation honesty check + gate-readiness signal.
- **Phase 1 Defense is tier-aware** per CLAUDE.md memory `entry-tier-applied-recognition.md`. Senior FDE candidates defend production-shape decisions; Entry FDE candidates defend applied-recognition of the same patterns.
- **No new tech after Wed.** Thu is deepening (LangGraph HITL) against tech already met. Gate-week discipline.
- **Density:** Tue/Wed/Thu run at 8 topics (W3 cap per CLAUDE.md — AIOps + Capstone weeks 5–8; W3 trends to 8 due to multi-agent + KG/CG density). Pre-session readings scoped tight.
