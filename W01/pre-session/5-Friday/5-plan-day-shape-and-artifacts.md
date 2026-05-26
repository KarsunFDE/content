---
week: W01
day: Fri
topic_slug: plan-day-shape-and-artifacts
topic_title: "Plan Day shape + six required planning artifacts"
parent_overview: W01/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://adr.github.io/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/articles/scaling-architecture-conversationally.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://github.com/architecture-decision-record/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Plan Day shape + six required planning artifacts

> The overview frames this against the cohort's first Plan Day on W2 Mon and the specific Scenario Design Planning artifact the pair will author. This reading is the generic case: why "no code lands on Plan Day" is a discipline (not a ceremony), why ADRs are the load-bearing artifact, and how the six artifacts compose into a defensible plan-spec. The §0 plan retrospective pattern is *not yet* introduced — that formally starts W3 Mon. W2 Mon has no prior plan-spec to retro against; do not waste 30 minutes looking for one.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Articulate why "Plan Day" is a *forcing function* for surfacing assumptions before code locks them in.
- Name the six artifacts a Plan Day should produce and what each one is *for* (the consumer, not the format).
- Write a single-page ADR in any of the three common templates (Y-statement, MADR, Nygard) and explain why the format matters less than the discipline.
- Identify the failure modes that signal a Plan Day was performative ceremony rather than load-bearing planning.

## 2. Introduction

"Plan Day" is the term this programme uses for a planning ritual that has been independently invented by every long-lived engineering org and given dozens of names — "spike day," "design day," "RFC day," "ADR session," "kickoff." The shape is consistent across the industry: at the start of a significant work increment (a new system, a major feature, a quarter), the team blocks a day to think out loud before writing code. The output is a small set of artifacts that capture *what the team decided and why* — not as documentation produced for an audit, but as forcing functions that surface assumptions before code embeds them.

The cost of skipping Plan Day is well-documented. Andrew Harmel-Law's 2021 essay on scaling architecture conversationally describes the failure mode: decisions get taken implicitly inside pull requests, with no record of the alternatives considered, no consequences captured, and no path back when those decisions turn out to be wrong [Fowler/Harmel-Law, 2021]. Six months later someone asks "why did we pick Postgres over DynamoDB here?" and no one can answer. The technical debt isn't the choice — it's the lack of context around the choice.

Plan Day exists to invert that pattern. The team produces the context *before* the code. The artifacts are short — single-page ADRs, not 40-page design documents — but they are *committed*. The next person can read them and understand why the system looks the way it does.

## 3. Core Concepts

### 3.1 The forcing function

The reason Plan Day works is not that the artifacts are inherently valuable (most ADRs are read maybe twice in their lifetime). The reason is that *writing them* forces the team to confront questions they otherwise route around. "Which embedding model?" feels like a free choice until you sit down to write the ADR — and then you discover you don't know what your retrieval evaluation looks like, you don't know the cost-per-query, and you don't know whether the choice locks you into a single LLM vendor. The ADR makes the unknowns visible.

The discipline is "no code lands on Plan Day." If the team is allowed to start coding while still planning, planning becomes a sidebar to coding, and the assumptions ride into the codebase un-surfaced. Holding the boundary — Monday is planning, Tuesday is coding — forces the planning to be complete enough to start from.

### 3.2 The six artifacts

The exact list varies by programme; the common shape is:

**(1) Requirements synthesis.** One or two pages restating the problem in the team's own language. The consumer is the future team member reading this in a month. The signal that it's good is: someone outside the team could read it and tell you what the system is for, without asking follow-up questions.

**(2) System map / 6R diagram.** A single diagram of the system's components and how they connect. The "6R" in some teams is rehost / replatform / refactor / re-architect / rebuild / retire — a framework for naming the disposition of *existing* components when modernizing brownfield code. For greenfield, the map is just "what calls what."

**(3) ADRs — typically two or three.** One per architecturally-significant decision. The format is short (one page), the structure is roughly *context / decision / consequences*, and the focus is on the *alternatives considered* and the *rationale for picking one* — not on the implementation detail. ADRs are the load-bearing artifact of Plan Day; the others support them [adr.github.io, 2026].

**(4) Estimate.** Effort (person-days) and cost (compute, licenses, third-party services). The estimate is wrong on first authoring — that's expected. What matters is that you wrote one down, so you can compare the actual against it and learn for next time.

**(5) Risk register.** What could go wrong, how likely, how bad, and what the team will do about it. The risk register is where you put the things that don't fit in an ADR — things you don't have to *decide* about but should *watch*.

