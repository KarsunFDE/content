---
template: weekly-retro
week: W04
scope: <pair-1 | pair-2 | pair-3 | cohort>
written_at: 2026-06-19 (Fri W4 EOD)
prs_merged_this_week: <integer>
codex_findings_unaddressed_carried_forward: <integer>
scenario_alternatives_completed: <int — W04-SA-1/2/3, ADRs due Thu EOD; Live Defenses move to W5 Fri>
assessment_results: { mcq_tier: <mid|entry>, mcq_score: <pct>, live_defense_pass: <bool — scoped to amended plan-spec post-Surprise>, design_planning_pass: <bool> }
artifacts_flagged_stale_90d: [<list>]
last_verified: 2026-05-23
retro_shape: post-incident (Mid-Sprint Surprise replaces standard retro framing)
---

# W04 Weekly Retro — Post-Incident framing

> **W4 retro shape differs from W1/W2/W3.** The Mid-Sprint Surprise is the dominant artifact of the week — Friday's incident IS the retro material. Standard `templates/weekly-retro.md` sections still apply, but §4 and §5 specifically reflect post-incident framing per `war-room/Fri-mid-sprint-surprise.md` §7 debrief prompts.

## 1. By the numbers

| Metric | Value |
|--------|-------|
| PRs merged this week (acquire-gov + pair-project) | <int> |
| Codex P0/P1 findings unaddressed (carried into W5) | <int> |
| Scenario-alternatives ADRs completed (W04-SA-1/2/3) | <int of 3> |
| Modernization PRs reaching `stage-4.0` green CI | <int of 3 — one per pair / service> |
| Future-hop ADRs with Pass 3 evidence (Java 25 + SF7 minor releases) | <int of 6 — 2 per pair> |
| AI Security PRs closing Item 9 (4 stored-content surfaces) | <int of 4> |
| HITL #6 authority-boundary table complete (8 endpoints × 4 cols) | <yes/no> |
| Mid-Sprint Surprise: time to first symptom diagnosis | <minutes — target ≤20> |
| Mid-Sprint Surprise: time to root-cause identification (Item 3) | <minutes — target ≤40> |
| Mid-Sprint Surprise: audit-gap (Item 2) noticed unprompted | <yes/no> |
| Mid-Sprint Surprise: 1-line bulkhead shipped + W5 tickets opened | <yes/no> |
| Mid-Sprint Surprise: OIG-facing answer drafted with calibrated uncertainty | <yes/no> |
| Live Defense (amended plan-spec) outcome | <pass / not-pass> |
| MCQ score (senior tier) | <pct> |
| MCQ score (entry tier) | <pct> |
| Scenario Design Planning (Mon) outcome | <pass / not-pass> |

## 2. What went well

- <Specific. Reference a PR, a Mid-Sprint Surprise discipline moment, or a Wed AI Security exercise.>
- <Specific. Was the §0 retro on W3 Phase 1 Defense sharply argued?>
- <Specific. Did the Tue debt-item-12 discovery land cleanly — finding filed, lint re-enabled, fallout owned?>

## 3. What did not

- <Specific. Where did codex Full strictness catch something the cohort would have called acceptable at Ramping?>
- <Specific. Did the Mid-Sprint Surprise expose a planning gap from Mon? Name the gap.>
- <Specific. Did any pair confuse Wed JWT-skip with Fri load incident? That's a curriculum-delta-check failure — name it.>

## 4. Post-incident framing — the Mid-Sprint Surprise debrief (cohort-scoped retros)

Per `war-room/Fri-mid-sprint-surprise.md` §7, instructor probed each pair with the 6-question debrief during Live Defense. Aggregate observations:

