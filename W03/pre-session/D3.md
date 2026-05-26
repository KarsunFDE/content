---
template: pre-session-reading
week: W03
day: Wed
phase: Agentic
topic: "Multi-agent patterns + Knowledge/Context graphs + HITL #4 (multi-agent handoffs) + scenario research"
estimated_total_minutes: 60
last_verified: 2026-05-23
fde_situations: [3, 4, 5, 7, 11]
tech: [LangGraph, Neo4j, Postgres recursive CTE, NetworkX, CrewAI (compare-set only)]
sources_research_briefs: [research/multi-agent-supervisor-pattern.md, research/knowledge-graph-acquisitions.md]
author: instructor
---

# Pre-session reading — Multi-agent patterns + KG/CG thinking + HITL #4

Week 3, Day Wed. Estimated total time on task: ~60 minutes. Last verified: 2026-05-23.

## 1. Why this matters

Wednesday is the pivot day. Mon-Tue's single-agent intake-triage handoffs over to a harder shape: **evaluator-agent → consensus-agent → SSA-review-agent** (Wed-Fri per D-060). Multi-agent introduces real failure modes: chatty handoffs that explode latency, lost context across the supervisor boundary, audit fan-out that makes Item 2's audit-log race catastrophic, and — most importantly — **HITL gates at agent handoff boundaries**. That's touchpoint #4 of seven. Wednesday also lands the substantial Knowledge Graph + Context Graph thinking placement (per D-030) — the 17-entity acquire-gov schema is your graph, and CO query patterns are your traversal exercises.

Wednesday afternoon is the **dedicated scenario-research slot** (per D-040). You'll research three scenario-alternatives prompts in pair-collaborative mode.

## 2. Core concept in 5 minutes — supervisor-worker is the default multi-agent shape

The supervisor-worker pattern: one **supervisor agent** decides *which worker to call next* and *when the work is done*; the workers do narrow, well-defined tasks. For the evaluation flow:

- **Supervisor:** decides — score next proposal? Aggregate consensus? Hand off to SSA?
- **Evaluator-agent (worker × N):** scores one proposal against Section M factors. Pure read + reason.
- **Consensus-agent (worker × 1):** aggregates per-evaluator scores into a panel consensus + narrative.
- **SSA-review-agent (worker × 1, HARD GATE):** produces the SSDD tradeoff narrative for SSA review. **This agent doesn't fire without a human approval at the entry — per FAR 15.308 the SSA decision authority cannot delegate.**

The HITL #4 question Wednesday answers: *at which supervisor → worker handoff is human approval required before the supervisor delegates?* Not all of them — `supervisor → evaluator-agent(proposal_N)` is reversible (reviewable read) and doesn't need a gate. But `consensus-agent → SSA-review-agent` is the irreversible-handoff boundary; that's where HITL lives.

The anti-pattern to avoid: **chatty handoffs**. If your supervisor agent re-invokes Bedrock between every worker step "to check progress," you've turned a 5-call flow into a 30-call flow with no observability win.

## 3. Knowledge graphs + context graphs — your 17-entity schema is a graph

The acquire-gov schema (Vendor ↔ Proposal ↔ Evaluation ↔ Award ↔ ContractModification ↔ Cpar — 17 entities total per `training-project/feature-inventory-target.md`) is a *graph*, not a tree. CO queries that take advantage of the graph:

- *"Show me every contract this vendor has won with red CPARs"* — Vendor → Award → Contract → Cpar with `rating=red` filter.
- *"Which evaluators have scored proposals from vendors with QASP findings in their portfolio?"* — Vendor → Award → Contract → QaspFinding back to Evaluator via EvaluationScore.
- *"Are there contract modifications on awards where the original SSDD never approved a tradeoff above the IGCE?"* — Award → SSDD → Evaluation → IGCE ↔ ContractModification with value filter.

These are graph traversal queries. You have three viable backends — Wed's third scenario-alternative compares them:

- **Neo4j** — purpose-built graph DB; Cypher queries are natural; operational overhead is real.
- **Postgres recursive CTE** — your existing Postgres can do it; recursive `WITH RECURSIVE` queries are powerful but unfamiliar to most engineers.
- **NetworkX in-process** — load the graph into Python on every query; fine for low cardinality, breaks at scale.

There's no universally right answer; the constraint shape (data residency, FedRAMP boundary, ops budget) drives the pick.

## 4. What to read or watch tonight

