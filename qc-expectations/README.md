---
folder: qc-expectations
purpose: learner-facing prep briefs for the 4 QC audit + exam checkpoints
last_verified: 2026-06-03
---

# QC Audit Expectations — Cohort #1

Learner-facing prep briefs for the **four checkpoint assessments** in the Karsun-FDE 6-Week Intensive. Each brief is a 10-15 min read covering format, topics, example scenario shapes, and prep guidance.

> These briefs tell you **what to expect**. They do not contain the actual exam or audit questions. Memorising scenarios from these documents will not help you — internalising the *thinking shape* will.

## The four checkpoints

| # | Brief | Date | Covers |
|---|-------|------|--------|
| 1 | [Checkpoint 1](checkpoint-1.md) | Tue 9 Jun 2026 | W1 LLM Engineering Essentials + W2 RAG Architecture |
| 2 | [Checkpoint 2](checkpoint-2.md) | Tue 16 Jun 2026 | W3 Agentic Systems |
| 3 | [Checkpoint 3](checkpoint-3.md) | Mon 22 Jun 2026 | W4 AI-Native SDLC + AI Security + Brownfield Modernization |
| 4 | [Checkpoint 4](checkpoint-4.md) | Fri 26 Jun 2026 | W5 AIOps + Governance + Deployment Readiness |

C1 + C2 land on **Tuesday** in the standard checkpoint slot. C3 lands **Monday** (collides with W5 Plan Day); C4 lands the **Friday of W5** (the same day as the Final Adversarial Review PR). Both shift exam time to early afternoon. See each brief for specifics.

## Two assessment surfaces

| Surface | Audience | Format | When |
|---------|----------|--------|------|
| **Exam** | Trainee competency test, two-tier (Senior FDE / Entry FDE) | 90-min written, scenario-grounded MCQ + (Senior) short scenario | In-person, inside the 2-hr block on the checkpoint date |
| **Audit interview** | Leadership / external auditor view of each candidate's depth + decision-making | 30-min Zoom 1:1, candidate is pressed on tech + decisions | Async during the same week; you book your own slot |

Two auditors split the cohort 3-and-3 for each checkpoint. Audit interview is **interview pressure**, not artifact walkthrough — the auditor opens with a tech-grounded scenario, lets you work the surface for 3-5 minutes, then presses sideways through follow-ups.

## The 7 thinking lenses (used in audit follow-ups)

| # | Lens | Press shape |
|---|------|-------------|
| T1 | System Thinking | Whole-system impact, upstream/downstream, second-order effects |
| T2 | Architecture Thinking | Boundaries, coupling, evolvability, structural trade-offs |
| T3 | Design Thinking | API contracts, module/data shapes, failure modes in the response |
| T4 | Data & Cloud Thinking | Storage fit, cloud-native patterns, IAM, cost-shape |
| T5 | Reliability & Security Thinking | Failure modes, blast radius, OWASP awareness |
| T6 | SDLC & Delivery Thinking | Tests, CI, rollback, observability, branch hygiene |
| T7 | AI Thinking | LLM/RAG/agent-aware: grounding, eval, HITL, autonomy gating |

You don't need to memorise the labels. Recognise the *shape* of the press when it comes.

**AI Thinking floor per checkpoint** (minimum count of T7-tagged questions the auditor will pick for you):

| Checkpoint | AI Thinking floor |
|------------|-------------------|
| 1 | 4 — LLM + RAG is the content; AI Thinking is the spine |
| 2 | 5 — Agentic systems week; AI Thinking dominates |
| 3 | 3 — Modernization week; SDLC + Reliability also surface heavily |
| 4 | 3 — AIOps; Reliability + Cloud + SDLC carry heavier load |

## Tier handling

Both tiers see the same question bank in the audit. Each question has two bars — a **Senior bar** and an **Entry bar**:

- **Entry candidates** meeting the Entry bar confidently **meet bar**
- **Senior candidates** meeting only the Entry bar are **approaching bar** (not a fail, a flag)
- Either tier landing at the Senior bar **exceeds bar** at the Entry tier and **meets bar** at the Senior tier

You'll get a per-checkpoint composite verdict: **meets / partial / below**. Per-question feedback comes through your instructor afterward, not on the audit call.

## What the briefs do NOT contain

- The actual exam questions
- The actual audit questions
- Answer keys

Example scenarios in each brief are **deliberately distinct** from the curriculum's war-room incidents. Use them to test your opening, not as a study bank.

## Refresh cadence

These briefs update when:
- The checkpoint date moves (calendar adjustments)
- Hot-tech references go stale (auditor presses on technology that's evolved past what's in the brief)

`last_verified` frontmatter on each brief shows the most recent review date. If you're reading more than 30 days post that date, flag your instructor — federal-AI moves fast, and references may have drifted.

## How to use these briefs

1. **Skim the index** (this page) before any checkpoint week starts
2. **Deep-read the relevant brief** at the start of the week the checkpoint lands in — read it once on Monday, again the morning of the checkpoint
3. **Use the example scenarios as opening practice** — read one, set a 3-min timer, and verbalise (out loud, to a wall or a pair partner) how you'd open. Don't try to nail the "right" answer; nail the *first move*
4. **Use the prep checklists at the bottom of each brief** — they're ordered by leverage, not by topic

If you have a calendar conflict for the exam date or for booking an audit slot, raise it with your instructor **at least 24 hours before** the checkpoint.

---

*Briefs maintained by the programme lead. Cohort #1 of Karsun-FDE 6-Week Intensive.*
