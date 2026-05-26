---
week: W03
day: Mon
topic_slug: iterative-spec-driven-dev-and-section-zero-retro-discipline
topic_title: "Iterative spec-driven dev + the §0 retro discipline"
parent_overview: W03/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://github.com/github/spec-kit
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.atlassian.com/team-playbook/plays/retrospective
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/bliki/ConversationalStories.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://adr.github.io/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://en.wikipedia.org/wiki/Retrospective
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Iterative spec-driven dev + the §0 retro discipline

## 1. Learning Objectives

By the end of this reading, the learner can:

- State what *spec-driven development* means and how it differs from "writing a doc."
- Explain why the spec is a living artefact that retrospects on itself, not a one-shot upfront design.
- Run a focused 15-minute "§0 retrospective" against last cycle's spec and produce three named carry-forwards.
- Distinguish a spec from an architecture decision record (ADR) and explain how the two interact.
- Recognise the failure modes of unrevisited specs (silent drift, frozen assumptions, fictional baselines).

## 2. Introduction

Spec-driven development is the discipline of *writing down what you intend to build before you build it, and revisiting what you wrote after you finish.* The first half (write a spec before you code) is widely accepted in principle and frequently skipped in practice. The second half (revisit the spec at the end of the cycle and update it with what actually happened) is the part that turns specs from upfront-design theatre into a living, falsifiable artefact.

