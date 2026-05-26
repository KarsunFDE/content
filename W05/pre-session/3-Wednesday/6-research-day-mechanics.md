---
week: W05
day: Wed
topic_slug: research-day-mechanics
topic_title: "Research Day mechanics — running a comparative-evaluation slot"
parent_overview: W05/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://www.itonics-innovation.com/blog/technology-evaluation
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://libguides.law.ucdavis.edu/c.php?g=1035998&p=10361056
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://libguides.lib.vt.edu/BCHM1014/EvaluatingInfoSources
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://www.yomu.ai/blog/evaluating-source-credibility-guidelines-for-identifying-reliable-research-materials
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://arxiv.org/html/2509.01306v1
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Research Day mechanics — running a comparative-evaluation slot

## 1. Learning Objectives

By the end of this reading, the learner can:

- Describe a **research-day discipline** that produces a defensible comparative ADR in a single afternoon — scoping the question, splitting the work, converging on a recommendation.
- Apply the **CRAAP / authority-accuracy-currency-relevance-objectivity** heuristic to filter sources during fast-paced research, and identify three failure modes the heuristic catches.
- Build a **comparison matrix** that holds 2–4 candidate technologies against 4–7 evaluation dimensions and articulate why dimension choice (not score) carries most of the ADR's weight.
- Recognise common **research-day failure modes** (vendor-marketing capture, confirmation bias, recency over-weighting, false-consensus) and name a mitigation for each.

## 2. Introduction

A "research day" is a block of time — usually half a day to a day — when an engineering team or pair pauses delivery work to evaluate a contested technology or design decision. The deliverable is a defensible comparative artifact: an ADR, a tech-evaluation matrix, or a short white paper. The constraint is time: by end-of-day, the team needs something defensible enough to act on Monday.

Research days fail in characteristic ways. The most common failure is **vendor-marketing capture**: the pair lands on a vendor's product page, the page is impressive, the pair carries the vendor's framing into the ADR, and the comparison ends up describing the chosen vendor against weakened versions of the alternatives. The second most common failure is **confirmation bias**: the pair starts with an instinct, finds three sources that support it, declares the question answered, and never seeks the disconfirming case. The third is **recency over-weighting**: a blog post from last week feels more authoritative than the practitioner literature from six months ago, even when the older source has more depth.

This reading is a short discipline against those failure modes. It is not about *what to research* (the topic does that); it's about *how to run the half-day well* so the artifact you produce is one a sceptical reviewer would respect on Monday morning.

## 3. Core Concepts

### 3.1 Scope the question before opening a browser

The single most common research-day mistake is starting with the search bar. A useful five-minute pre-step:

1. **Write the question as a one-sentence comparison.** "Compare A vs B vs C on dimensions D1–D4 for context X." If you can't write that sentence, the question isn't scoped yet.
2. **Name the deliverable shape.** ADR? Matrix? Memo? Knowing the shape changes how you read sources.
3. **Name the audience.** A platform team's ADR has different rigour than an internal Slack post.
4. **Name the *out-of-scope* list explicitly.** What questions are you deliberately not answering today? Putting them in writing prevents drift mid-afternoon.

The pair commits this scope before either person opens a tab. Five minutes of friction here saves an hour of drift later.

### 3.2 Source filtering — the CRAAP / authority heuristic

The library-research literature has converged on a small set of source-credibility heuristics. The most cited is **CRAAP** (Currency, Relevance, Authority, Accuracy, Purpose):

- **Currency.** Is the source current enough for the question's recency window? For hot-tech topics, three months is the practical horizon; for foundation-stable, twelve months.
- **Relevance.** Does the source address the specific question, or only a related one?
- **Authority.** Who wrote it? Is the author identifiable? Is the publishing institution reputable?
- **Accuracy.** Is the claim supported by evidence the source itself shows? Can the claim be cross-checked against another source?
- **Purpose.** Why does this source exist? Is it a vendor marketing page, an independent practitioner blog, an academic paper, or a community standard?

In a research day, you don't apply CRAAP fully to every source — you skim it as a first-pass filter and skip sources that fail two or more dimensions. A vendor product page fails Purpose by definition; that doesn't disqualify it, but it does demote it relative to an independent practitioner's comparison ([UC Davis Law Library: *Evaluating Sources*](https://libguides.law.ucdavis.edu/c.php?g=1035998&p=10361056), retrieved 2026-05-26).

### 3.3 The comparison-matrix shape

