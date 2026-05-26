---
week: W04
day: Fri
topic_slug: amendment-tradeoff-evaluations-further-reading
topic_title: "Amendment tradeoff evaluations — further reading (extended cases, retro prompts)"
parent_overview: W04/pre-session/5-Friday/1-DailyTopicOverview.md
parent_topic: W04/pre-session/5-Friday/6-amendment-tradeoff-evaluations.md
estimated_minutes: 6
sources:
  - url: https://rootly.com/sre/2025-sre-incident-management-best-practices-checklist
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://hyperping.com/blog/incident-post-mortem
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://upstat.io/blog/incident-anti-patterns
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Amendment tradeoff evaluations — further reading

> **Companion to [6-amendment-tradeoff-evaluations.md](./6-amendment-tradeoff-evaluations.md).** Primary covers the 2×2 severity-by-effort matrix, the three failure modes (heroism, avoidance, spec drift), the EOD retro discipline, and two real-world cases (fintech tradeoffs, e-commerce 16:30 audit). This appendix adds gaming (avoidance-check IC intervention) and healthcare (compliance-expanded Q1) cases, plus an extended retro-prompt library.

## Extended Real-world Patterns

**Gaming — matchmaker afternoon tradeoffs.** A multiplayer studio's incident afternoon had only one candidate item, but it was Q2 (high-severity, high-effort cross-service work). The pair's instinct was heroism: try to ship the cross-service fix by 18:00. The incident commander intervened: stage the work for the next sprint, keep the existing single-region throttle running through the weekend, ship the impact assessment as the day's artifact. *The avoidance check at 16:30 was the load-bearing intervention — the team initially thought "ship nothing" felt wrong, but the IC reframed the impact assessment + stage-with-named-owner as the legitimate shipment.* On Monday, the cross-service fix shipped through normal sprint review, properly tested.

The lesson: the avoidance check is sometimes mis-triggered by a heroism instinct in disguise. "Ship nothing feels wrong" is the team's heroism speaking, not its avoidance speaking; the IC's job is to recognise which is which.

**Healthcare — clinical-rule afternoon tradeoffs.** A hospital network's incident produced three candidate items, two of which involved touching the clinical-rule engine. The team's compliance posture meant that high-severity items affecting patient-facing recommendations *could not* be staged — staging would constitute an audit-trail gap. The Q1 quadrant therefore expanded to include one item that would normally have been Q2. *The matrix's quadrants are not domain-neutral; the regulated-industry team's "low-effort" definition included "and passes the compliance-team's same-day review."* The team shipped two of three items same-day with extended review attendance, and staged the third with explicit compliance sign-off on the deferral.

The pattern: in regulated industries, the matrix axes are interpreted through a compliance lens. "Severity" includes audit-trail completeness; "effort" includes same-day compliance review. Teams operating in these contexts should re-anchor their quadrant boundaries explicitly each Friday.

## Extended retro prompt library

The primary reading's worksheet contains four reflective prompts. A larger library of prompts, drawn from 2026 SRE retro practice ([Hyperping](https://hyperping.com/blog/incident-post-mortem); [Rootly](https://rootly.com/sre/2025-sre-incident-management-best-practices-checklist)) — pick the two most relevant per Friday:

1. **Mental-model shift.** "What changed in your pair's mental model of 'production' between 09:00 and 17:00 today?"
2. **Observable-signal anchoring.** "What single observable signal would have given your pair 15 more minutes earlier?"
3. **Weekend-anxiety calibration.** "Which of your STAGE items concerns you most going into the weekend, and why?"
4. **One-week delta.** "What did you ship today that you would *not* have shipped a week ago, and what changed?"
5. **Heroism resistance.** "Which Q2 item did you most want to collapse into Q1, and what stopped you?"
6. **Avoidance resistance.** "Which Q1 item did you most want to demote to Q2, and what kept it in Q1?"
7. **Spec-drift catch.** "Which of your shipped fixes nearly went out without its ADR amendment, and what caught it?"
8. **Owner-quality check.** "For each STAGE item, can you name the owner without checking the worksheet? If not, why not?"

The library exists because the *same* prompt every Friday produces shallow answers by week 3 — the team starts pattern-matching to "what did we say last time." Rotating two of eight prompts keeps the retro generative.

## Extended Best Practices

- **Rotate retro prompts weekly.** Pick two from the library; do not run the same two two Fridays in a row.
- **Have the IC run the 16:30 audit, not the pair.** The pair has been making the trade-off calls all afternoon; the IC has the distance to spot the heroism/avoidance pulls.
- **Make the retro a 15-minute timebox.** Writing-not-discussing is partly enforced by the timebox — there is no room for discussion that doesn't write.
- **File the retro and the impact assessment side-by-side in the incident folder.** Three months later, the postmortem author wants both artifacts open at once.
- **Treat the matrix as a coaching artifact for new pairs.** A pair running their first amendment-tradeoff worksheet should run it with an experienced reviewer who can flag mis-classifications before the 16:30 audit.

## Extended Hands-on Exercise

After completing the primary reading's exercise (A–E quadrant walk), extend with:

1. **The avoidance-vs-heroism distinction.** For your sketched B, C, and E items, identify which one the pair would most likely default to *over*-staging (avoidance) and which to *over*-shipping (heroism). What would the IC's reframe look like in each case?
2. **The compliance-expanded Q1.** Assume your chosen industry has a compliance dimension (HIPAA, SOX, FedRAMP, FAR, etc.). Pick one Q2 item from your sketch and identify what compliance pressure would expand it into Q1 (audit-trail completeness, SLA breach, regulatory reporting threshold).
3. **The 4-Friday weekly retro arc.** Sketch what four consecutive Friday retro paragraphs might look like for the same pair — picking different prompts each week. Where would you expect the team's mental-model shift to evolve?

## Extended Sources

1. [Rootly — 2025 SRE Incident Management Best Practices Checklist](https://rootly.com/sre/2025-sre-incident-management-best-practices-checklist) — retrieved 2026-05-26
2. [Hyperping — Blameless Post-Mortems](https://hyperping.com/blog/incident-post-mortem) — retrieved 2026-05-26
3. [Upstat — Common Incident Anti-Patterns That Slow Response](https://upstat.io/blog/incident-anti-patterns) — retrieved 2026-05-26
4. [CSA — Rethinking Incident Response as an Engineering System](https://cloudsecurityalliance.org/blog/2026/04/23/rethinking-incident-response-as-an-engineering-system-addressing-7-operational-gaps) — retrieved 2026-05-26

Last verified: 2026-05-26
