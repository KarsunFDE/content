---
week: W04
day: Fri
topic_slug: amended-plan-spec
topic_title: "Amended plan-spec — how an in-place ADR amendment becomes the audit trail"
parent_overview: W04/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 13
sources:
  - url: https://adr.github.io/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://github.com/joelparkerhenderson/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/adr-process.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://github.com/github/spec-kit/blob/main/spec-driven.md
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Amended plan-spec — how an in-place ADR amendment becomes the audit trail

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish the two canonical mechanisms for evolving an Architectural Decision Record (ADR) — **amendment** (extend the existing document with a dated block) and **supersession** (a new ADR that replaces the old one) — and explain when each applies.
- Define an **amended plan-spec** as the in-place revision of Monday's ADRs after a mid-sprint event, name the four required slots of an amendment block (which decision, what changed, the amended decision, ship-or-stage), and explain why each is load-bearing.
- Articulate why the discipline of **amend rather than rewrite** preserves the audit trail in a way that deleting Monday's words does not.
- Explain why iterative spec-driven development under stress is the same discipline as the calm-condition version — the *act* of amending, not the artifact's prose, is the deliverable.
- Identify the regulated-industry contexts in which the original ADR + amendment block IS the compliance artifact for downstream auditors.

## 2. Introduction

A team that practises spec-driven dev in calm conditions develops an intuition: the spec is the durable artifact, and code is its implementation. Write the spec Monday, ship through the week, revisit the following Monday with lessons learned. This works as long as the events between Mondays are predictable.

When a mid-week event invalidates a Monday decision — a production incident, an unanticipated constraint, a discovery in adjacent work — the team faces a procedural choice that has more consequences than it appears. The instinct is often to (a) write a new "post-incident notes" document on the side, or (b) edit Monday's spec in place and pretend Monday got it right. Both are wrong. The first creates an artifact graveyard. The second destroys the audit trail — the record of what the team believed on Monday, why, and what they learned to change their mind.

The discipline the industry has converged on is the **amended plan-spec**: open the same Monday document, leave Monday's words intact, append a dated amendment block underneath each decision that needs revision. Original and amendment live together; git history shows both authors and timestamps; the reviewer three months later sees the decision *and* the revision *and* the reason.

