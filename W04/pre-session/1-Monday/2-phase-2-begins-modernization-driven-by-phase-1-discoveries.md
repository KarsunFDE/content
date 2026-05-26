---
week: W04
day: 1-Monday
topic_slug: phase-2-begins-modernization-driven-by-phase-1-discoveries
topic_title: "Phase 2 begins — modernization driven by Phase-1 discoveries"
parent_overview: W04/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://docs.aws.amazon.com/prescriptive-guidance/latest/strategy-modernizing-applications/phases.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-phased-approach/process.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://learn.microsoft.com/en-us/azure/architecture/patterns/strangler-fig
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.thoughtworks.com/en-us/insights/articles/embracing-strangler-fig-pattern-legacy-modernization-part-one
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.taazaa.com/blog/the-90-day-roadmap-from-legacy-modernization
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Phase 2 begins — modernization driven by Phase-1 discoveries

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the three artifacts a discovery phase must produce before an execution phase starts, and explain why "incomplete discovery + complete execution-plan" is the most common failure mode.
- Distinguish a *Phase 1 finding* from a *Phase 2 scope item* — and articulate why every Phase 2 scope item must trace back to a Phase 1 finding.
- Apply the strangler-fig principle to a phase boundary: what does Phase 1 keep alive, what does Phase 2 route around, and what proves the routing landed cleanly?
- Explain why scope discipline at a phase boundary is more valuable than ambition, and what the cost of skipping the retrospective looks like at week 4 of a 6-week effort.

## 2. Introduction

Most modernization programmes do not fail because their target architecture is wrong. They fail at the seam where one phase hands off to the next. A team finishes its discovery work, the engagement clock keeps ticking, and the next phase begins from a generic "modernize the platform" mandate rather than from the specific findings the discovery phase produced. By the time someone notices, the new phase has spent two weeks on work that the discovery phase had already concluded was lower priority.

This pattern is well-documented. AWS Prescriptive Guidance frames application modernization as an iterative Assess-Modernize-Manage loop where the Assess phase output — a prioritised backlog with complexity scores and modernize/lift-and-shift/retire dispositions — is the *only* legitimate input to the Modernize phase. The same shape appears in IBM's reimagined brownfield approach and Thoughtworks' strangler-fig framing: discovery findings are not a one-time artifact, they are the trace evidence that justifies every later decision. When that trace breaks, the modernization stops being driven by reality and starts being driven by whatever the team finds easiest to talk about on a Monday morning.

The interlock between phases is therefore a *deliberate ceremony*, not a transition. It looks backward at what the prior phase produced, forward at what the next phase will commit to, and explicitly draws the lines between findings and scope items. Skipping it is cheap on the day; expensive at the end.

## 3. Core Concepts

### 3.1 The discovery output is a contract, not a report

Every credible modernization framework draws the same boundary: discovery ends when it has produced (a) an inventory of components with current-state facts, (b) a prioritised backlog of candidates with disposition (modernize / lift-and-shift / retire / defer), and (c) a stated set of unknowns the next phase must close. AWS Prescriptive Guidance is explicit that the Assess output is a prioritised backlog with complexity scores, each item carrying a rationale. IBM's brownfield modernization guidance treats the discovery deliverable as a *signed contract* between the discovery and execution phases — a later phase cannot retroactively add scope items the discovery phase did not surface.

What this means in practice: when execution begins, the team should be able to point to any work item and answer "which discovery finding justifies this?" If no finding exists, the work item is on the wrong list.

### 3.2 Findings vs. scope items

A *finding* is a fact about the current state: "the lint job in CI is conditionally disabled" or "service A authenticates inbound calls but service B trusts the header without verification." A *scope item* is a commitment to do something about a finding within a bounded timeline: "we will enable the lint job and burn down the lint debt over Thursday afternoon" or "we will add header verification to service B and add a contract test."

The translation from finding to scope item is the load-bearing judgement of a phase boundary. It is not 1:1 — most findings will not become scope items in the next phase. Some are deferred with rationale ("the secrets-in-config finding is real but will be handled by W5 OWASP exercises"), some are cut ("the monorepo-vs-polyrepo finding is interesting but outside this engagement's mandate"), some are accepted as living debt ("the test coverage gap is documented; we are not closing it"). The phase-boundary artifact records all four dispositions: *committed*, *deferred-with-rationale*, *cut-with-rationale*, and *accepted-as-debt*.

### 3.3 The strangler-fig principle applied to a phase boundary

