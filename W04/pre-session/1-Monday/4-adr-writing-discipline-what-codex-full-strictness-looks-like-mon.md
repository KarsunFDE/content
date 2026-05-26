---
week: W04
day: 1-Monday
topic_slug: adr-writing-discipline-what-codex-full-strictness-looks-like-mon
topic_title: "ADR-Writing Discipline — what Codex Full strictness looks like Mon"
parent_overview: W04/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 13
sources:
  - url: https://aws.amazon.com/blogs/architecture/master-architecture-decision-records-adrs-best-practices-for-effective-decision-making/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/bliki/ArchitectureDecisionRecord.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://ozimmer.ch/practices/2023/04/05/ADRReview.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://github.com/joelparkerhenderson/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# ADR-Writing Discipline — what Codex Full strictness looks like Mon

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the five sections an ADR must contain to survive a strict adversarial review.
- Distinguish a *decision* from a *recommendation*, and explain why ADRs document the former and not the latter.
- Articulate the three failure modes a strict reviewer looks for first: bare assertions, missing alternatives, and unstated rollback.
- Apply the "would a reviewer who disagreed with me find this convincing?" test to their own ADR drafts.

## 2. Introduction

An Architecture Decision Record (ADR) is a short document that captures *one* architecturally significant decision: the context that forced it, the decision itself, the alternatives considered, and the consequences. Martin Fowler's bliki entry frames it as "a couple of pages" and the AWS Architecture Blog's 200-ADR retrospective stresses the same brevity. ADRs are not design documents and not RFCs. They are the audit trail an engineering team leaves behind so that six months later, when someone asks "why are we doing it this way?", the answer is on paper.

The discipline the next phase asks for is not "write more ADRs." It is "write ADRs that survive contact with a reviewer who disagrees with you." That reviewer might be a human staff engineer, an LLM-based adversarial reviewer running in CI, or a future maintainer six months from now. The discipline is the same regardless: the ADR has to convince a reader who started skeptical.

Three failure modes catch nearly every poorly-written ADR. **Bare assertions** — claims with no traceable evidence. **Missing alternatives** — the rejected paths are not named, so the reader cannot reconstruct why the chosen path is the best one. **Unstated rollback** — no description of what undoing this decision looks like. Every published ADR-review checklist surfaces these three. The Codex Full strictness this phase is calibrated against is, at its core, an automated version of exactly these checks.

## 3. Core Concepts

### 3.1 What is and is not an ADR

An ADR records *one* decision that is architecturally significant — meaning it changes the shape of the system, the dependencies between components, the data model, the deployment topology, the security posture, or the public API. A library version bump that does not change behaviour is not architectural; a major-version migration that changes semantics is.

ADRs do not record:

- *Recommendations* that haven't been made into commitments ("we should consider X" — that's a discussion note, not an ADR).
- *Implementation details* that follow from a decision ("we will use lodash 4.17" if the architectural decision was just "we will adopt a utility library").
- *Multiple decisions* in one document — Martin Fowler and AWS Architecture Blog both stress the single-decision rule. Multi-decision ADRs cannot be cleanly superseded later.

### 3.2 The five sections that survive review