A useful comparison matrix has 2–4 columns (candidates) and 4–7 rows (dimensions). The discipline is in the *dimension choice*, not the scoring:

- **Dimensions should be picked before reading any vendor's marketing.** If you let a vendor's framing pick your dimensions, you have already lost.
- **Each dimension should be falsifiable.** "User experience" is not a dimension; "supports SSO via SAML 2.0" is.
- **Each dimension should map to a real cost or risk.** A dimension that does not change the recommendation if it changes does not belong in the matrix.

A typical shape (illustrative, vendor-neutral):

| Dimension | Candidate A | Candidate B | Candidate C |
|---|---|---|---|
| Production maturity (GA date, customers) | | | |
| Total cost of ownership at our scale | | | |
| Vendor-portability of policy/config | | | |
| Integration with our current stack | | | |
| Security/compliance posture | | | |
| Operational burden (24×7 on-call expectations) | | | |
| Exit cost if we leave in 18 months | | | |

The matrix is not the deliverable — the *recommendation paragraph that explains which dimensions dominated* is the deliverable. A reader who only reads the recommendation paragraph should understand the *shape* of the decision; the matrix is the backing evidence.

### 3.4 Failure modes and mitigations

| Failure mode | What it looks like | Mitigation |
|---|---|---|
| Vendor-marketing capture | Recommendation uses the chosen vendor's framing in every dimension; alternatives appear as weakened straw versions | Pick dimensions before reading any vendor page; read independent practitioner sources first |
| Confirmation bias | Pair finds three sources supporting their pre-existing instinct; never seeks disconfirming case | Pair member assigned the *opposite* position for 30 minutes; report back |
| Recency over-weighting | Last week's blog post carries more weight than six-month-old practitioner literature | Apply the recency window appropriate to the topic; older does not mean wrong |
| False consensus | Three blog posts all cite the same primary source; the appearance of three independent sources is illusory | Trace cited sources to their origin; count unique primary sources, not unique URLs |
| Single-domain anchoring | Sources come exclusively from one industry vertical; conclusions don't generalise | Deliberately seek out one source from a different domain |

Each of these has produced a bad ADR somewhere. The team-level discipline is to name them out loud at the start of the day and watch for them at the end.

### 3.5 The Sub-30-Day source decision

Some research days surface a source dated within the last 30 days — a vendor announcement, a brand-new release, a recently published paper. Sub-30-day sources have a problem: there has not been time for the community to disagree with them. The fix is not to exclude them but to handle them deliberately:

- **Adopt.** The source is so authoritative and clear that you're willing to cite it knowing the disagreement hasn't surfaced yet.
- **Defer.** The source is interesting but not yet validated; flag it in a "watch list" and revisit in 30 days.
- **Omit.** The source is too speculative for an ADR; skip.

Picking one of these three deliberately is the discipline. Silently citing a sub-30-day source is not.

## 4. Generic Implementation

A worked example outside federal acquisitions: a **healthcare scheduling-platform engineering team** running a half-day research session to pick between three event-streaming platforms (Kafka, Pulsar, Redpanda) for the patient-notification pipeline.

**Pre-step (10 min).** The pair writes: *"Compare Kafka, Pulsar, Redpanda on durability, operational burden at our scale (50M events/day), exit cost, and on-call expertise availability for our scheduling pipeline by 2026-Q4."* They name an ADR as the deliverable, the platform-engineering manager as the audience, and explicitly mark *latency optimisation* as out-of-scope for today.

**Source pass (90 min).** Each person picks two candidates and reads three independent practitioner sources and one vendor-page source per candidate. They apply CRAAP as a skim filter — sources fail Currency if they predate 2025; sources fail Purpose if they are vendor-marketing pages with no engineering detail. They take notes against the four dimensions they pre-committed.

**Disconfirming-position pass (30 min).** Each person argues *against* their leading candidate for fifteen minutes. The other person plays devil's advocate.

**Matrix and recommendation (60 min).** They build the matrix, write a recommendation paragraph that names the one or two dimensions that drove the decision, and explicitly call out which dimensions were a tie. They list one open question they could not resolve today and propose a follow-up step.

**Total: 3.5 hours.** The artifact lands in the team's docs repo by EOD.

> [!instructor-review]
> The example is illustrative and avoids advocating any specific event-streaming platform. The point is the process shape: pre-scope, source pass, disconfirming pass, matrix, recommendation. Substitute the cohort's actual topic and dimensions.

