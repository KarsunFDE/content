---
checkpoint: 2
title: "QC Audit Expectations — Checkpoint 2: Agentic Systems"
covers: ["W03 Agentic Systems"]
delivery_date: 2026-06-16
last_verified: 2026-06-03
read_time_min: 11
audience: learner
---

# Checkpoint 2 — What to Expect (Tue 16 Jun 2026)

> Learner-facing prep brief. Covers **W3 Agentic Systems** — single agent → multi-agent → LangGraph HITL → KG/CG thinking. Lands the Tuesday after Phase 1 Gate, so the auditor knows you've just defended Phase 1 and will press on the agentic decisions you made there.

## At a glance

| Surface | Format | Duration | When |
|---|---|---|---|
| **Exam (in-person)** | Scenario-grounded MCQ + (Senior) short scenario. Heavier emphasis on multi-agent topology + HITL placement than C1. | 90 min | Tue 16 Jun, inside the 2-hr block |
| **Cohort debrief** | Instructor walks the flow + schedules audits | 30 min | Same block |
| **Audit interview (1:1)** | Zoom, 30 min. Tech-grounded opener, 7-thinking-dim follow-ups. **AI-Thinking floor raised to 5** for this checkpoint — agentic IS the AI-thinking content. | 30 min/candidate | Async slot you book |

---

## What's different about C2

Checkpoint 1 tested whether you could reason about a single LLM call grounded in a retrieval. Checkpoint 2 tests whether you can reason about **a graph of LLM calls where state, handoffs, and human gates are all design surfaces**. The auditor opens with a topology decision, presses on what each node owns, and ladders into who-has-authority + how-do-you-test-this.

**Three HITL touchpoints landed in W3** — they will come back as audit fodder:
- **#3 (Mon)** — interrupt-node boundaries in the agent graph (where a human MUST be in the loop)
- **#4 (Wed)** — multi-agent handoffs (correlation IDs across agent boundaries — debt item 6)
- **#5 (Thu)** — LangGraph HITL deep-dive (state machines + interrupt → resume patterns)

If you can defend *where* the HITL gate sits in your pair's agent graph + *why there and not earlier/later*, you've prepared for ~40% of the audit pressure.

---

## Topics in scope

Everything in `W03/pre-session/` + war-room incidents + your pair's W3 ADRs. The PLAN file shape:

- [W3 PLAN — week shape](../W03/PLAN.md)
- W3 pre-session: Mon (Agentic systems framing + ADR set), Tue (ReAct workflows + single-agent intake-triage), Wed (multi-agent design patterns + KG/CG thinking + scenario research), Thu (state machines + LangGraph HITL deep-dive), Fri (Phase 1 Defense — no normal pre-session). Files in `W03/pre-session/`.
- W3 war-room incidents (Mon→Thu) — `W03/war-room/`. Mon is Plan Day mode; Tue/Wed/Thu are incident-mode.

### Cross-cutting

- **Single-agent vs multi-agent decision boundary** — when do you actually need more than one agent, and what does the wrong answer cost you
- **State machine framing** (LangGraph) — nodes, edges, conditional edges, interrupts, checkpoint state
- **Knowledge Graph vs Context Graph** thinking — how the 17-entity KG (Vendor / Proposal / Evaluation / Award / ContractModification / CPAR / etc) shows up in retrieval and agent decision-making
- **Handoff observability** — correlation IDs across agent boundaries; what audit-log rows must land for a CO-or-OIG audit later
- **FAR citation in HITL design** — FAR 15.206 (CO amendment authority), FAR 15.308 (SSA decision authority — cannot be delegated)

---

## Example scenarios (illustrative — distinct from your week's incidents)

### Example A — Multi-agent split decision (T2 / T7)

> *"Your pair built a single ReAct agent for `POST /agent/intake-triage` that ingests an incoming proposal, classifies it against the solicitation type, extracts vendor metadata, and routes to the evaluator queue. The CO is asking why it occasionally mis-routes Section M evaluation criteria into the Section L instructions bucket. Your tech lead wants to split this into three agents. Defend or push back."*

**What an opening answer should touch:**

- Name the **load-bearing question** — is the mis-routing a *prompt/context* problem or a *cognitive boundary* problem? A single agent juggling 3 concerns isn't automatically wrong
- Frame the split as a **trade**: single agent has one context window + one set of failure modes; three agents have orchestration overhead, three context-window costs, more handoff surface, more places state can drift
- The auditor isn't looking for "split it" or "don't split it" — they're looking for *what evidence would tip the decision* (eval delta on misroute rate vs latency budget vs cost budget vs observability gain)
- HITL question: where would a human gate land in either topology? Same place? Different place?

**Likely follow-up presses:**
- *T6 — what eval set would you build to actually decide this?*
- *T5 — if you split, what new failure modes do you introduce at the handoff boundary?* (Lost state, double-classification, partial commits.)
- *T7 — if the SSA decision step is in this graph, does FAR 15.308 force a particular interrupt-node placement?* (Yes — SSA decision authority cannot be delegated; the model can prepare the decision, the SSA must commit it.)

### Example B — LangGraph interrupt + resume contract (T3 / T7 / T6)

> *"Your evaluator agent processes a proposal, scores it against the SSDD criteria, then hits an interrupt node that waits for the SSA review. The SSA reviews on Thursday, but Wednesday night a process crashes. When the platform restarts Thursday morning, the SSA's review queue is empty even though no one signed anything. Walk me through what's broken and how you'd build this so it can't happen."*

