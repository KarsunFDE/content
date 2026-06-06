---
week: W03
day: Wed
topic_slug: research-discipline-scenario-alternatives
topic_title: "Research discipline — scenario alternatives via /web-research"
parent_overview: W03/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://aws.amazon.com/blogs/architecture/master-architecture-decision-records-adrs-best-practices-for-effective-decision-making/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/bliki/ArchitectureDecisionRecord.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://arxiv.org/pdf/2509.04499
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://umassglobal.libguides.com/c.php?g=1267504&p=9295343
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-06-06
---

# Research discipline — scenario alternatives via /web-research

> [!NOTE]
> **From earlier:** Mon's ADR framing named the question and constraints. Today's D-040 slot is the research execution — three scenario-alternatives prompts open this afternoon. This topic is the method behind that work.

## 1. Learning Objectives

- Frame an architectural decision as a *scenario-alternatives* exercise: name the question, enumerate 2–4 alternatives, evaluate against consistent criteria, recommend with documented trade-offs.
- Apply the source-evaluation rubric (authority, currency, accuracy, relevance, objectivity) to candidate sources.
- Recognise the recency-window discipline and apply it to today's SA-1/SA-2/SA-3 research.
- Identify known-bad-pattern sources for current technologies and route around them.

## 2. Introduction

Decisions made under deadline pressure are usually made on intuition; documentation becomes post-hoc rationalization. The *scenario-alternatives* shape prevents that: a named question, enumerated viable options, consistent criteria, and a recommendation you can rebuild from the doc alone.

LLMs compress research but produce sources that look authoritative yet are stale or hallucinated. Deep-research agents misattribute claims — correct-statement ratios ~70–82% in 2026 benchmarks. Source evaluation stays a human discipline.

## 3. Core Concepts

### 3.1 The scenario-alternatives shape

Four parts, in order:

1. **The question** — concrete, at least one quantitative constraint. "Which KG backend?" is too vague; "Which meets sub-200ms p95 at acquire-gov cardinality within a single-Postgres ops budget?" is well-formed.
2. **The constraints** — non-negotiables: performance targets, regulatory requirements, ops-budget caps, architecture commitments.
3. **The alternatives** — 2–4 distinct architectural shapes. Two forces comparison; more than four signals a poorly-framed question.
4. **The criteria** — evaluation dimensions *consistent across all options*. Inconsistent criteria produce apples-to-oranges comparisons.

> [!IMPORTANT]
> **The recommendation is the *output* of the exercise, not the input.** Picking the recommendation first and then justifying it is the failure mode the shape exists to prevent. Wed PM's SA-1/SA-2/SA-3 sketches are not ADRs yet — they are research + one finding per SA + one open question. Full ADRs due EOD Thu W4.

### 3.2 Source-evaluation rubric

| Dimension | What to check | LLM-era note |
|-----------|--------------|--------------|
| **Authority** | Practitioner / maintainer / vendor / SEO site? | Primary sources outrank secondary tutorials |
| **Currency** | Published/updated when? Tech still current? | Framework churn makes 2-year-old tutorials misleading |
| **Accuracy** | Cross-verifiable against independent sources? | LLM summaries often misattribute claims |
| **Relevance** | Addresses your question? | Adjacent content wastes research budget |
| **Objectivity** | Balanced or product pitch? | Vendor docs are primary with obvious bias — name it |

Authority and currency dominate for LLM-era tech.

### 3.3 Recency windows by category

| Category | Window | Examples |
|----------|--------|---------|
| Hot-tech | 3 months | LangGraph, LangChain v1.0, Bedrock catalog, OWASP LLM Top 10 |
| Federal-regulatory | 6 months | FAR/DFARS amendments, FedRAMP baselines, OMB/OIG guidance |
| Foundation-stable | 12 months | Java LTS, Postgres, Spring Boot 3.x, mature web standards |