## 5. Real-world Patterns

**Fintech (mid-sized payments processor, 2025 vendor evaluation).** The team's published 2025 retrospective on a vendor selection describes the discipline of *picking dimensions before reading vendors*. Their stated lesson: "Two of the four dimensions in our final matrix did not appear in any of the three vendors' marketing — we had to ask them directly. That asymmetry was the most informative part of the evaluation."

**Logistics (parcel-tracking platform, monorepo-vs-polyrepo decision).** The team's public engineering blog frames their decision as a comparison matrix with five dimensions, of which only one (build-tooling support) appeared on either approach's marketing pages. The blog explicitly credits the discipline of picking dimensions first for avoiding the vendor-framing trap.

**E-commerce (online retailer, search-engine selection).** The team ran a one-day research session comparing three search platforms. Their published artifact notes that the pair assigned a *disconfirming reviewer* — a third person who read the matrix and the recommendation and was tasked specifically with finding the weakest dimension argument. That role caught two over-reaches in the recommendation before it was published.

**Gaming (live-service studio, observability stack selection).** The studio's 2026 talk on their observability re-platform names *exit cost* as the dominant dimension in their decision. None of the three candidates' marketing mentioned exit cost; the team had to compute it for each candidate by talking to teams that had migrated off. The talk's stated lesson: "If we had let the vendors' dimensions drive our matrix, we'd have picked the one we're most locked into."

## 6. Best Practices

- **Pick dimensions before reading vendor pages.** This is the single highest-leverage discipline.
- **Apply CRAAP as a skim filter, not a deep evaluation.** A two-minute pass per source is enough to demote a vendor-marketing page.
- **Assign someone the disconfirming position.** A 30-minute steel-manning of the opposite case catches confirmation bias.
- **Trace citations to unique primary sources.** Three blog posts citing the same paper is one source, not three.
- **Handle sub-30-day sources deliberately.** Adopt, defer, or omit — never silent inclusion.
- **Make the recommendation paragraph load-bearing, not the matrix.** A reader who only reads the paragraph should understand the shape of the decision.
- **Name the open question.** Every research-day artifact lists at least one question it could not resolve and proposes a follow-up.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** Imagine you have a research-day slot tomorrow afternoon. Pick *any* contested technology decision from your prior experience that you wish you'd researched better.

Write out:

1. The one-sentence comparison question.
2. The four dimensions you would commit to *before* reading any vendor page.
3. Two of the dimensions you suspect *no vendor's marketing would cover* — the asymmetry that makes the dimension valuable.
4. The disconfirming-reviewer assignment: who would steel-man the position you instinctively reject?
5. One open question you would explicitly call out as out-of-scope for today.

**What good looks like.** The two-dimensions-no-vendor-covers row is the diagnostic for whether you've internalised the discipline. If your dimensions are all things every vendor markets ("performance, ease of use, scalability"), the matrix will be vendor-captured before you start. If two dimensions are things like "exit cost in 18 months" or "on-call expertise available in our timezone", you're picking dimensions that *discriminate* between candidates rather than just describing them.

## 8. Key Takeaways

- *What is the discipline that runs a research day to a defensible artifact in a single afternoon?*
- *Which dimensions of CRAAP are most useful as a skim filter, and which require deeper analysis?*
- *Why is dimension choice more load-bearing than scoring in a comparison matrix?*
- *Which four failure modes ruin research days, and what is one mitigation for each?*

## Sources

1. [Technology Evaluation: A Guide for R&D and Innovation Teams — ITONICS](https://www.itonics-innovation.com/blog/technology-evaluation) — retrieved 2026-05-26
2. [Evaluating sources — UC Davis Law Library Research Guides](https://libguides.law.ucdavis.edu/c.php?g=1035998&p=10361056) — retrieved 2026-05-26
3. [Evaluating Sources — Criteria — Virginia Tech University Libraries](https://guides.lib.vt.edu/BCHM1014/EvaluatingInfoSources) — retrieved 2026-05-26
4. [Evaluating Source Credibility: Guidelines for Identifying Reliable Research Materials — Yomu AI](https://www.yomu.ai/blog/evaluating-source-credibility-guidelines-for-identifying-reliable-research-materials) — retrieved 2026-05-26
5. [Re3: Learning to Balance Relevance & Recency for Temporal Information Retrieval — arXiv 2509.01306](https://arxiv.org/html/2509.01306v1) — retrieved 2026-05-26

Last verified: 2026-05-26
