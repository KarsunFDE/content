---
week: W03
released_at: Wed 10 Jun 2026 09:00 (morning of scenario-research slot per D-040)
adr_due: EOD Thu 18 Jun 2026 (W4 Thu — after Phase 1 Gate)
live_defense_scheduled: Fri 19 Jun 2026 (W4 Fri — but note this collides with Mid-Sprint Surprise per D-049; instructor calls timing day-of)
last_verified: 2026-05-23
---

# W3 scenario-alternatives — index

Per D-040, W3 has 2-3 scenario-alternatives prompts released on the dedicated Wednesday research slot. ADR deliverable lands inside W4 (post-Phase-1-Gate); Live Defense scheduled inside W4 *if scheduling allows* — the W4 Fri Mid-Sprint Surprise (per D-049 — acquire-gov prod incident) takes priority and Live Defense may slide to W5 Fri at the instructor's call.

| # | Title | Constraint shape | Pair lead |
|---|-------|-------------------|-----------|
| W03-SA-1 | Single-agent vs multi-agent for intake-triage | CO operational simplicity vs evaluator-flow consistency | TBD (assigned Wed morning) |
| W03-SA-2 | LangGraph vs LangChain agents vs hand-rolled | Framework lock-in + ai-orchestrator FastAPI integration | TBD |
| W03-SA-3 | KG/CG representation — Neo4j vs Postgres recursive CTE vs NetworkX in-process | CO query latency + ops budget + FedRAMP boundary | TBD |

## Pair-collaborative mode (per D-040)

Each pair takes ONE scenario as their lead. The other two scenarios get a *brief* pair-look during the Wed afternoon slot (not a full ADR — just a one-paragraph "if we had to defend this, here's our gut take"). Pair-lead scenarios get full ADRs by EOD Thu W4. The brief looks become input for the W4 Mon Plan Day cross-pair discussion.

## Live Defense scheduling note

Per `pipeline/PIPELINE.md` §12 + the W3 PLAN.md: Live Defense Friday slots are reserved for *prior week's* scenarios. W3 scenarios' Live Defense is scheduled for **W4 Fri 19 Jun 2026** — BUT the W4 Mid-Sprint Surprise (per D-049 — acquire-gov prod incident) is also Fri. Instructor judgment day-of:

- If the Mid-Sprint Surprise wraps cleanly by 14:00, Live Defense runs 15:00–17:00 (compressed: 1 defender per pair × 10 min × 3 pairs + 30 min retro).
- If the Mid-Sprint Surprise consumes the day, Live Defense slides to **W5 Fri 26 Jun 2026** which is the Final Adversarial Review PR day — Live Defense runs in the morning, Final Adversarial Review PR in the afternoon.
- Either way, the W3 scenario ADRs are *due* EOD Thu W4 regardless of when the defense fires.

Cohort #1 calendar: Phase 1 Gate Fri 12 Jun → W4 starts Mon 15 Jun → W3 scenario ADRs due Thu 18 Jun → tentative Live Defense Fri 19 Jun.
