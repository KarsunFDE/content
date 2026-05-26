---
week: W04
day: Fri
topic_slug: amendment-tradeoff-evaluations
topic_title: "Amendment tradeoff evaluations — the severity-by-effort matrix, heroism vs avoidance, what gets cut"
parent_overview: W04/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://canny.io/blog/product-prioritization-frameworks/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://userpilot.com/blog/feature-prioritization-matrix/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://upstat.io/blog/incident-anti-patterns
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.rock.so/blog/eisenhower-matrix
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://rootly.com/sre/2025-sre-incident-management-best-practices-checklist
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Amendment tradeoff evaluations — the severity-by-effort matrix, heroism vs avoidance, what gets cut

## 1. Learning Objectives

By the end of this reading, the learner can:

- Apply a 2×2 **severity-by-effort** tradeoff matrix to the staged work coming out of a mid-sprint event, and explain why each quadrant has its own correct disposition.
- Name and describe the three common cohort-failure modes — **heroism**, **avoidance**, and **spec drift** — and the structural reasons each appears under stress.
- Distinguish **legitimate deferral** (staged, owned, tracked, ADR-amended) from **avoidance** (everything pushed off without a tracking artifact), and explain why the documents look similar but the outcomes are different.
- Articulate why the EOD post-incident retro is a separate artifact from the impact assessment and the postmortem, and what each is for.
- Explain why the discipline of *what gets cut* is harder than the discipline of *what gets shipped*, and where the cognitive load comes from.

## 2. Introduction

The hardest engineering call of a mid-sprint event afternoon is not "what shall we fix?" but "what shall we *not* fix today?" The fixes a team has energy and time for, by mid-afternoon, are usually a subset of what the amended plan-spec wants to ship. Choosing the subset is the **amendment tradeoff evaluation** — and the discipline behind it distinguishes a team that ships sustainable engineering work from one that ships a Friday-afternoon mess and pays for it on Monday.

