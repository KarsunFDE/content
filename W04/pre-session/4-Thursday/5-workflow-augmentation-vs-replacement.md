---
week: W04
day: Thu
topic_slug: workflow-augmentation-vs-replacement
topic_title: "Workflow Augmentation vs Replacement"
parent_overview: W04/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://hbr.org/2018/07/collaborative-intelligence-humans-and-ai-are-joining-forces
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/articles/exploring-gen-ai/i-still-care-about-the-code.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/articles/exploring-gen-ai.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.openrewrite.org/recipes/java/spring/boot3/upgradespringboot_3_0
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/articles/patterns-legacy-displacement/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Workflow Augmentation vs Replacement

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish *augmentation* from *replacement* as workflow design choices, and recognise the operational signals each one optimises for.
- Apply the augmentation/replacement framing to two surfaces: tool-assisted code migration (codemods, refactor recipes) and AI-assisted software delivery (coding agents).
- Identify which decisions a human must own under augmentation, and articulate the kinds of mistakes that are dangerous to delegate.
- Name two anti-patterns of replacement-first thinking (over-automation, single-step migrations) and one anti-pattern of augmentation-first thinking (perpetual manual override).
- Use the framing to write a one-page workflow proposal that names who owns judgment, who owns mechanics, and the seam between the two.

## 2. Introduction

The argument over whether machines (or tools) *augment* or *replace* humans is older than software. It surfaces today in two adjacent debates: how aggressively to automate large-scale code transformations (codemods, OpenRewrite recipes, IDE refactors), and how to integrate AI coding assistants into the day-to-day developer loop. The vocabulary is shared; the operational stakes are different.