- [LangGraph — Multi-agent systems overview](https://langchain-ai.github.io/langgraph/concepts/multi_agent/) (~15 min read), published 2025-12-22 (verified 2026-05-23 via /web-research). Focus on the **supervisor pattern** section and the comparison with handoffs-between-agents-without-supervisor. Wednesday's anchor.
- [LangGraph — Human-in-the-loop with multi-agent](https://langchain-ai.github.io/langgraph/concepts/human_in_the_loop/) (~10 min read, focus on §3 "Review tool calls"), published 2025-12-15 (verified 2026-05-23 via /web-research). The HITL #4 pattern: supervisor reviews the *tool call* (i.e., the proposed worker invocation) before the supervisor fires it.
- [FAR 15.308 — Source selection decision](https://www.acquisition.gov/far/15.308) (~6 min read, **second read this week** — Mon was the framing, Wed it becomes implementation), retrieved 2026-05-23 via /web-research. Today's read focus: *"The source selection authority's (SSA) decision shall be based on a comparative assessment ... The SSA's independent judgment ... shall not be delegated."* This is the non-negotiable hard-gate justification for the SSA-review-agent.
- [Neo4j Cypher manual — traversal patterns](https://neo4j.com/docs/cypher-manual/current/queries/concepts/) (~10 min read), retrieved 2026-05-23 via /web-research. Focus on variable-length path matching for the "every contract this vendor won with red CPARs" pattern.
- *(optional)* [Postgres recursive CTE](https://www.postgresql.org/docs/current/queries-with.html#QUERIES-WITH-RECURSIVE) (~12 min read), retrieved 2026-05-23 via /web-research. *Skip if you're comfortable with recursive CTEs.* Useful for Wednesday's third scenario-alternative.

## 5. Two questions to come in with tomorrow morning

1. For the **evaluator-agent → consensus-agent** handoff (after all N evaluators have scored): is this a hard interrupt, a soft interrupt, or no interrupt? What's the FAR/regulatory anchor for your answer?
2. The "show me every contract this vendor has won with red CPARs" query needs to be sub-200ms p95. Of {Neo4j, Postgres recursive CTE, NetworkX in-process}, which would you pick given the data volumes in `training-project/feature-inventory-target.md` (one tenant, ~100 vendors, ~500 proposals, ~80 active contracts)? Defend in one sentence.

## 6. Glossary refresh (terms you'll hear today)

- **Supervisor-worker pattern** — multi-agent shape where one supervisor agent delegates to workers + collects results.
- **Hierarchical orchestration** — supervisors of supervisors. Useful when worker groups have their own coordination logic.
- **Parallel fan-out** — supervisor dispatches N worker invocations in parallel; collects N results before next decision.
- **Chatty handoff anti-pattern** — supervisor invokes the LLM between every worker step "to check progress." Latency + cost balloon, no value.
- **Audit fan-out anti-pattern** — every agent in the flow writes its own AuditEvent. When Item 2's audit-log race fires, you lose multiple rows, not one. Defense: single audit-writer agent + correlation_id threading.
- **Knowledge graph (KG)** — explicit entity-and-relationship graph. Lookups are traversals.
- **Context graph (CG)** — graph-shaped *runtime context* assembled per request (vs static KG). E.g., per-request "this proposal's context = the solicitation + its 3 amendments + the vendor's prior CPARs."
- **Supervisor-as-tool-caller** — implementation pattern where the supervisor calls workers *as if they were tools* (LangGraph's `ToolNode` makes this natural).

## 7. Scenario-research slot (Wed afternoon, per D-040)

Three scenario-alternatives released this morning (`scenarios/W03-SA-1.md` through `W03-SA-3.md`):

1. **SA-1 — Single-agent vs multi-agent for intake-triage.** Mon-Tue you built it single-agent. Should it stay that way or transition to multi-agent? Tradeoffs.
2. **SA-2 — LangGraph vs LangChain agents vs hand-rolled.** Framework comparison constrained to the evaluator→consensus→SSA flow shape and the ai-orchestrator FastAPI + Pydantic + Bedrock constraint.
3. **SA-3 — KG/CG representation: Neo4j vs Postgres recursive CTE vs NetworkX.** The CO query traversal patterns drive the comparison.

ADR due EOD Thu W4 (after the Phase 1 Gate). Live Defense scheduled inside W4.

## 8. What you'll ship today

- Multi-agent supervisor-worker scaffolding in `ai-orchestrator/agents/evaluation_flow.py` — supervisor + 3 worker types defined, handoff logic stubbed.
- HITL #4 implementation: `interrupt_before=["ssa_review_ssdd"]` on the supervisor → SSA-review-agent edge, plus a supervisor approval gate before parallel evaluator fan-out (soft).
- KG/CG whiteboard sketch + three ADR-shaped notes in pair-project repo (SA-1/SA-2/SA-3 — not yet full ADRs; full ADRs due Thu W4).
- LangSmith traces showing the multi-agent flow (you'll see the supervisor → worker hand-offs as nested runs).

## 9. Anti-patterns to avoid today

- **"Just one more agent"** — every additional agent multiplies the failure surface. Defend each agent's existence.
- **Hand-rolled multi-agent on Day 3.** Use LangGraph's primitives. Hand-rolling is a Wed scenario-alternative *to evaluate*, not the path you commit to today.
- **Putting HITL #4 only on the SSA boundary.** That's HITL #5's anchor (Thu). HITL #4 is *between worker handoffs* — the supervisor approves the next worker invocation. If you only gate the SSA, you've conflated the two touchpoints.
- **Ignoring the audit fan-out anti-pattern.** When Wed's traces light up with five agents each writing their own AuditEvent, the cohort's W1 Tue brownfield-debt inventory item 2 race becomes a multi-row problem, not a one-row problem.

## 10. Optional deep-dive (not required to participate)

- [Multi-agent collaboration — LangGraph patterns](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/multi-agent-collaboration/) (~20 min read), published 2026-01-15 (verified 2026-05-23 via /web-research). Useful for tomorrow's full-implementation deep-dive.