The decision frame is a 2×2 matrix with two axes — **severity** and **effort** — borrowed from the prioritization-matrix tradition ([Canny 2026](https://canny.io/blog/product-prioritization-frameworks/); [Userpilot](https://userpilot.com/blog/feature-prioritization-matrix/); [Eisenhower](https://www.rock.so/blog/eisenhower-matrix/)) and sharpened for engineering work coming out of an incident. Each quadrant has one correct answer; the work is putting each candidate in the right quadrant before reading off the answer.

The matrix is half the discipline. The other half is **recognising the failure modes the team is most likely to slip into under fatigue.** 2026 incident-management literature ([Rootly](https://rootly.com/sre/2025-sre-incident-management-best-practices-checklist); [Upstat](https://upstat.io/blog/incident-anti-patterns)) names three: heroism, avoidance, and spec drift.

## 3. Core Concepts

### 3.1 The severity-by-effort 2×2 matrix

Map every candidate amendment item onto two axes. **Severity** = residual user-visible impact if unfixed (high if users still see degraded behaviour; low if existing mitigation has contained the symptom). **Effort** = work to ship today including review and validation (low for narrow hot-patch in one file; high for cross-cutting change across services or schema migration).

|  | Low effort | High effort |
|--|-----------|-------------|
| **High severity** | **Ship today (Q1, priority 1).** Narrow, reversible, validated. Textbook narrow-hot-patch path. | **Stage for next sprint with named owner + ADR amendment (Q2, priority 2).** Keep mitigation running through the weekend. Do *not* widen the patch shape under pressure. |
| **Low severity** | **Bundle into next clean PR (Q3).** Document the gap; do not rush. Rides along with planned next-sprint work. | **Stage and deprioritise (Q4).** Note in retro that the available budget did not reach this. May surface only if other signals re-elevate severity. |

Structural property: **only one quadrant produces a same-day ship.** Three of four quadrants stage. The matrix is calibrated so "ship today" is the rare default — review bar doesn't relax under stress, and most candidates won't survive strict review at 16:45.

The Eisenhower formulation ([2026](https://www.rock.so/blog/eisenhower-matrix/)) maps the same shape: Q2 (important but not urgent) is the protected-roadmap slot, and the discipline is to *defend* Q2 from collapsing into Q1 under heroic pressure.

### 3.2 Failure mode 1 — heroism

**Heroism** is the pair trying to ship every fix before 17:00. The pull comes from the instinct "we should be able to fix this today" — correct for low-effort-high-severity work, wrong for the other three quadrants, where it produces:

- A wide patch that doesn't pass review at 16:50 (ships unreviewed or fails to ship).
- A multi-service change the team can't fully validate before EOD, deploying state with unknown coupling.
- A "real fix" rushed against the calendar that breaks something else by Saturday morning.

2026 literature ([CSA](https://cloudsecurityalliance.org/blog/2026/04/23/rethinking-incident-response-as-an-engineering-system-addressing-7-operational-gaps); [Brent Chapman](https://greatcircle.com/blog/2026/04/21/self-blame-isnt-blameless/)) treats heroism as a culture-level failure: valorising "heroes who saved the system" produces incentives for heroism next time. The counter-discipline: **publicly value the team that staged appropriately** — write "staged with named owner and ADR amendment" into the retro as a *positive* outcome.

Structural test: at 16:30, can the pair point at a passing strict-review of every fix about to ship? If yes, ship; if not, stage.

### 3.3 Failure mode 2 — avoidance

**Avoidance** is the inverse: stage everything, ship nothing. The pull is over-correction to heroism — "don't rush" rotated into "ship nothing." The outcome is also bad:

- Impact assessment written, but no narrow-hot-patch shipped even when the low-effort-high-severity quadrant had viable work.
- Existing mitigation (flag, rollback) left running indefinitely, accumulating operational debt.
- Team confidence drops — responded all day with no shipped artifact.

The [Rootly 2025 SRE checklist](https://rootly.com/sre/2025-sre-incident-management-best-practices-checklist) rule: **at least one critical fix or one impact assessment must ship by EOD.** When every fix is correctly staged, the impact-assessment itself is the deliverable. Avoidance is failing to recognise the artifact-shipped *is* a same-day shipment.

Structural test: at EOD, does the team have *at least* a fully-written impact assessment and (where applicable) one narrow-hot-patch shipped?

### 3.4 Failure mode 3 — spec drift

**Spec drift** is the subtlest: fix code but never amend the Monday ADRs. Code in `main`; Monday's spec unchanged; the two diverged silently. Git history shows what changed in *code*; nothing shows what changed in *intent*.

The pull is cognitive load — writing an amendment block requires opening the ADR and writing four structured paragraphs. Under fatigue: "we'll do it Monday." Counter-discipline: **treat the amendment as part of the fix shipment, not a follow-up.** A fix without its ADR amendment is a partial shipment, like code without tests. PR checklist includes "ADR amendment authored at {path}" as required. Structural test: for every shipped fix, is there a corresponding ADR amendment in the same or paired PR?

### 3.5 The EOD post-incident retro — separate from the postmortem

The day-closing retro is **not the postmortem**. Three artifacts on three timescales: **impact assessment** (same-day, five-slot, engineering record by the pair); **EOD retro** (same-day, prompt-driven, *meta* record of how the team performed); **postmortem** (days later, durable narrative for the wider organisation).

The retro captures the team's **mental-model shift** before it fades. Classic prompt: *"What changed in your pair's mental model of 'production' between 09:00 and 17:00 today?"* — a coaching artifact for next week's plan and the next on-call's pre-load. Skipping it means the qualitative shift evaporates.

### 3.6 What gets cut is harder than what gets shipped

The deepest discipline is **cutting**. The team has energy for a known fraction of the available work; choosing which fraction is structurally hard because (a) loss aversion is asymmetric — the fix the team cut is visible as "we didn't ship X," but the Saturday-morning incident heroism would have produced is invisible; (b) status pressure rewards visible work — three fixes "looks" more productive than one fix + three structured stagings; (c) documentation burden falls on the stager — tickets, ADR amendments, owner assignments don't *feel* like engineering but are.

Counter-discipline: **treat staging artifacts as first-class engineering output.** The retro counts "staged with named owner + ADR amendment" as a same-day shipment, equal to a code change.

## 4. Generic Implementation

A canonical amendment-tradeoff worksheet, framework-agnostic, looks like the following. Each candidate item from the amended plan-spec gets one row.

```markdown
# Amendment Tradeoff Worksheet — {incident id}

## Candidates from the amended plan-spec
| # | Item | Severity (high/low) | Effort (low/high) | Quadrant | Disposition | Owner | Tracking |
|---|------|---------------------|--------------------|----------|-------------|-------|----------|
| 1 | {short name of item} | high | low | Q1 ship-today | SHIP | {name} | {ticket/deploy} |
| 2 | {short name} | high | high | Q2 stage-priority-2 | STAGE | {name} | {ticket} |
| 3 | {short name} | low | low | Q3 bundle-next-PR | BUNDLE | {name} | {ticket} |
| 4 | {short name} | low | high | Q4 stage-deprioritise | STAGE-DEPRIO | {name} | {ticket} |

## Failure-mode self-audit (run at 16:30)
- [ ] HEROISM check: every "SHIP" item passes strict review with reviewer available
  in the remaining window. If any does not, downgrade to STAGE.
- [ ] AVOIDANCE check: at least one of (a) impact-assessment artifact authored,
  (b) one SHIP item shipped. If neither, escalate to incident commander.
- [ ] SPEC-DRIFT check: every SHIP item has a corresponding ADR amendment in the
  PR or paired PR. If any SHIP item has no amendment, add it before shipping.

## EOD reflective prompts (15 min, written paragraph per prompt)
1. What changed in your pair's mental model of "production" between 09:00 and 17:00 today?
2. What single observable signal would have given your pair 15 more minutes earlier?
3. Which of your STAGE items concerns you most going into the weekend, and why?
4. What did you ship today that you would *not* have shipped a week ago, and what changed?
```

Deliberate properties: the quadrant is computed *before* the disposition is read off (forces classification rather than vibe-classification); the 16:30 self-audit is a checklist not a discussion (catches the pulls while correctable); the reflective prompts are written not discussed (writing captures the qualitative shift before it fades).

> [!instructor-review]
> **Severity/effort calibration is team-specific.** "Low effort" for a Spring Boot team with a strong Resilience4j muscle differs from one that has never used it. Working definitions should be set in the morning war-room and re-anchored each Friday — first-week definitions won't match fourth-week definitions.

## 5. Real-world Patterns

**Fintech — payment-incident afternoon tradeoffs.** A consumer-fintech team's incident produced four candidate amendment items by 14:00. The pair ran the worksheet: two items mapped to Q1 (high-severity, low-effort narrow patches in single validators), one to Q2 (a cross-service timeout reconfiguration), and one to Q4 (a refactor of a deprecated client library — large effort, low residual severity once Q1 shipped). They shipped Q1, staged Q2 with the SRE team as named owner, and Q4-deprioritised the refactor. *The EOD retro recorded the team felt the pull toward shipping Q2 under heroic pressure and explicitly resisted; the staged-with-owner outcome was treated as the day's third "shipment."*

**E-commerce — checkout-flow afternoon tradeoffs.** A retailer's incident produced six candidate items. The team's initial worksheet mapped four to Q1 and two to Q2 — textbook signal of over-confident classification. The pair did the 16:30 audit: of the four Q1 items, only two had reviewers available in the remaining window; two had natural diffs exceeding narrow-hot-patch shape. The pair downgraded those two to Q2. *The audit was the artifact — it converted the optimistic morning classification into a realistic afternoon shipment plan, before optimism produced a 16:55 unreviewed merge.*

Two further cases — gaming matchmaker afternoon (avoidance-check IC intervention) and healthcare clinical-rule afternoon (Q2-expanded-to-Q1 under compliance pressure) — appear in the [further-reading appendix](./6-amendment-tradeoff-evaluations-further-reading.md).

## 6. Best Practices

- **Run the 2×2 matrix explicitly, in writing, before deciding what to ship.** Writing the severity and effort columns forces defensible classification rather than vibe-based.
- **Treat Q1 as the rare default.** Three of four quadrants stage; the matrix makes "ship today" the exception.
- **Run the failure-mode self-audit at 16:30, every Friday.** Heroism, avoidance, and spec drift surface in the last 90 minutes; the audit catches them while correctable.
- **Count "staged with named owner + ADR amendment" as a same-day shipment in the retro.**
- **Write the EOD reflective prompts, do not discuss them.** Writing captures the qualitative shift before it fades.
- **Re-calibrate "low effort" and "high severity" weekly.** Team confidence and muscle-memory shift; the matrix quadrants shift with them.
- **Never collapse Q2 into Q1 under heroic pressure.** Q2 is the protected slot; collapsing produces the next Saturday-morning incident.

## 7. Hands-on Exercise

**Task (10–15 min):** Pair coming out of a mid-sprint event (NOT federal acquisitions — pick fintech, healthcare, gaming, e-commerce, or logistics). 14:00; mitigation stable. Five candidates A–E:

- **A.** Null-check in one validator (1 file, 4 lines).
- **B.** Cache-invalidation hook for the stale-data class (2 files, one service, ~40 lines).
- **C.** Circuit-breaker timeout reconfiguration across three services (multi-service config change).
- **D.** Renaming an internal-only debug endpoint (1 file, cosmetic).
- **E.** Refactor of deprecated client library (large, multi-day).

Walk the worksheet: (1) quadrant each candidate, (2) disposition each using the matrix, (3) run the 16:30 self-audit (reviewable in window? impact assessment or code shipment on track? every SHIP paired with ADR amendment?), (4) write the EOD reflective paragraph for prompt "What changed in your pair's mental model of 'production' between 09:00 and today's end?"

**What good looks like.** A → Q1 (ship), B → Q2 (stage with named owner), C → Q2 (multi-service is structurally not a same-day fix), D → Q3 (bundle into next clean PR), E → Q4 (stage + deprioritise). The 16:30 audit catches whether A's reviewer is actually available. The reflective paragraph names a *specific* shift — not generic "production is hard" — e.g., "we used to see cache TTL as a tuning knob; we now see it as a coupling surface between vendor data and user prices."

## 8. Key Takeaways

- **Two axes, four dispositions.** Severity × effort. Q1 (high/low) → ship; Q2 (high/high) → stage with named owner + ADR amendment; Q3 (low/low) → bundle into next clean PR; Q4 (low/high) → stage and deprioritise.
- **Why Q1 is the rare default.** Three of four quadrants stage. "Ship today" requires the rare combination of low effort *and* high severity *and* available review window.
- **Three failure modes.** Heroism (ship everything, ship fragile), avoidance (stage everything, ship nothing including artifacts), spec drift (fix code, never amend ADR). Each has its own counter-discipline and 16:30 self-audit check.
- **EOD retro vs. postmortem.** Retro captures the team's mental-model shift while fresh (qualitative, same-day, prompt-driven). Postmortem is the durable narrative for the wider org (multi-page, days later). Skipping the retro means the shift evaporates.
- **Why "what gets cut" is harder than "what gets shipped."** Loss aversion is asymmetric, status pressure rewards visible diffs, documentation burden falls on the stager. Counter-discipline: count staging artifacts as first-class engineering output.

## Sources

1. [Canny — 2026 Guide to Product Prioritization Frameworks](https://canny.io/blog/product-prioritization-frameworks/) — retrieved 2026-05-26
2. [Userpilot — Feature Prioritization Matrix 2026](https://userpilot.com/blog/feature-prioritization-matrix/) — retrieved 2026-05-26
3. [Upstat — Common Incident Anti-Patterns That Slow Response](https://upstat.io/blog/incident-anti-patterns) — retrieved 2026-05-26
4. [Rock.so — The Eisenhower Matrix (2026)](https://www.rock.so/blog/eisenhower-matrix/) — retrieved 2026-05-26
5. [Rootly — 2025 SRE Incident Management Best Practices Checklist](https://rootly.com/sre/2025-sre-incident-management-best-practices-checklist) — retrieved 2026-05-26

Last verified: 2026-05-26
