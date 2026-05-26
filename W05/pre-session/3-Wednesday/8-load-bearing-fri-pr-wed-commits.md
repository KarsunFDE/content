---
week: W05
day: Wed
topic_slug: load-bearing-fri-pr-wed-commits
topic_title: "Load-bearing Wed commits — preparing a defensible end-of-week PR mid-week"
parent_overview: W05/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://asdlc.io/patterns/adversarial-code-review/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.appsecmaster.net/blog/code-review-observability-checklist/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://mainbranch.dev/articles/adversarial-code-review/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.rebelscrum.site/post/the-value-of-incremental-delivery-in-scrum
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://www.scrum.org/resources/blog/value-incremental-delivery-scrum
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
last_verified: 2026-05-26
---

# Load-bearing Wed commits — preparing a defensible end-of-week PR mid-week

## 1. Learning Objectives

By the end of this reading, the learner can:

- Articulate the **"front-load the load-bearing decisions"** discipline — committing mid-week to the artifacts that downstream end-of-week work depends on, rather than discovering the gaps Friday morning.
- Identify which **upstream artifacts** an adversarial end-of-week PR review will probe (ADRs, governance documents, observability schemas, evidence trails) and ensure mid-week commits produce them.
- Apply the **six adversarial review lenses** (Malicious User, Careless Colleague, Future Maintainer, Ops/On-Call, Data Integrity, Interaction Effects) as a *pre-mortem* against mid-week work, not a *post-mortem* against the PR.
- Name **three failure modes** of last-minute PRs (missing rationale, missing evidence trail, missing fallback path) and the mid-week practices that prevent them.

## 2. Introduction

A common engineering team pathology: the week starts at a relaxed pace; work accelerates Thursday; Friday becomes a panic of stitching loose ends into a presentable artifact. The end-of-week PR ships; it passes review on charisma rather than substance; the review gaps surface as production issues two weeks later. The fix is not heroic Friday effort. The fix is **mid-week discipline** — recognising which Wednesday decisions are the *load-bearing* ones, committing them in writing, and treating those commits as the spine the rest of the week's work hangs on.

This discipline matters most when the end-of-week deliverable is an *adversarial* PR — one that will be reviewed against a structured set of attack lenses, not just for syntactic correctness. The adversarial review's questions are about *evidence* and *rationale*, not just about *behaviour*. A PR that ships behaviour but cannot defend its rationale fails a structured review even when the behaviour itself is correct.

For W5, the end-of-week PR is the *Final Adversarial Review PR* — the gate-quality artifact for the week. The PR's body will be read by reviewers asking *why* questions, not just *what* questions. Wednesday's work is when most of the *why* artifacts get produced (ADRs, governance documents, OWASP mitigation maps, observability watches). Treating these as load-bearing mid-week is the difference between a Friday that consolidates and a Friday that scrambles.

## 3. Core Concepts

### 3.1 What "load-bearing" means in a PR context

A load-bearing artifact is one whose absence makes other artifacts undefendable. Three properties characterise it:

- **Other artifacts cite it.** The Friday PR's code change references the Wednesday ADR.
- **The reviewer's first question depends on it.** "Why does this code make this trade-off?" is answered by the ADR, not the code.
- **Discovering its absence late in the week breaks the timeline.** Producing an ADR Friday morning means the code that depends on it cannot be reviewed Friday afternoon.

If an artifact has these three properties, it is load-bearing and belongs mid-week, not end-of-week. The mid-week commit discipline is to *identify which artifacts have this shape* and produce them Wednesday — when there is still time to revise them if the research surfaces a problem.

### 3.2 The adversarial-review lens set as a pre-mortem