**(6) Open questions.** The list of things you do not know yet. The discipline is that this list exists and gets revisited — not that it is empty.

### 3.3 ADR formats (and why they don't matter much)

Three common templates [adr.github.io, 2026; joelparkerhenderson, 2026]:

**Nygard's original (2011).**

```markdown
# Title

## Status
Proposed | Accepted | Deprecated | Superseded by ADR-NNN

## Context
What is the issue we are seeing that is motivating this decision?

## Decision
What is the change we are actually proposing or doing?

## Consequences
What becomes easier or harder as a result?
```

**MADR (Markdown Any Decision Records).** Adds explicit "Considered Options" and "Pros and Cons" sections — slightly more rigorous, mildly more ceremony.

**Y-statement (Zdun et al.).** A single sentence:

> In the context of *X*, facing *Y*, we decided for *Z* and against *A*, to achieve *B*, accepting *C*.

The Y-statement is striking because it forces every part of an ADR into a single sentence — context, problem, decision, alternatives, rationale, consequences. Sometimes that's the right shape; sometimes you need the longer form. The format matters less than the discipline of capturing the alternatives and the rationale.

The Microsoft Azure Well-Architected Framework explicitly recommends ADRs as part of the architect role and links to the Nygard format as a default [Microsoft Azure, 2024].

### 3.4 The "advice process" — for teams beyond two

Andrew Harmel-Law's contribution is naming the social pattern around ADRs. In a small team, the team writes the ADR together and that's enough. In a larger org, *anyone* can take an architectural decision provided they have:

1. **Asked for advice** from everyone meaningfully affected and from anyone with relevant expertise.
2. **Captured the advice** they received, including the dissents.
3. **Made the decision** themselves and recorded it as an ADR.

The advice process scales decision-making to the people closest to the work without devolving into design-by-committee. The ADR is the durable record of "I asked, I heard, here is what I decided and why" [Fowler/Harmel-Law, 2021].

### 3.5 What signals a performative Plan Day

Failure modes to watch for:

- **Code commits with timestamps inside the Plan Day window.** The discipline broke.
- **ADRs with no "Considered Alternatives" section, or with one alternative ("we considered X, we picked Y because Y is better").** The alternative was not actually considered.
- **Risk register with three risks: "scope creep," "people leaving the team," "third-party dependencies."** Nothing system-specific. The team did not actually think about what could go wrong.
- **Open Questions list is empty.** This is almost never true; an empty list signals that the team did not surface the things they don't know.
- **Estimate without a unit.** "About a sprint" is not an estimate; "12 person-days, ±30%" is.

## 4. Generic Implementation

A worked ADR outside federal acquisitions — an e-commerce team deciding their session store.

```markdown
# ADR 0007: Use Redis for session storage

## Status
Accepted (2026-04-15)

## Context
Our session data — cart contents, user preferences, in-flight checkout state — is currently
held in the Postgres database that also holds order and product data. Read latency on
session lookups is dominating the API p99, and cart-abandonment traces show users hitting
slow first-page renders during peak hours. Sessions are read 20× more often than they are
written; they expire in 24 hours and are not queried for analytics.

## Considered Alternatives
1. **Stay on Postgres** with a covering index — lowest operational change, no progress on
   the read-latency problem at peak scale.
2. **DynamoDB with TTL** — managed, scales horizontally, but pulls our deploy region's
   data-residency surface into AWS-managed-state territory we currently avoid.
3. **Redis (managed)** — purpose-built for the access pattern, sub-millisecond reads,
   TTL native, fits our existing managed-services posture.
4. **In-process cache** — fastest possible reads, but breaks our blue/green deploy story
   and complicates session affinity at the load balancer.

## Decision
Adopt managed Redis as the session store. Postgres remains the source of truth for any
session data we want to persist past 24 hours (settlement, fraud review).

## Consequences
Easier: API p99 drops by an estimated 40 ms based on a load-test spike. Session-TTL
expiry is automatic. Failover is well-trodden — managed Redis has been in our stack since
Q1 for caching.

Harder: One more service to monitor. Operational runbook needs a Redis failover section.
Engineers must remember that anything in Redis is ephemeral — we will add a lint rule
that flags writes to Redis without a TTL.

Open: We have not benchmarked the cost at peak season's projected QPS. Tracked as risk
R-12 on the program risk register.
```

This is the shape. A page, sometimes a little more or less. Context names the problem; alternatives are *real* (not strawmen); the decision is explicit; consequences cover both wins and costs; the open question is captured.

## 5. Real-world Patterns

