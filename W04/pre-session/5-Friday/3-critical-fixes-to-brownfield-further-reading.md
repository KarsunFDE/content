---
week: W04
day: Fri
topic_slug: critical-fixes-to-brownfield-further-reading
topic_title: "Critical fixes to brownfield — further reading (extended cases, compliance framing)"
parent_overview: W04/pre-session/5-Friday/1-DailyTopicOverview.md
parent_topic: W04/pre-session/5-Friday/3-critical-fixes-to-brownfield.md
estimated_minutes: 6
sources:
  - url: https://martinfowler.com/bliki/StranglerFigApplication.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-decomposing-monoliths/strangler-fig.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://tianpan.co/blog/2026-04-12-brownfield-ai-integrating-llm-features-into-legacy-codebases
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Critical fixes to brownfield — further reading

> **Companion to [3-critical-fixes-to-brownfield.md](./3-critical-fixes-to-brownfield.md).** The primary reading covers the five-rank reversibility-first model, the 12-line rule, and two real-world cases (fintech timeout revert, gaming flag-flip). This appendix adds two more real-world cases and the regulated-industry framing around staged compliance artifacts.

## Extended Real-world Patterns

**Healthcare — PACS upload validator.** A radiology platform discovered a NullPointerException in its image-upload validator after a routine library bump. The team's first instinct was a multi-service refactor to "do null-safety properly." The compliance officer intervened: ship a one-line null-check in the affected validator (rank 3, single file, 1 line), stage the broader null-safety work as a sprint item with an ADR amendment. The narrow hot-patch shipped in 40 minutes, passed adversarial review, and the broader refactor went through normal sprint review over the following six weeks. *The narrow patch shipped on Friday; the wide refactor shipped over the next sprint cycle, where it belonged.*

The pattern illustrates the discipline's most counter-intuitive call: the team that *wanted* to do "the right thing" architecturally was overridden by the discipline that says "the right thing now is the one-line patch; the architectural fix is a Tuesday-morning conversation."

**E-commerce — search-index drift.** A retailer's product search began returning stale results after a schema migration. The natural fix involved touching three services (indexer, query service, cache invalidator). The team explicitly declined the rank-4 cross-cutting patch and instead **staged**: they enabled a manual cache-flush endpoint (a pre-existing rank-2 mitigation), opened a ticket for the proper fix, and amended the migration ADR with the divergence. The proper fix shipped two sprints later, after the migration had stabilised. *The decision to stage rather than ship under pressure was the engineering call; the audit trail of the staged decision satisfied the post-mortem reviewer.*

## Extended Best Practices

- **Practise the rank walk on a non-production drill.** A team that has never run the five-rank conversation will run it slowly the first time, often skipping rank 1 in favour of the more-familiar rank 3. Quarterly drills are the practitioner default.
- **Pre-populate the rank-1 revert artifact at deploy time.** The deploy pipeline should emit a "last known good = {tag}" record into a known location, so the rank-1 revert is a copy-paste command, not a CI-history hunt.
- **Treat flag hygiene as a CI obligation, not a backlog item.** Flags that go a quarter without exercise should produce a CI warning. Stale flags are a leading cause of rank-2 unavailability under pressure.
- **Include the ADR-amendment-path as a PR template field for hot-patches.** A PR template that asks "ADR amendment at?" before merge catches spec drift at the gate, not in the next Monday's plan retrospective.

## Compliance / regulated-industry framing

In regulated industries — financial services, healthcare, federal contracting, defence — the audit trail of the staged decision IS the compliance artifact. The auditor's question is not "did you fix everything immediately?" The question is "when you chose not to fix something immediately, did you document the choice in a way I can read?"

Three structural properties of a defensible staged decision in a regulated industry:

1. **Named owner.** Not a team, not a TBD — a named individual with authority over next-sprint backlog. Absence of a name turns the staged item into an audit finding.
2. **Named tracking artifact.** A ticket ID, a sprint identifier, an ADR amendment path. The artifact has to be cite-able in the auditor's review.
3. **Named mitigation in place.** What is running today instead of the proper fix, when did it start running, and what is its operational cost?

A staged decision with those three named properties is, in audit framing, a *complete decision* — not a deferred one. The completeness lets the audit close the finding before the proper fix ships, because the audit's interest is in the team's documented change-control discipline, not in whether the team ships everything on the calendar.

## Extended Hands-on Exercise

After completing the primary reading's exercise (the rank-walk for a write-endpoint refactor incident), extend with:

1. **Rank-walk on a real PR from your pair-project repo.** Pick a recent PR; classify it on the five-rank scale as if it had been an incident response. Would it have been ship-now or stage? What ADR amendment would have accompanied it?
2. **Pre-flight the deploy artifact tagging.** Verify that your pair-project's deploy pipeline emits a "last known good" record at a known location. If it doesn't, propose the smallest change that would make rank-1 revert one command.
3. **Audit your flag hygiene.** List your pair-project's feature flags (if any). For each, name the last time it was flipped off in CI or production. Anything older than a quarter is no longer reliably rank-2.

## Extended Sources

1. [Martin Fowler — Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html) — retrieved 2026-05-26
2. [AWS Prescriptive Guidance — Strangler Fig Pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-decomposing-monoliths/strangler-fig.html) — retrieved 2026-05-26
3. [Brownfield AI — Integrating LLM Features into Legacy Codebases (Tian Pan, April 2026)](https://tianpan.co/blog/2026-04-12-brownfield-ai-integrating-llm-features-into-legacy-codebases) — retrieved 2026-05-26
4. [Flagsmith — 8 Feature Flag Deployment Strategies](https://www.flagsmith.com/blog/deployment-strategies) — retrieved 2026-05-26

Last verified: 2026-05-26