The strangler fig pattern is usually taught as a deployment strategy — route a slice of traffic to new code while old code keeps serving the rest. The same shape applies to a phase boundary. Phase 1 *keeps alive* the components the team has now characterised; Phase 2 *routes around* a small, named subset; the remaining components stay in Phase 1's running state. Each phase boundary expands the set of routed-around components incrementally.

Microsoft's Azure Architecture Center frames this as "boundary selection is one of the most consequential decisions in modernization planning." A phase boundary that picks too much fails to deliver any one slice well; a phase boundary that picks too little produces ceremony without progress. The discipline is to pick a slice the team can land in the phase's time budget *with proof* — not the slice that would be most impressive if it landed.

### 3.4 The retrospective as interlock

A modernization retrospective at a phase boundary is not a meeting; it is a written artifact that does three things in order:

1. **Re-reads the prior phase's spec** — what did we say we would learn? What did we say we would produce?
2. **Names the deltas** — where did the spec under-predict? Where did it over-predict? What did we discover that the spec did not name?
3. **Commits the deltas to scope** — for each delta, choose committed / deferred / cut / accepted, with rationale.

The retrospective output is the *only legitimate input* to the next phase's planning. If a next-phase scope item does not trace back to a finding named in the retrospective, that scope item is suspect.

### 3.5 The cost of skipping

The cost of a missed phase-boundary retrospective is rarely visible the week it is skipped. It surfaces two weeks later, when the team is mid-execution on work that does not solve a problem anyone documented. The discovery phase's findings drift into oblivion; the execution phase's scope drifts into "what felt natural to start with." Modernization Intel's 2022-2025 analysis of strangler-fig projects found a 76% success rate when phase-boundary discipline held — and the dominant failure mode in the other 24% was discovery findings that never made it into execution scope.

## 4. Generic Implementation

A phase-boundary artifact for any modernization programme can fit on a single page. Below is a generic template — replace the domain terms with your context.

```markdown
# Phase N → Phase N+1 Interlock — <YYYY-MM-DD>

## §0 Retro on Phase N's spec
Spec said we would: <bullets from the prior phase's stated goals>
Spec under-predicted: <bullets — what was harder/more than expected>
Spec over-predicted: <bullets — what was easier/less than expected>
Discovery the spec did not name: <bullets — emergent findings>

## §1 Findings inventory (Phase N output)
| ID | Finding | Source artifact | Severity |
|----|---------|-----------------|----------|
| F-01 | <one-line current-state fact> | <where this came from> | <H/M/L> |
| F-02 | ... | ... | ... |

## §2 Phase N+1 scope commitments
| Scope item | Traces to | Why now | Rollback |
|------------|-----------|---------|----------|
| S-01 | F-01, F-03 | <why this phase, not later> | <if it breaks, we...> |

## §3 Deferred-with-rationale
| Finding | Why deferred | When revisited |

## §4 Cut-with-rationale
| Finding | Why cut |

## §5 Accepted as living debt
| Finding | Why accepted | Owner |
```

The shape is deliberately boring. Its value is that every Phase N+1 scope item has a column traceable to a Phase N finding. If that column is empty for any row, the row should not be in §2.

## 5. Real-world Patterns

**E-commerce platform replatforming (Shopify-to-headless migration, retail, mid-2024).** A specialty retailer running a monolithic Shopify Plus storefront chose a phased migration to a headless commerce stack. Their discovery phase ran 8 weeks and produced a 47-item findings inventory. The interlock retrospective cut 31 items as deferred-or-out-of-scope and committed 16 to execution Phase 1. The cuts list — and *the rationale on each cut* — was what kept the next 14 weeks bounded. Modernization Intel's case-study aggregation flagged this as the dominant pattern in successful retail replatformings: the cuts are more load-bearing than the commits ([Thoughtworks — strangler-fig part one](https://www.thoughtworks.com/en-us/insights/articles/embracing-strangler-fig-pattern-legacy-modernization-part-one)).