The pattern is documented across major ADR guidance. The [ADR community canonical guidance](https://adr.github.io/) treats ADR text as not-to-be-altered but allows **amendment** as the standard evolution mechanism. [AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/adr-process.html): "Sometimes an ADR is not superseded by a new ADR, instead it is extended or amended by another ADR." [Joel Parker Henderson's ADR repository](https://github.com/joelparkerhenderson/architecture-decision-record) distinguishes amendment (additive, in-place) from supersession (new ADR + status change).

## 3. Core Concepts

### 3.1 Amendment vs. supersession — two mechanisms, two weights

The ADR community recognises two formal mechanisms:

- **Amendment.** Extend the existing ADR with a dated block underneath the original decision. Original status remains "accepted." Git history shows both authors and timestamps.
- **Supersession.** Create a new ADR. Mark the old "superseded by ADR-NNN" with a link. The new ADR contains the new decision in full.

Amendment is appropriate when the original is **substantially correct** but needs refinement (a constraint discovered after-the-fact, a threshold that changed, a new failure-mode clause). Supersession is appropriate when the original is **structurally wrong** (wrong pattern, wrong database, wrong framework). [Microsoft Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record): "If there are clarifications, then updating your existing ADR may be appropriate."

Under stress, **default to amendment**. Supersession is heavier (new file, new review, new status); amendment is lighter (in-place addition). Most mid-sprint changes are refinements, and amendment is the correct default.

### 3.2 The four-slot amendment block

A well-formed amendment block has four slots, in this order:

1. **Which decision is being amended.** Reference the specific section / sub-decision / numbered choice. "Amending the choice under §3.2 — 'Circuit-breaker library: Resilience4j' — to add a constraint."
2. **What you observed today that invalidates or refines it.** The evidence — symptom, data point, surprise. Without this slot the amendment is unmotivated and the future reader cannot reconstruct why the change happened.
3. **What the amended decision is.** The complete new statement. "Resilience4j circuit-breaker, with timeout configuration sourced from Spring Cloud Config rather than hard-coded." Not "we'll use it differently."
4. **Ship-or-stage status.** Today's deploy or staged for next sprint? Name the tracking artifact. This slot makes the amendment **actionable** — telling the next reader if the decision is live or planned.

The four-slot shape fits in five lines of markdown. The amendment is a structured patch, not a mini-essay; sprawling amendments lose their falsifiability.

### 3.3 Why "amend, do not rewrite" preserves the audit trail

The temptation under incident pressure is to **rewrite Monday's words** ("the new decision is what we actually believe, so the document should say that"). The reasoning is wrong, structurally. Rewriting destroys three things:

1. **Evidence of what Monday's team believed.** The future reader cannot tell whether the team got it right and revised on new evidence, or got it wrong and revised because the original was bad. Deleting the original collapses both into "always knew this."
2. **Evidence of *why* the change happened.** The amendment's reason slot answers "what surprised us." Without the original next to the amendment, the surprise is invisible.
3. **The chain auditors expect.** A rewritten document is no longer an audit trail; it is a fresh document. The reviewer's question "show me what changed and when" has no answer.

2026 spec-driven-dev literature ([GitHub spec-kit](https://github.com/github/spec-kit/blob/main/spec-driven.md)) names this property: specs are "versioned, created in branches, and merged" — updates are tracked changes, not destructive rewrites. Git history shows both versions; audit trail intact.

### 3.4 Iterative spec-driven dev under stress is the same discipline

A team that has practised §0 plan-retrospective four times in calm conditions (W2 Mon, W3 Mon, W4 Mon, plus the Tuesday workshop) has internalised the cadence of amending the spec with what was learned. The mid-sprint event is the **fifth iteration** — first done under stress.

The discipline does not change. Same four slots, same reason-evidence shape, same ship-or-stage criteria. Four prior practice runs provide the muscle memory for the fifth. This is the same insight as fire-drill training, simulator training in aviation, and tabletop exercises in incident response: performance under stress is not improvised, it is recalled from prior practice.

### 3.5 The amended plan-spec as compliance artifact

In regulated industries, the amended plan-spec is the compliance artifact. The reviewer's question is not "did you predict the event?" (no team can be expected to). The question is "when reality diverged from your plan, did you document the divergence in a way I can read?"

The amended plan-spec answers exactly that. Original on the page, amendment block on the page, dates visible, reason named, ship-or-stage explicit. The auditor three months later can reconstruct the team's mental state at both decision moments and verify the team followed its own documented change-control process. Rewriting Monday's words destroys this — the document becomes indistinguishable from a forecast that happened to be correct, which is not what happened.

## 4. Generic Implementation

A canonical amended-ADR template, framework-agnostic, looks like the following. The example uses generic naming and applies regardless of the underlying technology stack.

```markdown
# ADR-NNN: {original decision title}

**Status:** accepted (with amendment YYYY-MM-DD — see below)
**Date:** YYYY-MM-DD (original)
**Authors:** {names of original deciders}

## Context
{The original problem framing — left unchanged.}

## Decision
{The original decision — left unchanged.}

## Consequences
{The original consequences analysis — left unchanged.}

---

## Amendment (YYYY-MM-DD)

**Status:** amendment to ADR-NNN — accepted
**Amendment authors:** {names of authors of the amendment}

### 1. Which decision is being amended
{Reference the specific section / sub-decision / numbered choice above.
e.g., "The choice under 'Decision' to use Library X is being amended to add
a configuration constraint."}

### 2. What we observed that invalidates or refines the original
{The evidence: the symptom, the data point, the constraint discovered. Cite
the impact assessment or the incident ID if applicable. One paragraph; no
narrative essay.}

### 3. The amended decision
{The complete new statement of the decision, including the original substance
plus the new constraint. The reader should be able to follow ONLY the amended
decision if they read top-to-bottom skipping the original — but the original
is still there for the audit trail.}

### 4. Ship or stage
{One sentence: "Ships today in {deploy id}" OR "Staged for {sprint id},
tracked as {ticket id}, owner {name}, amendment to take effect at deploy
of that work."}
```

Deliberate properties: the amendment **appends** (no original line replaced), has its own status/date/authors (a separate decision moment), uses numbered slots to enforce order (skipped pieces are visible), and links slot-4 directly to the downstream tracking artifact.

> [!instructor-review]
> **Multiple amendments to one ADR.** Real ADRs can accumulate amendments over time. The convention is to append each underneath the previous with its own date and status, never inter-leaving slots. If a third amendment refines the first (not the original), its slot 1 should reference the first amendment explicitly. Some teams promote to supersession after 3+ amendments — confirm cohort's working convention.

## 5. Real-world Patterns

**Fintech — payment-gateway timeout ADR.** A consumer-fintech team had an ADR specifying "30-second timeout on the payment-gateway client." After a production incident where the timeout interacted badly with a gateway upgrade, the team amended with the four-slot block: which decision (the 30-second timeout), what they observed (the gateway upgrade reduced p99 from 12s to 800ms, making the 30s timeout unnecessarily slow on failure paths), the amended decision (3-second timeout with a documented escalation path for known-slow operations), and ship-or-stage (staged for next sprint with a feature-flagged rollout). The original decision stayed on the page; the amendment contained the new one. *Three months later, when a compliance audit asked why the team had reduced the timeout, the amended ADR was the answer — audit trail complete, change justified, no separate "post-incident notes" needed.*

**Logistics — route-optimisation ADR.** A delivery platform had an ADR specifying "route optimisation runs every 5 minutes." A traffic spike during a holiday weekend showed the 5-minute interval was inadequate. The team's first instinct was to **rewrite the ADR** to say "1-minute interval" — but the engineering lead caught the move and required an amendment block instead. The amendment named the holiday-traffic-spike observation, the new 1-minute interval, and a staged sprint item to make the interval dynamic. *When the same situation recurred six months later, the new on-call team read the amendment, saw the prior reasoning, and immediately implemented the staged dynamic-interval work — audit trail as coaching artifact, not just compliance one.*

Two additional cases — healthcare clinical-rule engine fail-open/fail-closed amendment, and e-commerce search-relevance BM25 amendment — appear in the [further-reading appendix](./5-amended-plan-spec-further-reading.md).

## 6. Best Practices

- **Default to amendment under stress; reserve supersession for structural rewrites.** Amendment is lighter and right for refinements; supersession is heavier and right when architecture changes.
- **Never rewrite the original decision; always append.** The audit trail is the document's structural integrity, not its current prose.
- **Number the four slots explicitly.** Out-of-order amendment blocks tend to skip slots; numbering makes missing pieces visible.
- **Cite the impact assessment in the slot-2 reason.** The chain symptom → hypothesis → amendment must be reconstructable.
- **Tag the slot-4 ship-or-stage status to a concrete artifact (deploy id, sprint id, ticket id).**
- **Treat each amendment as its own decision moment** with its own status and named authors.
- **Re-review the parent ADR every 2–3 amendments and consider supersession.** Accumulated amendments often signal that the original needs a structural rethink.

## 7. Hands-on Exercise

**Task (paper or markdown sketch, 10–15 min):** Pick an industry (NOT federal acquisitions). Existing ADR from two weeks ago:

> **ADR-014: Cache TTL for product-catalogue lookups**
> **Decision:** Cache product-catalogue lookups in Redis with a 60-second TTL. On cache miss, the read hits the database and writes to the cache with TTL=60s.

This morning's incident: during a flash sale, the 60-second TTL caused stale price information to persist after a vendor's mid-sale price update, leading to ~7 minutes of incorrect pricing on the homepage.

Draft the four-slot amendment block for ADR-014. Choose ship-or-stage realistically.

1. **Slot 1.** Which element of ADR-014's decision is being amended?
2. **Slot 2.** What did you observe that invalidates or refines it? Cite a hypothetical impact-assessment by ID.
3. **Slot 3.** State the complete amended decision with the new constraint integrated.
4. **Slot 4.** Ship today or stage? If stage, name the tracking artifact and owner.

Then: would you choose **amendment** or **supersession**, and why?

**What good looks like.** A correct sketch names the specific clause being amended (not "the whole ADR"), cites a specific symptom from a referenced impact assessment, states a complete amended decision (e.g., "60-second TTL with explicit cache-bust hook on vendor-initiated price updates, debounced at 5 seconds"), and chooses a defensible ship-or-stage status. Amend-vs-supersede should usually be **amendment** for this scenario — the cache mechanism is correct, only the staleness invariant changed. Common mistakes: free-form paragraph instead of four numbered slots; defaulting to supersession because "the original was wrong" (most refinements are right-for-original-conditions but in need of update).

## 8. Key Takeaways

- **The two canonical mechanisms.** Amendment (in-place dated block, original remains accepted) for refinements; supersession (new ADR with old marked superseded) for structural rewrites. Default to amendment under stress.
- **The four slots, in order.** (1) which decision is being amended, (2) what you observed that invalidates or refines it, (3) the amended decision in full, (4) ship-or-stage with tracking artifact.
- **Why "amend, do not rewrite" is load-bearing.** Rewriting destroys the evidence of what the team originally believed, why it changed, and the structural integrity that makes the document an audit trail.
- **Why iterative spec-driven dev under stress is the same discipline.** The mid-sprint amendment is the fifth iteration of a cadence practised four times in calm; the team executes from muscle memory.
- **Why the amended plan-spec is the compliance artifact.** The auditor's question is "when reality diverged from plan, did you document the divergence in a way I can read?" — and amendment-in-place answers exactly that.

## Sources

1. [Architectural Decision Records — community canonical site](https://adr.github.io/) — retrieved 2026-05-26
2. [Joel Parker Henderson — architecture-decision-record GitHub](https://github.com/joelparkerhenderson/architecture-decision-record) — retrieved 2026-05-26
3. [AWS Prescriptive Guidance — ADR Process](https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/adr-process.html) — retrieved 2026-05-26
4. [Microsoft Azure Well-Architected Framework — Architecture Decision Record](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record) — retrieved 2026-05-26
5. [GitHub spec-kit — Spec-Driven Development](https://github.com/github/spec-kit/blob/main/spec-driven.md) — retrieved 2026-05-26

Last verified: 2026-05-26