**Spotify's "RFC" process.** Spotify uses an RFC (Request for Comments) document that fills a similar role to an ADR — proposed in writing, circulated to affected teams, comments collected, then decided. The format is heavier than a Nygard ADR but the discipline is the same: capture the decision, the alternatives, and the rationale before code lands [Fowler/Harmel-Law, 2021 — referenced pattern].

**Thoughtworks Tech Radar.** Beyond per-project ADRs, Thoughtworks maintains a portfolio-level "Tech Radar" that names which technologies they are adopting, trialing, assessing, or holding back from. The radar is the org-level companion to per-team ADRs and prevents teams from independently picking the same wrong-for-the-org tool [Fowler/Harmel-Law, 2021].

**Healthcare — Epic / Cerner customizations.** Hospital IT teams modernizing on top of large EHR platforms produce ADRs around customizations (e.g., "Use FHIR R4 for inter-org data exchange, defer R5 until vendor support stabilizes"). The ADRs are the audit-survivable record of why each integration looks the way it does.

**Open source — Rust language RFCs, Python PEPs.** Both Rust and Python use a public, version-controlled RFC/PEP process for language-level decisions. The shape is much heavier than a team ADR (community comment window, formal review board), but the underlying pattern is identical: capture the decision, the alternatives, and the rationale in writing before changing the code.

## 6. Best Practices

- **Write ADRs short and write them now.** A one-page ADR written during the decision is worth a 20-page design document written six months later from memory.
- **Always include the alternatives you considered**, and consider real ones — not strawmen you knew you would reject.
- **State consequences honestly, including the costs.** ADRs that read like sales pitches lose the value of the artifact; the next team can't see the trade-offs.
- **Number ADRs and never delete them.** When a decision is reversed, mark the old one "Superseded by ADR-NNN" and write a new one. The historical chain is part of the value.
- **Hold the "no code on Plan Day" boundary**. Once that boundary erodes, planning collapses into the coding loop and the artifacts stop earning their keep.
- **Treat the Open Questions list as a real backlog item**, not a documentation bullet. Each open question should have an owner and a target date for resolution.

## 7. Hands-on Exercise

**Whiteboarding exercise (15 minutes).** Pick one significant decision from a past project — language choice, framework choice, data store choice, deploy target choice. Write a one-page ADR in the Nygard format:

- Title (numbered).
- Status (Accepted, presumably, since you actually did this).
- Context (1 paragraph — what was the problem?).
- Considered Alternatives (at least three real options, with one-line summaries of each).
- Decision (1 paragraph — what you picked, stated plainly).
- Consequences (2 paragraphs — what got easier, what got harder, what's still open).

**What good looks like.** The ADR is short enough to read in three minutes. The alternatives include at least one option you almost picked instead — not a parade of obviously-wrong options. The consequences section is balanced — both the wins and the new operational costs are named. If the reader can't tell from the ADR which alternative you almost picked instead, the alternatives section is not doing its job.

## 8. Key Takeaways

- Can I name the six Plan Day artifacts and what each one is *for* (its consumer)?
- Do I understand why "no code on Plan Day" is a forcing function, not ceremony?
- Can I write a single-page ADR in Nygard or Y-statement format from memory?
- Do I know the difference between an ADR with real alternatives and one with strawmen?
- Can I recognize the failure modes of a performative Plan Day (empty Open Questions, generic risk register, code commits in the planning window)?

## Sources

1. [Architectural Decision Records (ADRs) — adr.github.io](https://adr.github.io/) — retrieved 2026-05-26 via /web-research. Authoritative landing page for the ADR community, with links to template variants (Nygard, MADR, Y-statement) and the Zdun et al. "Sustainable Architectural Decisions" paper.
2. [Scaling the Practice of Architecture, Conversationally — Andrew Harmel-Law, martinfowler.com](https://martinfowler.com/articles/scaling-architecture-conversationally.html) — retrieved 2026-05-26 via /web-research. Source for the advice-process framing, the four supporting mechanisms (Decision Records, Advisory Forum, Team-sourced Principles, Tech Radar), and the failure modes of un-recorded decisions.
3. [Architecture decision record (ADR) examples — joelparkerhenderson on GitHub](https://github.com/architecture-decision-record/architecture-decision-record) — retrieved 2026-05-26 via /web-research. Reference collection of ADR templates and worked examples (15.9k stars; widely-cited community resource).
4. [Architecture decision record — Microsoft Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record) — retrieved 2026-05-26 via /web-research (referenced via adr.github.io). Enterprise-cloud-vendor recommendation of ADR practice as part of the architect role.

Last verified: 2026-05-26
