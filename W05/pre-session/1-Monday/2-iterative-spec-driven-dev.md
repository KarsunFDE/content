---
week: W05
day: Mon
topic_slug: iterative-spec-driven-dev
topic_title: "Iterative Spec-Driven Dev — §0 retro as a load-bearing discipline"
parent_overview: W05/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://github.com/github/spec-kit
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.productbuilder.net/learn/spec-driven-development
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.infoq.com/articles/enterprise-spec-driven-development/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://aws.amazon.com/blogs/industries/from-spec-to-production-a-three-week-drug-discovery-agent-using-kiro/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Iterative Spec-Driven Dev — §0 retro as a load-bearing discipline

## 1. Learning Objectives

By the end of this reading, the learner can:

- Explain why spec-driven development (SDD) is *iterative* rather than *waterfall*, and how that distinction shapes day-to-day cadence.
- Describe the four canonical SDD phases (spec → plan → tasks → code) and where human-review gates sit between them.
- Justify why every plan cycle should open with a retrospective on the previous cycle's spec, not on the previous cycle's code.
- Identify the artifact-level signals that a spec is "earning its keep" versus the signals that it has decayed into shelfware.

## 2. Introduction

Spec-driven development is a methodology where structured, versioned specifications — not code — are the source of truth, and code is generated or maintained against those specs by humans and AI coding agents working together. The premise is older than the current AI tooling wave (formal-methods communities have argued for it for decades), but the AI-assisted-engineering era has dramatically lowered the cost of writing and revising specs, which is the chief reason the practice is back in vogue ([Microsoft Developer Blog, 2026-05-26](https://developer.microsoft.com/blog/spec-driven-development-spec-kit)).

SDD is explicitly *iterative within a change scope* — you specify, plan, and implement a single feature or bugfix, not the entire system upfront. Critically, humans review at every phase boundary: the spec is reviewed before the plan, the plan is reviewed before tasks, tasks are reviewed before implementation. The phase boundaries are what make SDD predictable ([GitHub Spec Kit, 2026-05-26](https://github.com/github/spec-kit/blob/main/spec-driven.md)).

The thing teams routinely miss is that SDD is a *loop*, not a sequence. Each iteration's spec is informed by what the previous iteration's spec got wrong. Without a closing retrospective on the spec itself, you end up with a fresh spec every cycle and zero compounding learning — the artifacts pile up but the discipline rots. The §0 retrospective on the last cycle's plan-spec is what keeps the loop closed.

This reading is the generic framing of that discipline. The day's overview ties it back to W4's plan-spec and the Mid-Sprint Surprise; here we look at the practice across industries and at the tooling landscape circa 2026.

## 3. Core Concepts

### 3.1 Four canonical SDD phases

The phases vary in naming across tools but cluster the same way ([GitHub Blog, 2026-05-26](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/)):

| Phase | Question it answers | Primary artifact |
|-------|---------------------|------------------|
| Spec | What and why | High-level description, no implementation detail |
| Plan | How (technical blueprint) | Architecture + design decisions traced to the spec |
| Tasks | Steps | Ordered, scoped work items |
| Code | The implementation | Production code + tests, generated/written against the tasks |

Each phase boundary is a human-review checkpoint. Skipping the boundary — especially the spec → plan one — is the single most common SDD anti-pattern, because it lets implementation drift outpace any artifact that explains it.

### 3.2 The "constitution" or "steering" file

Mature SDD tooling layers a *constitution* (Spec Kit's term) or *steering file* (Kiro's term) on top of the per-feature spec. The constitution captures the project-level non-negotiables: testing approach, deployment posture, performance budgets, security baselines. These are the rails every per-feature spec runs on so the loop doesn't relitigate them each cycle ([Microsoft Developer Blog, 2026-05-26](https://developer.microsoft.com/blog/spec-driven-development-spec-kit)).

A retro that revises the constitution is a different kind of event from a retro that revises a feature spec — the first changes the rails, the second just trims the track.

### 3.3 The retrospective gate

The §0 retrospective on the last cycle's spec asks four questions:

1. What did the spec correctly predict that the implementation honoured?
2. What did the spec miss that the implementation discovered?
3. What did the spec over-specify (specifying things the code never needed)?
4. What in the spec turned out to be the wrong abstraction entirely?

Question 2 is the load-bearing one. A spec that gets surprised by production reality has done its job — it surfaced the gap. A spec that's never surprised is either trivially correct or quietly disconnected from the system. The retro is the place to name that.

### 3.4 Iteration cadence

SDD does not prescribe a cadence — teams pick the loop length that fits their delivery rhythm. Common shapes ([InfoQ enterprise SDD, 2026-05-26](https://www.infoq.com/articles/enterprise-spec-driven-development/)):

- **Per-feature** (most common): one spec per shippable feature; retro at feature close.
- **Per-sprint**: one spec per sprint; retro at sprint retro.
- **Per-plan-day**: one spec per planning interval, retro opens the next interval.

The shape matters less than the closure. A team that ships specs but never retros them is doing waterfall-with-extra-steps; a team that retros every cycle is doing genuine iterative SDD even on a fast cadence.

### 3.5 What lives in the spec vs the code

The drift trap: as code grows, decisions that "should" be in the spec accumulate in code comments, PR descriptions, or tribal knowledge. The retrospective is where you catch this — a spec that hasn't moved in three cycles while the code has shipped three meaningful design changes is a spec that's lost the source-of-truth crown. The fix is to demote the spec (mark it stale) or backport the decisions; pretending nothing happened is the failure mode.

## 4. Generic Implementation

A minimal spec template for any team, in any domain. Substitute your domain terms; the structure carries.

```markdown
# Spec: <feature name>

## Why (problem statement)
<2-4 sentences: what user/system problem this addresses>

## What (behaviour)
<bulleted list of observable behaviours; no implementation>

## Non-goals
<things this spec deliberately doesn't address>

## Constraints
<perf, security, compliance, integration>

## Acceptance signals
<how we will know it works in production>

## Risks + open questions
<the items the next planning session must resolve>

## Retrospective slot (filled at next cycle's §0)
- Predictions that held:
- Surprises:
- Over-specifications:
- Wrong abstractions:
```

The retrospective slot at the bottom is what makes the artifact iterative. When the next cycle opens, the §0 retro fills in that block on the *previous* cycle's spec before the new cycle's spec is even sketched. Tooling like Spec Kit, Kiro, OpenSpec, BMAD, and Tessl ([Spec-Driven Development 2026, 2026-05-26](https://www.productbuilder.net/learn/spec-driven-development)) each have their own templates, but the retro slot pattern is portable across all of them.

## 5. Real-world Patterns

**Healthcare — drug discovery agent (industry)**. A pharma team built a discovery-pipeline agent in three weeks using Kiro's spec-first flow. The team's report flags the same insight that matters for any iterative SDD adopter: the per-week spec retro was where they caught that early specs over-specified the molecular-screening criteria and under-specified the audit-trail format the FDA submission would later demand. The retro changed the steering file (FDA audit-trail format became a project-level non-negotiable) and trimmed the per-feature specs. Without the retro, the second-week spec would have repeated the first's mistake ([AWS for Industries — Kiro drug discovery, 2026-05-26](https://aws.amazon.com/blogs/industries/from-spec-to-production-a-three-week-drug-discovery-agent-using-kiro/)).

**E-commerce — feature-flag retrospective**. A large e-commerce team running per-sprint SDD discovered through retros that their specs consistently under-specified rollback behaviour. The retros surfaced the pattern across three cycles before the team formalised "rollback contract" as a constitution-level requirement. The lesson is the meta-pattern: a single missed spec is a bug; the same gap three retros running is a constitution change waiting to be made ([Spec-Driven Development at enterprise scale, InfoQ, 2026-05-26](https://www.infoq.com/articles/enterprise-spec-driven-development/)).

**Fintech — settlement reconciliation**. A fintech reconciliation team uses per-feature SDD with a strict §0 retro discipline. The artifact that matters to them is not the spec itself but the *retrospective delta* — the diff between what each spec predicted and what production revealed. After ~18 cycles the team's retro-deltas trended toward zero on settlement-window edge cases (the constitution had absorbed them) and stayed non-zero on counterparty-API quirks (uncontrollable variability). That signal lets them target hardening work where it pays back ([Spec-Driven Development guide, BCMS, 2026-05-26](https://thebcms.com/blog/spec-driven-development)).

**Gaming — live-ops feature pipeline**. A live-ops team for a free-to-play title runs SDD on a daily plan cadence (deploys are constant). Their retro discipline collapsed early without rails — daily was too tight to do retros well. They moved retros to weekly while keeping specs daily, which produced the right rhythm: specs stayed close to the work, retros stayed thoughtful. The cadence-decoupling lesson is portable: the spec loop and the retro loop don't have to share a clock.

## 6. Best Practices

- **Open every planning cycle with §0 retro on the previous cycle's spec.** Not the code, not the velocity — the spec. The retro asks what the spec predicted, what it missed, what it over-specified, what abstraction was wrong.
- **Treat the constitution / steering file as a different artifact from the per-feature spec.** Don't relitigate project-level non-negotiables in every feature spec; promote recurring spec surprises into the constitution so the next cycle inherits the learning.
- **Make the retro slot a structural part of the spec template**, not a separate doc. A spec that has nowhere to record its own post-mortem will not be retroed.
- **Ship specs at the same cadence you ship code.** A spec that hasn't moved in three cycles while the code has shipped meaningful changes is stale; either backport the changes or mark the spec deprecated.
- **Keep human-review gates at every phase boundary.** Spec → plan, plan → tasks, tasks → code. Skipping any boundary defeats the predictability that makes SDD worth the overhead.
- **Resist the urge to over-specify.** A spec that pins implementation choices the code doesn't need is dead weight; the retro is where you catch this and trim.
- **Distinguish "surprise" from "failure" in the retro.** A spec surprised by production reality has done its job — surfacing the gap is the point. Punishing surprise teaches teams to write defensive, untestable specs.

## 7. Hands-on Exercise

**Whiteboarding prompt (10 min).** Pick a feature you shipped in the last 30 days. On a single page, write the spec that would have predicted that feature's behaviour using the template in §4. Then fill in the retrospective slot honestly:

- What does the spec correctly predict?
- What did the spec miss that the implementation discovered?
- What is in the spec that the implementation never needed?
- What abstraction in the spec turned out to be wrong, and what would replace it?

**What good looks like.** A finished page has all four retro answers populated with one or two sentences each, and at least one of them surfaces a candidate for the project's constitution or steering file (a recurring concern that should be a project-level non-negotiable). If the retro section is empty, the spec is shelfware; if it surfaces a constitution-level candidate, the iteration paid for itself.

## 8. Key Takeaways

- *What makes SDD iterative rather than waterfall?* Per-feature scope, retrospective closure, and human-review gates at every phase boundary.
- *Where does compounding learning live in SDD?* In the §0 retro plus the constitution / steering file. Without both, the artifacts pile up but the discipline rots.
- *How do you know a spec is earning its keep?* Its retro produces non-trivial deltas, and recurring deltas migrate into the constitution.
- *What's the difference between spec surprise and spec failure?* Surprise is the spec doing its job — surfacing the gap. Failure is the spec drifting from the code without anyone noticing.

## Sources

1. [GitHub Spec Kit — toolkit and methodology](https://github.com/github/spec-kit) — retrieved 2026-05-26
2. [Spec-driven development with AI: a new open source toolkit (GitHub Blog)](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/) — retrieved 2026-05-26
3. [Spec-Driven Development (2026 Guide) — Product Builder](https://www.productbuilder.net/learn/spec-driven-development) — retrieved 2026-05-26
4. [Spec-Driven Development — Adoption at Enterprise Scale (InfoQ)](https://www.infoq.com/articles/enterprise-spec-driven-development/) — retrieved 2026-05-26
5. [From spec to production: a three-week drug discovery agent using Kiro (AWS for Industries)](https://aws.amazon.com/blogs/industries/from-spec-to-production-a-three-week-drug-discovery-agent-using-kiro/) — retrieved 2026-05-26

Last verified: 2026-05-26
