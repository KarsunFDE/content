---
week: W04
day: Fri
topic_slug: impact-assessments
topic_title: "Impact assessments — the five-slot artifact that turns chaos into a defensible record"
parent_overview: W04/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 13
sources:
  - url: https://sre.google/sre-book/example-postmortem/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://sre.google/workbook/postmortem-analysis/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://sreschool.com/blog/impact-assessment/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://oneuptime.com/blog/post/2026-01-30-impact-assessment/view
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://lightrun.com/blog/blast-radius-analysis/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Impact assessments — the five-slot artifact that turns chaos into a defensible record

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define an **impact assessment** as a structured per-incident artifact, distinct from a postmortem, and explain when each is the right deliverable.
- Name the five canonical slots of an impact assessment — symptom, root-cause hypothesis, confirming/refuting evidence, user-visible impact, hand-off scope — and explain why each is load-bearing.
- Distinguish **symptom** from **root cause** and explain why writing the symptom first prevents the most common diagnostic tunnel-vision.
- Explain why an impact assessment scores **honesty and calibration**, not "did the team fix it" — and how that scoring shapes the artifact's content.
- Articulate why the impact assessment is the substrate for downstream artifacts (postmortem, ADR amendment, follow-up ticket) rather than a parallel document to them.

## 2. Introduction

The artifact that usually survives an incident is a postmortem written days later, when the chaos has been sanded into a tidy narrative. The postmortem is useful but answers "what happened?" — when stakeholders on Monday morning are often asking a different question: **what does this *mean* for us, and what is staged for next sprint?**

The **impact assessment** is the artifact that answers the second question. It is a small, structured, same-day deliverable — typically one page — written by the responding pair while the incident is still warm. It converts the lived experience into five slots a reviewer can read in three minutes: symptom, root-cause hypothesis, confirming/refuting evidence, user-visible impact, hand-off scope. The slots are deliberately small because the artifact is meant to be readable by stakeholders who were not in the war room.

