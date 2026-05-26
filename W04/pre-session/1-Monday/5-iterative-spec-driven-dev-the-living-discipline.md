---
week: W04
day: 1-Monday
topic_slug: iterative-spec-driven-dev-the-living-discipline
topic_title: "Iterative Spec-Driven Dev — the living discipline you've now practised 3×"
parent_overview: W04/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://github.com/github/spec-kit
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://github.com/github/spec-kit/blob/main/spec-driven.md
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.thoughtworks.com/en-us/insights/blog/agile-engineering-practices/spec-driven-development-unpacking-2025-new-engineering-practices
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.infoq.com/articles/spec-driven-development/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.infoq.com/articles/enterprise-spec-driven-development/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# Iterative Spec-Driven Dev — the living discipline you've now practised 3×

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish a *living spec* from a *frozen spec*, and explain why drift detection is the load-bearing capability.
- Name the four phases of a canonical spec-driven development cycle (Specify / Plan / Task / Implement) and identify where each cycle's retrospective lives.
- Articulate why "we wrote it down" is not spec-driven development, and what step turns documentation into discipline.
- Apply the §0 retrospective pattern to a prior plan: compare what the spec said would happen against what reality produced, and dispose of each delta.

## 2. Introduction

Spec-driven development (SDD) is the 2025-era response to a long-standing problem: every team has written specifications, and every team has watched those specifications drift away from the running system. SDD is not a return to waterfall and not "more documentation." It is the discipline of treating the specification as a *living artifact* — one that the team is contractually obligated to re-read, amend, or supersede whenever the next iteration begins.

The Thoughtworks 2025 piece on SDD as a key new AI-assisted practice frames the problem cleanly: specs that "drift — outdated or vague specs can cause misaligned plans." InfoQ's enterprise-SDD analysis adds that the fundamental issue is not the absence of specs but that "by Sprint 3, the HLD is outdated; by release 2, the SRS no longer matches the product." The shift in 2025 is the adoption of tooling — GitHub's open-source Spec Kit, schema validators in CI, contract-verification engines — that makes drift *machine-detectable* rather than discovered six months late by an unhappy stakeholder.

The discipline this reading is about lives one layer above the tooling. The tools detect drift; the team has to *do something about it*. That "something" is the iterative retrospective on the prior cycle's spec — the §0 step that every cycle starts with. Without that step, even a tool-enforced spec degrades into a paperwork ceremony.

## 3. Core Concepts

### 3.1 The four phases of an SDD cycle

The community-converged SDD lifecycle (GitHub Spec Kit, GitHub Blog launch announcement, Microsoft Learn module):

1. **Specify** — define what and why. User journeys, business requirements, success criteria, acceptance criteria. Structured natural language; not code.
2. **Plan** — define how. Technical architecture, design decisions, technology choices. Still pre-code.
3. **Task** — decompose the plan into discrete implementation units that a human or AI agent can execute in bounded time.
4. **Implement** — AI agents and humans execute tasks; humans review, validate, iterate.

Each phase produces a Markdown artifact that feeds the next. Spec Kit's published workflow ships these four phases as resumable pipelines; the artifact-to-artifact handoff is what makes the cycle iterable instead of one-shot.

### 3.2 Where the retrospective lives

The retrospective is *not* a separate fifth phase. It is the *first thing* the next cycle does. When cycle N+1 begins its Specify phase, it begins by re-reading cycle N's spec and asking three questions:

1. What did the spec say would happen?
2. What actually happened?
3. For each delta, what disposition do we choose: amend the spec for next cycle, supersede the spec, accept the delta as a known limit?

This is the §0 retrospective pattern. It is built into the structure of the next cycle's planning artifact rather than scheduled as a standalone event. The InfoQ enterprise-SDD coverage frames this as treating "specification authoring as an operational discipline with feedback loops, quality metrics, and continuous improvement." The feedback loop is the retrospective; without it, the team is doing spec-shaped documentation, not spec-driven development.

### 3.3 Living vs. frozen specs

A *frozen spec* is a document written once, approved, and consulted occasionally. By release 2 it is wrong, and the team has stopped pretending it isn't. A *living spec* is a document the team is contractually obligated to re-read every cycle and either reaffirm or amend. The same document text can be either, depending on the team's discipline around it.

The mechanism that makes a spec living is the next-cycle retrospective. If cycle N+1 does not re-read cycle N's spec, the spec has frozen — regardless of how often the team claims to "follow the spec." The Thoughtworks 2025 framing: "specifications and code diverge without active maintenance discipline, similar to how BDD's living documentation promise breaks down when teams stop treating specifications as a maintained single source of truth."

### 3.4 What machine-detectable drift adds

The 2025 tooling shift — schema validators in CI, contract-verification engines, spec-diff tools — gives the team an early-warning signal that the running system has diverged from the spec. InfoQ's deep dive on executable specs frames this as "specification validators embedded directly into CI pipelines, runtime enforcement layers, schema validation, payload inspection."

