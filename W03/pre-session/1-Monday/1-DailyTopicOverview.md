---
template: pre-session-reading
week: W03
day: Mon
phase: Agentic
topic: "Agentic systems framing + HITL #3 interrupt-node boundaries ADR (Plan Day)"
estimated_total_minutes: 80
last_verified: 2026-06-06
fde_situations: [3, 4, 5, 7, 11]
tech: [LangGraph, LangChain v1.0, Bedrock InvokeModel, FAR 15.206, FAR 15.308]
sources_research_briefs:
  - research/langchain-v1-20260522.md
  - research/bedrock-claude-catalog-20260522.md
author: instructor
hitl_touchpoint: 3
---

# W3 Monday Pre-Session — HITL #3: Plan-Day ADR pattern (agentic adoption)

> [!NOTE]
> **From earlier:** W2 Fri's eval-harness gave you measurable signals on every RAG transition. Monday's HITL #3 ADR cites that harness as the calibration source for agent-level evals.

## 1. What you'll learn today

By the end of war-room you'll be able to:

- Classify any agent transition as no-gate / soft-gate / hard-gate with a one-sentence justification.
- Write a defensible HITL #3 ADR — decision, FAR citation, implementation primitive.
- Explain why agents are state machines and where HITL nodes fit in that model.
- Design a memory scope plan your pair can carry into Tuesday without revisiting.

## 2. The day at a glance

```mermaid
graph LR
    T2[Topic 2: Irreversibility] --> T3[Topic 3: State machines + HITL]
    T3 --> T4[Topic 4: Memory patterns]
    T4 --> T5[Topic 5: Eval-driven dev]
    T5 --> T6[Topic 6: §0 retro discipline]
    T6 --> T7[Topic 7: HITL #3 ADR]
    T7 --> WR[War-room + plan-spec]
```

| Topic | Focus | Reading min | Why you'll need it |
|-------|-------|------------:|--------------------|
| 2. Irreversibility | One-way doors, CO framing | ~10 min | The justification every ADR gate cites |
| 3. Agents as state machines | Nodes, edges, interrupts, checkpointers | ~10 min | Mental model for the whole week |
| 4. Memory patterns | Thread vs cross-thread vs shared | ~10 min | Memory choice lands in today's plan-spec |
| 5. Eval-driven dev | Transition-level signals, replay sets | ~10 min | ADR's eval-signal requirement |
| 6. §0 retro discipline | Spec-driven dev, carry-forwards | ~10 min | §0 runs first in the PM plan-spec block |
| 7. HITL #3 ADR | Gate types, ADR anatomy, primitives | ~10 min | The Plan-Day deliverable |

## 3. Threading

- **HITL programme thread:** Touchpoint **#3 of 7** — Plan-Day ADR. (#1 = W1 Fri LLM Essentials; #2 = W2 Thu RAG fallback; #4 = W3 Wed multi-agent handoffs; #5 = W3 Thu hard interrupt.)
- **Phase thread:** Phase 1 (AI Adoption) — final week. Phase-1 Defense lands Fri.
- **Pair-project:** Each pair commits a HITL #3 ADR in their pair-repo by 17:00 today. Three required ADRs: single-vs-multi-agent · LangGraph as framework · HITL #3 interrupt-node boundaries.
- **Decision anchors:** D-033 (LangChain v1.0), D-040 (Wed research slot for scenario-alternatives), D-043 (HITL 7-touchpoint thread), D-044 (Karsun-aspect anchoring), D-060 (`POST /agent/intake-triage` Mon–Tue flow).

## 4. Why today matters

W2 ended with a working RAG layer over the FAR/DFARS corpus. W3 turns that retrieval into something the platform can *act on*. The CO's framing anchors the week: 22 proposals arrived in a 90-minute TEP-week window; manual triage missed two amendments. She needs an automated intake-triage flow — and she has one non-negotiable constraint.

Today you translate that constraint into architecture. Topology and boundaries before code. The six pre-session topics are not independent — they form a single design chain: irreversibility (Topic 2) justifies the gate, state-machine model (Topic 3) gives it a home, memory (Topic 4) keeps it resumable, evals (Topic 5) make it measurable, §0 retro (Topic 6) keeps the spec honest, and the ADR (Topic 7) is the deliverable.

> [!IMPORTANT]
> **War-room anchor (10:00–12:00).** The CO opens the war-room with a single sentence: *"I — and only I — fire the irreversible actions."* Your job: sketch the agent topology on the whiteboard, label each transition reversible or irreversible, and walk into the PM plan-spec block with a draft HITL #3 ADR. This is the gate-design muscle you need to carry through W3 Wed (#4) and Thu (#5).

## 5. How to read this

- Read topics 2–7 in order — each builds on the prior.
- Self-checks at end of each topic — take 30s before expanding answers.
- Deeper-dives optional but recommended for senior FDEs.
- Hands-on exercises feed into morning war-room.
- Total expected time: **~80 min at 100 wpm**.

> [!TIP]
> **Plan Day starts with §0 retro.** The first 15 minutes of the PM plan-spec block are §0 — re-read last week's spec before writing the new one. Do not start the three ADRs until §0 carry-forwards are named. This is the living discipline the cohort practises every Plan Day (D-029, D-036).

> [!CAUTION]
> **Cross-topic trap:** The three HITL touchpoints this week (#3 Mon, #4 Wed, #5 Thu) are distinct shapes — ADR design, soft-interrupt implementation, hard-interrupt + FAR audit row. Don't collapse them today. Import only intake-triage scope into the HITL #3 ADR; FAR 15.308 (SSA non-delegation) belongs to HITL #5 on Thu.

## 6. Two questions to walk in with tomorrow

1. Which transitions in your pair-project's intake-triage topology are genuinely irreversible — and what external system observes the action?
2. For each gate you commit in today's ADR (soft or hard), which FAR clause or reversibility argument anchors it — and what eval signal tells you the gate is calibrated right?

<details>
<summary>Topic-to-war-room map</summary>

- Topic 2 → War-room whiteboard (~20 min): Classify transitions as one-way/two-way doors; label irreversibility.
- Topic 3 → War-room topology sketch (~20 min): Draw state machine, place HITL nodes.
- Topic 4 → Plan-spec PM (~10 min): Memory scope decision lands in §4 Approach.
- Topic 5 → Plan-spec PM (~10 min): Each ADR names its eval signal.
- Topic 6 → Plan-spec PM §0 (~15 min): §0 retro runs before the three ADRs.
- Topic 7 → Plan-spec PM §3 (~30 min): HITL #3 ADR is the anchor deliverable.

</details>

<details>
<summary>Consolidated sources</summary>

- LangChain v1.0 + LangGraph: research/langchain-v1-20260522.md — retrieved 2026-05-22
- Bedrock InvokeModel: research/bedrock-claude-catalog-20260522.md — retrieved 2026-05-22
- FAR 15.206 + FAR 15.308: acquisition.gov — retrieved 2026-05-26 (FAC 2026-01 effective 2026-03-13)
- LangGraph interrupts: https://docs.langchain.com/oss/python/langgraph/interrupts — retrieved 2026-05-26
- LangGraph persistence: https://docs.langchain.com/oss/python/langgraph/persistence — retrieved 2026-05-26
- Anthropic Building Effective Agents: https://www.anthropic.com/research/building-effective-agents — retrieved 2026-05-26

</details>

Last verified: 2026-06-06
