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
last_verified: 2026-05-26
---

# Research discipline — scenario alternatives via /web-research

## 1. Learning Objectives

By the end of this reading, the learner can:

- Frame an architectural decision as a *scenario-alternatives* exercise: name the question, enumerate 2–4 alternatives, evaluate against consistent criteria, recommend with documented trade-offs.
- Apply a research-source-evaluation rubric (authority, currency, accuracy, relevance, objectivity) to candidate sources before citing them.
- Recognize the recency-window discipline (hot-tech 3 months / federal-regulatory 6 months / foundation-stable 12 months) and why a sub-30-day source warrants explicit handling.
- Identify common known-bad-pattern sources for current technologies (stale framework idioms, deprecated APIs, superseded model identifiers) and route around them.
- Produce an ADR (or ADR-sized sketch) that another engineer could review for substance, not just style.

## 2. Introduction

Architectural decisions made under deadline pressure are usually made on intuition; the documentation, if it exists, is post-hoc rationalization. The discipline that gets you out of that mode is the *scenario-alternatives* shape — a named question, an enumerated set of viable options, consistent evaluation criteria, and a recommendation that you (or your future self) can rebuild from the doc alone [1][2].

The 2025–2026 problem is that the same LLMs that compress the research work also make it easier to produce sources that look authoritative but are stale, incorrect, or hallucinated. Deep-research agents — those that orchestrate multi-step web exploration into citation-rich reports — have been shown in 2026 evaluation benchmarks to misattribute claims, cite irrelevant sources, and produce confident summaries of patterns the underlying frameworks no longer endorse [4]. The implication is that you cannot outsource source-evaluation; it stays a human discipline even when the search is automated.

The combined discipline — *scenario-alternatives* framing plus *source-evaluation* rigor plus *recency-window* hygiene — is what makes architectural research durable across the rapid version-churn of LLM-era tooling. A research session that produces three viable options, each backed by sources you can defend by author, date, and primary-vs-secondary status, gives you an ADR that survives both peer review and a six-month-later question of "why did we pick this?"

## 3. Core Concepts

### 3.1 The scenario-alternatives shape

A well-formed scenario for an architectural research exercise has four parts:

1. **The question** — concrete enough to evaluate options against. "Which vector database?" is too vague; "Which vector database meets <p95 latency target>, <data-residency constraint>, <existing-skills constraint> for our workload?" is well-formed.
2. **The constraints** — non-negotiables that narrow the option set. Includes performance targets, regulatory requirements, team skill ceilings, ops-budget caps, and pre-existing architectural commitments.
3. **The alternatives** — typically 2–4, each a distinct architectural shape (not minor variants). Two is enough to force comparison; more than four becomes unwieldy [2].
4. **The criteria** — the dimensions on which alternatives are evaluated, *consistent across all options*. Without consistent criteria, comparisons become apples-to-oranges.

The recommendation is the *output* of the exercise, not the input. Picking the recommendation first and then justifying it is the failure mode the shape exists to prevent [3].

### 3.2 The source-evaluation rubric (CRAAP / authority-currency-accuracy)

Library and information-science literature converges on roughly the same source-evaluation rubric — variants include CRAAP (Currency, Relevance, Authority, Accuracy, Purpose) and the five-question authority-currency-accuracy-relevance-objectivity model [5]:

- **Authority.** Who wrote/published this? Are they a recognized practitioner, project maintainer, vendor, or an aggregator/SEO site? Primary sources (project docs, RFCs, official blog posts) outrank secondary sources (tutorials, summaries, AI-generated overviews).
- **Currency.** When was it published or last updated? Is the technology it discusses still current?
- **Accuracy.** Can the claims be cross-verified against other independent sources?
- **Relevance.** Does it actually address your question, or is it adjacent?
- **Objectivity.** Is it making a balanced comparison or selling a product?

For LLM-era technologies (frameworks, model catalogs, agent libraries), authority and currency dominate. A two-year-old tutorial on a framework that has had three major releases since is actively misleading even if accurate at the time.