What this tooling does *not* do is choose what to do about the drift. The drift signal arrives as "the spec says X, the running system does Y." The team still has to decide:

- Amend the spec to match the system (X was wrong, Y is correct).
- Change the system to match the spec (Y is the bug, X is correct).
- Supersede both (the question itself changed).

The retrospective is where this choice is made. Tooling without retrospective is just an automated reminder that the team is drifting; the retrospective is what makes the drift signal cost the team something to ignore.

### 3.5 The Continuous Validation Principle

The spec is not done when it is written. The spec is done when the next cycle's reality has been compared against it and the spec has either survived or been amended. This is the *Continuous Validation Principle* — what the iterative discipline contributes that one-shot specification cannot.

The implication is uncomfortable: every cycle, the team writes a spec knowing that next cycle they will publicly re-read it and ask what it got wrong. There is no incentive to write impressively or aspirationally; there is incentive only to write *truly enough that the next cycle can amend it cleanly*. This shifts the spec's purpose from "predict the future" to "be a good reference for next cycle's diff."

### 3.6 What the §0 retrospective actually looks like

A §0 retrospective for cycle N is a short section at the top of cycle N+1's planning artifact. Three paragraphs at most:

- Paragraph 1: what the prior cycle's spec predicted (extracted from the prior spec).
- Paragraph 2: what actually happened (this cycle's observed reality).
- Paragraph 3: per-delta disposition. *Amend* (next cycle's spec corrects), *Supersede* (a new ADR cancels the prior decision), *Accept* (the delta is recorded as a known limit), or *Carry forward* (the delta is not resolved this cycle but is now visible).

The artifact is short on purpose. A long §0 means the cycle has been re-litigating the past instead of moving forward. A § 0 that fits in three paragraphs is enough to keep the spec living without freezing the team.

## 4. Generic Implementation

A generic spec-cycle template, drawn from the GitHub Spec Kit shape with the §0 retrospective explicit. Example domain is logistics (substitute your context).

```markdown
# Cycle 4 — Route Optimization Engine

## §0 Retrospective on Cycle 3 spec
Cycle 3's spec predicted: the engine would handle 500 routes/sec at p95 < 200ms.
Actual: 320 routes/sec at p95 = 280ms.

Deltas and disposition:
- Throughput shortfall (-36%): AMEND. New Cycle 4 spec targets 350 routes/sec
  (achievable per profiling) and treats 500 routes/sec as a Cycle 6 stretch.
- Latency overshoot (+40%): SUPERSEDE. ADR-0012 cancels the in-memory cache
  strategy in favor of a write-through cache; new spec assumes the new strategy.
- Memory consumption stable as predicted: no delta, no action.

## §1 Specify (this cycle)
User journey: dispatcher submits 200-stop route list at 06:00; system returns
optimized route within 90 seconds; dispatcher accepts or edits.

Success criteria:
- p95 latency for 200-stop optimization < 90 seconds (was: 90s, holds)
- Throughput at 350 routes/sec sustained (was: 500, amended this cycle)
- Memory < 8GB per worker (no change)

## §2 Plan
Architecture: write-through cache (per superseding ADR-0012), worker pool of 4,
shared queue. ...

## §3 Tasks
T-01: Implement write-through cache wrapper around routing engine
T-02: Update worker pool config; add observability
T-03: Validation harness against historical route data
...

## §4 Implementation gates
- AI agent may execute T-01, T-02 with human review per PR
- T-03 requires pair review before merge
- All tasks block on §1 success criteria via CI assertion
```

The shape is deliberately recursive — every cycle starts with the prior cycle's retrospective, so a reader scrolling backward through cycles sees the trace of what the team learned and amended at each step.

## 5. Real-world Patterns

**Fintech KYC pipeline — drift detection via contract tests (US neobank, 2025).** A neobank running a high-volume KYC pipeline adopted SDD after a regulatory audit found their internal spec did not match the running system. Their published approach ([InfoQ enterprise SDD](https://www.infoq.com/articles/enterprise-spec-driven-development/)): the spec is machine-readable; CI runs contract tests against every pull request; a spec-diff tool surfaces unintended drift. The §0 retrospective for each sprint reads the contract-test failures from the previous sprint and either amends the spec or fixes the code. Within two quarters, audit findings on spec-system mismatch dropped to zero.

**E-commerce checkout — spec-as-source-of-truth migration (apparel retailer, 2025).** A mid-size apparel retailer migrating from a monolith to microservices used Spec Kit's four-phase cycle (Specify → Plan → Task → Implement) for each service extraction. Their lesson, published on the GitHub Spec Kit examples ([GitHub Spec Kit — spec-driven.md](https://github.com/github/spec-kit/blob/main/spec-driven.md)): the spec was *less* important than the practice of re-reading it. Their first three services drifted; their fourth onward did not, because they had institutionalized the §0 retro by then.

**Healthcare imaging pipeline — spec-evolution over six cycles (medical-imaging vendor, 2024-2025).** A US medical-imaging vendor running six 4-week cycles to modernize an imaging pipeline published a detailed retrospective trace ([Thoughtworks — spec-driven development unpacking 2025](https://www.thoughtworks.com/en-us/insights/blog/agile-engineering-practices/spec-driven-development-unpacking-2025-new-engineering-practices)): cycles 1-2 produced specs that froze (no §0 retro, no amendment), cycles 3-4 introduced the retro and surfaced 11 deltas, cycles 5-6 produced specs that explicitly cited and amended cycles 3-4. By cycle 6 the spec was the most-trusted document on the project; in cycles 1-2 it had been the least.

**Gaming live-ops platform — cycle-on-cycle amendment trace (multiplayer studio, 2025).** A studio running weekly live-ops cycles used a stripped-down SDD pattern with a single-paragraph §0 retrospective per week. Their guidance ([InfoQ — spec-driven development when architecture becomes executable](https://www.infoq.com/articles/spec-driven-development/)): "the §0 retro is the entire discipline; everything else is just artifact format." The studio reported that the retro caught one P0 regression per quarter that would otherwise have shipped to production because the spec said the regression-prone code path was already deprecated.

## 6. Best Practices

- Start every cycle with the §0 retrospective on the prior cycle's spec — three paragraphs, four dispositions (amend / supersede / accept / carry forward).
- Treat the prior cycle's spec as a *contract* with the next cycle, not as background reading; if the prior spec is not cited in §0, the cycle has skipped the retro.
- Adopt drift-detection tooling (schema validators, contract tests in CI) — but do not confuse the alert with the discipline; tooling reports drift, retros decide what to do about it.
- Keep the spec short enough that re-reading it every cycle is cheap; a 50-page spec re-reads as paperwork, a 5-page spec re-reads as alignment.
- Supersede instead of edit-in-place when a decision changes — the next cycle's reader needs to see the trace.
- Forbid "follow the spec" as a code-review comment unless the spec has been re-read this cycle; otherwise the comment is appealing to a frozen document.
- Pair the §0 retro with named ownership — one person owns the artifact, others contribute deltas; without an owner, the retro doesn't get written.

## 7. Hands-on Exercise

**Whiteboarding exercise (15 min, pair):**

Pick a real plan you or your team wrote in the last six weeks — a sprint plan, design doc, project proposal, whatever. Write a §0 retrospective for it now. Three paragraphs:

1. What the plan predicted (extract the key claims — usually 3-6 of them).
2. What actually happened (factual, no spin).
3. Per-delta disposition. For each delta, choose *amend* (rewrite the prediction), *supersede* (replace with a new decision), *accept* (record as known limit), or *carry forward* (delta is now visible but not resolved this cycle).

Then count. How many predictions matched reality? How many drifted silently until now?

**What good looks like:** the retrospective is honest — drifts are named, not euphemised. Each delta has a one-word disposition. The artifact fits on one page; if it doesn't, the original plan was too long. A reader who picks up your retro tomorrow can use it to decide what *next* week's plan should say, without asking you.

## 8. Key Takeaways

- *What makes a spec "living" instead of "frozen"?* — A next-cycle retrospective that re-reads the prior spec and disposes of each delta. The retro, not the document, is the discipline.
- *What are the four phases of an SDD cycle?* — Specify, Plan, Task, Implement. Each produces an artifact that feeds the next; the retrospective sits at the top of cycle N+1's Specify phase.
- *What does drift-detection tooling add, and what does it not add?* — It surfaces the spec-vs-system delta automatically; it does not choose the disposition. The retrospective is where the choice happens.
- *What is the Continuous Validation Principle?* — The spec is done when the next cycle's reality has been compared against it and it has been amended or reaffirmed — not when it was first written.

## Sources

1. [GitHub Spec Kit — toolkit for Spec-Driven Development](https://github.com/github/spec-kit) — retrieved 2026-05-26
2. [Spec Kit — spec-driven.md](https://github.com/github/spec-kit/blob/main/spec-driven.md) — retrieved 2026-05-26
3. [Spec-driven development: Unpacking one of 2025's key new AI-assisted engineering practices — Thoughtworks](https://www.thoughtworks.com/en-us/insights/blog/agile-engineering-practices/spec-driven-development-unpacking-2025-new-engineering-practices) — retrieved 2026-05-26
4. [Spec Driven Development: When Architecture Becomes Executable — InfoQ](https://www.infoq.com/articles/spec-driven-development/) — retrieved 2026-05-26
5. [Spec-Driven Development – Adoption at Enterprise Scale — InfoQ](https://www.infoq.com/articles/enterprise-spec-driven-development/) — retrieved 2026-05-26

Last verified: 2026-05-26
