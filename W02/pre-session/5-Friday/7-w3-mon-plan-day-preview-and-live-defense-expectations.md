---
week: W02
day: Fri
topic_slug: w3-mon-plan-day-preview-and-live-defense-expectations
topic_title: "W3 Mon Plan Day preview + Live Defense expectations"
parent_overview: W02/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://intent-driven.dev/blog/2026/04/29/spec-driven-development-with-adr/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://thebcms.com/blog/spec-driven-development
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://asana.com/resources/sprint-retrospective
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://juicebox.ai/blog/rubrics-for-interviews
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://dev.to/finalroundai/i-reviewed-final-round-ai-for-technical-interviews-heres-what-actually-matters-in-2026-47gd
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# W3 Mon Plan Day preview + Live Defense expectations

## 1. Learning Objectives

- By the end of this reading, the learner can describe what a "§0 plan retrospective on a multi-day plan-spec" is and why it differs from a generic sprint retrospective.
- By the end of this reading, the learner can list the categories a plan retrospective covers (what shipped, what dropped, what surprised, what to carry forward) and explain why "what got dropped" is the most-skipped category.
- By the end of this reading, the learner can articulate the structure of an oral technical defense and name the dimensions a defense rubric weights.
- By the end of this reading, the learner can describe rubric design that mitigates bias and what "citations as load-bearing, not decorative" means in a defense context.

## 2. Introduction

Two skills are being introduced ahead of next Monday: spec-driven plan retrospectives and live oral defense of technical decisions. Both are practiced disciplines, not naturally-acquired ones — they take repetition and rubric-driven feedback to land.

The plan-retrospective discipline answers a question most teams never formally ask: *did the plan we wrote on Monday survive contact with the work we did Tue–Fri?* Sprint retrospectives ask what went well and what didn't; plan retrospectives ask the more pointed version — *what was wrong with the plan itself*. The distinction matters because a plan-spec is a reviewable artifact (committed to git, cited in PRs, anchored to ADRs). When the work diverges from the spec, the divergence is data about the planning skill, not just about the work.

The live oral defense discipline answers a different question: *can you articulate the trade-off you made, in real time, under questioning, without strawmen?* This is the skill that separates an engineer who picked a vector store from an engineer who can defend why this vector store, against these alternatives, given these constraints. The rubric weights the articulation, not the conclusion — different defenders can land on different conclusions and both score well.

This reading is the preview for Monday. Specifics of W2's plan and this week's scenario-alternatives live in the daily overview; the discipline here is generic.

## 3. Core Concepts

### 3.1 What a plan retrospective is