**Healthcare records platform — phased FHIR migration (US health system, 2025).** A regional hospital network migrating from a proprietary EHR data model to FHIR R4 produced phase-boundary retrospectives every 4 weeks across a 36-week programme. The retrospectives caught a recurring pattern: each phase's discovery work was surfacing 2-3 findings about *the prior phase's modernization choices* that needed remediation. Treating those as a separate "phase-N-remediation" backlog (rather than re-opening phase N) kept the forward motion intact while making the rework visible ([AWS Prescriptive Guidance — Modernize phase](https://docs.aws.amazon.com/prescriptive-guidance/latest/strategy-modernizing-applications/phases.html)).

**Fintech payments platform — Java 8 to 17 hop (2025).** A payments processor running Spring Boot 1.5 on Java 8 used a strangler-fig phase structure to migrate. Phase 1 (discovery, 6 weeks) produced 23 findings, ranking deprecated-API usage and security-CVE exposure as the top categories. Phase 2 (execution, 10 weeks) committed only 9 items — but each had a named Phase 1 finding and a named rollback path. The team's published retrospective ([Taazaa — 90-day modernization roadmap](https://www.taazaa.com/blog/the-90-day-roadmap-from-legacy-modernization)) attributed the on-time landing to the *small* commit list rather than to engineering velocity.

**Logistics carrier — warehouse-management system rewrite (2024-2025).** A US carrier modernizing a 20-year-old WMS used phase boundaries at 6-week intervals. Their published lesson: the temptation at every phase boundary is to add scope ("while we're in here, we could also..."); the discipline is to refuse. Every "while we're in here" addition cost an average of 2.3× its initial estimate. Their final retrospective ([Microsoft Learn — Strangler Fig pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/strangler-fig)) framed the phase boundary as "the only place where the team is allowed to say no."

## 6. Best Practices

- Treat the prior phase's findings inventory as the *only* legitimate input to the next phase's scope — if a scope item has no traced finding, cut it before the phase starts.
- Write the retrospective as an artifact before the next phase's planning meeting, not during it; planning runs against the retro, not alongside it.
- Make the cuts and deferrals as visible as the commits — readers should be able to see what *did not* make it into the phase and why.
- Name a rollback for every scope item before execution starts; "we ran out of time" is not a rollback.
- Time-box the retrospective itself — for a 4-6 week phase, a retro that takes more than half a day is a sign the prior phase's findings were not documented clearly enough to read.
- Keep the phase boundary symmetric: every commit comes with a deferral or a cut. A phase that commits to everything has not been planned, it has been wished for.

## 7. Hands-on Exercise

**Architecture-drawing prompt (15 min, pair):**

Take a programme you have worked on (current or prior). Pick one phase boundary — a real point where one stage of work handed off to the next. On a single page, write:

1. The three things the prior phase's spec said it would learn or produce.
2. The two most important things the prior phase actually surfaced that the spec did not name.
3. For each of those two, a one-sentence disposition: *committed to Phase N+1*, *deferred to Phase N+2 with revisit date*, *cut with rationale*, or *accepted as living debt with owner*.
4. One scope item the next phase committed to that, in hindsight, had no clear trace back to a finding — and what it cost.

**What good looks like:** the page reads as a contract, not as a narrative. A reader who joined the team a week later can use it to answer "why are we doing this and not that?" without asking anyone. The Phase 1-to-Phase 2 trace is explicit on every committed scope item; the cut and deferral lists are at least as long as the commit list; rollbacks are named.

## 8. Key Takeaways

- *What is the only legitimate input to the next phase's scope?* — The prior phase's findings inventory plus the interlock retrospective that disposes of each finding.
- *What is the difference between a finding and a scope item?* — A finding is a current-state fact; a scope item is a bounded commitment to do something about a finding. Most findings do not become scope items.
- *What does the strangler-fig principle say about a phase boundary?* — Route around a small, named subset of components with proof; keep the rest in the prior phase's running state.
- *What is the most common cause of phase-boundary failure?* — Discovery findings that never make it into execution scope, and execution scope items that have no traced finding.

## Sources

1. [Phases of the modernization process — AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/strategy-modernizing-applications/phases.html) — retrieved 2026-05-26
2. [Modernization process — AWS Prescriptive Guidance phased approach](https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-phased-approach/process.html) — retrieved 2026-05-26
3. [Strangler Fig pattern — Microsoft Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/strangler-fig) — retrieved 2026-05-26
4. [Embracing the Strangler Fig pattern for legacy modernization (part one) — Thoughtworks](https://www.thoughtworks.com/en-us/insights/articles/embracing-strangler-fig-pattern-legacy-modernization-part-one) — retrieved 2026-05-26
5. [The 90-Day Roadmap from Legacy Modernization — Taazaa](https://www.taazaa.com/blog/the-90-day-roadmap-from-legacy-modernization) — retrieved 2026-05-26

Last verified: 2026-05-26