Name the recency category for each source you cite. A source outside its window is not automatically wrong — but you should know you're using it and justify it.

### 3.4 Sub-30-day sources

A sub-30-day source warrants an explicit *adopt / defer / omit* decision. Surface the date, name the risk class, escalate per CLAUDE.md hard rule.

> [!CAUTION]
> **Sub-30-day vendor announcements** may not yet be validated by independent practitioners. Never silently include — make the explicit adopt/defer/omit call.

### 3.5 Known-bad patterns and deep-research caution

`known-bad-patterns.yml` lists patterns dominating older tutorials: LCEL-as-foundational, `Chain` class advocacy, LCEL `|` pipe syntax. Flag for instructor review; do not silently include.

Deep-research-agent output is a **starting point, not a citable conclusion**. Cite the primary source you verified, not the agent that found it.

## 4. Generic Implementation

Minimal scenario-alternatives ADR template — feature-flag system selection (generic SaaS, not federal-acquisitions):

```markdown
# ADR-2026-04-12: Feature-flag system

## Status
Proposed (decision needed before sprint 14)

## Context
~30 new features over 6 months; need per-user targeting, gradual rollout,
audit trail, self-host option (EU data-residency constraint).

## Decision drivers (consistent across all alternatives)
1. Self-host viable (data residency)
2. Engineering integration effort (weeks)
3. Cost at ~50k MAU
4. Audit-log fidelity
5. SDK support (Python + TypeScript)

## Considered alternatives

| Criterion | LaunchDarkly (managed) | Unleash (self-hosted OSS) | Build in-house |
|-----------|----------------------|--------------------------|---------------|
| Self-host | No (deal-breaker) | Yes | Yes |
| Integration | ~1 week | ~2 weeks | ~4 weeks |
| Cost | $$$ | Infra only | Salary |
| Audit | Strong | Strong | As built |

## Decision
Option B (Unleash). Self-host requirement disqualifies A; B vs C — B
costs ~2× less engineering time with comparable capability.

## Consequences
- We take on ops for one more component.
- 2-week integration cost accepted; alternative was a data-residency deal-breaker.
- Revisit if MAU > 500k (different cost profile).

## Sources verified
1. Unleash official docs, retrieved YYYY-MM-DD (hot-tech, in-range).
2. LaunchDarkly pricing, retrieved YYYY-MM-DD (hot-tech, in-range).
```

Decision flows from the criteria table — the recommendation is the output, not the input. Consequences section names the second-order effects you're accepting.

> [!NOTE]
> **D-040 PM slot opens today.** SA-1 / SA-2 / SA-3 scenario-research prompts release this morning. The method in this topic file is the discipline behind that work — frame the question first, then search. Full ADRs due EOD Thu W4.

## 5. Real-world Patterns

**Fintech — payment gateway ADRs.** 3–4 alternatives, ~8 consistent criteria (latency, fees, PCI scope, vendor lock-in). ADR survives 5+ years when reopened.

**Enterprise SaaS — observability retro.** Vendor picked by intuition; cost issues at scale forced a reopen. Scenario-alternatives produced a migration path the intuition-based original couldn't support.

## 6. Best Practices

- **Frame the question before searching.** A vague question generates undirected research.
- **Cite primary sources.** Secondary sources find primary ones; they are not endpoints.
- **Tag every source with retrieval date and recency category.** A claim's truth has a half-life.
- **Treat deep-research-agent output as a candidate-source list.** Verify each source independently.
- **Surface sub-30-day sources for explicit adopt/defer/omit.** Never silently include.
- **Keep alternatives between 2 and 4.** More than four signals a poorly-framed question.

> [!WARNING]
> **Anti-pattern: `model-knowledge-as-research`.** Using model-internal knowledge without `/web-research` citations violates D-046 — produces vibes-based decisions that fail the Phase 1 gate. Every current-state claim must trace to a retrieved source with a date. "The model said Neo4j is faster" is not a citation.

