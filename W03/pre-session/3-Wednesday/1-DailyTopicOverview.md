---
template: pre-session-reading
week: W03
day: Wed
phase: Agentic
topic: "Multi-agent design patterns + HITL #4 + KG/CG + scaffold orchestrator"
estimated_total_minutes: 67
last_verified: 2026-05-26
fde_situations: [3, 4, 5, 7, 11]
tech: [LangGraph, LangChain v1.0, Neo4j, Postgres recursive CTE, NetworkX, LangSmith, Bedrock]
sources_research_briefs:
  - research/langchain-v1-20260522.md
  - research/bedrock-claude-catalog-20260522.md
author: instructor
---

# Pre-session reading — Multi-agent design patterns + HITL #4 + KG/CG

Week 3, Day Wed. Estimated total time on task: ~67 minutes. Last verified: 2026-05-26.

> **Wed is the pivot day.** Mon–Tue's single-agent intake-triage handoffs into the Wed–Fri evaluator → consensus → SSA-review flow (D-060). Three of seven programme HITL touchpoints land this week; **#4 lands today** as a *soft* interrupt at the supervisor → worker handoff — different shape from Mon's #3 ADR and different from Thu's #5 hard interrupt + FAR audit. Wednesday afternoon is the **dedicated scenario-research slot** (D-040). Below are the seven topics that frame the day, then the day-shape callout, ship list, and optional deep-dive.

## 1. Why this matters

Three concurrent evaluations kick off today (RFP-2026-GSA-1184 cloud / 1199 cyber / 1204 mobile). Six evaluators each. The SSA is asking for something new: **visibility into the supervisor's decisions before consensus completes** — approve or override what's *about to* be delegated, not wait until the SSDD review. That's HITL #4: a *soft* interrupt at the supervisor → worker handoff, distinct from Thu's hard interrupt at the SSDD/award boundary.

Today's load surfaces two brownfield-debt items in real time. **Item 2 (audit-log race)** — ~72 audit rows fanning into one Postgres write if every agent writes its own AuditEvent — becomes the *Multi-Agent Anti-Patterns* topic, not an abstraction. **Item 6 (correlation IDs)** — without a threaded `audit_correlation_id` per `evaluation_id`, the multi-agent trace is unreconstructable. Both combine into the W4 Mid-Sprint Surprise; you feel the precursor today. Wednesday afternoon's dedicated scenario-research slot (D-040) cashes in on three SA prompts released this morning — including SA-3, where the 17-entity acquire-gov schema becomes the **KG/CG** reference exercise.

## 2. Supervisor-worker — the default multi-agent shape (10 min)

The supervisor-worker pattern: one **supervisor agent** decides *which worker to call next* and *when the work is done*; the workers do narrow, well-defined tasks. For the Wed–Fri evaluation flow there are four roles:

- **Supervisor** — decides: score next proposal? Aggregate consensus? Hand off to SSA?
- **Evaluator-agent (worker × N)** — scores one proposal against Section M factors. Pure read + reason; no state-mutating side effects.
- **Consensus-agent (worker × 1)** — aggregates per-evaluator scores into a panel consensus + narrative.
- **SSA-review-agent (worker × 1, hard gate Thu)** — produces the SSDD tradeoff narrative for SSA review. **This agent doesn't fire without human approval at the entry** — FAR 15.308 says the SSA's independent judgment cannot be delegated. Today you scaffold it; Thursday you wire the hard gate.

The *delegation contract* is the load-bearing primitive: every supervisor → worker arrow says "what state does the worker read, what state does it write, and how does the supervisor decide when to fire it?" The contract is what the HITL #4 interrupt makes visible to the SSA.

LangGraph implementation: the supervisor is a node that calls Bedrock to decide `state['next_worker']`; the workers are nodes that read state, call tools, write back. The supervisor-as-tool-caller pattern (workers exposed to the supervisor *as if they were tools*) makes LangGraph's `ToolNode` a natural fit.

## 3. Hierarchical orchestration + parallel fan-out (8 min)

Two extensions of the supervisor pattern you need to recognize today (and pick one for the morning build):