### 3.3 Recency windows by category

Not all technology research has the same recency requirements. A useful three-band framing:

- **Hot-tech (3-month window).** Fast-moving frameworks (LLM orchestration, agent libraries), model catalogs and pricing, governance/security advisories. Anything older than 3 months has significant risk of having been superseded.
- **Federal-regulatory (6-month window).** Regulations, agency policies, compliance guidance. These change slower than tech but faster than foundational standards; 6-month-old guidance often holds but needs verification.
- **Foundation-stable (12-month window).** Language LTS versions, mature databases, foundational web standards, networking primitives, OS-level patterns. Twelve-month-old foundational content is usually fine.

The discipline is to *name the recency category* for each source you cite. A source from outside its category's window is not automatically wrong, but you should know you're using it and justify it.

### 3.4 The sub-30-day source

A source dated within the last 30 days is special. It may be:

- A newly released framework version with breaking changes,
- A vendor announcement that has not yet been validated by independent practitioners,
- A blog post by an early adopter whose experience may not generalize,
- A regulatory advisory whose interpretation is still being established.

The right discipline for sub-30-day sources is not "exclude" but "explicit handling": surface the source's date, name the risk class, and make an explicit *adopt / defer / omit* decision. Adopting a 12-day-old blog post might be correct — but it should be a decision, not an accident [4].

### 3.5 Known-bad patterns

For any given technology family, there are *known-bad patterns* — patterns that were once correct, are now superseded, but still dominate older tutorials and AI-generated summaries because the model's training data is older than the framework's current state. Examples (generic, not framework-specific):

- *Deprecated API idioms.* The old way to call a function that still appears in tutorials but raises deprecation warnings or has been removed.
- *Superseded model identifiers.* Older model IDs that no longer exist in the current catalog.
- *Abstract patterns the framework has moved away from.* Foundational primitives that have been replaced by simpler shapes; tutorials still teach the old foundation because they predate the change.

Maintaining a project-local *known-bad-patterns* list — and checking candidate code/patterns against it — is one of the cheapest interventions against stale-source contamination. When a candidate pattern matches, you don't have to re-litigate why it's wrong; the list tells you what to substitute and points at current docs [4].

### 3.6 The deep-research-agent caution