## 7. Hands-on Exercise

Pick a plausible architectural decision for your pair project (not federal-acquisitions domain directly). Produce a sketch: (1) question with one quantitative constraint; (2) 3–5 measurable criteria; (3) 2–4 alternatives in a criteria table; (4) one source per alternative with retrieval date and recency category; (5) one-sentence recommendation keyed to criteria; (6) one consequence you're accepting.

> [!NOTE]
> **Self-check** (30s — answer mentally before expanding)
>
> 1. A deep-research agent returns a summary saying "LangGraph is the best choice because of its LCEL foundation." What two problems does this source have?
> 2. A source for your SA-2 ADR was published 18 days ago on LangChain's official blog announcing LangGraph v0.3. What is the correct handling per the research protocol?

<details>
<summary>Show answers</summary>

1. Two problems: (a) LCEL-as-foundational is a known-bad pattern — LangGraph v1.0 docs do not centre LCEL/Runnables; the summary is citing a stale framing. (b) Deep-research-agent output is not a citable conclusion — you must verify the underlying primary source (LangGraph official docs) independently before citing. The summary is a candidate-source lead, not a citation.
2. Sub-30-day source handling: surface the publication date, name the risk class (vendor announcement, not yet independently validated), and make an explicit adopt/defer/omit decision with instructor sign-off per CLAUDE.md hard rule. If the announcement describes a breaking change or new API, defer until at least one independent practitioner report validates the behavior. Do not silently include.

</details>

> [!IMPORTANT]
> **The recommendation is the *output*, not the input.** Picking the answer first then justifying it is the failure mode the scenario-alternatives shape exists to prevent. Wed PM SA sketches are research + one finding + one open question per SA — not pre-decided ADRs.

## 8. Key Takeaways

- Scenario-alternatives: question + constraints + 2–4 alternatives + consistent criteria → recommendation as output, not input.
- Source rubric: authority and currency dominate; primary outranks secondary.
- Recency windows: hot-tech 3 months, federal-regulatory 6 months, foundation-stable 12 months.
- Deep-research-agent output is a starting point — verify each source independently before citing.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record — retrieved 2026-05-26 — foundation-stable
- https://aws.amazon.com/blogs/architecture/master-architecture-decision-records-adrs-best-practices-for-effective-decision-making/ — retrieved 2026-05-26 — foundation-stable
- https://martinfowler.com/bliki/ArchitectureDecisionRecord.html — retrieved 2026-05-26 — foundation-stable
- https://arxiv.org/pdf/2509.04499 — retrieved 2026-05-26 — hot-tech (DeepTRACE: Auditing Deep Research AI Systems)
- https://umassglobal.libguides.com/c.php?g=1267504&p=9295343 — retrieved 2026-05-26 — foundation-stable

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

**Consequences section as the discipline test.** The consequences section of an ADR is where intellectual honesty lives. Every architectural decision has trade-offs; an ADR with no negative consequences is incomplete and produces over-confident future decisions. Senior FDEs reviewing peer ADRs should check the consequences section first — if it reads as entirely positive, the author has either found a unicorn decision or hasn't looked hard enough. Common consequence classes: additional ops burden, skill-ceiling risk (team doesn't fully know the technology yet), performance cliff at a specific scale threshold, vendor-lock exposure, migration cost if the decision is revisited.

**Research-without-citation-date as an audit trail gap.** In federal-acquisitions work, the audit trail for an architectural decision includes the sources used and their dates. A source cited without a retrieval date is not an audit-grade citation — if the source is later amended or retracted, there is no way to reconstruct what the source said at the time of the decision. The `last_verified` + `/web-research` citation discipline in this programme is not bureaucracy; it is the same provenance discipline that federal audit teams apply to procurement decisions. Senior FDEs should treat the citation format as a first-class deliverable, not a formality.

</details>

Last verified: 2026-06-06
