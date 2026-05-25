---
template: weekly-retro
week: W03
scope: cohort (Mid-Program Retro — end of Phase 1)
written_at: 2026-06-12 (Fri W3, 13:00–15:00)
phase: end-of-Phase-1 (Mid-Programme Gate Day)
gate_outcome: <to fill — number of pairs Pass / Conditional / No-Pass>
prs_merged_this_week_cohort: <int>
codex_findings_unaddressed_carried_forward_cohort: <int>
scenario_alternatives_assigned: 3 (W03-SA-1, SA-2, SA-3 — ADR due Thu W4)
scenario_alternatives_completed: 0 (W3 ADRs land in W4 per scenarios/_index.md)
hitl_touchpoints_landed_cohort_wide: 5 of 7 (W1 Fri #1, W2 Thu #2, W3 Mon #3, W3 Wed #4, W3 Thu #5)
mcq_outcome: { senior_avg: <pct>, entry_avg: <pct>, senior_pass_count: <N>, entry_pass_count: <N> }
artifacts_flagged_stale_90d: []
last_verified: 2026-05-23
---

# Mid-Program Retro — End of Phase 1 (W3 Fri)

Written: 2026-06-12, 13:00–15:00. Scope: cohort-wide. Phase 1 (W1 Thu – W3 Fri) concludes.

> **Mid-Program Retro is the load-bearing retro of the programme.** It does three things at once: (a) closes Phase 1 by naming what was *adopted* and what was *discovered* (per D-039); (b) authors the input to W4 Mon §0 plan retrospective — the modernization plan is written *against* this retro; (c) tees up the second half of the programme (Phase 2 = W4–W6 modernization + AIOps + deliverability).

## 1. By the numbers — Phase 1 cohort-wide

| Metric | Value |
|--------|-------|
| Pairs Pass / Conditional / No-Pass (Phase 1 Defense) | <N> / <N> / <N> |
| Cohort PRs merged across Phase 1 (W1 Thu through W3 Fri, 12 working days) | <int> |
| HITL touchpoints landed cohort-wide so far | **5 of 7** (#1 W1 Fri, #2 W2 Thu, #3 W3 Mon, #4 W3 Wed, #5 W3 Thu) |
| Codex strictness progression | W1 Light → W2 Ramping → W3 Near-full (per D-034) |
| MCQ averages by tier (Senior / Entry) | <pct> / <pct> |
| Brownfield-debt items inventoried W1 Tue but not yet modernized | 12 of 12 (modernization starts W4) |

## 2. What worked across Phase 1

- **HITL thread is real.** Five touchpoints in three weeks; the cohort can name them by number and recognize them in code. (Probe: ask the cohort to list #1-#5 from memory. They should not need to look.)
- **§0 retro discipline.** Cohort ran it twice (W2 Mon retro'd W1; W3 Mon retro'd W2). By the W3 retro the pattern is internalized — D-036's spec-driven-as-living-discipline framing holds.
- **Codex calibration ramp worked.** Light W1 → Ramping W2 → Near-full W3 was the right shape; pairs internalized the pattern rather than getting overwhelmed by Day 3.
- **acquire-gov brownfield as substrate.** Real federal-acquisitions surface — FAR citations matter, OIG framing is live, multi-tenant boundary is concrete. The cohort hasn't been training against a toy app.
- <Add cohort-specific observations during the actual retro.>

## 3. What did not work across Phase 1

- <Specific. Reference where the cohort missed a Codex finding, where W2's RAG plan-spec was wrong about effort, where the W3 multi-agent shape took longer than estimated.>
- <Specific.>
- <Specific.>

## 4. Phase-1 discoveries — cohort synthesis (REQUIRED — drives W4 Mon §0)

> Per D-039 + D-058 (interlock pattern), the cohort synthesizes the discoveries from the three pair retros into a *cohort-wide* discoveries list. This list becomes the input to next week's W4 Mon §0 plan retrospective and explicitly drives the W4 modernization plan.

**Pair-level discoveries (rolled up from `W03-pair-N-retro.md` §6 across pair-1 + pair-2 + pair-3):**

1. <Discovery cluster 1 — likely about audit-log race (Item 2) catastrophic behavior under multi-agent fan-out, surfaced Wed W3.>
2. <Discovery cluster 2 — likely about correlation-ID gaps across the agent → tool → service → audit chain (Item 6), surfaced Tue W3 and exacerbated Thu W3.>
3. <Discovery cluster 3 — likely about multi-tenant filtering surface area (Item 10) exploding when you add LangGraph thread namespaces + tool-call filters.>
4. <Discovery cluster 4 — likely about LangChain v0.x legacy patterns (Item 5) still wired into 3 entry points; the cohort migrated some W2 but not all.>
5. <Discovery cluster 5 — likely about no-circuit-breaker eval→solicitation calls (Item 3) under TEP-week load; surfaced Wed W3, full incident Fri W4.>

**Cohort-level discoveries (cross-pair patterns the instructor + cohort name together):**

- <e.g., "Phase 1 surfaced that AI capability adoption *amplifies* every brownfield-debt item — Item 2 isn't 'a missed audit row' anymore, it's 'six missed audit rows per evaluation under TEP-week load'. Phase 2 modernization should sequence the debt-items that AI capability MOST amplifies.">
- <e.g., "Phase 1 surfaced that HITL gates are useless without the audit row schema being right. The first thing W4 Wed (HITL #6 AI Security) needs is the OWASP LLM06 framing for *why* HITL is the answer to Excessive Agency, not just the framework primitive for pausing execution.">

## 5. HITL thread reflection (5 of 7 landed)

The cohort has now seen HITL in five explicit shapes:

| # | Week | Shape | Cohort tag for it |
|---|------|-------|-------------------|
| 1 | W1 Fri | LLM Essentials — flag-and-review surface for hallucinated outputs | <how the cohort talks about it> |
| 2 | W2 Thu | RAG fallback — escalate to CO when retrieval misses / faithfulness drops | <cohort tag> |
| 3 | W3 Mon | Plan Day ADR — interrupt-node boundaries committed BEFORE code | <cohort tag> |
| 4 | W3 Wed | Multi-agent handoff — supervisor's worker invocation reviewed by SSA | <cohort tag> |
| 5 | W3 Thu | LangGraph deep-dive — hard `interrupt_before` for FAR-anchored gates with 18-hour resume | <cohort tag> |

**Coming next:**
- #6 W4 Wed — AI Security framing (OWASP LLM06 Excessive Agency).
- #7 W5 Wed — AIOps auto-remediation authority boundary.

**Question for the cohort:** of these five, which felt *most natural* to land? Which felt *most artificial*? Why? (The answers shape how the instructor frames #6 + #7.)

## 6. Pipeline-health flags (cohort-scoped)

- Artifacts older than 90 days still in active use: <list with paths>
- Research briefs flagged for refresh by `tech-research`: <list — likely none at end-of-Phase-1>
- War-room scenarios used twice: <should be zero — list if any>
- Pipeline templates needing revision based on Phase 1: <list with suggested edits — likely: extend `live-defense-rubric.md` to formalize the Gate variant for W6>
- Density check: Tue/Wed/Thu ran at 8 topics (W3 cap). Did the cohort feel it? If yes, demote one topic for cohort #2 W3.

## 7. Cohort retro probe questions (instructor reads aloud during the 13:00–15:00 block)

After the by-the-numbers + discoveries synthesis, the instructor poses these:

1. *"Across Phase 1, what discovery are you most confident about? What discovery are you most worried you're WRONG about?"* (Calibration check.)
2. *"Which of the five HITL touchpoints felt most natural? Most artificial?"* (Feeds W4 Wed + W5 Wed framing.)
3. *"Where did Claude Code help most across Phase 1? Where did it actively hurt?"* (Cohort-pattern feedback.)
4. *"What's the one thing about acquire-gov modernization (W4-W6 scope) you're most uncertain about heading into Phase 2?"* (Sets up weekend reading.)
5. *"If you had Phase 1 to do over with the knowledge you have now, what would you skip? What would you spend MORE time on?"* (Spec-driven dev meta-retro.)

## 8. EOD Friday actions (15:00–16:30)

- 3 pair-retro files finalized + committed.
- This file finalized + committed.
- **Phase 1 freeze tag** on each pair-project repo (`phase1-freeze-pair-N-20260612` per D-056).
- W3 MCQ administered (30 min — see `assessments/W03-MCQ.md`).
- W4 Mon Plan Day briefing distributed to cohort channel for weekend skim.
- Cohort-wide one-line summary committed to the cohort board (see §9).

## 9. One-line summary for the cohort board (Monday W4 morning)

> <One sentence the cohort sees Monday W4 morning. Honest, specific, forward-looking. E.g., "Phase 1 shipped — three pairs defended AI Adoption with real FAR-anchored HITL gates; Phase 2 starts Mon with audit-writer consolidation + correlation-ID rollout against acquire-gov as the cohort's #1 and #2 modernization priorities.">

## 10. References

- `pipeline/DECISIONS.md` D-038 (Phase 1 = W1 Thu – W3 Fri), D-039 (defend what was adopted + what was discovered), D-043+D-044 (HITL thread), D-060 (both flows sequentially this week).
- `pipeline/PIPELINE.md` §13 (assessment battery), §14 (`weekly-retro` skill).
- `assessments/W03-Phase1-Defense-rubric.md` — gate rubric whose outcomes populate §1 above.
- `training-project/feature-inventory-target.md` §W3 (anchor features touched).
- W4 Mon `pre-session/Mon.md` (when Agent E authors it) — should reference §4 of this file as the W4 §0 retro input.