A plan retrospective is a structured review of a prior plan-spec, conducted at the start of the next planning cycle, using the intervening work as evidence. In spec-driven development the plan-spec is a versioned artifact (under `openspec/` or equivalent), and the retrospective consumes the commit history, PR comments, ADRs, and eval results from the period since the spec was written ([Architectural Decision Records with Spec-Driven Development — intent-driven.dev, Apr 2026](https://intent-driven.dev/blog/2026/04/29/spec-driven-development-with-adr/), retrieved 2026-05-26).

The structure (these are the questions the retrospective answers, in order):

1. **What did the plan-spec say?** A literal re-reading of the spec at start of cycle. Treat it as a contract, not as an aspiration.
2. **What actually shipped?** What landed on `main`, with what shape, on what schedule.
3. **Where did reality diverge from the plan?** The interesting category — where the plan over-predicted, under-predicted, or named the wrong thing.
4. **What got dropped from the plan?** The most-skipped category. Items that were in the plan-spec at Monday and are not in the shipped work by Friday. Sometimes the drop was correct (the scope was wrong); sometimes the drop was a failure (the team got distracted). The retrospective names which.
5. **What carries forward into next week's plan?** Carrying forward is not the same as "we will do it next week" — it is an explicit decision about whether the dropped item is still in scope, deferred, or cancelled.

### 3.2 Why this differs from a sprint retrospective

A sprint retrospective asks "what went well, what didn't, what should we change about how we work?" A plan retrospective asks the narrower, more specific question — "was the plan itself a good plan?" The artifact under review is the plan-spec, not the team's process.

Both are useful and they happen at different cadences. The Asana 2026 sprint-retrospective guidance emphasises five steps and psychological safety; the plan retrospective borrows the safety discipline but adds the rigor of comparing against a committed artifact ([Asana — Sprint Retrospective 2026](https://asana.com/resources/sprint-retrospective), retrieved 2026-05-26). The plan retro is closer to a code-review of the plan than to a team-process review.

### 3.3 What got dropped — the load-bearing category

The "what got dropped" category is the one that exposes the planning skill most directly. Three sub-cases:

- **Dropped correctly.** The team realised mid-week that the scoped item was not the right scope; the plan over-promised. The retrospective names this and updates the planning heuristic ("next time, scope a research-heavy item to half-a-day, not a full day").
- **Dropped silently.** The team did not realise the item was dropped until the retrospective named it. This is the failure case. Either the plan was not consulted during the week (the spec was decorative) or the team was not honest about progress.
- **Dropped under pressure.** The team knew the item was at risk; the choice to drop was deliberate; the trade-off was made consciously. The retrospective surfaces what was traded against what.

The discipline that keeps "what got dropped" honest: the retrospective output is a written delta in the spec repo, not just a verbal acknowledgment. The delta names the item, the sub-case, and the carry-forward decision ([Spec-Driven Development with ADR — intent-driven.dev](https://intent-driven.dev/blog/2026/04/29/spec-driven-development-with-adr/), retrieved 2026-05-26).

### 3.4 The live oral defense — structure

The format that has settled into 2026 technical-skills evaluation:

- **2 min opener.** The defender states the decision, the key trade-off, and the most important alternative they rejected. No setup, no preamble — directly into the substance.
- **5–6 min probes.** The examiner asks pointed questions drawn from a rubric. Questions probe articulation depth, alternative awareness, trade-off honesty, and citation discipline.
- **2 min closer.** The defender summarises what they would do differently with hindsight, or what would change their decision if a key assumption shifted.

The total is typically 10 minutes per defender. The format compresses what would otherwise be a 30-minute conversation into a high-density signal pass. The structure matters because it disciplines the defender — there is no time for hand-waving — and it disciplines the examiner — the rubric is the script, not improvisation.

### 3.5 What the rubric weights

A 2026-typical technical defense rubric weights these dimensions (the specific weights vary by program):

- **Tech-stack fluency.** Does the defender know the actual mechanics of the system they chose? Specific config flags, default behaviors, performance characteristics. This is the dimension that catches "I chose X because the blog post said so" defenders.
- **Trade-off articulation without strawmen.** Does the defender represent the alternative options fairly, including their genuine strengths? Strawman alternatives (presenting only the weak version of a competitor) score poorly.
- **Domain framing.** Does the defender connect the technical choice to the domain it serves? A vector-store choice for a regulated industry has different constraints than the same choice for a casual chatbot.
- **Citation discipline.** Did the defender ground their reasoning in real, retrievable sources rather than vibes? "Citations as load-bearing, not decorative" means a defender who cannot produce the source for a claim under questioning loses points.
- **Constraint awareness.** Does the defender know what they don't know? Does the defender name the conditions under which their choice would be wrong?

Each dimension scores on an anchored 1–5 scale — anchors describe observable behavior at each band, not vague adjectives. Anchor design is the bias-mitigation tool ([Juicebox — Rubrics for Interviews 2026](https://juicebox.ai/blog/rubrics-for-interviews), retrieved 2026-05-26).

### 3.6 Bias and calibration in oral defense

Oral assessment is inherently susceptible to charisma effects, halo bias, and central-tendency clustering. The mitigation strategies that have shown up in 2026 hiring and assessment literature:

- **Behavioral anchors at every band.** A 5/5 on articulation has a specific definition: "the defender articulates complex ideas with exceptional clarity and proactively structures answers" ([Final Round AI Review — DEV, 2026](https://dev.to/finalroundai/i-reviewed-final-round-ai-for-technical-interviews-heres-what-actually-matters-in-2026-47gd), retrieved 2026-05-26). Without anchors, raters cluster at 3.
- **Evidence-based scoring.** A score is paired with a specific observation: "scored 4 on trade-off articulation because the defender named the genuine strength of Pinecone before explaining why Atlas was the better fit *for this constraint*." Vague scoring ("good energy") gets thrown out.
- **Multi-rater calibration.** When more than one rater is available, they score independently first, then compare. Disagreements over 2 bands trigger a calibration discussion. Single-rater scoring is a known weak point but is sometimes unavoidable.
- **Scores not returned live.** The defender does not see scores during the defense; scores are returned in a separate feedback session. This protects the defender from in-flight defensiveness and protects the rater from charisma-driven score creep.

## 4. Generic Implementation

A plan-retrospective template, in markdown — versioned in the same repo as the plan-spec it reviews:

```markdown
# Plan retrospective — Plan-Spec [WXX-Mon, dated YYYY-MM-DD]

## §0 What the plan said
[Quote the original plan-spec verbatim — the goals, the named work items, the open questions, the success criteria.]

## §1 What shipped
[List, with PR links: what landed on main between Monday and the retrospective.]

## §2 Where reality diverged
| Plan item | What actually happened | Why |
| --- | --- | --- |
| ... | ... | ... |

## §3 What got dropped
| Item | Sub-case (correctly / silently / under-pressure) | Carry-forward decision |
| --- | --- | --- |
| ... | ... | ... |

## §4 Carries forward into next plan-spec
[Bulleted, with the next-spec section each will appear in.]

## §5 Planning heuristic updates
[What did we learn about how to write the next plan? Concrete, prescriptive.]
```

The template is a forcing function — it cannot be filled in with hand-waving. Every cell either has evidence or is left explicitly blank. Left-blank cells are themselves a signal ("we have no evidence about X — that is a gap").

## 5. Real-world Patterns

**Healthcare — clinical-pathway specification.** A regional health network adopted a spec-first development discipline for clinical-decision-support changes. Plan-specs (typically 1-week cycles) are reviewed at the start of each new cycle against the shipped PRs and the eval delta. The discipline caught a pattern where the team consistently over-scoped corpus-extraction work and under-scoped prompt-engineering work; six weeks of retrospectives let them calibrate the next plan's scoping more accurately ([Spec-Driven Development 2026 Guide — BCMS](https://thebcms.com/blog/spec-driven-development), retrieved 2026-05-26).

**Fintech — engineering interviews.** A consumer-banking team adopted a structured-defense format for senior-engineer hiring after measuring a 3x predictive-validity improvement of structured interviews over unstructured ones. The rubric weighted system-design articulation, cost-trade-off awareness, and constraint reasoning — the same dimensions a programme defense rubric would weight, applied to interviews rather than coursework ([Juicebox — Rubrics for Interviews 2026](https://juicebox.ai/blog/rubrics-for-interviews), retrieved 2026-05-26).

**Education — academic oral defense.** Engineering graduate programs have used oral defense rubrics for decades; the 2026 evolution has been the addition of "AI infrastructure literacy" dimensions to defenses for ML and systems-engineering students. The defense now probes for "distributed-systems thinking, cost awareness, monitoring, fallback strategies, and a clear grip on what AI can't guarantee" — directly parallel to the dimensions a programme defense weights ([Final Round AI Review — DEV, 2026](https://dev.to/finalroundai/i-reviewed-final-round-ai-for-technical-interviews-heres-what-actually-matters-in-2026-47gd), retrieved 2026-05-26).

## 6. Best Practices

- Write the plan retrospective against the *committed* plan-spec, not against memory of what the plan said; the artifact is the contract.
- Cover all five categories (said / shipped / diverged / dropped / carry-forward); skipping "dropped" is the most common failure.
- Output the retrospective as a versioned markdown file in the same repo as the plan-spec; verbal-only retrospectives don't compound.
- Prepare opener-and-closer language before a live defense; the 2-minute time bound is not the moment to compose thoughts.
- Treat citations in a defense as load-bearing — be able to produce the source when asked, including the retrieval date and the specific claim it backs.
- Use rubrics with behavioral anchors at every band; vague rubrics cluster scores at 3 and lose signal.
- Score evidence-by-evidence, not impression-by-impression; the rationale paired with each score is the audit trail.
- Multi-rater calibrate when possible; if single-rater is unavoidable, name it as a known limitation.

## 7. Hands-on Exercise

**Whiteboard exercise (12 min):** Pick a recent technical decision you made (or watched a team make) within the last 3 months — choice of database, framework, deployment platform, anything. Sketch:

1. The 2-minute opener you would deliver if you had to defend the decision in a live defense format. Write the actual sentences.
2. The single most likely probing question an examiner would ask, and the 60-second answer you would give.
3. The closer — what would you do differently with hindsight, or what assumption would change your decision?

Then evaluate your own answers against the five rubric dimensions (fluency, articulation without strawmen, domain framing, citation discipline, constraint awareness). For each dimension, score yourself 1–5 with the anchor that justifies the score.

**What good looks like:** Your opener gets to the substance in the first sentence — no "so basically..." preambles. Your probing question is the question you would *not* want to be asked (the genuine weak spot of the decision), and your 60-second answer acknowledges the weakness honestly rather than deflecting. Your closer names a specific assumption that, if changed, would flip your decision. Your self-scores are paired with observable anchors ("4 on fluency because I named three specific config flags and their defaults"), not vague impressions ("3 because it felt okay").

## 8. Key Takeaways

- What is a §0 plan retrospective, and why does it differ from a generic sprint retrospective?
- Which category in a plan retrospective is most often skipped, and why does that category expose planning skill more than any other?
- What structure does a 10-minute live oral defense follow, and what does each segment (opener / probes / closer) test?
- Which dimensions does a 2026-typical defense rubric weight, and what bias-mitigation patterns does a well-designed rubric carry?

## Sources

1. [Architectural Decision Records with Spec-Driven Development using OpenSpec — intent-driven.dev (Apr 2026)](https://intent-driven.dev/blog/2026/04/29/spec-driven-development-with-adr/) — retrieved 2026-05-26
2. [Spec-Driven Development (SDD): The Definitive 2026 Guide — BCMS](https://thebcms.com/blog/spec-driven-development) — retrieved 2026-05-26
3. [Sprint Retrospective: How to Run an Effective Retro [2026] — Asana](https://asana.com/resources/sprint-retrospective) — retrieved 2026-05-26
4. [The Complete Guide to Using Rubrics for Interviews in 2026 — Juicebox](https://juicebox.ai/blog/rubrics-for-interviews) — retrieved 2026-05-26
5. [I Reviewed Final Round AI for Technical Interviews: What Actually Matters in 2026 — DEV Community](https://dev.to/finalroundai/i-reviewed-final-round-ai-for-technical-interviews-heres-what-actually-matters-in-2026-47gd) — retrieved 2026-05-26

Last verified: 2026-05-26