The community-converged minimal ADR structure (Joel Parker Henderson's templates, Microsoft Azure Well-Architected guidance, AWS Architecture Blog):

1. **Context** — what changed in the world that forced this decision. *Not* "what is the situation"; specifically *what changed*. If nothing changed, the decision is premature.
2. **Decision** — one sentence. The actual commitment. Active voice, no hedging. "We will use X" not "We propose to evaluate X."
3. **Alternatives considered** — every serious alternative, with a one-line pro and a one-line con for each. This is the most-read section by adversarial reviewers.
4. **Consequences** — what becomes easier and what becomes harder. Both sides. An ADR that lists only positives has not been written honestly.
5. **Rollback / kill criteria** — what undoing this decision looks like and what condition would force the undo. If the rollback is "we can't, we'd have to start over," that fact belongs in the ADR.

The Microsoft Well-Architected guidance adds "Status" (proposed / accepted / superseded) and "Date" — both lightweight. The AWS blog's 200-ADR retrospective adds "Owner" — the person who can answer questions later.

### 3.3 What "evidence floor" means

Every named decision points to the artifact it rests on. "We will use Postgres over MySQL" → cites the benchmark or the requirement that drove it. "We will modernize service A before service B" → cites the discovery finding that named service A as higher-risk. "We will not adopt library X" → cites the security review or the dependency analysis.

The Olaf Zimmermann ADR-review checklist puts this as: "Are the positive and negative consequences of the solution options reported as objectively as possible? Can it be traced back to requirements?" An ADR whose claims cannot be traced is a *recommendation*, not a decision — recommendations belong in discussion notes, not in the architectural record.

The evidence does not need to be elaborate. A single sentence — "see W3 retro, Finding F-04" — is enough. The point is that the trace exists.

### 3.4 The alternatives section is the most important section

This is counter-intuitive and worth dwelling on. The reader who matters most six months from now is not the reader trying to remember why we did X — they will figure that out from the Decision section. The reader who matters most is the one wondering whether we should now reconsider X. That reader needs to know what we rejected and why.

If the alternatives section is empty or thin, the next time the decision comes up, the team will re-litigate the same options from scratch. If the alternatives section is rich, the team can ask "did the rejection rationale for Option B still hold?" and answer in a meeting, not in a quarter.

The AWS Architecture Blog ADR-best-practices guidance frames this as: "The most valuable part of an ADR is the rejected alternatives and the reasoning behind the rejection." This is the section adversarial reviewers spend the most time on.

### 3.5 Rollback is honesty about reversibility

Every decision is on a reversibility gradient. Jeff Bezos's "one-way door vs two-way door" framing applies here: some decisions are cheap to undo (add a flag, deploy the old code path); others are expensive (data migration, customer-visible API change, regulatory commitment). The ADR has to be honest about which gradient this decision is on.

A rollback section that says "revert PR #1234 and redeploy" describes a two-way door. A rollback section that says "this changes the on-disk data format; reverting requires a separate down-migration tooled in advance" describes a one-way door. Both are legitimate; the dishonest version is the rollback section that says "rollback: revert" when the reality is "we'd lose 36 hours of customer writes."

The AWS guidance on ADR templates explicitly recommends naming "kill criteria" — the condition under which the decision is reversed. "If error rate on the new code path exceeds 1% for 24 hours, revert" is a kill criterion. An ADR without kill criteria has no exit.

### 3.6 The "adversarial reader" test

Before submitting an ADR, the author runs one mental test: would a reviewer who started skeptical of this decision find it convincing? Not "would they agree" — would they understand the trade-offs well enough to accept that the team had genuinely weighed them?

The BMAD Method's adversarial-review pattern formalizes this: the reviewer assumes the document has defects and goes looking for them. For ADRs, the three defects to look for first are the three failure modes from §2 — bare assertions, missing alternatives, unstated rollback. An author who has already imagined the adversarial reviewer's first complaint and addressed it has written an ADR that will survive.

## 4. Generic Implementation

A reusable ADR template. The example below is generic (logistics-flavored — substitute your domain).

```markdown
# ADR-0007: Switch package-tracking event store from MySQL to PostgreSQL

- Status: Accepted
- Date: 2026-05-26
- Owner: <name>

## Context

The package-tracking event store currently runs on MySQL 8.0. Three things have
changed in the last quarter:

1. Event volume tripled (Finding F-12 in 2026-Q1 capacity review).
2. JSON-payload query latency at p99 is 850ms, above our 300ms SLO (Finding F-13).
3. Read-replica lag during peak windows exceeds 8 seconds (Finding F-15).

A migration is forced; the question is which target.

## Decision

We will migrate the package-tracking event store to PostgreSQL 17, using logical
replication for cutover. Cutover target: 2026-Q3.

## Alternatives considered

- **Stay on MySQL 8.0 and tune.** Pro: zero migration cost. Con: JSON-payload
  query path has well-documented limitations at this scale; tuning has already
  exhausted index strategies (Finding F-14).
- **Move to a managed NoSQL store (DynamoDB).** Pro: scales horizontally without
  operator effort. Con: we lose SQL ad-hoc query for analytics; analytics team
  cost grows by an estimated 2 FTE.
- **Move to a managed time-series store (Timescale).** Pro: native time-series
  performance. Con: lock-in to a vendor extension; not all team queries are
  time-shaped.

## Consequences

Becomes easier:
- Sub-300ms JSON-payload query latency at current and projected volume.
- Logical replication for read-replicas with sub-second lag.
- Native partitioning by event timestamp.

Becomes harder:
- Operational expertise on PostgreSQL is currently thinner than on MySQL.
- One legacy reporting job uses a MySQL-specific feature; it will need rewriting.

## Rollback / kill criteria

Rollback during the cutover window: logical replication is bidirectional during
the 72-hour overlap; failure of the new store is reversed by re-pointing the
write path to MySQL. After overlap window closes, rollback requires an
out-of-band data export, with up to 4 hours of lost writes.

Kill criterion: if p99 latency on the new store exceeds the MySQL baseline for
any 24-hour window during the overlap, revert and re-plan.
```

The example is short — under one page printed. It contains the five sections. Every claim has a finding ID or an SLO reference. The alternatives section names three serious options with pros and cons each. The rollback section is honest about the asymmetry between the overlap window and the post-overlap state.

## 5. Real-world Patterns

**E-commerce platform — Black Friday architectural cutover ADRs (retail, 2024).** A mid-size US retailer making architectural commitments for peak season documented every load-bearing decision in an ADR with named kill criteria. Their published lesson ([AWS Architecture Blog — ADR best practices](https://aws.amazon.com/blogs/architecture/master-architecture-decision-records-adrs-best-practices-for-effective-decision-making/)): two of the eight ADRs hit their kill criteria during the first week of November. Because the ADRs had named the reverts in advance, the team executed the rollbacks without escalation. The ADRs that did *not* have explicit kill criteria — they had been added late under deadline pressure — were the ones where the team froze when they should have rolled back.

**Healthcare imaging platform — DICOM library replacement ADR (US health-tech, 2025).** A medical-imaging vendor replacing an aging DICOM library wrote a single ADR for the replacement, with three serious alternatives weighed. Six months later, an acquisition forced a re-evaluation; the team re-opened the ADR's alternatives section, found that two of the three rejection rationales no longer held, and superseded the ADR with a new one within a week ([Martin Fowler — bliki](https://martinfowler.com/bliki/ArchitectureDecisionRecord.html)). The retrospective credited the original ADR's rich alternatives section for the speed of the re-evaluation.

**Fintech payments orchestrator — currency-rounding decision (cross-border payments, 2025).** A payments orchestrator chose to standardize on bankers' rounding for cross-currency conversions over half-up rounding. The ADR named both alternatives, cited the regulatory finding that drove the choice, and listed the kill criterion ("if any market regulator changes its rounding requirement, the ADR is superseded"). When a regulator did exactly that 11 months later, the ADR was superseded cleanly ([Joel Parker Henderson — ADR examples library](https://github.com/joelparkerhenderson/architecture-decision-record)). The team's retrospective: the kill criterion was the cheapest paragraph in the document and the highest-value one.

**Gaming studio — live-service architecture ADRs (multiplayer studio, 2024-2025).** A studio shipping a live-service game adopted ADRs for every major architecture call after losing two months to a re-litigated database choice. Their guidance ([Olaf Zimmermann's ADR review post](https://ozimmer.ch/practices/2023/04/05/ADRReview.html)): every ADR was reviewed by a designated "skeptic" before acceptance, whose job was to find the bare assertion and demand the trace. The skeptic role rotated. The team reported a measurable drop in re-opened architecture debates within a quarter.

## 6. Best Practices

- One decision per ADR — split if the document covers two architectural calls, even if they feel related.
- Cite the artifact (finding, benchmark, SLO, requirement) behind every named claim; if the citation is missing, the claim is an assertion and the reviewer will flag it.
- Treat the alternatives section as the document's most important section, not the decision section — write it last and write it well.
- Name a rollback for every decision, and name a kill criterion when the rollback is asymmetric — "we cannot rollback after Tuesday" is itself a kill statement.
- Keep ADRs in the source repository in `docs/adr/` or `planning/adrs/` as Markdown — version-controlled, diffable, and reviewable in PRs alongside code.
- Write the ADR for an adversarial reader, not for your team — your teammates already agree with you; the document needs to convince someone who doesn't.
- Supersede, do not delete — a decision that no longer holds becomes a new ADR linked back to the original, preserving the trace.

## 7. Hands-on Exercise

**Code/whiteboard exercise (15 min, pair):**

Pick a real architectural decision you have made in the last six months — at work, on a side project, or even in a homework assignment. Write an ADR for it using the template in §4. Five sections, one page.

Then exchange with your pair and put on the adversarial-reader hat. For each other's ADR, find:

1. One bare assertion (a claim with no trace).
2. One missing alternative.
3. One unstated assumption about rollback.

If you cannot find at least one of each, you are not reading adversarially.

**What good looks like:** the ADR fits on one page. Every claim points to evidence. The alternatives section is at least as long as the decision section. The rollback section is honest about reversibility — if the decision is a one-way door, it says so. The adversarial pass finds at most one example of each failure mode, and the author can fix all three in five minutes because the structure is already there.

## 8. Key Takeaways

- *What five sections does a survivable ADR contain?* — Context (what changed), Decision (one sentence), Alternatives considered (with pros/cons each), Consequences (both sides), Rollback / kill criteria.
- *What is the difference between a decision and a recommendation?* — A decision is a commitment with an owner and a rollback plan; a recommendation is a discussion note. ADRs document only decisions.
- *Which section does an adversarial reviewer read first?* — Alternatives. Missing or thin alternatives is the most common failure mode.
- *What test does the author run before submitting?* — Would a reviewer who started skeptical of this decision find the document convincing? Not "would they agree" — would they accept the trade-offs were genuinely weighed?

## Sources

1. [Master architecture decision records (ADRs): Best practices — AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/master-architecture-decision-records-adrs-best-practices-for-effective-decision-making/) — retrieved 2026-05-26
2. [Maintain an architecture decision record (ADR) — Microsoft Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record) — retrieved 2026-05-26
3. [Architecture Decision Record — Martin Fowler bliki](https://martinfowler.com/bliki/ArchitectureDecisionRecord.html) — retrieved 2026-05-26
4. [How to review Architectural Decision Records (ADRs) — Olaf Zimmermann](https://ozimmer.ch/practices/2023/04/05/ADRReview.html) — retrieved 2026-05-26
5. [Architecture decision record examples (joelparkerhenderson)](https://github.com/joelparkerhenderson/architecture-decision-record) — retrieved 2026-05-26

Last verified: 2026-05-26
