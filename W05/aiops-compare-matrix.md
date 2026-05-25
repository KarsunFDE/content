---
week: W05
day: Thu (morning workshop)
title: "AIOps platform compare-matrix — cohort #1 consolidated"
authored_at: 2026-06-25T11:00:00Z (target — fills in during Thu workshop)
authored_by: cohort + instructor consolidation
last_verified: 2026-05-23
recency_window: hot-tech 3-month — re-verify pre-cohort
input_artifacts:
  - weeks/W05/scenarios/W05-SA-1.md  # research prompt
  - per-pair Wed-research ADRs (live during cohort #1; not pre-authored)
output_consumes:
  - weeks/W05/war-room/Thu.md  # workshop session shape
  - weeks/W05/war-room/Fri-final-adversarial-pr.md  # Fri PR ADR cites this matrix
---

# W5 AIOps Platform Compare-Matrix — Cohort #1 Consolidated

> Authored during Thu morning compare-matrix workshop (per `war-room/Thu.md`). Fills in live with cohort + instructor consolidation. Template structure below — cohort #1's actual content is added during the workshop. The cohort defends the Datadog hands-on choice rigorously, with full knowledge of where each alternative would beat it.

## Frontmatter (filled during workshop)

```yaml
attended_pairs: [pair-1, pair-2, pair-3]
datadog_owner_pair: pair-N
dynatrace_owner_pair: pair-N
newrelic_owner_pair: pair-N
coralogix_owner_pair: pair-N (or instructor)
cohort_verdict: Confirm Datadog | Pivot to <alternative>
verdict_consensus: unanimous | majority | split
```

## Compare-matrix (12 dimensions × 4 platforms)

| Dimension | Datadog AI | Dynatrace Davis AI | New Relic AI Monitoring | Coralogix AI Observability |
|-----------|-----------|---------------------|--------------------------|----------------------------|
| Cohort hands-on (Tue PM install) | **Yes** | No | No | No |
| AI-native anomaly detection product | **Watchdog AI** | **Davis AI (causation)** | Lookout / Errors Inbox AI | Streama AI |
| LLM observability first-class? | **Yes (LLM Obs)** | Limited | **Yes (AI Monitoring)** | **Yes (AI Observability)** |
| Causation graph (vs correlation)? | No | **Yes** | No | No |
| Token/cost-per-request first class? | **Yes** | Manual | **Yes** | **Yes** |
| Conversational AI for on-call? | **Bits AI** | No (assistant-only) | NR Assistant | No |
| FedRAMP authorization | **High (GovCloud)** | Moderate | Moderate | Moderate |
| Federal-market presence | High | High | Medium | Lower |
| Pricing model | Per-host + ingest | Per-host (DPS) | Per-user + ingest | **Per-GB ingest** |
| Open-source friendly (OTel ingestion) | Yes | Yes | Yes | Yes |
| Karsun-licensed (assumption) | **Yes** | Maybe | Maybe | No |
| Datadog-Bedrock-traceparent join out-of-box | **Yes** | Partial | Partial | Partial |

**Bold cells = where the platform either uniquely wins on that dimension or matches the cohort's hands-on assumption.**

## Per-platform win conditions (cohort-authored)

### Datadog AI (hands-on choice; default)

Wins on: hands-on familiarity for cohort, FedRAMP High GovCloud, LLM Observability first-class, conversational on-call (Bits AI), Karsun-licensed assumption.

Loses (would pick another) when: engagement size is 20+ services where Dynatrace's causation graph reasoning would beat correlation; log-volume-dominated economics make Coralogix's per-GB pricing the better fit.

### Dynatrace Davis AI

Wins on: **causation graph** for complex multi-service root-cause. Where Davis builds the topology + reasons over it for "this caused that", Datadog provides correlations the on-call interprets.

Loses (would pick another) when: cohort #1's scale (5 components) doesn't activate the causation-graph advantage; FedRAMP Moderate doesn't qualify for high-boundary workloads.

### New Relic AI Monitoring

Wins on: AI-as-assistant rather than AI-as-agent framing (good for cohorts new to AIOps); per-user pricing fits teams with many seats vs many hosts; per-transaction AI summaries useful for developer-facing observability.

Loses (would pick another) when: AIOps maturity is high (the assistant framing is less interesting); host-count > user-count makes per-user pricing more expensive.

### Coralogix AI Observability

Wins on: **streaming-pipeline-native** processing decouples storage from query cost; per-GB ingest dramatically cheaper for log-heavy workloads (e.g., federal-acquisitions audit volume).

Loses (would pick another) when: cohort isn't log-volume-dominated; smaller federal-market footprint = thinner reference base; cohort wants Karsun-licensed familiarity.

## Cohort #1 verdict template

```markdown
**Cohort verdict.** [Confirm Datadog AI | Pivot to <X>] for `acquire-gov` cohort #1.

**Reasoning.**
- [Win condition that applies to cohort #1]
- [Win condition that applies to cohort #1]
- [Win condition that applies to cohort #1]

**Where we'd pivot for a different engagement shape.**
- [Specific shape] → [Platform] because [win condition].
- [Specific shape] → [Platform] because [win condition].

**Open question.**
- [Anything the cohort couldn't resolve via /web-research; surface to skill lead for Cohort #2 prep.]
```

## How Fri's PR cites this matrix

Each pair's Fri PR includes an ADR (linked from PR description) that references this consolidated matrix + commits the platform choice for the pair's project. The PR ADR doesn't re-derive the matrix; it cites this file + makes the engagement-specific call.

## Re-verification cadence

This matrix lives in the `hot-tech 3-month` recency window per `pipeline/RESEARCH-PROTOCOL.md`. **Re-verify pre-cohort** (within 30 days of W5 Mon start) — vendor product pages ship updates monthly. Specifically:

- Datadog LLM Observability — features ship monthly; verify Watchdog AI on LLM endpoints is still accurate.
- Dynatrace Davis AI — Davis ships major updates twice yearly; verify causation-graph still the differentiator.
- New Relic — AI Monitoring product evolves rapidly; verify the per-transaction summaries claim.
- Coralogix — Streama AI features; verify per-GB pricing model still applies.

## Sources (all retrieved 2026-05-23 via `/web-research`; re-verify pre-cohort)

- Datadog AI / Watchdog AI — https://docs.datadoghq.com/watchdog/
- Datadog LLM Observability — https://docs.datadoghq.com/llm_observability/
- Datadog Bits AI — https://www.datadoghq.com/blog/bits-ai-incident-management/
- Datadog FedRAMP — https://www.datadoghq.com/security/
- Dynatrace Davis AI — https://www.dynatrace.com/platform/artificial-intelligence/
- Dynatrace Public Sector — https://www.dynatrace.com/solutions/public-sector/
- New Relic AI Monitoring — https://newrelic.com/platform/ai-monitoring
- New Relic Public Sector — https://newrelic.com/solutions/industry/public-sector
- Coralogix AI Observability — https://coralogix.com/ai-observability/
- Coralogix Streama pricing — https://coralogix.com/pricing/
- `pipeline/DECISIONS.md` D-060 (Datadog hands-on rationale)
