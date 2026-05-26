---
week: W04
day: Fri
topic_slug: impact-assessments-further-reading
topic_title: "Impact assessments — further reading (extended cases, reassessment cadence)"
parent_overview: W04/pre-session/5-Friday/1-DailyTopicOverview.md
parent_topic: W04/pre-session/5-Friday/4-impact-assessments.md
estimated_minutes: 6
sources:
  - url: https://sreschool.com/blog/impact-assessment/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://censinet.com/perspectives/audit-trails-support-regulatory-compliance
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://oneuptime.com/blog/post/2026-01-30-impact-assessment/view
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Impact assessments — further reading

> **Companion to [4-impact-assessments.md](./4-impact-assessments.md).** Primary reading covers the five-slot artifact, the scoring rubric, and two real-world cases (fintech model regression, e-commerce cascade). This appendix adds the compliance-load-bearing case from healthcare and the refuting-evidence case from gaming, plus the reassessment-during-incident cadence and an extended exercise.

## Extended Real-world Patterns

**Healthcare — clinical-decision-support latency spike.** A hospital network's clinical decision-support service had a 23-minute window where p99 latency exceeded the SLA. The impact slot's **compliance sub-dimension** was the load-bearing element: the SLA breach triggered a regulatory-reporting obligation under the network's audit framework. The assessment's hand-off slot staged the regulatory notification as a tracked item with the compliance team as owner. *Without the compliance sub-dimension in the impact slot, the team would have treated the incident as a pure engineering issue and missed the reporting obligation* ([Censinet — Audit Trails in Healthcare](https://censinet.com/perspectives/audit-trails-support-regulatory-compliance)).

The lesson generalises: in any regulated industry (healthcare, finance, federal), the impact slot's third sub-dimension (business / compliance) is the slot most likely to surface obligations the engineering team would otherwise miss. The discipline is to default to *naming* the compliance dimension even when the team's initial read is "no compliance impact" — the act of naming forces verification rather than assumption.

**Gaming — leaderboard desync.** A multiplayer studio's leaderboard service desynced from the game state during a live event. The assessment named the symptom precisely (a specific desync metric on the leaderboard dashboard) and rejected the obvious-but-wrong hypothesis (the leaderboard service itself) on the evidence slot's strength: a refuting data point showed the leaderboard's read replica was healthy throughout. The actual cause turned out to be in the upstream event-aggregator. *The discipline of writing the refuting evidence saved the team an hour of investigation against the wrong hypothesis* — the kind of saving that only shows up in well-structured assessments.

## Reassessment cadence during long incidents

2026 practitioner guidance ([SRE School](https://sreschool.com/blog/impact-assessment/); [OneUptime](https://oneuptime.com/blog/post/2026-01-30-impact-assessment/view)) treats the impact assessment as **living during the incident**, not a single same-day deliverable written at the end:

- **Every 15–30 minutes during an active incident:** reassess the five slots. Has the symptom shape changed? Has new evidence shifted the hypothesis? Has the user-visible impact dimension grown or shrunk? Has any handoff item materialised that wasn't there at the start?
- **At the end of each shift handoff:** the assessment is the artifact handed over, not a verbal walk-through. The next shift inherits the slots as they stood; their first act is a reassessment.
- **At EOD:** the final version of the assessment is committed. This is the artifact that downstream postmortem, ADR amendment, and live defense build on.

The discipline matters because a single end-of-day assessment loses signal that was present mid-incident. The hypothesis at 11:00 may have been different from the hypothesis at 13:30, and the evidence that ruled out the earlier hypothesis is part of the reasoning chain — not noise to be sanded out at 17:00.

## Extended Best Practices

- **Reassess every 15–30 minutes during a long incident.** The assessment is living during the incident; the final version is what's committed at EOD.
- **Default to naming the compliance sub-dimension even when initial read is "no impact."** The act of naming forces verification rather than assumption — and in regulated industries, the missed compliance impact is the audit finding.
- **Treat refuting evidence as load-bearing, not as garnish.** A well-structured slot 3 includes the alternative hypotheses *ruled out* with the data points that ruled them out — this is the artifact's defence against second-guessers.
- **Hand off in writing across shifts; treat the assessment as the handoff artifact.** Verbal walk-through is not a handoff; the next shift inherits whatever is on the page.
- **Calibrate slot granularity to team size.** A pair runs five slots; a team of 5+ adds a responder-roster slot; a solo on-call may collapse slots 2+3.

## Extended Hands-on Exercise

After completing the primary reading's exercise (the same-day five-slot draft for a hypothetical incident), extend with:

1. **The 15-minute reassessment.** Re-draft slot 2 (hypothesis) and slot 3 (evidence) as they might have looked 15 minutes before the rollback, when the team was still uncertain. How is the calibration label different? What evidence was *not yet available*?
2. **The compliance-dimension audit.** For your chosen industry, walk through slot 4's three sub-dimensions and identify the one most likely to be missed by an engineering-only reading. What artifact (SLA, audit framework, regulation) would be the source of truth for that dimension?
3. **The downstream-artifact mapping.** Identify which lines of your assessment would feed which downstream artifact: which line goes into the postmortem timeline, which informs the ADR amendment, which is the substrate for a live-defense interrogation.

The extensions move the reading from "I can write the artifact" to "I can use the artifact as the substrate for everything that comes after."

## Extended Sources

1. [Google SRE — Example Postmortem (Shakespeare outage)](https://sre.google/sre-book/example-postmortem/) — retrieved 2026-05-26
2. [SRE School — Impact Assessment Guide (2026)](https://sreschool.com/blog/impact-assessment/) — retrieved 2026-05-26
3. [OneUptime — How to Build Impact Assessment (January 2026)](https://oneuptime.com/blog/post/2026-01-30-impact-assessment/view) — retrieved 2026-05-26
4. [Censinet — Audit Trails Support Regulatory Compliance in Healthcare](https://censinet.com/perspectives/audit-trails-support-regulatory-compliance) — retrieved 2026-05-26
5. [Lightrun — Blast Radius Analysis](https://lightrun.com/blog/blast-radius-analysis/) — retrieved 2026-05-26

Last verified: 2026-05-26