The framing has had a recent renaissance because LLM-assisted coding makes spec quality matter more, not less. GitHub's *Spec-Kit* project frames it directly: the spec is "the lingua franca" — the artefact humans and AI agents both consume to drive implementation, and the quality of the system reflects the quality of the spec ([GitHub — spec-kit](https://github.com/github/spec-kit), retrieved 2026-05-26). The same idea, in a pre-LLM register, runs through Martin Fowler's writing on conversational stories: requirements documents that are revised by the conversation, not frozen by it ([Martin Fowler — Conversational Stories](https://martinfowler.com/bliki/ConversationalStories.html), retrieved 2026-05-26).

The §0 retrospective discipline is the lightweight ritual that keeps specs honest: every Plan Day starts by re-reading the previous cycle's spec against what actually shipped, naming the carry-forwards, then writing the new spec on top of *what we now know*, not what we thought a week ago.

## 3. Core Concepts

### 3.1 What a spec actually contains

A useful working shape for a spec — drawing on GitHub Spec-Kit's framing and the broader literature — has six sections:

1. **Context** — what is the situation, who are the stakeholders, what was the previous state.
2. **Goals** — what does success look like, in measurable terms when possible.
3. **Non-goals** — what we explicitly are *not* doing in this cycle.
4. **Approach** — the architecture sketch, the major components, the key trade-offs (often with sub-ADRs).
5. **Risks and unknowns** — what could go wrong, what we do not know yet.
6. **Acceptance signals** — what evidence will tell us the cycle is done.

Spec-Kit's framing emphasises that the spec is "explicit, executable, and evolving" — it should be specific enough to drive implementation but loose enough to absorb learning ([GitHub — spec-kit](https://github.com/github/spec-kit), retrieved 2026-05-26).

### 3.2 The spec vs the ADR

These are not the same artefact. The relationship is:

- A **spec** describes *what we are building this cycle* and *the shape of how*.
- An **ADR** records a *single architecturally significant decision* — its context, options considered, choice made, consequences ([adr.github.io](https://adr.github.io/), retrieved 2026-05-26).

A spec for a single cycle typically *references* one or more ADRs (e.g., "we chose a supervisor-worker shape — see ADR-0011"). The ADR is the durable, atomic record. The spec is the bounded, time-boxed planning document. The two age differently: ADRs accumulate forever; specs are superseded by the next cycle's spec.

### 3.3 What "iterative" actually means in spec-driven dev

The opposite of iterative is *waterfall* — a spec written once, frozen, and treated as the contract. Iterative spec-driven dev keeps the spec as the contract within a cycle but reopens it at the cycle boundary:

```
Cycle N:
  §0  read Cycle N-1 spec + what shipped
  §0  name carry-forwards
  §1  write Cycle N spec
  §2  build against Cycle N spec
  §3  ship; observe; capture deltas
Cycle N+1: repeat, with §0 driven by Cycle N's actual results
```

The single most important property is that *§0 happens before §1* — you do not write the new spec until you have re-read the old one. This is what stops the slow drift of fictional baselines, where each cycle's spec is built on assumptions the previous cycle invalidated but no-one wrote down.

### 3.4 Anatomy of a §0 retrospective

Borrowing from agile retrospective practice, a focused §0 has three questions, time-boxed to 10–15 minutes ([Atlassian — Sprint Retrospective playbook](https://www.atlassian.com/team-playbook/plays/retrospective), retrieved 2026-05-26; [Retrospective — Wikipedia](https://en.wikipedia.org/wiki/Retrospective), retrieved 2026-05-26):

1. **What did the previous spec under-estimate?** (i.e., the work we thought was small turned out to be large; the constraint we thought was loose turned out to be binding.)
2. **What did the previous spec over-engineer?** (i.e., the affordance we built but did not use; the gate we added but no-one tripped; the abstraction that did not pay off.)
3. **Which decision from the previous spec turned out to be load-bearing?** (i.e., the thing the team now relies on; the assumption everyone is implicitly carrying.)

Each answer produces a carry-forward — a sentence that lands in the new spec's §1 (Context) or §5 (Risks). The §0 is not for blame, not for celebration; it is for naming the deltas that should change the new spec.

### 3.5 Failure modes the §0 corrects

- **Silent drift.** The spec was right at the start; it is wrong now, but no-one revised it. The team is following stale guidance and does not know it.
- **Frozen assumptions.** The spec said "the data is uniform Latin-1 strings" — in practice 8% of the data turned out to be Unicode emoji and the code has been silently dropping rows ever since.
- **Fictional baselines.** Each new spec cites metrics from the previous spec ("we improved latency 30%") that were never measured properly. The numbers compound into a fiction.
- **Goal creep without acknowledgement.** The team started building feature A; mid-cycle they pivoted to feature B; the spec still says A and nobody updated it. The next cycle is planned against a non-existent baseline.

The §0 catches all four because it forces the question *what is actually true today, against what the spec says*.

### 3.6 Writing the new spec on top of carry-forwards

After a §0, the new cycle's spec §1 (Context) explicitly enumerates the carry-forwards:

> Carrying forward from Cycle N: (1) the retrieval boundary needs multi-tenant filtering — discovered Tue when tenant A queries returned tenant B documents; (2) the hybrid retrieval + reranker we built was unnecessary at this scale — basic kNN handles the load; (3) the load-bearing ADR was ADR-0007 (we picked Postgres pgvector) — every subsequent decision assumes it.

That paragraph would be invisible without the §0 and is the single most useful artefact at the start of a cycle. It tells the new spec where to apply pressure (carry-forward #1 → new component), what to remove (carry-forward #2 → delete the reranker), and what is locked (carry-forward #3 → do not revisit pgvector this cycle).

## 4. Generic Implementation

A worked example outside federal-acquisitions: a small B2B SaaS team running two-week cycles on a customer-success platform.

**End of Cycle N — they wrote this Cycle-N spec two weeks ago:**

```markdown
# Cycle N Spec — Churn-risk score MVP
## §1 Context
Customer-success team currently triages renewals manually...
## §2 Goals
- Ship a per-account churn-risk score (0-100) by end of cycle.
- Display the score in the CS dashboard.
## §3 Non-goals
- We are not building automated outreach this cycle.
## §4 Approach
- Train a logistic regression on the 18-month renewal dataset.
- Expose via a `/score/{account_id}` endpoint.
- ADR-0014: per-tenant model vs shared model — chose shared for cycle 1.
## §5 Risks
- Data quality on the 18-month dataset is unknown.
## §6 Acceptance signals
- 80% of accounts have a non-null score by cycle end.
```

**Start of Cycle N+1 — §0 retrospective on the above (12 minutes):**

```markdown
## §0 Retrospective on Cycle N
### What did the previous spec under-estimate?
- The data-quality risk (§5) materialised harder than expected.
  18-month dataset had 35% of accounts with missing usage signals.
  We patched with imputation but it is fragile.

### What did the previous spec over-engineer?
- The logistic regression pipeline is more flexible than we needed.
  In practice 4 features explain 90% of the score; the other 12
  add complexity without much signal.

### Which decision turned out to be load-bearing?
- ADR-0014 (shared model). Every CS rep now relies on cross-tenant
  comparison ("this account looks like 5 other accounts that churned").
  Do not revisit shared-vs-per-tenant this cycle.

## §1 Context (Cycle N+1)
Carrying forward from Cycle N: (1) data-quality remediation is the
critical-path work this cycle; (2) we will prune the model to the
top 4 features rather than adding more; (3) ADR-0014 is locked.

## §2 Goals
- Replace imputation with a real data-quality pipeline.
- Prune the model and re-validate; ship a v2 score.
- ADR-0017: choose between dropping vs imputing missing accounts.

... (rest of spec)
```

The shape: §0 names three things, §1 carries them forward, §2 onward responds to them. The whole §0 fits on one page.

## 5. Real-world Patterns

**Software product management — Atlassian, GitLab, Linear all run lightweight cycle-boundary retros.** Modern product teams have largely converged on a "what worked / what did not / what to change" three-question shape, with the recurring discipline that the *output* of the retro lands as concrete commitments in the next cycle's plan — not as bullet points filed away forever ([Atlassian — Sprint Retrospective](https://www.atlassian.com/team-playbook/plays/retrospective), retrieved 2026-05-26).

**Hardware engineering — design reviews and lessons-learned databases.** Mature hardware programmes treat post-mortems as a living artefact: each design review opens with "lessons we are carrying forward from the previous build." NASA's lessons-learned databases are the institutional ancestor of the §0 retro — they exist because the cost of forgetting a previous cycle's failure in hardware is measured in lives, not story points.

**Manufacturing — Toyota's kaizen and the A3 report.** The Toyota Production System made "look at what just happened, then plan the next iteration" load-bearing decades before software borrowed the idea. The A3 report — a single-page write-up that includes the current state, the target state, and the gap-closure plan — is structurally a spec with a built-in §0. Software has reinvented the form many times since.

**Open-source — RFC repositories with explicit supersession.** Communities like Rust and Python publish RFCs that explicitly reference and supersede earlier RFCs. The pattern matches spec-driven dev with §0: each RFC opens by citing the prior art it is updating or replacing, so the historical chain is reconstructable. The lesson: institutional memory does not maintain itself; you have to write it down at the boundary.

## 6. Best Practices

- Run the §0 retro *before* writing any of the new cycle's §1; reversing the order defeats the purpose.
- Time-box the §0 to 10–15 minutes — it is not a generic retrospective, it is a focused read of one document against one set of outcomes.
- Capture carry-forwards as one-sentence claims, not paragraphs — if you cannot say it in a sentence, you have not understood the delta.
- Treat ADRs as the durable layer and specs as the time-boxed planning layer; do not conflate the two.
- Update the previous cycle's spec *before* archiving it — note what shipped, what did not, what changed, so a future reader can trust it.
- Resist the urge to skip §0 on a "small" cycle — small cycles drift fastest because no-one feels the cost of skipping.
- Make the §0 visible to the team that did the work, not just to the author writing the new spec — collective memory is the asset.

## 7. Hands-on Exercise

**Whiteboarding / writing prompt (15 minutes).** Pick a feature you (or your team) shipped in the last month. Pretend it was Cycle N. Now write the §0 retrospective for the next cycle — three questions, one paragraph each, ending with three one-sentence carry-forwards.

Expected components:

- Question 1: a concrete under-estimation, with one piece of evidence from the cycle (a specific incident, a metric, a bug).
- Question 2: a concrete over-engineering, with one specific affordance that did not pay off.
- Question 3: one decision the team now implicitly relies on, with a sentence on why revisiting it would be disruptive.
- Three sentences at the bottom labelled "Carry forward to Cycle N+1" — each a falsifiable claim, not an aspiration.

**What good looks like.** Your §0 fits on half a page. Each carry-forward names something specific enough that a teammate could pick the spec up tomorrow and act on it without asking you to elaborate. You did not turn the §0 into a blame log or a victory lap — it is calibration, not judgement. You can defend each carry-forward against the question *what evidence supports this?*

## 8. Key Takeaways

- Can you describe a spec's six sections and the difference between a spec and an ADR?
- Can you state what makes spec-driven dev *iterative* and why §0 has to come before §1?
- Can you run a focused 15-minute §0 retro and produce three carry-forwards a teammate could act on?
- Can you name the four failure modes (silent drift, frozen assumptions, fictional baselines, goal creep without acknowledgement) and explain how §0 catches each?
- Can you defend the discipline to a sceptical engineer who wants to "just start coding" the new cycle?

## Sources

1. [GitHub — spec-kit (Spec-Driven Development toolkit)](https://github.com/github/spec-kit) — retrieved 2026-05-26
2. [Sprint Retrospective: How to Hold an Effective Meeting — Atlassian](https://www.atlassian.com/team-playbook/plays/retrospective) — retrieved 2026-05-26
3. [Conversational Stories — Martin Fowler](https://martinfowler.com/bliki/ConversationalStories.html) — retrieved 2026-05-26
4. [Architectural Decision Records (ADRs)](https://adr.github.io/) — retrieved 2026-05-26
5. [Retrospective — Wikipedia](https://en.wikipedia.org/wiki/Retrospective) — retrieved 2026-05-26

Last verified: 2026-05-26
