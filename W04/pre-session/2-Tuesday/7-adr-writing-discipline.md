---
week: W04
day: Tue
topic_slug: adr-writing-discipline
topic_title: "ADR Writing Discipline — Brownfield ADR shape vs Scope ADR shape"
parent_overview: W04/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://adr.github.io/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://adr.github.io/madr/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/welcome.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://github.com/joelparkerhenderson/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# ADR Writing Discipline — Brownfield ADR shape vs Scope ADR shape

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish a *scope ADR* from a *brownfield-analysis ADR* by purpose, audience, and lifetime.
- Choose between the Nygard, MADR, and Y-statement templates based on the decision's complexity.
- Identify the three anti-patterns AWS Prescriptive Guidance names for missing or malformed ADRs.
- Name the four characteristics that make an ADR adversarially reviewable.
- Critique an example ADR against the discipline's load-bearing criteria.

## 2. Introduction

This is the second ADR reading on Tuesday — the first (on brownfield-analysis ADRs for a single module) covered the *contents* of one specific ADR shape. This reading covers the *discipline* of writing ADRs across multiple shapes: when do you write a scope ADR versus a brownfield-analysis ADR versus a future-hop ADR, how do you choose a template, and what does adversarial review look for in each shape.

ADRs are the cheapest high-leverage artifact in software engineering. They take 15–30 minutes to write, fit on one screen, and answer the questions that future engineers will otherwise spend hours archaeologizing for. Michael Nygard's 2011 *Documenting Architecture Decisions* (Cognitect) popularized the format; the GitHub `adr` community site (adr.github.io) now catalogs seven templates, ranging from Nygard's original 5-section shape to MADR's expanded 8-section template to the Y-statement single-paragraph form.

The discipline of ADR writing is not the same as the template choice. The discipline is the *practice* of writing them at the right cadence, in the right shapes, with the right rigor for adversarial review. AWS Prescriptive Guidance (March 2022, retrieved 2026-05-26) names three ADR anti-patterns to avoid: no decision is made out of fear; a decision is made without justification; the decision is not captured in a repository. All three are practice failures, not template failures.

The reason this matters specifically in brownfield modernization is that modernizations produce multiple ADR shapes in the same week. A scope ADR sets the iteration's commitments; per-module brownfield-analysis ADRs detail each hop; future-hop ADRs document evidence for hops planned but not yet executed. A team that uses one template for all three muddles the audience and weakens the artifact's value. The discipline is in knowing which shape to use when.

## 3. Core Concepts

### 3.1 Three ADR shapes that show up in modernization weeks

Three distinct ADR shapes accumulate over a modernization week. Each has a different purpose, audience, and lifetime:

**Scope ADR** — what we are committing to do this iteration and why. Audience is the team plus its reviewers. Lifetime is the iteration. Contains the in-scope work items, the alternatives considered (and explicitly rejected), the success criteria, and the rollback plan. Example title: "W4 Modernization Scope — solicitation-service Jakarta migration + AI security threat model."

**Brownfield-analysis ADR** — what one specific module IS and what one specific hop will change. Audience is the executing pair plus the adversarial reviewer. Lifetime is the hop. Contains the six sections detailed in the prior reading (module under analysis, current state pinned, target state pinned, tools and recipes, pre-flight checks, hand edits required) plus optional rollback. Example title: "solicitation-service Spring Boot 2.7 → 3.5 hop."

**Future-hop ADR** — evidence collected for a hop *planned but not yet executed*. Audience is the team plus future iterations. Lifetime is until the hop is executed or formally deferred. Contains dry-run evidence, representative-module compile attempts, named breaking changes, and at least one hand edit identified. Future-hop ADRs are *paper exercises* that ship as evidence in defense or review without committing the team to execution. Example title: "Future hop — Spring Boot 3.5 → 4.0 evidence collection."