- **Hierarchical orchestration** — supervisors of supervisors. Worker groups have their own internal coordination logic (e.g., an `evaluator-supervisor` that manages the N evaluator-agents for a single proposal, sitting beneath the top-level supervisor that manages multiple proposals). Useful when there are coordination concerns within a worker subtree that don't need to bubble up.
- **Parallel fan-out** — supervisor dispatches N worker invocations in parallel, collects N results before the next decision. Today's evaluator-agents fan out across **three concurrent evaluations × six evaluators each** = 18 parallel scoring tasks before fan-in to consensus.

The build choice today: **flat supervisor with parallel fan-out of evaluator-agents** (simpler, sufficient for the morning). Hierarchical is a Thu/Fri refactor candidate if the supervisor's prompt grows past usefulness.

LangGraph's reducer patterns (e.g., `Annotated[list, add_messages]` on state keys that multiple parallel workers write to) are what keep parallel writes safe; you'll meet them in code Thu morning.

## 4. Multi-agent anti-patterns — chatty handoffs, audit fan-out, lost context (10 min)

Three named anti-patterns. Today's load surfaces all three if you let them — they're not abstract.

- **Chatty handoffs.** Supervisor re-invokes Bedrock between every worker step "to check progress." A five-call flow balloons into thirty calls with no observability win. Latency multiplies, cost multiplies, the LangSmith trace becomes unreadable. *Defense:* the supervisor decides once per worker call, not once per worker thought.
- **Audit fan-out.** Every agent writes its own `AuditEvent`. Three evaluations × six evaluators × ~four events each ≈ **72 audit rows fanning into one Postgres write surface**. Item 2's audit-log race becomes catastrophic — you lose multiple rows, not one. *Defense:* single audit-writer pattern — either one dedicated `audit_writer` node that drains `EvaluationState.pending_audit_events` on transitions, or one utility called from one place. Pick one shape; defend it.
- **Lost context across the supervisor boundary.** Supervisor passes a stripped-down state to the worker; the worker can't recover the framing it needs to do its job; results come back inconsistent. *Defense:* a documented worker-input contract per worker type. Don't let "supervisor decides what worker sees" become "supervisor invents what worker sees per call."

These three differ from Tue's anti-patterns (which were wiring-discipline reminders for single-agent ReAct). Today's are **new multi-agent-specific concepts** — they don't exist at single-agent scope. Recognize them by name; recognize the W1 Tue brownfield-debt items they activate.

## 5. HITL #4 — supervisor → worker soft interrupt (12 min) **⭐ HITL touchpoint #4**

**Touchpoint #4 of seven. Soft interrupt. Different from #3 and #5 — do not pre-collapse.**

