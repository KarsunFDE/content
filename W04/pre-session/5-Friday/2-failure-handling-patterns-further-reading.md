---
week: W04
day: Fri
topic_slug: failure-handling-patterns-further-reading
topic_title: "Failure handling patterns — further reading (extended cases, best practices)"
parent_overview: W04/pre-session/5-Friday/1-DailyTopicOverview.md
parent_topic: W04/pre-session/5-Friday/2-failure-handling-patterns.md
estimated_minutes: 6
sources:
  - url: https://sre.google/sre-book/emergency-response/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://sre.google/sre-book/postmortem-culture/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://incident.io/blog/incident-management-best-practices-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Failure handling patterns — further reading

> **Companion to [2-failure-handling-patterns.md](./2-failure-handling-patterns.md).** The primary reading covers the four-phase cadence (detect → respond → root-cause → fix-or-stage), the generic mitigations, and two real-world patterns at recognition depth. This appendix carries the additional case studies, extended best practices, and the regulated-industry edge case where "stage" is structurally unavailable.

## Extended Real-world Patterns

**Gaming — multiplayer matchmaker latency.** A multiplayer studio's matchmaker started routing players into matches that crashed on load. The team's runbook had a generic-mitigation entry: "if match-crash rate > 5%, throttle matchmaker to single-region." The throttle dropped crash rate to 0.3% while the team investigated. Root cause turned out to be a corrupted asset bundle that one CDN edge had cached; clearing the edge and refreshing the bundle resolved it. The fix-or-stage decision was *ship*: the bundle refresh was a single CDN API call, reversible, with no blast-radius concern. Total user-visible degradation: nineteen minutes. Without mitigation-first, the studio estimates two hours.

The lesson the studio drew from the post-incident retro is the one Google SRE's Emergency Response chapter ([Google SRE](https://sre.google/sre-book/emergency-response/)) calls out: the runbook's pre-named generic mitigation is what made the response fast. Without the runbook entry, the team would have had to invent the mitigation on the spot, and the time-to-mitigate would have been measured in tens of minutes rather than tens of seconds.

**Healthcare imaging — PACS upload failures.** A radiology platform's image-upload path began failing intermittently. The team's compliance posture mandated audit-log completeness, so "stage with a documented gap" was not an option for the underlying race condition — the gap would have been a compliance finding. The team mitigated by routing uploads to a single replica (eliminating the race), then shipped the proper fix the same day with extended testing.

The pattern illustrates that "fix vs. stage" is not domain-neutral: regulated industries have categories of work where stage is structurally unavailable. The runbook needs to name those categories in advance, not improvise them under stress. The healthcare-network's compliance officer treats the runbook's "categories that cannot be staged" list as a compliance artifact — every category must have a defined mitigation path that can run indefinitely until the proper fix ships.

## Extended Best Practices

- **Practise the cadence on a non-production incident before you need it.** Game-days and tabletop exercises are how teams learn that mitigation is faster than diagnosis; the lesson does not transfer from reading alone. The Google SRE workbook's chapter on Incident Response includes recommended frequency targets: at least quarterly tabletop, monthly chaos-engineering exercise.
- **Pre-name the categories of fix that cannot be staged.** In regulated industries, list the work classes where audit-trail gaps or SLA breaches make deferral structurally unavailable. The list is itself a compliance artifact and a coaching artifact for new on-calls.
- **Tie the postmortem to the runbook directly.** Each postmortem's action-items section should produce at least one runbook update — a new symptom panel, a new generic mitigation entry, a new "category that cannot be staged" line. The postmortem that produces no runbook delta is one whose lessons have not been internalised.
- **Test the runbook itself in CI when possible.** A runbook entry like "if rate > 5%, throttle to single-region" should have a smoke-test that exercises the throttle path on a staging cluster; a runbook entry whose mitigation has never been exercised is a runbook entry that may not work under pressure.
- **Run the cadence quarterly even when no incident has happened.** A team that has not exercised the cadence in three months will execute it slowly the next time. Practitioner guidance from [Google SRE Blameless Postmortems](https://sre.google/sre-book/postmortem-culture/) treats the cadence as a perishable skill.

## Extended Hands-on Exercise

After completing the primary reading's exercise (the four-phase sketch for a hypothetical SaaS API incident), extend the work with:

1. **Pre-naming non-stageable categories.** For your chosen industry, list two or three fix categories where stage would constitute a compliance gap or SLA breach. Justify each in one sentence. Compare with a pair partner.
2. **Runbook-to-test mapping.** For your generic-mitigation choice in the primary exercise, propose one CI smoke-test that would have caught a regression in the mitigation path itself. Note whether your team currently runs anything like it.
3. **Postmortem-to-runbook delta.** Assume the incident produces a postmortem. Draft the one-line runbook update the postmortem's action items would require. Identify which of the four phases (detect / respond / root-cause / fix-or-stage) the update strengthens.

These extensions move the reading from "I understand the cadence" to "I know how my team would integrate the cadence into its existing ops and review surfaces."

## Extended Sources

1. [Google SRE — Emergency Response](https://sre.google/sre-book/emergency-response/) — retrieved 2026-05-26
2. [Google SRE — Blameless Postmortems / Postmortem Culture](https://sre.google/sre-book/postmortem-culture/) — retrieved 2026-05-26
3. [incident.io — Incident Management Best Practices 2026](https://incident.io/blog/incident-management-best-practices-2026) — retrieved 2026-05-26
4. [PagerDuty — Incident Response Documentation](https://response.pagerduty.com/) — retrieved 2026-05-26

Last verified: 2026-05-26