- **At 09:15, what did the cohort know vs suspect?** <observation across pairs — be specific about which pair leaned hypothesis-first vs symptom-first>
- **The 1-line bulkhead vs full Resilience4j decision.** <which pair shipped the bulkhead; which pair tried to do more and broke something; which pair shipped nothing and that's a finding>
- **Did the cohort tell the Karsun delivery lead the SSDD-sign event might not be in the audit log?** <yes/no/when — calibrated uncertainty was the artifact>
- **OIG observer cover-story reveal.** <how did cohort react to the reveal that the observer was cover-story; did they accept the chaos-injection framing>
- **W5 Mon ticket for the audit-log race detector.** <was it actually written by EOD; did it have acceptance criteria>
- **3-month-later SSDD-proof question.** <which pair had a defensible answer; which pair froze>

## 5. FDE situations this week stressed

Per `pipeline/COVERAGE.md` FDE-situation mapping:

- **Situation 3 (Architectural commits under pressure):** Mon Plan Day + Thu OpenRewrite hop + Fri amended plan-spec. <how the cohort handled each>
- **Situation 5 (Spec-driven dev discipline):** Tue workshop discovery. <how the cohort responded to "your PRs weren't actually linted">
- **Situation 6 (Cross-service correlation under failure):** Fri Mid-Sprint Surprise. <did the cohort hit the W3C-traceparent gap as the proximate W5 deliverable>
- **Situation 7 (Production-quality LLM under load):** Fri surprise — Item 3 is LLM10:2025 in practice. <did the cohort name this connection>
- **Situation 8 (Brownfield modernization):** Thu OpenRewrite hop (SB 2.7→4.0 + J11→21). <which pair reached `stage-4.0` green CI>
- **Situation 9 (Security threat modeling):** Wed AI Security Day. <was the OWASP 2025 vs 2023 ID delta cleanly handled>
- **Situation 11 (HITL discipline under authority gating):** Wed HITL #6 + Fri amended-plan-spec. <was HITL #6's authority-boundary table actually defended Fri>

## 6. Carry-forward into W5

> W5 is AIOps anchor week (per CLAUDE.md programme grid). The Mid-Sprint Surprise carry-forward is the load-bearing W5 hand-off.

- Codex findings still open (modernization + AI Security PRs): <list with PR links>
- Scenario-alternatives Live Defenses (W04-SA-1/2/3) deferred to W5 Fri: <confirm scheduled>
- W5 Mon Plan Day §0 retro must absorb: (a) amended plan-spec from this Friday, (b) audit-log race detector design (W5 Wed Datadog AI hands-on per D-060), (c) Resilience4j W5 anchor work, (d) W3C traceparent end-to-end fix
- Pair commitments restated: <list — each pair names what they own coming into W5>
- Any planning-artifact re-do scheduled: <yes / no — date; rare for W4>

## 7. Pipeline-health flags (cohort-scoped retros only)

- Artifacts older than 90 days still in use: <list with paths>
- Research briefs flagged for refresh by `tech-research`: <list — OWASP LLM Top 10 should re-verify pre-W5 in case OWASP publishes 2026 update>
- War-room scenarios used twice: <should be zero — Mid-Sprint Surprise re-use is expected for Cohort #2+ per D-049, but within Cohort #1, MSS ran once>
- Pipeline templates needing revision based on this week: <list — e.g., did `templates/scenario-design-planning.md` need a W4-specific §0 retro callout that the W4 brief surfaced as missing?>

## 8. Decision-log update candidates

- Did the Fri Mid-Sprint Surprise expose anything that should become a D-061+ decision? Candidate areas: pair-project per-pair-unique debt collision with the canonical load-incident shape; whether the OIG observer cover-story should be made literal (actual instructor-played observer); whether the 1-line bulkhead should be promoted to a defined intermediate state in the chaos-injection.

## 9. One-line summary for the cohort board

> <One sentence the cohort sees on Monday morning. Honest, specific, forward-looking. Example: "W4 closed Phase 2 modernization scope cleanly; the Mid-Sprint Surprise exposed the W5 audit-log race + Resilience4j hand-offs; HITL #6 authority-boundary table is binding for W5 AIOps work.">