Harvard Business Review's "Collaborative Intelligence" framing argues that the most durable applications of AI in workplaces are those where humans and machines amplify each other's strengths rather than substituting for each other. The authors observe that across many industries, "AI will radically alter how work gets done and who does it, [but] the technology's larger impact will be in complementing and augmenting human capabilities, not replacing them" ([Collaborative Intelligence](https://hbr.org/2018/07/collaborative-intelligence-humans-and-ai-are-joining-forces), HBR, retrieved 2026-05-26).

In the codemod world, the same principle takes a more pedestrian form: OpenRewrite recipes handle a defined and deterministic share of a migration; the remainder requires judgment calls a human owns. The split is not a failure of the tool; it is the design of the workflow. Engineers who try to push the ratio to 100% automation usually discover the residual was the part that actually needed thinking ([OpenRewrite UpgradeSpringBoot_3_0 recipe](https://docs.openrewrite.org/recipes/java/spring/boot3/upgradespringboot_3_0), retrieved 2026-05-26).

This reading frames the choice as a workflow design question — when does augmentation fit, when does replacement fit, and what does the seam between them look like in practice?

## 3. Core Concepts

### 3.1 The two workflow shapes

**Augmentation.** The tool does the mechanical bulk; the human owns the judgment calls and the final commit. The tool may produce a complete-looking output, but its output is treated as a *proposal* until a human evaluates it. The seam is named: "this is the part the tool decides, this is the part I decide."

**Replacement.** The tool replaces a step that a human used to do, end-to-end. Output goes to production without human review on each invocation. Replacement is appropriate when (a) the step is well-specified, (b) the failure modes are bounded and observable, and (c) the rollback is cheap and tested.

The two shapes are not opposites; they are the endpoints of a spectrum. Most real workflows are augmentation in the small (per-task) and replacement in the large (the *kind* of step is automated, but specific instances may still be reviewed).

### 3.2 Three diagnostics for choosing

| Question | Augmentation if… | Replacement if… |
|---|---|---|
| Is the output specification complete? | No — judgment is needed about edge cases | Yes — the spec covers every case |
| Are the failure modes bounded and observable? | Failures are open-ended; some are silent | Failures are loud and catchable |
| Is rollback cheap and pre-rehearsed? | Rollback requires investigation | Rollback is a single command |

If any answer points to "augmentation," the workflow needs a human in the loop. The most common workflow design error is to treat all three as "yes" when at least one is "no."

### 3.3 The augmentation seam in tool-assisted migrations

When OpenRewrite runs the SB 2.7 → 3.0 composite recipe, roughly two-thirds to three-quarters of the diff lands deterministically — imports, deprecated APIs, Maven/Gradle coordinates, configuration-key renames ([UpgradeSpringBoot_3_0 recipe](https://docs.openrewrite.org/recipes/java/spring/boot3/upgradespringboot_3_0), retrieved 2026-05-26). The residual concentrates in three places: custom servlet `Filter` wiring, `WebSecurityConfigurerAdapter` subclasses, and distributed-tracing configuration. Each of those requires a decision about *intent* — the recipe can rewrite imports but cannot infer what the team wanted the filter chain to enforce.

The mature workflow names the seam: codemod runs first, captures its patch, and the team treats the residual as the surface for judgment. The tooling does not "fail" when it cannot rewrite the residual; the *workflow* fails if the team treats the seam as a defect rather than a design feature.

### 3.4 The augmentation seam in AI-assisted delivery

The analogous question for AI coding assistants is sharper because LLM outputs are not deterministic. Birgitta Böckeler's Thoughtworks memo argues for a specific test: *would you be comfortable being on-call for a 1,000-5,000-line change set you did not read?* If the answer is no, the workflow is augmentation, not replacement — the seam includes test code at minimum, and almost certainly the code itself ([I still care about the code](https://martinfowler.com/articles/exploring-gen-ai/i-still-care-about-the-code.html), 9 July 2025, retrieved 2026-05-26).

The memo notes the foundational distinction: "LLMs are NOT compilers, interpreters, transpilers or assemblers of natural language, they are inferrers" — a compiler takes a structured input and produces a repeatable, predictable output; an LLM does not. That property does not disqualify replacement workflows for narrow, well-spec'd tasks, but it raises the bar for what counts as "well-spec'd."

### 3.5 Anti-patterns on both sides

- **Over-automation.** Pushing the augmentation/replacement seam past the point of bounded failure modes. A common symptom: the team finds itself debugging mass-generated code at 2am.
- **Single-step migration.** Treating a multi-stage migration as a single tool invocation. Even when the recipe handles 95% of the diff, the remaining 5% benefits from a named human stage.
- **Perpetual manual override.** The opposite failure: refusing to let a deterministic recipe run because "I want to do it by hand to learn." Augmentation has a finite half-life; if the team never trusts the tool, the recipe never proves out.
- **Augmentation theatre.** A human "reviews" a 5,000-line tool-generated diff in five minutes and ships it. The seam exists on paper but not in practice. The discipline that prevents this is naming what the human is reviewing *for*, in advance.

### 3.6 Augmentation is the same principle as HITL authority

Workflow augmentation is the SDLC-week framing of the same principle that the agentic-systems weeks named as HITL authority. In the agent-tools world, HITL authority answers "which actions does a human always approve?" In the SDLC-tools world, augmentation/replacement answers "which steps does a human always review?" The vocabulary is different; the operational habit is the same.

## 4. Generic Implementation

The sketch below is a one-page workflow proposal template for any team introducing a new automation step. Replace the verbs as needed.

```
WORKFLOW: <name of the step being automated>

1. Inputs.
   - What does the step take in?
   - What's the source of truth for the inputs?

2. Mechanical core (what the tool decides).
   - The deterministic portion of the step.
   - The tool's output is the candidate output.
   - Example: codemod patch, generated config, scaffolded code.

3. Judgment seam (what the human decides).
   - The portion of the step where intent is required.
   - Named in advance — what is the human reviewing *for*?
   - Example: filter-chain wiring intent, test-code review,
     handling of a transitive dependency the tool does not see.

4. Failure modes.
   - The two or three ways this step is most likely to be wrong.
   - For each, what's the detection signal? (CI failure, lint
     warning, runtime alert, missing test.)

5. Rollback.
   - The exact command(s) to undo the step.
   - Has the rollback been rehearsed? (yes / no)
   - If no, schedule the rehearsal before the first real run.

6. Augmentation today / replacement tomorrow.
   - What's the criterion for promoting this from augmentation
     to replacement? (Usually: N successful runs, M observed
     failures all caught by the detection signals in step 4.)
```

The proposal is a half-page if the team is honest. The discipline is filling in section 3 ("what the human decides") in concrete enough language that the next reviewer knows what they are reviewing for.

## 5. Real-world Patterns

**Healthcare imaging — radiology workflow with AI triage.** Several published case studies on hospital radiology workflows describe the pattern: an AI model triages incoming images and flags candidate findings; a radiologist reviews each flagged case before the report goes out. The model replaces the *triage* step (which used to be done by humans queueing studies) but augments the *diagnosis* step. The seam is named explicitly in the workflow doc — radiologists review for findings the model missed and findings the model raised in error ([Collaborative Intelligence](https://hbr.org/2018/07/collaborative-intelligence-humans-and-ai-are-joining-forces), retrieved 2026-05-26).

**E-commerce — automated content moderation with human review queue.** Most large platforms run a two-stage moderation pipeline: an automated classifier handles bulk decisions (replacement workflow for clear-cut cases) and routes ambiguous cases to a human reviewer (augmentation workflow for the residual). The platform tunes the threshold based on observed false-positive and false-negative rates; the seam moves over time as the classifier improves, but it never disappears.

**Gaming — procedural content generation with art-direction review.** Game studios using procedural generation for assets (terrain, NPC variants, dialogue branches) describe the same seam: the generator produces candidate content at scale; an art director or narrative designer reviews and either accepts, edits, or regenerates. The replacement-vs-augmentation split is on the *kind* of asset, not the per-asset decision — generated terrain may ship unreviewed; generated dialogue does not.

**Fintech — fraud-detection model with analyst escalation.** Card-issuer fraud teams describe a tiered workflow: the model auto-declines on high-confidence fraud (replacement), routes medium-confidence cases to an analyst queue (augmentation), and lets low-confidence transactions through with a logged signal (replacement, with audit). The seam is concrete: the analyst's review notes feed back into the next model retrain — augmentation produces the training signal that improves the replacement portion.

## 6. Best Practices

- Write down where the seam is *before* you ship the workflow; "we'll figure it out" is the failure mode.
- Name what the human is reviewing for, in concrete terms — "review the filter chain wiring for unintended permissive rules," not "review the diff."
- Treat the residual the tool cannot handle as the workflow's product, not its defect; the residual is where judgment lives.
- For tool-assisted refactors, separate the mechanical sweep from the manual residual into two named stages with a rescue branch between them.
- For AI-assisted authoring, require the human to read at least the test code; if the change is too large to read, it is too large to ship.
- Promote a step from augmentation to replacement only after named criteria are met — number of successful runs, failure modes observed and all caught, rollback rehearsed.
- Revisit the seam quarterly; tooling improves and the seam moves, but the seam never disappears completely.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** Choose one automated step from your own day-to-day work (CI test runner, code formatter, deployment script, AI coding-assistant invocation). Fill in the six-section template from Section 4 for that step. Do not skip section 3 — name what the human reviews for, even if you have to invent the answer.

When you finish, swap with a partner and review each other's drafts. Look specifically for: (a) is the judgment seam concrete enough that a stranger could enforce it? (b) are the failure modes observable, or are some silent? (c) has the rollback been rehearsed?

**What good looks like:** section 3 reads like a checklist, not a paragraph; failure-mode detection signals are specific (a named CI job, a specific log alarm); the rollback section names a specific commit, tag, or feature flag the team has actually used in the last quarter.

## 8. Key Takeaways

- *What distinguishes an augmentation workflow from a replacement workflow?* Augmentation keeps a named seam where a human owns judgment; replacement automates the step end-to-end based on a complete spec, bounded failure modes, and a cheap rollback.
- *How do you decide which shape to use?* Apply the three diagnostics — spec completeness, failure-mode observability, rollback cheapness — and treat any "no" as a signal that augmentation is the right shape today.
- *Where does the seam live in a codemod-driven migration?* In the residual the deterministic recipe cannot handle, which concentrates in the parts of the system where intent is encoded (security wiring, observability config, custom integrations).
- *What does the seam look like for AI-assisted coding?* At minimum, the human reads the test code; for most teams in 2026 it includes the application code as well, because LLMs are inferrers, not compilers.
- *Why does the augmentation/replacement framing matter for legacy modernization specifically?* Because every modernization is a sequence of small workflow choices — what does the tool handle, what do we own — and the team that names those choices ships; the team that hopes the tool handles everything stalls.

## Sources

1. [Collaborative Intelligence: Humans and AI Are Joining Forces](https://hbr.org/2018/07/collaborative-intelligence-humans-and-ai-are-joining-forces) — retrieved 2026-05-26
2. [I still care about the code](https://martinfowler.com/articles/exploring-gen-ai/i-still-care-about-the-code.html) — retrieved 2026-05-26
3. [Exploring Generative AI (index)](https://martinfowler.com/articles/exploring-gen-ai.html) — retrieved 2026-05-26
4. [OpenRewrite — UpgradeSpringBoot_3_0 Recipe](https://docs.openrewrite.org/recipes/java/spring/boot3/upgradespringboot_3_0) — retrieved 2026-05-26
5. [Patterns of Legacy Displacement](https://martinfowler.com/articles/patterns-legacy-displacement/) — retrieved 2026-05-26

Last verified: 2026-05-26