Confusing the shapes produces predictable problems. A scope ADR with brownfield-ADR detail level is unfocused (too much per-module specificity); a brownfield-analysis ADR with scope-ADR breadth is unauditable (claims about multiple modules can't all be verified in one review); a future-hop ADR with scope-ADR commitment language gets the team accused of scope creep.

### 3.2 Template choice — Nygard vs MADR vs Y-statement

The seven templates the `adr` community catalogs span a complexity range:

- **Nygard template (2011)** — minimal: Title, Status, Context, Decision, Consequences. Five sections. Suited to decisions where the team trusts each other and just needs the rationale captured. Reads in two minutes.
- **MADR (Markdown Architectural Decision Records, 4.0 as of 2024)** — expanded: Context and Problem Statement, Decision Drivers, Considered Options, Decision Outcome, Consequences, Confirmation, Pros and Cons of the Options, More Information. Eight sections plus optional metadata. Suited to decisions with multiple defensible alternatives. Reads in five minutes.
- **Y-statement** — single paragraph: "In the context of *X*, facing *Y*, we decided for *Z* (and against *A*, *B*), to achieve *W*, accepting *consequences*." Suited to decisions that are clear enough to compress into one sentence. Reads in 30 seconds.

The discipline is to choose the *minimum sufficient* template. A decision where everyone agrees on the rationale and the alternatives are obviously inferior gets a Y-statement. A decision with three real options and trade-offs gets MADR. A decision in the middle gets Nygard. The wrong template choice is usually MADR-when-Nygard-would-do — eight sections worth of writing that nobody reads because the decision didn't justify the breadth.

For brownfield-analysis ADRs specifically, the six-section custom shape from the prior reading is generally more useful than any of the off-the-shelf templates. The custom shape exists because brownfield work demands current-state and target-state pinning that the generic templates don't include.

### 3.3 The three anti-patterns AWS Prescriptive Guidance names

AWS Prescriptive Guidance's *Using architectural decision records to streamline technical decision-making* (Kunce and Goby, March 2022, retrieved 2026-05-26) names three anti-patterns the discipline targets:

1. **No decision is made at all, out of fear of making the wrong choice.** The ADR practice forces a decision out of indefinite deferral by giving it a structural form. "We chose to defer for these specific reasons" is itself a decision worth capturing.
2. **A decision is made without any justification, and people don't understand why it was made.** The ADR's Context and Decision sections make the rationale explicit; subsequent teams don't re-litigate the same trade-off because the record exists.
3. **The decision isn't captured in an architectural decision repository, so team members forget or don't know that the decision was made.** A team's collection of ADRs is its decision log; without it, decisions live only in slack threads and memory, both of which degrade.

The discipline of ADR writing addresses all three by making the *artifact* the unit of decision-making rather than the *conversation*.

### 3.4 Four characteristics of adversarially-reviewable ADRs

For a brownfield-analysis or scope ADR to survive adversarial review (a senior engineer, codex, an external reviewer), it should exhibit four characteristics:

- **Evidence over assertion.** Every claim links to evidence. "Tests pass" → link to CI run. "Dependency is supported" → link to vendor's lifecycle page. An ADR full of unanchored claims is paper.
- **Alternatives explicitly considered.** Listing 3+ alternatives and naming why each was rejected demonstrates the decision wasn't made by default. An ADR with only one option is suspect.
- **Reversibility named.** What happens if this decision turns out wrong? The rollback path, the cost of reversal, the trigger for re-evaluation. Decisions framed as irreversible without justification are red flags.
- **Specificity over generality.** "Spring Boot 2.7.18 → 3.5.4" beats "upgrade Spring Boot." Specificity makes the claim falsifiable.

These four are the lenses an adversarial reviewer brings. An ADR that handles all four is harder to attack and more durable as an organizational artifact.

### 3.5 The cadence — when to write each shape

A useful cadence rule for modernization weeks:

- **Scope ADRs** are written *before* the iteration starts (or in the first hour of it). They constrain what the iteration commits to.
- **Brownfield-analysis ADRs** are written *during* iteration planning, before any per-module work begins. They authorize each hop.
- **Future-hop ADRs** are written *at the end* of the iteration, capturing what was investigated but not executed. They preserve the work for later.

Out-of-order writing degrades the discipline. A brownfield-analysis ADR written *after* the hop is a post-mortem, not a contract. A scope ADR written *during* execution is a justification, not a constraint. The cadence is what makes the artifacts adversarially useful.

## 4. Generic Implementation

A worked example: a generic team modernizing a fictional `notifications-service` writes three ADRs in one week. Generic naming throughout.

**Scope ADR (written Monday morning):**

```markdown
# ADR-019: notifications-service modernization scope — Q2 W2

## Status
Accepted (2026-05-25, reviewers: senior-eng + codex Full)

## Context
notifications-service runs on Spring Boot 2.5.4 + Java 11 + AWS SDK v1.
Spring Boot 2.5 OSS support ended; AWS SDK v1 is in maintenance mode
with a v2 strongly preferred path. Two hops are required this quarter;
W2 commits to the first.

## Decision
This week we execute:
  1. Spring Boot 2.5.4 → 3.5.4 hop on notifications-service.
  2. Java 11 → 17 hop concurrent with (1).
  3. javax.* → jakarta.* package flip (driven by 1+2).
Out of scope this week: AWS SDK v1 → v2 (deferred to W4 per
ADR-022, the future-hop ADR).

## Alternatives Considered
  - Big-bang upgrade including AWS SDK v2 — rejected because the
    hand-edit volume crosses our team's risk tolerance for one
    iteration.
  - Defer all to Q3 — rejected because Spring Boot 2.5 OSS support
    has already ended; we are accumulating CVE risk.
  - Spring Boot 2.5 → 2.7 intermediate — rejected because 2.7 OSS
    support also ends in our quarter; double-hop saves one round
    of testing.

## Consequences
  + notifications-service exits two end-of-life runtimes in one week.
  - Build container image grows ~180MB.
  - Spring Cloud Sleuth references in custom tracing code require
    hand edit to Micrometer Tracing (counted in brownfield-analysis
    ADR-020).

## Rollback
  Tag legacy-baseline-pre-jakarta created at the start of the hop.
  Reverting the W2 PRs returns to the tag.
```

**Brownfield-analysis ADR (Monday afternoon — see the prior reading for the full six-section shape):** ADR-020 details the per-module hop. ~100 lines.

**Future-hop ADR (Friday end-of-day):**

```markdown
# ADR-022: Future hop — AWS SDK v1 → v2 on notifications-service

## Status
Evidence collected (2026-05-29). Not yet executed.

## Context
notifications-service uses AWS SDK v1 (com.amazonaws.*) in three
client classes: SnsClient, SqsClient, S3Client. v1 is in maintenance;
v2 is the supported path. Hop deferred to W4 per scope ADR-019.

## Evidence Collected
  - OpenRewrite recipe org.openrewrite.java.migrate.UpgradeAwsSdkV1ToV2
    available at version 3.35.0 (verified 2026-05-29).
  - mvn rewrite:dryRun output for notifications-service: 47 file edits
    across 12 files. Patch artifact at
    artifacts/W2/dryrun-aws-sdk-v2.patch.
  - One representative module (sns-publish) compiled against v2
    locally; 3 hand edits required (PublishRequest builder pattern,
    PublishResponse field access, async client lifecycle).
  - Breaking changes named: client lifecycle is now AutoCloseable,
    PublishRequest is immutable, model classes use builders.

## Status of Execution
  Scheduled for W4 per ADR-019. This ADR is paper evidence — no
  source code modified this week.
```

Notice the language: the future-hop ADR says "evidence collected" and "not yet executed." It does not commit the team to W4 execution unconditionally; it commits the team to having done the homework so that the W4 scope ADR can authorize the hop with confidence.

## 5. Real-world Patterns

**E-commerce platform — ADRs became the modernization budget tool.** A direct-to-consumer retailer's architecture council reviewed modernization-budget allocation quarterly using ADR throughput as the primary metric. Scope ADRs landed each iteration; brownfield-analysis ADRs were counted per-module hop; future-hop ADRs were tallied as planning capital. Teams that produced few ADRs but executed many changes were flagged for review (was the discipline being skipped?); teams that produced many ADRs but executed few were also flagged (was the team analysis-paralysed?). The metric was crude but useful; over two years the company tracked steady ADR throughput growth alongside steady modernization throughput growth.

**Travel-booking platform — Y-statement ADRs unblocked fast decisions.** A hotel-booking aggregator's API team adopted Y-statement ADRs for small decisions ("In the context of *date parsing across timezones*, facing *DST ambiguity*, we decided for *ISO 8601 + offset always required* and against *implicit local time*, to achieve *unambiguous booking timestamps*, accepting *slightly more verbose API contracts*"). The team produced 30+ Y-statements per quarter; senior reviewers could ratify or push back on each in under a minute. The same team reserved MADR for decisions touching three or more services, which kept the heavier template from being applied to trivial choices.

**Government services platform — scope ADRs interfaced cleanly with procurement.** A state-government IT modernization program required quarterly scope ADRs to be filed alongside procurement requests. Each scope ADR named the modules in flight, the alternatives considered, and the rollback path. The procurement office's review of scope ADRs replaced what had been a verbal sign-off process; subsequent disputes about scope reduced significantly because the artifact named what was and wasn't committed. The discipline migrated outward from engineering into the organization's decision rituals.

**Manufacturing IoT platform — adversarial review of an ADR triggered a substantial replan.** An industrial sensor company's brownfield-analysis ADR for a database hop was rejected in adversarial review because section 6 (hand edits required) said only "some Spring Data adjustments expected." The reviewer's note: *"if you cannot name them in advance, you have not yet earned the right to execute the hop."* The team came back two weeks later with section 6 enumerating four specific repository-method changes, two custom JPQL queries that needed rewriting, and one entity-graph annotation that had no v2 equivalent. The hop shipped cleanly. Without the rejection, the team would have discovered the unknowns mid-hop and rolled back.

## 6. Best Practices

- **Choose the minimum sufficient template** — Y-statement when one paragraph fits; Nygard for typical decisions; MADR when multiple options need detailed trade-off discussion; custom brownfield-analysis shape for module hops.
- **Write the scope ADR before the iteration commits** — not during, not after.
- **Write per-module brownfield-analysis ADRs before any code change** — the ADR is the authorization, not the documentation.
- **Treat ADR rejection as a feature** — an ADR rejected in adversarial review is the discipline working; a hop avoided is cheaper than a hop rolled back.
- **Keep future-hop ADRs honest** — paper evidence is fine; scope-creep commitments dressed as evidence are not.
- **Reference evidence, don't assert it** — every load-bearing claim links to a CI run, a vendor page, a dry-run patch, a benchmark.
- **Archive ADRs with the code, not in a separate wiki** — the ADR's value compounds when it lives next to the code it constrains.
- **Resist the urge to retroactively fix an ADR after the hop** — the ADR is a contract written before; after-the-fact edits weaken the artifact's evidentiary value.

## 7. Hands-on Exercise

**Exercise: Critique a deliberately weak ADR (15 min).**

Below is a poorly-written ADR. Identify at least five problems against the discipline's load-bearing criteria.

```markdown
# ADR-077: Upgrade Spring Boot

## Status
Accepted

## Context
We should upgrade Spring Boot because it's old.

## Decision
Upgrade Spring Boot.

## Consequences
It will be newer.
```

**What good looks like:** your critique names at least these problems:

1. Title is non-specific — which version to which version?
2. No date or reviewer attribution.
3. Context lacks evidence — what does "old" mean? Lifecycle status? CVE pressure?
4. Decision is not falsifiable — "upgrade" doesn't say to what version, on what timeline, for which modules.
5. No alternatives considered — the AWS anti-pattern of "made without any justification."
6. No mention of cost, risk, or rollback.
7. Consequences are tautological ("newer").
8. No linkage to scope ADR or brownfield-analysis ADR; the artifact is decoupled from execution.

After identifying the problems, rewrite the ADR using either Nygard or MADR with the same intent. The rewrite should be 30–80 lines and survive an adversarial review.

A common surprise: the rewrite reveals the original decision wasn't really made at all — when forced to name the target version, the alternatives, and the rollback plan, the team finds gaps in their own reasoning. That's the discipline working.

## 8. Key Takeaways

- *What are the three ADR shapes in a modernization week?* — Scope ADRs (iteration commitments), brownfield-analysis ADRs (per-module hop contracts), future-hop ADRs (paper evidence for deferred work).
- *How do you choose a template?* — Minimum sufficient: Y-statement for simple, Nygard for typical, MADR for multi-option, custom brownfield-analysis for module hops.
- *What three anti-patterns does the discipline target?* — Decisions not made out of fear, decisions made without justification, decisions not captured in a repository.
- *What makes an ADR adversarially reviewable?* — Evidence over assertion, alternatives explicitly considered, reversibility named, specificity over generality.
- *What is the cadence?* — Scope ADR before iteration; brownfield-analysis ADRs before per-module work; future-hop ADRs at end of iteration. Out-of-order writing degrades the discipline.

## Sources

1. [Architectural Decision Records (ADRs) — community site](https://adr.github.io/) — retrieved 2026-05-26
2. [MADR — Markdown Architectural Decision Records](https://adr.github.io/madr/) — retrieved 2026-05-26
3. [AWS Prescriptive Guidance — Using architectural decision records to streamline technical decision-making](https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/welcome.html) — retrieved 2026-05-26
4. [joelparkerhenderson/architecture-decision-record (15.9k stars)](https://github.com/joelparkerhenderson/architecture-decision-record) — retrieved 2026-05-26

Last verified: 2026-05-26