In 2026, deep-research agents (Gemini Deep Research, Manus, Perplexity Deep Research, OpenAI's research tool) have become common; cohort engineers will use them. The 2026 evaluation benchmarks reveal a consistent failure mode: even high-quality agents misattribute claims, cite irrelevant sources, or summarize patterns the source actually contradicts. Reported correct-statement ratios range from ~70% to ~82% across leading systems [4].

The implication for your own research workflow: deep-research output is a *starting point*, not a citable conclusion. Use it to surface candidate sources, then verify each source independently using the rubric above. Cite the primary source you verified, not the agent that found it.

## 4. Generic Implementation

A minimal scenario-alternatives ADR template, applied to a generic non-federal example (choosing a feature-flag system for a mid-sized SaaS):

```markdown
# ADR-2026-04-12: Feature-flag system

## Status
Proposed (decision needed before sprint 14)

## Context
We are deploying ~30 new features over the next 6 months and need:
- per-user / per-tenant flag targeting,
- gradual rollout with percentage gates,
- audit trail (who flipped what, when),
- a self-host option (data-residency constraint from EU customers).

## Decision drivers (criteria, consistent across alternatives)
1. Self-host viable (data residency)
2. Engineering effort to integrate (weeks of dev time)
3. Cost at our scale (~50k MAU)
4. Audit-log fidelity
5. SDK support for our stack (Python + TypeScript)
6. Operational burden (do we operate it, or vendor does?)

## Considered alternatives

### Option A: LaunchDarkly (managed SaaS)
- Self-host: No (deal-breaker for EU subset)
- Integration: ~1 week
- Cost: $$$
- Audit: Strong
- SDKs: Strong
- Ops burden: None
Source: LaunchDarkly docs (vendor, hot-tech, retrieved YYYY-MM-DD).

### Option B: Unleash (open source, self-hosted)
- Self-host: Yes
- Integration: ~2 weeks
- Cost: Infra only
- Audit: Strong
- SDKs: Strong
- Ops burden: We run it
Source: Unleash docs (project, hot-tech, retrieved YYYY-MM-DD); independent
operations write-up from Company X (secondary, foundation-stable, retrieved YYYY-MM-DD).

### Option C: Build in-house
- Self-host: Yes (trivially)
- Integration: ~4 weeks initial + ongoing maintenance
- Cost: Salary
- Audit: As good as we build it
- SDKs: As good as we build them
- Ops burden: Highest
Source: N/A — internal capability assessment.

## Decision
Option B (Unleash). Self-host requirement disqualifies A; B and C have
similar capability profile but C costs ~4× the engineering time and we
have no differentiated reason to own the implementation.

## Consequences
- We take on ops for one more component (mitigated by Unleash's managed-cloud
  fallback if we change our mind on data residency).
- We accept a 2-week integration cost; alternative was a deal-breaker for
  the EU customer segment.
- We revisit if our MAU grows past ~500k (different cost profile).

## Sources verified
1. Unleash official docs, retrieved YYYY-MM-DD (hot-tech window, in-range).
2. LaunchDarkly pricing page, retrieved YYYY-MM-DD (hot-tech, in-range).
3. Company X operations write-up, published 2025-08, retrieved YYYY-MM-DD
   (foundation-stable, 9-month-old, in-range; vendor-independent so authority OK).
```

What each piece does:

- **Status + context** make the decision *time-localized* — a future reader knows what was true when this was decided.
- **Decision drivers** are the consistent criteria — every alternative is scored on the same dimensions.
- **2–4 alternatives** including a build-it-yourself option as a forcing function.
- **Decision** flows from the drivers; it is the *output*, not the input.
- **Consequences** name the second-order effects you're accepting.
- **Sources verified** lists primary sources with retrieval dates and recency-category labels.

## 5. Real-world Patterns

**Fintech — payment-gateway selection (Stripe/Adyen/Braintree).** Production ADRs at payment companies show a consistent pattern: 3–4 alternatives, ~8 consistent criteria (latency, fees, regional coverage, dispute-handling, PCI scope, currency support, vendor lock-in, SDK quality), one explicit "build in-house" option that's typically rejected on cost. The ADR survives 5+ years and is the load-bearing reference when the question is reopened on contract renewal [2].

**Healthcare — interoperability standard (FHIR vs HL7 v2 vs proprietary).** Hospital IT departments document interop-standard choices as scenario-alternatives ADRs because the choice is sticky on the scale of decades. The criteria typically include regulatory mandate, ecosystem support, IT-staff skills, vendor coverage of the standard, and migration cost. Sources are heavily primary (standards-body docs, CMS publications, EHR vendor docs) because secondary tutorials lag the regulatory environment too much [1].

**Enterprise SaaS — observability stack (Datadog vs Grafana stack vs in-house).** A 2026 production retro described a company that initially picked Datadog by intuition, hit cost issues at scale, and reopened the decision two years later as a scenario-alternatives exercise. The retro's lesson: the original "decision" was a vendor demo + a quick proof-of-concept, not a documented comparison. The reopened decision produced an ADR with five criteria, three alternatives, and a recommendation to migrate selectively — a path the original intuition-based decision could not have supported [3].

**Open-source — build-tool migration (Gradle vs Bazel vs Buck).** Large open-source projects publish their build-tool ADRs as part of their contribution docs. The pattern: criteria emphasize community size, build correctness guarantees, incremental-build speed, and team-familiarity cost. The ADRs typically cite primary sources (the build tools' own docs, benchmarks from teams that have migrated) and explicitly note which secondary tutorials were *not* used because of recency/authority issues [3].

## 6. Best Practices

- **Frame the question before searching.** A vague question generates undirected research. Constrain the question by performance target, regulatory boundary, and existing-architecture commitment.
- **Cite primary sources.** Project docs, RFCs, official blog posts, regulatory filings. Use secondary sources to find primary ones, not as endpoints.
- **Tag every source with retrieval date and recency category.** A claim's truth has a half-life; the date is part of the citation.
- **Treat deep-research-agent output as a candidate-source list, not a conclusion.** Verify every cited source independently using the rubric.
- **Surface sub-30-day sources for explicit decision.** Adopt, defer, or omit — never silently include.
- **Maintain a known-bad-patterns list per project / per technology family.** Cheap to maintain, expensive to skip.
- **Keep alternatives between 2 and 4.** Two forces comparison; more than four becomes unmanageable and signals a poorly-framed question.
- **Write the consequences section honestly.** Every architectural decision has trade-offs; an ADR with no negative consequences is incomplete.

## 7. Hands-on Exercise

**ADR sketch (15 min).** Pick a real architectural decision you face (or imagine one for your current project — but not from the federal-acquisitions domain). Examples: queue technology, deployment platform, frontend framework, monitoring tool, payment processor, search backend.

Produce a scenario-alternatives ADR sketch with:

1. The question, phrased concretely (1 sentence including at least one quantitative constraint).
2. 3–5 decision-driver criteria, each one objectively measurable.
3. 2–4 alternatives, each evaluated against all criteria (use a table).
4. One source per alternative, with author/publisher, date, recency category, and primary/secondary classification.
5. A recommendation with one-sentence justification keyed to the criteria.
6. One consequence you're accepting and one you'd revisit later.

**What good looks like:** Your question is specific enough that two engineers reading it would evaluate the same alternatives. Your criteria are observable (e.g., "p95 latency under 100ms," not "fast enough"). Your alternatives include at least one option you don't recommend — and you've taken its case seriously. Every source has a date and a recency category in-window. Your recommendation cites the criteria that drove it, not personal preference. You name a real consequence (this is the discipline test — there is always a consequence; if you can't name one, you haven't looked hard enough).

## 8. Key Takeaways

- *What are the four parts of a well-formed scenario-alternatives exercise?* (Question, constraints, 2–4 alternatives, consistent evaluation criteria.)
- *What does the source-evaluation rubric (authority/currency/accuracy/relevance/objectivity) buy you, and why is authority especially load-bearing for LLM-era tech?* (It filters stale or hallucinated content; primary sources outrank secondary ones, and version-churn makes secondary tutorials silently incorrect.)
- *Why does a sub-30-day source warrant explicit adopt/defer/omit handling?* (It may not have been validated by independent practitioners, and its risk class is different from older sources; an explicit decision is the right discipline.)
- *Why is deep-research-agent output a starting point, not a citable conclusion?* (Benchmarks show consistent misattribution and citation errors; verify each source independently before citing.)
- *Why must consequences be written honestly?* (Every architectural decision has trade-offs; an ADR with no negative consequences is incomplete and produces over-confident future decisions.)

## Sources

1. [Maintain an architecture decision record (Microsoft Azure Well-Architected Framework)](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record) — retrieved 2026-05-26
2. [Master architecture decision records (ADRs): Best practices (AWS Architecture Blog)](https://aws.amazon.com/blogs/architecture/master-architecture-decision-records-adrs-best-practices-for-effective-decision-making/) — retrieved 2026-05-26
3. [Architecture Decision Record (Martin Fowler's bliki)](https://martinfowler.com/bliki/ArchitectureDecisionRecord.html) — retrieved 2026-05-26
4. [DeepTRACE: Auditing Deep Research AI Systems for Tracking Reliability Across Citations and Evidence (arXiv)](https://arxiv.org/pdf/2509.04499) — retrieved 2026-05-26
5. [Criteria for Evaluating Sources (UMass Global Research Guides)](https://umassglobal.libguides.com/c.php?g=1267504&p=9295343) — retrieved 2026-05-26

Last verified: 2026-05-26
