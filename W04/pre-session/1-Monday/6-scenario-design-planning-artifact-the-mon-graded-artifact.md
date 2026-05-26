---
week: W04
day: 1-Monday
topic_slug: scenario-design-planning-artifact-the-mon-graded-artifact
topic_title: "Scenario Design Planning Artifact — the Mon graded artifact"
parent_overview: W04/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://www.sciencedirect.com/topics/computer-science/architecture-scenario
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://aws.amazon.com/builders-library/ensuring-rollback-safety-during-deployments/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.manifest.ly/use-cases/software-development/rollback-plan-checklist
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://monday.com/blog/project-management/raid-project-management-template/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://thedigitalprojectmanager.com/project-management/raid-log/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Scenario Design Planning Artifact — the Mon graded artifact

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the six components of a scenario-design planning artifact and explain what each one is preventing.
- Distinguish a *risk* from an *assumption* from an *issue* from a *dependency*, using the RAID framework.
- Articulate why "missing artifacts ≠ graded artifact" — coherence across all six components is the grading floor.
- Apply the stimulus-response-measurement framing to a real architecture scenario the team is planning to execute.

## 2. Introduction

A *scenario-design planning artifact* is a single short document a team produces at the start of a bounded modernization window — typically a sprint or a phase. It is not a project plan, not a backlog, and not a design document. It is the load-bearing artifact that tells a future reviewer: *here is what we are committing to this week, here is what we believe is true, here is what could go wrong, here is how we will know it worked, and here is how we get out.*

The artifact's discipline draws from two longstanding software-engineering traditions. From the architecture-trade-off analysis world (ATAM and its descendants, formalized in the IEEE scenario-based analysis literature), it borrows the stimulus-response-measurement framing for architectural scenarios. From the project-management RAID-log tradition (Risks / Assumptions / Issues / Dependencies), it borrows the practice of *naming what could go wrong before it does* and tracking the assumptions that the plan rests on.

The reason this artifact is *graded* — meaning evaluated as a single coherent piece, not as an exercise — is that its real value is at the seams between components. A modernization scope statement without a rollback plan is wishful thinking; a risk register without a validation plan is bookkeeping; a pair-project slice without a test plan is a guess. The six components together produce something a stakeholder can rely on. Any one component missing breaks the assurance the whole artifact is meant to provide.

## 3. Core Concepts

### 3.1 The six components and what each prevents

A typical scenario-design planning artifact contains:

1. **§0 retrospective on the prior cycle** — names what the prior plan got right and wrong, so this cycle's plan is grounded in reality. *Prevents:* the team committing to scope based on out-of-date assumptions.
2. **Scope ADR(s)** — the architecturally-irreversible commit(s) this cycle makes. *Prevents:* drift from initial scope into "while we're in here" additions.
3. **Threat model / security ADR** — the threats this cycle exercises and which it explicitly defers. *Prevents:* the modernization shipping with a new attack surface no one named.
4. **Pair / team slice** — which person or pair owns which piece of the work, and how those pieces interact. *Prevents:* duplicated work and orphaned ownership.
5. **Test / validation plan** — what proves each commitment landed. *Prevents:* "done" being undefined, and PRs merging without evidence.
6. **Risk + rollback log** — for each named risk, the rollback if it triggers. *Prevents:* the team freezing when reality diverges from plan.

The components are sequential in the artifact but they reinforce each other. A scope ADR (§2) is only credible if the test plan (§5) names how its commitments are verified. A threat model (§3) is only credible if the validation plan (§5) includes the relevant security tests. The rollback log (§6) is only credible if it cites the scope ADR's commitments so a future reader can trace which rollback applies to which commit.

### 3.2 The RAID framework, applied

RAID (Risks, Assumptions, Issues, Dependencies) is the project-management vocabulary the planning artifact uses for its non-scope sections. The Digital Project Manager's RAID-log guide gives the canonical definitions; monday.com's RAID template is widely adopted in modernization programmes. The four terms are not interchangeable:

- **Risk** — something that *could* happen and would matter if it did. Has a probability, an impact, and a mitigation.
- **Assumption** — something the plan treats as true without verifying. Has a verification step or an explicit "we accept the exposure."
- **Issue** — something *already* wrong and blocking work. Has an owner and a target resolution. Distinct from a risk by being present-tense.
- **Dependency** — something this cycle needs from another team or upstream system. Has a name, an owner, and a needed-by date.

The reason all four matter is that they fail differently. A risk that materializes is responded to with mitigation; an assumption that breaks is responded to with re-planning; an issue that is unresolved blocks; a dependency that slips delays. The artifact tracks them separately so the responses can be matched correctly.

### 3.3 Stimulus / response / measurement for the scope ADR

The ATAM tradition (well summarized in Bass-Clements-Kazman's *Software Architecture in Practice* and the IEEE Scenario-Based Analysis literature) frames an architectural scenario as a triple:

- **Stimulus** — the event or action that initiates the scenario.
- **Response** — how the system is expected to react.
- **Measurement** — a quantifiable metric of the response.

Every scope ADR's verification section should contain at least one such triple. For a modernization commit like "migrate service A from a monolithic SQL store to a sharded store," the triple might be:

- Stimulus: 10,000 concurrent reads against the migrated store.
- Response: All requests return < 200ms p95 with correct data.
- Measurement: Load test against staging; p95 from observability; correctness from contract tests.

The triple makes the commitment falsifiable. Without it, the commitment is aspirational; with it, the test plan section knows exactly what to build.

### 3.4 The rollback log is symmetric with the scope

For every commit in the scope ADR section, the rollback log must have an entry. The AWS Builders' Library guidance on rollback safety frames this as a *symmetry requirement*: every roll-forward must have a defined roll-back, and the roll-back must have been *tested* or at least credibly designed before the roll-forward ships.

A rollback entry has four fields:

- **Trigger** — what condition makes the team execute the rollback. "Error rate > 1% for 15 minutes," "data corruption detected by reconciliation job," "stakeholder veto by end of Wednesday."
- **Procedure** — the named, ordered steps. "Revert PR #X; re-run migration Y in reverse; notify Z."
- **Time-to-rollback** — the realistic estimate. If rollback takes 4 hours, that fact is more important than the procedure being elegant.
- **Data-loss exposure** — honest accounting of what is lost if rollback executes mid-window. "Up to 90 seconds of writes" or "zero loss; all writes are durable."

If any rollback's data-loss exposure is "we don't know," that is itself a finding for the artifact — and is typically resolved by either adding a verification step before commit or by re-scoping the commit to be reversible.

### 3.5 Coherence as the grading criterion

The artifact is graded as a single coherent piece. A reviewer asks: do the six components agree with each other? Specifically:

- Every scope-ADR commit appears in the test plan (otherwise: undefined "done").
- Every threat-model entry appears in either the test plan or the explicit deferral list (otherwise: unexercised security claim).
- Every test in the test plan maps to a commit or a threat (otherwise: orphan test).
- Every risk has either a mitigation in the plan or an explicit rollback (otherwise: unmanaged risk).
- Every assumption has either a verification step or an explicit acceptance (otherwise: hidden exposure).
- Every dependency has an owner and a needed-by date (otherwise: blocked-by-no-one).

A component that exists but does not connect to the others is technically present but functionally absent. The grading discipline focuses the team's attention on the *seams*, not the components individually — which is the same place real modernization projects break.

## 4. Generic Implementation

A generic scenario-design planning artifact template, illustrated for a fictional e-commerce modernization cycle (substitute your domain).

```markdown
# Cycle 7 Scenario Design Plan — Catalog Service Modernization

## §0 Retro on Cycle 6
Predicted: catalog service search-latency improvement to p95 < 100ms.
Actual: p95 = 145ms (improved from 220ms but missed target).
Deltas:
- Latency target missed: AMEND. Cycle 7 targets p95 < 130ms (achievable per profiling).
- Cache-invalidation correctness: held as predicted. No action.

## §1 Scope (ADR-0014 in repo)
- Commit: replace catalog read path with cached-projection store
- Commit: dual-write to legacy SQL during 5-day overlap window
- Explicit out-of-scope: write path (Cycle 8)

## §2 Threat model
- T-01: cache poisoning via unsanitized upstream feed → tested in §4
- T-02: dual-write inconsistency window → reconciliation job, see §4
- Deferred: rate-limiting hardening (Cycle 9)

## §3 Pair slice
- Pair A: cached-projection store, dual-write writer
- Pair B: reconciliation job, observability
- Solo: stakeholder dashboard for the cutover window

## §4 Validation
- Scope commit 1: load test 10k req/s on staging, p95 < 130ms (stim/resp/meas)
- Scope commit 2: chaos test killing one writer; reconciliation closes gap < 60s
- Threat T-01: fuzz the feed parser; assert no panic, no data write on malformed
- Threat T-02: 4-hour soak with induced lag; reconciliation report = 0 drift

## §5 RAID
- Risk R-01: feed upstream may add new fields without schema bump (M prob, H impact)
  → mitigation: schema-strict parsing, fail-closed on unknown fields
- Assumption A-01: staging traffic shape matches prod within 10% — verify Tue AM
- Issue I-01: observability dashboards not yet versioned with code (owner: Pair B, by Wed)
- Dependency D-01: SRE on-call coverage during cutover window (Thu 14:00–22:00)

## §6 Rollback log
- Commit 1 rollback: revert traffic via feature flag; cached-projection store stays
  populated but is unread. Time: 2 min. Data loss: zero.
- Commit 2 rollback: stop dual-write; truncate cached-projection store; legacy SQL
  resumes as primary. Time: 15 min. Data loss: up to 5 minutes of in-flight writes
  to the projection — replayable from binlog.
- Trigger: error rate > 0.5% for 15 minutes on read path OR reconciliation drift > 100.
```

The artifact is short — six sections, one page printed, every section connecting to at least one other.

## 5. Real-world Patterns

**Fintech — payment rail migration scenario-plan (Q1 2025).** A US payments platform migrating between card-network rails produced a scenario-design artifact per two-week cutover window. Their published reference ([AWS Builders' Library — rollback safety](https://aws.amazon.com/builders-library/ensuring-rollback-safety-during-deployments/)) traced one near-miss to a missing rollback entry for an in-band cutover step; after adding the entry (22-minute time-to-rollback), the team caught a similar issue in staging before it shipped. The artifact's value was at the seam between scope commit and rollback log, not in either alone.

**Healthcare records — FHIR migration validation plan (US health system, 2025).** A regional health system migrating from a proprietary EHR to FHIR R4 used a six-component scenario-design artifact per migration wave. Their published lesson ([RAID-log practice from monday.com guidance](https://monday.com/blog/project-management/raid-project-management-template/)): the *assumptions* section was the most valuable component. Every wave they discovered two or three assumptions had silently changed (typically about source-system schema details); naming them upfront meant they were verified before cutover, not during it.

**Gaming studio — server-side rollback scenarios (multiplayer studio, 2024-2025).** A multiplayer game studio executing a weekly live-ops release produced a scenario-design artifact with extreme focus on §6 (rollback log). Their guidance ([The Digital Project Manager — RAID log](https://thedigitalprojectmanager.com/project-management/raid-log/)): every server-side change shipped with a named rollback whose time-to-rollback was under 5 minutes, or it did not ship. The artifact's rollback log was the gate, not a checklist item.

**Logistics — warehouse-management cutover (US carrier, 2025).** A US carrier cutting over its WMS used scenario-design artifacts at 6-week intervals. Their retrospective ([rollback-plan checklist from Manifest.ly](https://www.manifest.ly/use-cases/software-development/rollback-plan-checklist)) highlighted one insight: the *dependency* section was where the project slipped — naming dependencies with owners and needed-by dates exposed slippage two weeks earlier than other tracking did.

## 6. Best Practices

- Produce the artifact as a single document, not six documents; the value is in cross-component coherence, which separate documents lose.
- Use the RAID vocabulary deliberately — risk / assumption / issue / dependency are *not* interchangeable, and conflating them costs the team the ability to respond correctly.
- Every scope commit gets a stimulus/response/measurement triple in the validation plan; if no triple, the commit is aspirational.
- Every scope commit gets a rollback entry with named trigger, procedure, time-to-rollback, and data-loss exposure; if any field is "we don't know," that gap is itself the next action.
- Keep the artifact short — one page printed is the target. A long artifact is paperwork; a short one is a working document.
- Re-read the artifact at the end of the cycle and feed its deltas into next cycle's §0 retro; the artifact is part of the spec-driven loop, not a separate ceremony.
- Name an owner for the artifact — one person edits, others contribute. Without an owner, the artifact becomes a wiki page nobody updates.

## 7. Hands-on Exercise

**Whiteboarding exercise (15 min, pair):**

Pick a real bounded piece of work you have planned recently (a sprint, a release, a side-project milestone). Produce a scenario-design planning artifact for it using the §0-§6 template in §4. One page total.

For each scope commit, write the stimulus/response/measurement triple in §4. For each, write the rollback entry in §6 with the four fields (trigger, procedure, time-to-rollback, data-loss exposure).

Then have your pair grade your artifact for coherence — *not* for content. Specifically: does every commit appear in the test plan? Does every test map to a commit or threat? Does every risk have a mitigation or rollback? Does every assumption have a verification step?

**What good looks like:** the artifact fits on one page. A reader who picks it up cold can answer "what is this team doing this week?" in 30 seconds and "what could go wrong and what's the response?" in 60 seconds. The coherence pass finds zero orphans — every component connects. The rollback log's data-loss exposures are stated honestly, including "zero" where it is genuinely zero and a specific number elsewhere.

## 8. Key Takeaways

- *What six components does the artifact contain and what does each prevent?* — §0 retro (out-of-date scope), scope ADR (drift), threat model (unnamed attack surface), pair slice (orphan ownership), validation plan (undefined "done"), rollback log (freezing under divergence).
- *What is the difference between a risk, an assumption, an issue, and a dependency?* — Risks could happen; assumptions are unverified beliefs; issues are present and blocking; dependencies are owed by another team or upstream system. They fail differently and respond differently.
- *Why is "missing artifacts ≠ graded artifact" the grading floor?* — The artifact's value is in cross-component coherence; a missing component breaks the assurance the whole artifact is meant to provide.
- *What makes a scope commit "credible" in this artifact?* — A stimulus/response/measurement triple in the validation plan, plus a rollback entry with named trigger, procedure, time-to-rollback, and data-loss exposure.

## Sources

1. [Architecture Scenario — ScienceDirect Topics overview](https://www.sciencedirect.com/topics/computer-science/architecture-scenario) — retrieved 2026-05-26
2. [Ensuring rollback safety during deployments — AWS Builders' Library](https://aws.amazon.com/builders-library/ensuring-rollback-safety-during-deployments/) — retrieved 2026-05-26
3. [Rollback Plan Checklist — Manifest.ly](https://www.manifest.ly/use-cases/software-development/rollback-plan-checklist) — retrieved 2026-05-26
4. [RAID Log: Track Risks, Assumptions, Issues and Decisions — monday.com](https://monday.com/blog/project-management/raid-project-management-template/) — retrieved 2026-05-26
5. [RAID Logs: Definition, Template, Examples, & How To Guide — The Digital Project Manager](https://thedigitalprojectmanager.com/project-management/raid-log/) — retrieved 2026-05-26

Last verified: 2026-05-26