The SSA's new ask: visibility into the supervisor's decisions *before* consensus completes — *not* at the SSDD-review boundary (that's Thu's hard gate, #5). She wants to see what the supervisor is *about to delegate*, approve or override, before it fires.

**Soft-interrupt shape:** the graph **pauses** at `supervisor_decide_next_worker`; the supervisor's proposed worker call (with a *suggested* default action) is surfaced; the SSA approves via `Command(resume={"approved": True})`, edits via `Command(resume={"override": "evaluator_2"})`, or rejects. After her response, the supervisor fires the (possibly-edited) call. LangGraph wiring:

```python
app = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["supervisor_decide_next_worker"],
)
```

**Audit schema for soft-interrupt resume** (`HITL_SOFT_RESUME`): one `AuditEvent` row with `before` (the supervisor's proposed call) + `after` (the SSA's edit or approval) + `actor_id = user:{ssa_id}` + the threaded `audit_correlation_id` from `EvaluationState`. Same row, two state snapshots, no fan-out.

**Three explicit distinctions** (don't conflate):

| HITL | When | Shape | FAR anchor |
|------|------|-------|------------|
| #3 (Mon) | Plan Day | **ADR** — name *which* transitions are gated and why | 15.206 amendments framing |
| **#4 (Wed)** | **Multi-agent handoffs** | **Soft interrupt** — suggest default, human approves/edits | (sets up 15.308) |
| #5 (Thu) | SSDD review + award | Hard interrupt — block indefinitely; non-regenerative resume | 15.308 SSA cannot delegate; 5.705 irreversible |

**Misconception to pre-empt before walking in:** *"If the SSA doesn't respond in 5 min, auto-approve so the system doesn't block."* **Wrong** — and the precise reason matters: auto-timeout-then-approve **is itself a delegation**. The W4 surface for this misread is Thu's hard-gate, but today's soft-gate inherits the same discipline — the human's calendar is the constraint, not the graph's patience.

## 6. Advanced memory patterns for multi-agent (7 min)

Three memory shapes for multi-agent flows (builds on Mon's single-agent memory topic):

- **Shared scratchpad** — all agents read/write a common state key (e.g., `EvaluationState.shared_notes`). Cheap, leaky; works when workers genuinely share a working context.
- **Per-agent thread** — each agent has its own isolated message history. Hermetic; good for evaluator-agents (they shouldn't see each other's scoring rationale until consensus).
- **Supervisor-curated context** — supervisor filters/summarizes what each worker sees. Most expressive; most expensive (an extra Bedrock call per delegation if naïvely implemented). Use when worker prompts need agent-specific framing that the worker shouldn't reconstruct itself.

The Wed–Fri evaluator flow uses **per-agent thread for evaluator-agents** (isolated scoring) + **supervisor-curated context for the consensus-agent** (the supervisor passes the panel's per-evaluator results with framing). Defend the pick today; the choice is in your morning scaffolding.

## 7. KG + CG + scaffold orchestrator — the 17-entity acquire-gov schema (15 min)

D-030's substantial mid-W3 placement. The acquire-gov schema is a *graph*, not a tree.

**The 17-entity KG:** Vendor ↔ Proposal ↔ Evaluation ↔ Award ↔ ContractModification ↔ Cpar, plus sub-entities (Solicitation, Amendment, EvaluatorScore, SSDD, ConsensusRound, QASP findings, …). Edges carry semantics (`vendor_submitted_proposal`, `evaluator_scored_proposal`, `award_modified_by`, `vendor_holds_cpar`). The schema is canonical from W1 Tue's brownfield-debt inventory.

**KG vs CG:**

- **Knowledge Graph (KG)** — static, persistent, the source-of-truth. Vendor A has won 12 contracts; 3 carry red CPARs. The KG owns that fact across requests.
- **Context Graph (CG)** — runtime, per-request, assembled by the agent. "This proposal's context = the solicitation + its 3 amendments + the vendor's last 5 CPARs + the panel's prior consensus on this vendor." The CG is what gets stitched into the prompt; the KG is the source it's stitched *from*.

**Three CO query traversal patterns** (today's whiteboard exercise, tomorrow's pair-project scaffolding):

1. *"Show me every contract this vendor has won with red CPARs"* — Vendor → Award → Contract → Cpar with `rating=red` filter (3 hops).
2. *"Which evaluators have scored proposals from vendors with QASP findings in their portfolio?"* — Vendor → Award → Contract → QaspFinding back to Evaluator via EvaluationScore (4 hops, includes a back-reference).
3. *"Are there contract modifications on awards where the original SSDD never approved a tradeoff above the IGCE?"* — Award → SSDD → Evaluation → IGCE ↔ ContractModification with value filter (irregular shape — recursive on ContractModification chains).

**Scaffold orchestrator pattern.** The supervisor isn't doing KG traversal itself — a dedicated `kg_query` tool wraps the chosen backend, and the supervisor (or a worker) invokes it. The orchestrator is what keeps the KG behind a stable contract while the *backend* underneath remains a tradeoff (SA-3 below).

**Three viable backends — SA-3 framing:**

- **Neo4j** — purpose-built graph DB. Cypher queries are natural for traversal (`MATCH (v:Vendor)-[:WON]->(a:Award)-[:GOT_CPAR]->(c:Cpar {rating: 'red'})`). Operational overhead is real — another DB to backup, secure, FedRAMP-scope.
- **Postgres recursive CTE** — your existing Postgres can do it. `WITH RECURSIVE` queries express traversal; powerful but unfamiliar to most engineers; performance depends on indexing strategy you don't naturally think about until queries get slow.
- **NetworkX in-process** — Python graph library, in-memory. Load the relevant subgraph per request; runs traversals in pure Python. Fine for low cardinality (one tenant, ~100 vendors, ~500 proposals, ~80 active contracts per `training-project/feature-inventory-target.md`); breaks at scale; doesn't survive process restart.

There's no universally right answer; data residency, FedRAMP boundary, ops budget, and query latency targets drive the pick. SA-3's ADR (due Thu W4) captures the defense. Today's expectation: *you can articulate the tradeoffs in your own words by EOD.*

## 8. Research discipline — scenario alternatives via `/web-research` (5 min, **Wed afternoon activity**)

Wed afternoon is the **dedicated scenario-research slot** per D-040 — not a topic to teach, an activity to do. Three scenario-alternatives prompts release this morning under `scenarios/`:

1. **SA-1** — Single-agent vs multi-agent for intake-triage. Re-opens Mon's decision after two more days of impl experience.
2. **SA-2** — LangGraph vs LangChain Agents vs hand-rolled. Framework comparison constrained to the evaluator → consensus → SSA flow + the `ai-orchestrator` FastAPI + Pydantic + Bedrock constraint.
3. **SA-3** — KG/CG representation: Neo4j vs Postgres recursive CTE vs NetworkX. Anchors to topic 7 above.

**Research discipline (non-negotiable):** all source-current claims route through `/web-research` per `pipeline/RESEARCH-PROTOCOL.md` + the CLAUDE.md hard rule. Sub-30-day sources require explicit instructor sign-off before inclusion (adopt / defer / omit). Known-bad-patterns in `skills/tech-research/references/known-bad-patterns.yml` (notably LCEL-as-foundational and `Chain` class advocacy for LangChain) flag for instructor review — don't silently include. Cite every retrieved source with a date.

ADRs due EOD Thu W4 (after the Phase 1 Gate). Live Defense scheduled inside W4. Today: research + sketch.

## 9. Two questions to come in with tomorrow morning

1. For the **evaluator-agent → consensus-agent** handoff (after all N evaluators have scored a proposal): is this a hard interrupt, a soft interrupt, or no interrupt? What's the FAR/regulatory anchor for your answer? *(Hint: this is not where #4 or #5 lives — defend the no-interrupt case carefully.)*
2. The "show me every contract this vendor has won with red CPARs" query needs sub-200ms p95 against the data volumes in `training-project/feature-inventory-target.md`. Of {Neo4j, Postgres recursive CTE, NetworkX in-process}, which would you pick? Defend in one sentence. *(Don't pre-commit your SA-3 ADR — but show you can sketch the answer.)*

## 10. Glossary refresh (terms you'll hear today)

- **Supervisor-worker pattern** — multi-agent shape; one supervisor delegates, workers do narrow tasks.
- **Hierarchical orchestration** — supervisors of supervisors; worker subtrees with their own coordination.
- **Parallel fan-out** — supervisor dispatches N workers in parallel; collects N results before next decision.
- **Delegation contract** — the per-arrow agreement of *what state the worker reads + writes + when the supervisor fires it*.
- **Chatty handoff** — anti-pattern: supervisor re-invokes the LLM between every worker step. Latency + cost balloon; no value.
- **Audit fan-out** — anti-pattern: every agent writes its own `AuditEvent`. Item 2's race becomes multi-row.
- **Lost context across supervisor boundary** — anti-pattern: worker can't recover framing; results inconsistent.
- **Soft interrupt** — graph pauses; suggests default; human approves/edits/rejects via `Command(resume=...)`.
- **Hard interrupt** — graph blocks indefinitely until human input (Thu's shape, not today's).
- **Knowledge Graph (KG)** — static, persistent entity-and-relationship graph. Lookups are traversals.
- **Context Graph (CG)** — runtime per-request graph stitched from the KG + retrieved evidence; what enters the prompt.
- **Scaffold orchestrator** — the supervisor calls a `kg_query` tool that wraps the chosen backend; the backend is a tradeoff under the stable contract.
- **Correlation ID (`audit_correlation_id`)** — one ID per `evaluation_id` threaded through every node + tool + AuditEvent row. Item 6's partial fix today; full fix W5 W3C `traceparent`.

## What today looks like (logistics, ~3 min)

- **09:00–12:00 — war-room** (`war-room/D3.md`): CO update + SSA cameo at minute 30 of Act 1 (first SSA appearance — she returns Thu as anchor); whiteboard the supervisor topology; identify the soft-interrupt point; scaffolding demo (~15 min live coding); cohort builds.
- **12:00–13:00 — lunch.**
- **13:00–16:00 — dedicated scenario-research slot (D-040)**: SA-1 / SA-2 / SA-3 in pair-collaborative mode. Mid-afternoon (~60 min) parallel KG/CG conceptual block on the whiteboard — instructor-led on the 17-entity schema + three CO query traversals.
- **16:00–17:00 — convergence + EOD ship.** Each pair shows the supervisor pause-and-resume in LangSmith; surfaces Item 3 (no circuit breaker `evaluator → solicitation-service`) in the pair-retro note.

## What you'll ship today

- Supervisor-worker scaffolding in `ai-orchestrator/agents/evaluation_flow.py` — supervisor + at least one worker type (evaluator-agent) end-to-end against `EvaluationState`.
- HITL #4 implementation: `interrupt_before=["supervisor_decide_next_worker"]` on the supervisor decision node; soft interrupt fires; resume via `Command(resume=...)` works in a unit test.
- Single audit-writer pattern committed — one of two shapes (dedicated `audit_writer` node, or one utility called from one place). No fan-out.
- LangSmith trace screenshot of the supervisor pause → SSA-approval → worker → result loop, in the pair-repo.
- Three SA prompt sketches (SA-1, SA-2, SA-3) — not full ADRs yet; one finding per SA + one open question. Full ADRs due EOD Thu W4.
- KG/CG whiteboard photo committed as `docs/W3-D3-kg-cg.png` in the pair-repo.
- Pair-retro note naming Item 3 (no circuit breaker `evaluator → solicitation-service`) failure mode observed under concurrent load. Fix lands W5 (Resilience4j); today's posture is *document and proceed with hand-rolled retry*.

## What to read or watch tonight

- [LangGraph — Multi-agent systems overview](https://langchain-ai.github.io/langgraph/concepts/multi_agent/) (~15 min). Focus on the **supervisor pattern** section and the comparison with handoff-without-supervisor. Source retrieved 2026-05-26 via /web-research; cross-verified against `research/langchain-v1-20260522.md`.
- [LangGraph — Human-in-the-loop with multi-agent](https://langchain-ai.github.io/langgraph/concepts/human_in_the_loop/) (~10 min, §3 "Review tool calls"). The HITL #4 pattern: supervisor reviews the *tool call* before firing. Source retrieved 2026-05-26 via /web-research.
- [FAR 15.308 — Source selection decision](https://www.acquisition.gov/far/15.308) (~6 min, **second read this week** — Mon was framing; Wed it becomes the soft-gate setup for Thu's hard-gate). Today's read focus: *"The SSA's independent judgment ... shall not be delegated."* Source retrieved 2026-05-26 via /web-research.
- [Neo4j Cypher manual — traversal patterns](https://neo4j.com/docs/cypher-manual/current/queries/concepts/) (~10 min). Focus on variable-length path matching for the "every contract this vendor won with red CPARs" pattern. Source retrieved 2026-05-26 via /web-research.
- *(optional, skip if comfortable with recursive CTEs)* [Postgres recursive CTE](https://www.postgresql.org/docs/current/queries-with.html#QUERIES-WITH-RECURSIVE) (~12 min). Useful for SA-3. Source retrieved 2026-05-26 via /web-research.

## Optional deep-dive (not required to participate)

- [Multi-agent collaboration — LangGraph patterns tutorial](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/multi-agent-collaboration/) (~20 min). Useful preview for Thu's full-implementation deep-dive (HITL #5 + parallel execution + checkpointing). Source retrieved 2026-05-26 via /web-research.

## Sources

- LangGraph multi-agent docs (canonical), retrieved 2026-05-26 via /web-research.
- LangGraph human-in-the-loop docs (canonical), retrieved 2026-05-26 via /web-research.
- FAR 15.308 — Source selection decision (acquisition.gov), retrieved 2026-05-26 via /web-research.
- Neo4j Cypher manual (canonical), retrieved 2026-05-26 via /web-research.
- Postgres recursive CTE docs (canonical), retrieved 2026-05-26 via /web-research.
- `research/langchain-v1-20260522.md` — LangChain v1.0 brief (hot-tech, 3mo window, in-range).
- `research/bedrock-claude-catalog-20260522.md` — Bedrock model IDs (hot-tech, 3mo window, in-range).
- `training-project/feature-inventory-target.md` — 17-entity acquire-gov schema (canonical, in-repo).

Known-bad-patterns checked against `skills/tech-research/references/known-bad-patterns.yml` (LCEL-as-foundational, `Chain` class advocacy, LCEL `|` pipe). None of today's reading or scaffolding promotes those patterns; LangGraph + plain-Python composition is the v1.0-aligned shape.

Last verified: 2026-05-26