The discipline descends from SRE postmortem practice ([Google SRE Postmortem Analysis](https://sre.google/workbook/postmortem-analysis/)) but is structurally lighter. A postmortem is comprehensive — timeline, causes, lessons, action items, six to eight pages. The impact assessment is **the next-day handoff**: what does the team that was not on call need to know, in the order they need to read it. 2026 practitioner writeups ([OneUptime](https://oneuptime.com/blog/post/2026-01-30-impact-assessment/view); [SRE School](https://sreschool.com/blog/impact-assessment/)) emphasise that a multi-dimensional structured assessment is what makes the incident's downstream consequences legible to people who were not present.

## 3. Core Concepts

### 3.1 Slot 1 — Symptom (what users / stakeholders / observers actually saw)

The symptom slot is filled with **evidence**, not hypotheses. Cite the surface — Grafana panel, log line, dashboard row, screenshot — that an outsider could open and reproduce. The criterion: **someone who was not in the war room can verify it independently** using the artifacts named in the slot.

Common failure modes:

- **Hypothesis dressed as symptom.** "The database was slow" is a hypothesis; the symptom is "p99 latency on the orders endpoint rose from 120ms to 1.4s between 14:32 and 14:47."
- **Vague timing.** "Around 2pm" is not adequate; "14:32 UTC, first elevated metric on {service} panel {N}" is. The timeline is forensic; rounding loses signal.
- **Missing the non-symptom.** Name what was *not* anomalous ("the database panel was green; the elevation was on the API service"). The non-symptom rules out tunnels that would have wasted time.

The Google SRE example postmortem ([Shakespeare outage](https://sre.google/sre-book/example-postmortem/)) opens with exactly this discipline.

### 3.2 Slot 2 — Root-cause hypothesis (one explanation, named)

The hypothesis slot contains **exactly one sentence**: the working explanation that accounts for every observed symptom. Not three hypotheses with caveats. Not a forensic narrative.

The one-sentence constraint exists because the hypothesis must be **falsifiable**. A hypothesis that says "it might be the database, the cache, or the new validator" is not falsifiable. Practitioner guidance from 2026 ([SRE School](https://sreschool.com/blog/impact-assessment/)) names this failure mode "plural-hypothesis paralysis."

When the team genuinely does not know the root cause yet, the slot says so honestly: "Working hypothesis: validator regression in `service-X` from deploy at 14:21; not yet confirmed. Alternative under investigation: downstream dependency timeout in `service-Y`." That is two hypotheses, but a *named* set with primary, alternative, and unresolved status explicit. Honesty about the state of knowledge, not theatrical certainty.

### 3.3 Slot 3 — Confirming/refuting evidence

The evidence slot contains the **specific data point** that confirmed or refuted the working hypothesis. *Hypothesis without evidence is a guess; evidence without hypothesis is noise.* The slot binds them. Three required sub-elements:

1. **The confirming data point.** Specific log line / metric query result / test output, with timestamps and source.
2. **The refuting data points considered and ruled out.** If the team also considered hypothesis B, the data point ruling B out belongs here. This is the artifact's defence against second-guessers.
3. **The unknowns.** Things the team could not test in the window, with reason.

A casual summary says "we figured out it was the validator." The assessment says "hypothesis was validator regression; confirming data point was the unit test on the null-handling path, run against the new commit and seen failing at line N; we ruled out the downstream-dependency hypothesis because that dependency's p99 was flat throughout the window."

### 3.4 Slot 4 — User-visible impact (concrete terms)

The impact slot names the cost in **concrete, audit-trail-readable terms**. Vague impact ("users had a degraded experience") is worse than no impact statement; concrete impact ("32% of orders between 14:32–14:47 returned 5xx and were retried by the client at 14:48") is the deliverable.

Three sub-dimensions ([Lightrun](https://lightrun.com/blog/blast-radius-analysis/); [OneUptime](https://oneuptime.com/blog/post/2026-01-30-impact-assessment/view)):

- **User impact.** How many users, how badly, in what way?
- **Service propagation.** How many upstream/downstream services pulled in? Local or cascade?
- **Business / compliance impact.** Audit-trail completeness, SLA, regulatory commitment, financial reconciliation? Regulated industries treat the assessment as a compliance artifact at this dimension.

### 3.5 Slot 5 — Hand-off scope (what is staged forward)

The fifth slot converts the lived incident into a **forward-looking commitment**: what work is staged, who owns it, when it will be picked up, against which artifact (ADR, runbook, sprint backlog).

Discipline: every staged item has a **named owner** and a **named tracking artifact**. "We will improve our monitoring" is not an item; "the {team} will add a panel for {metric} to the {service} dashboard by {date}, tracked as {ticket id}" is. The named owner + named artifact together make the staged item trackable. Without them, staged work has a half-life of three sprints before forgotten.

The hand-off slot connects to the fix-or-stage decision (separate topic). If today's mitigation is feature-flag-off and the underlying fix is staged, the slot records that decision for whoever picks up the proper fix.

### 3.6 Scoring rubric — honesty, calibration, evidence-chain

A well-written assessment scores on three axes, none of which is "did the team fix the bug":

- **Honesty.** Does the assessment describe what actually happened, including wrong turns and unresolved questions?
- **Calibration.** Does confidence match evidence? "Working hypothesis, not yet confirmed" with stated next tests is well-calibrated.
- **Evidence-chain integrity.** Does the chain symptom → hypothesis → evidence → impact → handoff hold together?

The assessment is the substrate for the postmortem and the live-defense conversation rather than a competitor with them. All three downstream artifacts (postmortem, ADR amendment, live defense) inherit the assessment's calibration.

## 4. Generic Implementation

A one-page impact-assessment template, framework-agnostic, looks like the following. Domain-specific framing (the actual symptoms, the actual surfaces, the actual SLAs) fills in once a team owns the template.

```markdown
# Impact Assessment — {incident id}

**Date:** YYYY-MM-DD  **Authored by:** {pair name}  **Window:** {HH:MM}–{HH:MM} UTC

## Slot 1 — Symptom (what users / stakeholders / observers saw)
{One paragraph. Cite the surface: dashboard panel, log line, screenshot. Name the
non-symptom — what was *not* anomalous. Include the timestamp range.}

## Slot 2 — Root-cause hypothesis (one sentence)
Primary: "{the one explanation that accounts for every observed symptom}"
Alternative under investigation (if any): "{alternative + status: confirmed / ruled out / unknown}"

## Slot 3 — Confirming / refuting evidence
- Confirming data point: {log line / metric query / test output, with timestamp + source}
- Refuting alternative considered: {alternative hypothesis} — ruled out by {data point + source}
- Unknowns (not yet tested): {item} — reason: {why not in window} — proposed test: {plan}

## Slot 4 — User-visible impact
- User impact: {N users affected, type of impact, severity}
- Service propagation: {services pulled into the failure; cascade or local}
- Business / compliance impact: {SLA / audit-trail / regulatory dimension, if any}

## Slot 5 — Hand-off scope
- Staged for next sprint:
  - {item} — owner: {name} — tracking: {ticket id} — artifact updated: {path}
- Already shipped today (if any):
  - {item} — owner: {name} — tracking: {ticket id} — artifact updated: {path}
- ADR amendments authored:
  - {path/to/ADR.md} — section: "Amendment (YYYY-MM-DD)" — see ticket {id}
```

The deliberate property of the template: each slot has a falsifiable shape. A symptom that does not cite a surface is incomplete; a hypothesis that is two sentences is over-cap; an evidence slot that lacks a confirming data point fails the integrity check. The template makes the failure modes visible to the author *while writing*, not only to the reviewer afterward.

> [!instructor-review]
> **Slot granularity should match team size.** The five-slot shape is calibrated for a pair. Teams of 5+ may add a "responder roster" sixth slot; solo on-calls may collapse slots 2+3. Instructor: confirm the cohort's working shape before treating five as universal.

## 5. Real-world Patterns

**Fintech — fraud-detection model regression.** A consumer-fintech platform's fraud-detection model began flagging an unusual fraction of legitimate transactions after a routine retraining. The assessment opened with a clean symptom slot ("false-positive rate on the fraud-flag dashboard rose from 0.4% to 3.1% between 09:15 and 09:45, while true-positive rate held flat"). The hypothesis slot named the retraining run explicitly; the evidence slot cited the AUC delta between model versions; the impact slot quantified user cost in declined-but-legitimate transactions (a number CX needed for outreach); the hand-off slot staged "rollback to previous model + add canary deploy for model updates" with a named owner. *The CX team used the impact-slot number for customer communication that day; engineering used the hand-off-slot ticket for next sprint.* Different audiences, same assessment.

**E-commerce — promotion-engine cascade.** A retailer's promotion engine silently dropped discount codes during a regional promo. The assessment's symptom slot caught what casual postmortems would have missed: the apparent symptom (revenue dip) was downstream of two separate failures — a cache invalidation 11 minutes earlier and a feature-flag flip 4 minutes later. The hypothesis named the cache invalidation as primary and the flag flip as exacerbating, *with the evidence slot citing the data points that ordered the two*. The hand-off staged two follow-up tickets owned by two different teams. Without the structured assessment, the postmortem would have conflated the failures.

Two additional cases — healthcare clinical-decision-support latency (compliance-dimension load-bearing) and gaming leaderboard desync (refuting-evidence saved the team) — appear in the [further-reading appendix](./4-impact-assessments-further-reading.md).

## 6. Best Practices

- **Write the symptom first, before the hypothesis.** Separating "what users saw" from "what we think caused it" prevents the most common diagnostic tunnel.
- **One hypothesis per assessment, named explicitly.** If the team has two candidates, name primary + alternative-under-investigation with status. Do not list three with caveats.
- **Cite specific surfaces — dashboard panel, log line, ticket id — not "we noticed."** The reader not in the war room must verify the symptom independently.
- **Score honesty and calibration, not "did the team fix it."** Reward the team that wrote "hypothesis not yet confirmed" honestly over one that overstated certainty.
- **Make the hand-off slot trackable: named owner + named tracking artifact for every staged item.**
- **Reuse the same assessment as input to postmortem, ADR amendment, and live defense.** The assessment is the substrate; the others are extensions.

## 7. Hands-on Exercise

**Task (paper or markdown sketch, 10–15 min):** You are a pair on-call for a SaaS product in an industry of your choice (NOT federal acquisitions — pick fintech, healthcare, gaming, e-commerce, or logistics). Your platform had an incident this morning. The mitigation is stable; users are back to normal; you are now writing the impact assessment before EOD.

Draft each of the five slots for the following scenario: "Between 09:14 and 09:51 today, the primary write endpoint of our service returned 5xx at an elevated rate (peak 11%). The cause appears to be a missing null-check in a validator that was changed in this morning's deploy. A rollback at 09:43 stabilised the symptom; the proper fix is staged for next sprint with a unit-test regression."

1. **Symptom.** Write the one-paragraph symptom citing surface, measurement, and non-symptom.
2. **Hypothesis.** Write the one-sentence working hypothesis. If you have a serious alternative, name it.
3. **Evidence.** Write the confirming data point and the refuting data point (or hypothesis ruled out).
4. **Impact.** Quantify user impact, service propagation, and business/compliance impact (use your chosen industry's framing).
5. **Hand-off.** Stage the proper fix with a named owner and a tracking artifact. Note any ADR amendment authored today.

**What good looks like.** A correct sketch keeps the symptom evidentiary (specific surface, specific timestamps, named non-symptom), commits to one hypothesis with an honest confidence label, cites a specific confirming data point *and* a refuted alternative, names user impact in concrete terms, and stages the follow-up work with a named owner. A common mistake is to merge the symptom and the hypothesis ("the validator broke") — note that as the anti-pattern. Another common mistake is a hand-off slot that says "we'll improve monitoring" without an owner — that is not yet a hand-off; that is a wish.

## 8. Key Takeaways

- **What is an impact assessment vs. a postmortem?** Same-day, one-page, five-slot artifact written while the incident is warm. The postmortem is multi-page, days later. The assessment is the substrate; the postmortem is the extension.
- **The five canonical slots:** Symptom, root-cause hypothesis, confirming/refuting evidence, user-visible impact, hand-off scope. Each is load-bearing.
- **Why symptom before hypothesis?** Separating "what we saw" from "what we think caused it" prevents diagnostic tunnel.
- **What does the scoring rubric reward?** Honesty, calibration, evidence-chain integrity. Not "did the team fix it."
- **What makes a hand-off slot trackable?** Named owner + named tracking artifact for every staged item.

## Sources

1. [Google SRE — Example Postmortem](https://sre.google/sre-book/example-postmortem/) — retrieved 2026-05-26
2. [Google SRE Workbook — Postmortem Analysis](https://sre.google/workbook/postmortem-analysis/) — retrieved 2026-05-26
3. [SRE School — Impact Assessment Guide (2026)](https://sreschool.com/blog/impact-assessment/) — retrieved 2026-05-26
4. [OneUptime — How to Build Impact Assessment (January 2026)](https://oneuptime.com/blog/post/2026-01-30-impact-assessment/view) — retrieved 2026-05-26
5. [Lightrun — Blast Radius Analysis](https://lightrun.com/blog/blast-radius-analysis/) — retrieved 2026-05-26

Last verified: 2026-05-26