**What an opening answer should touch:**

- LangGraph interrupts work against **persisted graph state** (checkpointer). If the state wasn't persisted to durable storage between the interrupt and the crash, the interrupt was lost — that's the bug
- Name the persistence contract: which checkpointer (in-memory? Postgres? Redis?) and what its durability story is. Most production patterns use a database-backed checkpointer with thread-id scoping
- Press the **idempotency** angle — when the system comes back up, can you safely resume? What if the agent partially-wrote to the audit log before the crash?
- The fix isn't "use Postgres checkpointer" alone — it's "use a durable checkpointer + the audit-log writes are idempotent + the resume contract is explicit"

**Likely follow-up presses:**
- *T5 — what's the worst-case scenario if you replay an interrupt that already partially committed?* (Double-award, double-CPAR-entry, regulatory issue.)
- *T6 — how would you test this in CI? You can't kill prod.* (Chaos test in a graph-replay fixture; interrupt + restart, assert state.)
- *T2 — does the checkpointer become a SPOF? What's the blast radius if Postgres goes down?* (Whole agent graph stalls — design for graceful degradation or fail-fast.)

### Example C — KG vs vector retrieval (T7 / T4)

> *"Your evaluator agent retrieves prior CPAR entries for the vendor under evaluation. Pure vector search over CPAR free-text gets you 'similar' vendors and 'similar' performance comments. Your pair lead is proposing you add the 17-entity KG into the loop so the agent can traverse Vendor → past Awards → linked CPARs → linked ContractModifications. Defend the trade."*

**What an opening answer should touch:**

- Vector search finds *semantic similarity* — useful for "have we seen language like this before?"
- A KG traversal finds *relational truth* — "what awards has this vendor held? Which were modified? Which had below-bar CPARs?" That's a different question
- These compose; they don't compete. The agent should know which question it's asking before it picks the retrieval surface
- Cost-shape: KG traversal latency is bounded by graph depth, not by index size; vector latency is bounded by recall+rerank cost. Different ops profile
- Naming the failure mode: vector-only retrieval missing the *fact* that this vendor had a contract terminated for cause in 2023 is a *correctness* issue, not a relevance issue

**Likely follow-up presses:**
- *T2 — where does the KG sit in your architecture? Neo4j? A property graph in Postgres? Built on Atlas?*
- *T6 — how do you keep the KG in sync with the source-of-truth records? Event-driven? Batch?*
- *T5 — if a CPAR record is corrected after-the-fact, how does the agent know to re-evaluate?*

---

## What "meets bar" looks like

For C2 specifically, the auditor is calibrating you on:

1. **Topology defensibility** — you can sketch the agent graph, name what each node owns, and defend the boundary placement against a "why not just one agent?" or "why not three more?" press
2. **HITL placement with FAR grounding** — when the auditor names a federal-acquisitions authority concept (FAR 15.206, FAR 15.308, SSA authority, CO discretion), you can map it to a specific interrupt node in the graph and explain why that's the load-bearing gate
3. **Handoff observability** — when state crosses an agent boundary, you can name what audit-log rows MUST land + what correlation ID anchors them. *"It works in the happy path"* is a fail — the auditor will press on the 3am-on-Saturday path
4. **State persistence contract** — for any LangGraph interrupt, you can name the checkpointer choice and defend it against the durability + idempotency questions

Common stuck points:

- **Conflating LangGraph with LangChain** — they're related but distinct; LangGraph is the runtime substrate for stateful + branching workflows, LangChain v1.0 is the LLM-call abstraction surface
- **"More agents = better"** — not the answer. Each agent boundary is an integration surface; integration surfaces have latency, cost, and failure-mode taxes
- **Forgetting the human** — if your topology answer doesn't have a HITL gate, the auditor will press T7 until one appears. For Karsun-FDE federal work, *something* requires CO or SSA authority
- **Vague observability answers** — *"we'd log it"* is not an answer. What event, what fields, what correlation ID, what consumer of that log

---

## How to prep

1. **Re-walk your pair's W3 ADR set** — the three required ADRs (single-vs-multi-agent split, LangGraph framework defence, HITL #3 interrupt-node boundaries). Know what you chose AND what you considered AND why
2. **Sketch your agent graph from memory** — if you can't draw the nodes + edges + interrupt placements without looking, fix that today. Auditors often hand you the whiteboard
3. **Read the [W3 PLAN](../W03/PLAN.md)** end-to-end + at least one war-room you didn't run yourself. Your pair worked one variant; the auditor may anchor on a different one
4. **Review FAR 15.206 + FAR 15.308 in plain English** — you don't need to cite section numbers from memory, but you should know which acquisition acts cannot be delegated and which require CO approval. Your domain-knowledge repo has the references
5. **Run the LangGraph interrupt → resume pattern** once locally if you didn't already. The "can my graph survive a crash" question is the C2 equivalent of last checkpoint's "can your RAG survive a vendor lookup miss"

---

## Logistics

Same as C1: 90-min exam in the 2-hr block; 30-min Zoom audit on an async slot you book; two auditors, Zoom auto-recorded.

**One thing to watch:** C2 lands the Tuesday *after* the Phase 1 Gate (Fri 12 Jun). You'll be tired. Treat the weekend before C2 as recovery, not as cramming. Your pair already defended Phase 1 successfully — most of what the auditor will press on, you already had to say out loud at the Gate.

---

*Brief last verified 2026-06-03. Re-verify LangGraph/LangChain hot-tech references against `research/` briefs if prepping more than 30 days post write-date.*
