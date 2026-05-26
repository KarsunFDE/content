---
week: W05
day: Mon
topic_slug: aiops-decisions-three-adrs
topic_title: "AIOps Decisions — the three ADRs you draft on Plan Day"
parent_overview: W05/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://github.com/joelparkerhenderson/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://aws.amazon.com/blogs/architecture/master-architecture-decision-records-adrs-best-practices-for-effective-decision-making/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://martinfowler.com/bliki/ArchitectureDecisionRecord.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://adr.github.io/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
last_verified: 2026-05-26
---

# AIOps Decisions — the three ADRs you draft on Plan Day

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define what an Architecture Decision Record (ADR) is, and articulate why immutability + status tracking are non-negotiable to the practice.
- Distinguish a load-bearing ADR (a commitment with downstream consumers) from an open-question note that pretends to be an ADR.
- Apply the Nygard template (Context / Decision / Consequences) to a real platform-choice decision and explain its trade-offs in the form a future engineer would need.
- Recognise the three ADR archetypes that typically anchor an AIOps adoption cycle: platform choice, managed-service migration boundary, and authority/HITL seed.

## 2. Introduction

An Architecture Decision Record is a short document that captures an important architectural decision along with its context and consequences ([Martin Fowler, 2026-05-26](https://martinfowler.com/bliki/ArchitectureDecisionRecord.html)). The format is old (Michael Nygard's original blog post is from 2011), the discipline is straightforward (one decision per record, in source control, append-only with status), and the practice is unevenly adopted.

Where it does land — well — teams report two compounding benefits: future engineers can read the *why* not just the *what*, and the structured format forces today's team to consider alternatives and trade-offs explicitly rather than coast on the loudest voice in the room.

For an AIOps adoption, three ADR archetypes recur across engagements:

1. **Platform-choice ADR** — which AIOps platform, with named pivot conditions.
2. **Managed-service migration boundary ADR** — when do you move a workload off the current stack onto a managed service, with the boundary explicit.
3. **Authority-seed ADR** — for any platform feature that automates remediation, what's the starting position for human-in-the-loop authority.

This reading explains the practice generically and walks each archetype in turn. The day's overview pins the three ADRs to the cohort's specific work this week; here we look at what makes each archetype durable.

## 3. Core Concepts

### 3.1 What an ADR is (and isn't)

An ADR is a one-page (or thereabouts) document with at minimum: title, status, context, decision, consequences ([Nygard's original template, 2026-05-26](https://github.com/joelparkerhenderson/architecture-decision-record)).

What it is:

- Immutable once accepted (you supersede with a new ADR, you don't edit).
- Single-decision (one core direction per record, not a wish list).
- Stored in the source repo it applies to (most commonly `doc/adr/` or `docs/adr/`).
- Status-tracked: proposed → accepted → superseded.

What it isn't:

- A design doc. ADRs reference design docs; they don't replace them.
- An RFC. RFCs gather opinion; ADRs commit the decision.
- A retrospective. The retrospective is a separate artifact that may surface an ADR-worthy change.
- A backlog. "We'll decide next week" is an open question, not an ADR.

### 3.2 The Nygard template

The widely used Michael Nygard framework — Context / Decision / Consequences — usually works, with cloud projects often extending it with a compliance section ([AWS Architecture Blog, 2026-05-26](https://aws.amazon.com/blogs/architecture/master-architecture-decision-records-adrs-best-practices-for-effective-decision-making/)):

```markdown
# ADR-NNN: <Title>

## Status
<Proposed | Accepted | Superseded by ADR-MMM | Deprecated>

## Context
<Why is this decision needed? What forces are at play?>

## Decision
<What did we decide? State it crisply.>

## Consequences
<What becomes easier? What becomes harder? What new work does this create?>

## Alternatives considered
<The serious alternatives, each with pros/cons and why rejected.>

## Compliance / regulatory notes (optional)
<Boundary calls — FedRAMP, HIPAA, SOX, etc.>
```

Variants exist: MADR (Markdown ADR) is a heavier template common in cloud teams; Y-Statements compress an ADR to a single sentence for indexing ([ADR templates index, 2026-05-26](https://adr.github.io/adr-templates/)). Pick one and stay consistent within a project.

### 3.3 Status as a load-bearing field

A team can write 30 ADRs in a year. Without status, the document set becomes archaeology. With status — and the rule that once accepted you supersede rather than edit — it becomes a navigable history: which decisions are live, which are obsolete, which obsoleted which. The discipline is what makes ADRs scale; without it the practice rots into shelfware ([Microsoft Azure Well-Architected, 2026-05-26](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record)).

### 3.4 The three AIOps-adoption archetypes

#### Archetype A: Platform-choice ADR

What it commits: which AIOps platform the team adopts as default.

What makes it load-bearing: it has *named pivot conditions* — under what circumstances the team would choose a different platform. A platform-choice ADR without pivot conditions is loyalty, not decision-making.

Typical structure:

- Context: the engagement shape, telemetry volume, FedRAMP requirement, team familiarity.
- Decision: platform X, hands-on this engagement.
- Consequences: which capabilities you gain, which you give up, the lock-in shape, the cost curve.
- Alternatives: each compare-set platform with one paragraph on why rejected here — and explicitly, the engagement shape that would flip the choice.

#### Archetype B: Managed-service migration boundary ADR

What it commits: whether (and where) a workload moves from a self-managed component to a cloud-managed equivalent.

What makes it load-bearing: the *boundary* — latency budget, multi-tenant isolation guarantee, FedRAMP boundary placement, vendor lock-in cost. "We'll decide later" is not an ADR; "we will migrate iff latency stays under N ms and tenant isolation can be enforced via metadata filters" is.

Typical structure:

- Context: the workload's current owner, its performance and isolation properties, the managed service's claimed boundary.
- Decision: migrate / hybrid / no-migrate, with the boundary stated.
- Consequences: what changes for ops, what changes for cost, what changes for compliance.
- Alternatives: stay-on-self-managed; abstract behind a port-and-adapter so the choice is reversible; full rip-and-replace.

#### Archetype C: Authority/HITL seed ADR

What it commits: a starting position for automated-remediation authority on a feature that has not yet been built or measured in production.

What makes it load-bearing: the *seed* is acknowledged provisional. The team commits a starting position (full-auto / propose-and-approve / escalate-only) knowing the *final* ADR lands after the feature ships and gets measured. The seed forces the conversation early; the final ADR closes it later with data.

Typical structure:

- Context: the feature, the risk of an automated action getting it wrong, the cohort/audience the action affects.
- Decision: seed position (one of the three modes), valid until the feature ships measurable signals.
- Consequences: what the seed enables this week; what it defers.
- Alternatives: each of the other authority modes, with the conditions under which each would be the right seed instead.

## 4. Generic Implementation

A worked ADR for a hypothetical e-commerce platform choosing a search engine — generic enough to translate to any platform-choice decision.

```markdown
# ADR-007: Adopt OpenSearch (self-managed) for product-catalog search

## Status
Accepted (2026-05-26)

## Context
The product-catalog search currently runs on a Postgres full-text index. Catalog
growth has pushed the index into degenerate query plans at peak; latency p95
exceeded the 300ms storefront budget twice in the last quarter. We need a
dedicated search engine. Two managed options (Elastic Cloud, Algolia) and two
self-managed options (OpenSearch, Meilisearch) were evaluated.

## Decision
Adopt OpenSearch on Kubernetes, self-managed, for the product-catalog search.
Migration completes by end of Q3; the Postgres FTS path is removed once
shadow-traffic comparison shows < 0.5% relevance regression.

## Consequences
- We own the operational surface (cluster, snapshots, version upgrades).
- We avoid per-query SaaS pricing that scales with catalog size.
- We must onboard the SRE team on OpenSearch's operational profile (rolling
  restarts, shard rebalancing, ingestion backpressure).
- Cost projection: 60% of Elastic Cloud equivalent at current scale; the curve
  crosses if catalog grows > 5×, at which point we re-evaluate.

## Alternatives considered
- **Elastic Cloud (managed):** lower ops overhead, higher per-document cost.
  Rejected at current scale; revisit when ops cost > $X/month or team headcount
  drops below threshold.
- **Algolia:** strongest relevance + speed defaults, lowest ops. Rejected on
  per-query pricing model — catalog has high cardinality of free-tier browsers
  which inflates query count.
- **Meilisearch:** simpler operational profile, smaller feature set. Rejected on
  faceting capability; faceted nav is core to the storefront.

## Pivot conditions
We would supersede this ADR and move to Elastic Cloud if:
1. Team headcount supporting search drops below 2 SREs, OR
2. Cluster reliability incidents exceed 1/quarter for two consecutive quarters, OR
3. Catalog scale crosses the cost-curve crossover identified above.
```

The pivot-conditions block is what makes this ADR durable: a future engineer reading it doesn't have to guess whether the original team was sure or hedging — they wrote down what would make them change their mind.

## 5. Real-world Patterns

**Healthcare imaging — platform choice ADR with explicit FedRAMP boundary**. A medical-imaging SaaS adopting a new APM platform wrote a platform-choice ADR that named FedRAMP High as a hard requirement. The ADR's pivot conditions were unusual: not "if pricing changes" but "if the vendor loses its FedRAMP authorization, this ADR is automatically deprecated and the team chooses from the still-authorized compare-set." The mechanism — auto-deprecation on regulatory event — made the ADR future-proof against vendor-specific risk ([AWS Architecture Blog on ADR best practices, 2026-05-26](https://aws.amazon.com/blogs/architecture/master-architecture-decision-records-adrs-best-practices-for-effective-decision-making/)).

**Fintech — managed-service migration ADR with explicit boundary**. A payment-processor evaluating a move from self-managed Kafka to a managed streaming service wrote an ADR whose decision was *hybrid*: critical settlement events stay on self-managed Kafka (latency budget unforgiving, compliance boundary inside the team's control), while telemetry and audit events move to the managed offering. The ADR's value was the explicit boundary; without it the team would have either over-migrated (latency cost) or under-migrated (ops cost) ([ADR practice overview, 2026-05-26](https://martinfowler.com/bliki/ArchitectureDecisionRecord.html)).

**Logistics — authority-seed ADR for auto-remediation**. A logistics platform deploying anomaly-driven auto-scaling wrote an authority-seed ADR for the auto-scaling action: seed position propose-and-approve (human acknowledges each scale-out), valid until 30 days of operation data accumulate, at which point a final ADR supersedes. After 30 days they had enough confidence to move to full-auto for scale-out (low blast radius) and kept propose-and-approve for scale-in (data-loss risk). The seed/final pattern is what kept the conversation from being either too cautious upfront or too cavalier ([Joel Parker Henderson ADR examples, 2026-05-26](https://github.com/joelparkerhenderson/architecture-decision-record)).

**E-commerce — three-archetype rollout for a search platform**. A multi-brand e-commerce platform rolling out a new search engine ended up writing all three archetypes in a single quarter: platform-choice (Elastic vs OpenSearch), managed-service-boundary (which workloads stay self-managed), and authority-seed (when can the system reindex automatically). The team's observation was that the three archetypes are not arbitrary categories — they show up together because adopting any non-trivial platform forces all three conversations.

## 6. Best Practices

- **One decision per ADR.** Avoid combining multiple architecture decisions; each ADR should address one core technical direction.
- **Immutable once accepted.** Don't edit; supersede with a new ADR and link forward. The history is the value.
- **Name pivot conditions explicitly.** A platform-choice ADR without conditions under which you'd change platforms is loyalty masquerading as a decision.
- **Differentiate seed from final.** Authority/HITL ADRs are often premature when written; commit a seed position with an explicit expiry condition, and write the final ADR after measurement.
- **Store ADRs alongside the code they govern.** Most commonly `doc/adr/` or `docs/adr/`; keep them in source control so they version with the system.
- **Run readout review meetings.** Spend 10–15 minutes of a meeting silently reading the proposed ADR, then collect written comments; this gets you reviewed records, not rubber-stamped ones.
- **Keep ADRs brief.** A single page or thereabouts. If it can't fit, the decision is probably two decisions in disguise — split it.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** Pick a real platform-choice decision from your past — a database, a queueing system, a UI framework, a CI/CD tool. Write the Nygard-template ADR for it after the fact:

- Context: what forces drove the decision when you made it?
- Decision: state it crisply.
- Consequences: what got easier? What got harder?
- Alternatives considered: name 2–3 serious alternatives and why rejected.
- Pivot conditions: under what circumstances would you change the decision today?

**What good looks like.** A finished page reads like the team's actual reasoning, not the ad copy of the chosen vendor. The alternatives section names real trade-offs (not strawmen). The pivot conditions are specific (a number, a date, an event) — not vague ("if it gets bad"). If the pivot conditions can't be written, the decision is probably loyalty, not a decision; flag it for revision.

## 8. Key Takeaways

- *What is an ADR?* A one-page record of one architectural decision, immutable once accepted, status-tracked, and stored alongside the code it governs.
- *Why is status load-bearing?* Without it, the ADR set rots into archaeology; with it, the history becomes navigable — which decisions are live, which superseded, which deprecated.
- *What separates a load-bearing platform-choice ADR from a loyalty statement?* Named pivot conditions — the specific events that would supersede the decision.
- *What three archetypes anchor an AIOps adoption cycle?* Platform choice, managed-service migration boundary, and authority/HITL seed — they recur together because adopting any non-trivial platform forces all three conversations.

## Sources

1. [Architecture Decision Records — Joel Parker Henderson examples and templates](https://github.com/joelparkerhenderson/architecture-decision-record) — retrieved 2026-05-26
2. [Master architecture decision records (ADRs): Best practices (AWS Architecture Blog)](https://aws.amazon.com/blogs/architecture/master-architecture-decision-records-adrs-best-practices-for-effective-decision-making/) — retrieved 2026-05-26
3. [Architecture Decision Record (Martin Fowler bliki)](https://martinfowler.com/bliki/ArchitectureDecisionRecord.html) — retrieved 2026-05-26
4. [Architectural Decision Records — adr.github.io](https://adr.github.io/) — retrieved 2026-05-26
5. [Maintain an architecture decision record (Microsoft Azure Well-Architected)](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record) — retrieved 2026-05-26

Last verified: 2026-05-26