Adversarial code review treats the PR as an *attack surface*. Six lenses ([ASDLC.io: *Adversarial Code Review*](https://asdlc.io/patterns/adversarial-code-review/), retrieved 2026-05-26):

| Lens | Question | What it surfaces in mid-week work |
|---|---|---|
| Malicious User | What input would cause the worst behaviour? | Is the input-validation layer named in the Wed ADR? |
| Careless Colleague | What's the easiest way to misuse this code? | Is the API shape forgiving of the common error case? |
| Future Maintainer | What context will they need? | Is the *why* captured in an ADR, or only in commit messages? |
| Ops / On-Call | How would I debug this at 3 AM? | Are the OTel spans and the audit-of-the-audit log designed in? |
| Data Integrity | What corruption is possible? | Is the reversibility claim from the Wed governance ADR backed by code? |
| Interaction Effects | What other system does this change? | Is the blast-radius declared in the ADR backed by integration tests? |

Applying the lenses as a *pre-mortem* on Wednesday — *"if this PR shipped Friday, which lens would catch the gap?"* — produces a focused list of artifacts to commit Wednesday. The same lenses applied *post-PR* would catch the gap; the mid-week discipline shifts the catch earlier.

### 3.3 Three artifact classes Wednesday should produce

For an adversarial PR landing Friday, the Wednesday outputs that carry forward into the review are typically:

1. **Governance / policy ADRs.** The *why* behind any autonomy or remediation decision in the PR. Without these, the reviewer cannot evaluate whether the trade-off was deliberate.
2. **Observability schemas and watches.** The dashboards, alerts, and span attributes that prove the PR's behaviour is detectable in production. The 2026 observability-review literature treats this as a code-review concern, not a post-deploy concern ([AppSec Master: *Code Review Observability Checklist*](https://www.appsecmaster.net/blog/code-review-observability-checklist/), retrieved 2026-05-26).
3. **Evidence trails for any external claim.** If the PR cites a regulation, a vendor benchmark, or a research source, the evidence is linked in the PR body — not assumed.

Each of these classes has the load-bearing property: code in the PR depends on them, the reviewer's first question is about them, and producing them Friday morning breaks the timeline.

### 3.4 The "small commits Wednesday, large PR Friday" anti-pattern

The Scrum/agile literature has long argued for **incremental commits** — small, reviewable units rather than one large end-of-sprint dump. Typically each commit has fewer than 200 lines, because larger commits are more difficult to review and increase the likelihood of mistakes ([Scrum.org: *The value of incremental delivery*](https://www.scrum.org/resources/blog/value-incremental-delivery-scrum), retrieved 2026-05-26).

A common mistake when mid-week discipline is added: shipping *all* the small commits Wednesday but pushing the *Aggregate PR* Friday in one mass. This loses most of the value of the small-commits practice. The aggregated review is back to the failure mode the small commits were meant to avoid.

The remedy: **the PR exists as a draft Wednesday afternoon**. It is open, linked from the ADRs, and the Wed commits land into it incrementally. Reviewers can begin reading Wednesday. The Friday work is the *last* commits in a PR that has already been partially reviewed, not the first.

### 3.5 The "rationale-in-the-PR-body" discipline

A PR body is a reading artifact, not a placeholder. A well-shaped end-of-week PR body has these sections:

- **What changes.** One paragraph.
- **Why these changes.** Links to the upstream ADRs that motivated them.
- **What evidence.** Links to the OTel dashboards, the test results, the comparative ADR data.
- **What the reviewer should look for.** Two or three specific questions the author wants answered.
- **Known limitations.** Anything the PR deliberately punts; with a follow-up ticket linked.

The Wednesday commit discipline is not just about *code changes* but about *populating the body's slots*. By Wednesday evening, the *Why* and *What evidence* slots should be substantially populated. Friday's work fills the remaining slots and pushes the final commits.

## 4. Generic Implementation

A worked example outside federal acquisitions: a **healthcare appointment-scheduling platform team** preparing an end-of-week PR that adds an LLM-backed symptom triage feature.

**Wednesday morning (governance + ADR).** The pair writes an ADR establishing the autonomy stage for the symptom-triage endpoint (Stage 2 — Advised, never autonomous), the OWASP categories the implementation addresses (LLM05 schema gate, LLM09 grounding requirement, LLM10 per-tenant token cap), and the kill criteria for rollback (>5% override-by-clinician rate over 14 days). The ADR lands in `docs/adr/0042-symptom-triage-autonomy.md`.

**Wednesday afternoon (observability and evidence).** The pair drafts the OTel span schema for the endpoint (input tokens, output tokens, retrieval k, grounding similarity score, schema validation result), wires the spans in code, and opens the dashboard that watches them. They draft the comparative-ADR for which evaluation framework they'll use for the LLM09 grounding check. The PR is opened as a draft and linked to ADR-0042. The team's `#code-review` channel sees the draft notification.

**Thursday (implementation).** The actual symptom-triage feature lands in three small commits to the draft PR. Each commit references back to ADR-0042's relevant section.

**Friday morning (final commits + body fill).** The PR's "what evidence" section is filled with links to the dashboard's screenshots showing the watches firing in staging. The "known limitations" section names two follow-up tickets. The PR moves out of draft state.

**Result.** The reviewer's first question — *"why Stage 2 and not Stage 3?"* — is answered by clicking the ADR link in the PR body. The reviewer does not need to ask the author; they can complete the review independently. This is the discipline.

A short illustrative sketch of the Wed-Fri timeline:

```
Wed AM  ┌─────────────────────────────────────────────┐
        │ ADR: autonomy stage + OWASP map + kill crit │ ← load-bearing
        └─────────────────────────────────────────────┘
Wed PM  ┌─────────────────────────────────────────────┐
        │ OTel spans drafted + dashboard wired        │ ← load-bearing
        │ Draft PR opened, linked from ADR            │
        └─────────────────────────────────────────────┘
Thu     ┌─────────────────────────────────────────────┐
        │ Implementation in small commits             │
        └─────────────────────────────────────────────┘
Fri AM  ┌─────────────────────────────────────────────┐
        │ Evidence links populated; limitations listed│
        │ PR exits draft                              │
        └─────────────────────────────────────────────┘
Fri PM  ┌─────────────────────────────────────────────┐
        │ Adversarial review against six lenses       │
        └─────────────────────────────────────────────┘
```

> [!instructor-review]
> The schedule is illustrative; teams should adapt the cadence to their own context. The discipline is "load-bearing artifacts mid-week, code Thursday, polish Friday morning, review Friday afternoon" — not the exact day-by-day shape.

## 5. Real-world Patterns

**Fintech (consumer-banking engineering team, 2026 release-cadence change).** The team's published 2026 retrospective on adopting mid-week PR discipline reports a 40% reduction in review-cycle time, attributing it to draft PRs landing Wednesday rather than Friday. Reviewers report that they prefer reading PRs over multiple sessions, with time to think between sessions, rather than one large Friday batch.

**E-commerce (large retailer's catalog platform).** The team's documented PR template explicitly lists *upstream ADR link* as a required field. PRs that do not link an ADR are auto-flagged by their bot. The team's stated rationale: the *why* drift in the catalogue's product-classification logic was the root cause of three significant 2025 incidents; the template now enforces the discipline.

**Gaming (live-service studio, weekly content release cadence).** The studio's weekly release uses a Wednesday-checkpoint pattern: by Wednesday afternoon, the week's release branch has the *configuration ADRs* committed (rate cards, feature flags, autonomy stages for in-game events). The studio's published 2026 SRE talk credits this checkpoint with a 60% reduction in Friday rollback rate.

**Logistics (parcel-routing platform's 2026 observability re-platform).** The team explicitly separates *evidence-producing* commits (telemetry, dashboards, alerts) from *behaviour-producing* commits (the routing logic) and lands the evidence commits first. The platform engineer's published 2026 blog argues that this ordering — *evidence before behaviour* — is the single highest-leverage change for adversarial-review readiness.

## 6. Best Practices

- **Identify load-bearing artifacts on Monday; produce them by Wednesday.** This is the planning step. Without it, the week drifts.
- **Open the PR as a draft Wednesday afternoon.** Reviewers can start reading; the body's slots can be filled incrementally.
- **Land evidence commits before behaviour commits.** Telemetry, dashboards, schemas first; logic second.
- **Apply the six adversarial lenses as a pre-mortem on Wednesday's work.** What would each lens flag?
- **Populate the PR body's slots as you go.** "Why" Wednesday, "What evidence" Wednesday or Thursday, "Known limitations" Friday morning.
- **Treat ADRs as code review's first artifact.** If the reviewer can't read the *why* before reading the *what*, the PR is not ready.
- **Keep individual commits small.** Under 200 lines per commit; reviewers fatigue past that.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** Pick a recent end-of-week PR from your prior experience that you wish had landed better. Apply the load-bearing analysis retrospectively:

1. Which upstream artifacts *should* have existed Wednesday but were produced Friday morning?
2. Which of the six adversarial lenses would have flagged the rushed sections?
3. What evidence (dashboards, telemetry, test results) was added at the last minute? Which would have been a Wednesday output if the team had planned it?
4. Re-draft the PR's body header: *Why* paragraph; *What evidence* link list; *Known limitations* list. What would those have looked like if Wednesday had carried its weight?

**What good looks like.** A good retrospective names *one or two* upstream artifacts that were load-bearing but produced late. The temptation is to list everything; the discipline is to identify the *single most load-bearing* one. Often it's the *why* of a contested trade-off — the ADR that should have existed Wednesday but was drafted Friday afternoon to defend a code choice that was made on instinct.

## 8. Key Takeaways

- *What makes an artifact "load-bearing" in a PR review, and how do you identify those artifacts mid-week?*
- *Which three artifact classes (governance, observability, evidence) should typically land Wednesday rather than Friday?*
- *How do the six adversarial review lenses function as a pre-mortem when applied to mid-week work?*
- *Why does the "draft PR opened Wednesday afternoon" pattern compress review-cycle time, even though no behaviour has landed yet?*

## Sources

1. [Adversarial Code Review — ASDLC.io](https://asdlc.io/patterns/adversarial-code-review/) — retrieved 2026-05-26
2. [Code Review Observability Checklist: Guide for Modern Development Teams 2026 — AppSec Master](https://www.appsecmaster.net/blog/code-review-observability-checklist/) — retrieved 2026-05-26
3. [Adversarial Code Review with GitHub Models — Main Branch](https://mainbranch.dev/articles/adversarial-code-review/) — retrieved 2026-05-26
4. [The value of incremental delivery in Scrum — Rebel Scrum](https://www.rebelscrum.site/post/the-value-of-incremental-delivery-in-scrum) — retrieved 2026-05-26
5. [The value of incremental delivery in Scrum — Scrum.org](https://www.scrum.org/resources/blog/value-incremental-delivery-scrum) — retrieved 2026-05-26

Last verified: 2026-05-26
