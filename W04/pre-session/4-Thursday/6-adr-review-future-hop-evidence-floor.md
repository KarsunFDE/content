---
week: W04
day: Thu
topic_slug: adr-review-future-hop-evidence-floor
topic_title: "ADR Review — future-hop ADRs with an evidence floor"
parent_overview: W04/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://adr.github.io/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://github.com/joelparkerhenderson/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/articles/scaling-architecture-conversationally.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.5-Release-Notes
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://spring.io/blog/2025/11/20/spring-boot-4-0-0-available-now
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# ADR Review — future-hop ADRs with an evidence floor

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define an Architectural Decision Record (ADR) and explain why it captures *decisions* rather than *designs*.
- Distinguish a normal ADR (decision being made now) from a *future-hop ADR* (decision being evaluated against a future state) and name the additional evidence the latter requires.
- Apply a five-point evidence floor when reviewing a future-hop ADR (dry-run output, captured patch, compile/build evidence, identified hand edits, cited primary source).
- Critique an ADR draft against the floor and identify when "looks straightforward" is the wrong conclusion.
- Distinguish *paper exercises* from *evidence-backed proposals* and explain why the latter is the bar for any ADR that will inform a future migration.

## 2. Introduction

Architectural Decision Records (ADRs) are short documents that capture *one* architecturally significant decision and the reasoning behind it. The ADR community defines them simply: "an ADR captures a single AD [architectural decision] and its rationale" ([adr.github.io](https://adr.github.io/), retrieved 2026-05-26). The collection of ADRs over a project's lifetime is its decision log — the answer to "why did we end up here?" that a future engineer can read instead of guessing.

Most ADRs you write are about decisions being made *now*: choosing a database, adopting an authentication scheme, deciding to split a service. A separate genre — the focus of this reading — is the *future-hop ADR*: a decision being evaluated against a state the system has not yet reached. "If we continued our framework upgrade to the next major in six months, what would change? What would we recommend?"

Future-hop ADRs are useful because they protect downstream decisions. The team that finishes a Spring Boot 2.7 → 3.0 hop today and writes a future-hop ADR for the 3.0 → 3.5 transition has done the work to make next quarter's planning concrete. But future-hop ADRs are *also* the easiest ADR genre to fake. The release notes are public, the migration guides are linkable, and a draft can read confidently without any evidence the author has actually exercised the hop. The discipline this reading proposes is the *evidence floor* — five concrete artifacts a future-hop ADR must cite, in addition to the prose.

## 3. Core Concepts

### 3.1 What an ADR is — and is not

An ADR is a short document — usually one to three pages — that captures: the decision being made, the context that forced the decision, the alternatives considered, the consequences of the choice, and the status (proposed / accepted / superseded). The canonical templates (Nygard's original, MADR, Henderson's collection) all converge on these elements ([architecture-decision-record GitHub collection](https://github.com/joelparkerhenderson/architecture-decision-record), retrieved 2026-05-26).

ADRs are *not* design documents. A design document describes how a system works; an ADR describes why a system works that way. The two complement each other; conflating them produces 50-page documents that nobody reads.

### 3.2 Future-hop ADRs vs ordinary ADRs

A future-hop ADR has all the elements of an ordinary ADR plus one more: it is evaluating a *target state* the project has not yet reached. The context section reads "we are on framework version X; version Y is available and version Z is announced." The decision section reads "we recommend hopping to Y next quarter, deferring Z until [criterion]."

The risk is that future-hop ADRs degrade into reading the release notes. A "we read the docs, looks fine" ADR is not a future-hop ADR — it is a wishlist. The evidence floor is what distinguishes the two.

### 3.3 The five-point evidence floor

A future-hop ADR meets the floor when it cites all five of the following:

1. **Dry-run output.** A real codemod or migration tool was run against a representative module at the project's current state. The output is captured (a patch file, a recipe report, a JSON manifest). The ADR links to it.
2. **Captured patch or recipe report.** The output above is committed to the repository (or to a wiki page reachable from the ADR). Future readers can inspect what the tool actually proposed, not just what the author claims it proposed.
3. **Compile or build evidence.** At minimum, one representative module compiles against the proposed target. If the tooling does not yet support the hop, the ADR captures the *blockers* observed (specific errors, missing recipes).
4. **At least one identified hand edit.** The ADR names at least one residual that the mechanical sweep does not handle. If the ADR claims the hop is fully mechanical, it is either wrong or trivially small.
5. **Cited primary source.** A primary source — official release notes, the framework's migration guide, the codemod's recipe definition — is linked with a retrieval date. Secondary blog posts are allowed in addition; they do not substitute.

An ADR that misses any of the five is not yet a future-hop ADR; it is a draft.

### 3.4 Why "looks straightforward" is the failure mode

When an author writes "the upgrade looks straightforward based on the release notes," what they usually mean is "I read the release notes and did not run anything." The release notes are written by the framework maintainers to be reassuring. They are not a substitute for the specific evidence of *your* codebase against the new version.

This is the single most common failure mode of future-hop ADRs, and it is also the easiest to catch in review: a reviewer who asks "what did the dry run actually produce?" will surface the missing evidence within one round of review.

### 3.5 ADR review as a conversation

Andrew Harmel-Law's Thoughtworks article on scaling architecture argues that ADRs work best inside a conversation-driven practice — an Advice Process, where the author consults the people the decision affects and the people with relevant expertise before publishing ([Scaling the Practice of Architecture, Conversationally](https://martinfowler.com/articles/scaling-architecture-conversationally.html), 15 Dec 2021, retrieved 2026-05-26). The evidence floor fits into this practice naturally: it is what the author brings to the conversation, not the conversation's substitute.

### 3.6 The two-future-hop pattern

When a team finishes a framework hop, they often write *two* future-hop ADRs: one for the next minor (a low-risk continuation) and one for the next major (a high-risk jump). Both ADRs cite the same evidence floor; what differs is the recommendation. The pattern is useful because it forces the team to confront the trade-off in writing: a smaller incremental hop with shorter runway, or a bigger jump with longer runway.

For example, a project that has just landed at Spring Boot 3.0 might write:

- "Future-hop ADR — 3.0 to 3.5": shorter runway because Spring Boot 3.5's OSS-support window has a known end date in the near term ([Spring Boot 3.5 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.5-Release-Notes), retrieved 2026-05-26).
- "Future-hop ADR — 3.0 to 4.0": longer runway since Spring Boot 4.0 GA'd in November 2025 and represents a new generation built on Spring Framework 7 with Java 25 first-class support ([Spring Boot 4.0.0 available now](https://spring.io/blog/2025/11/20/spring-boot-4-0-0-available-now), retrieved 2026-05-26).

The recommendation may be either path; what matters is that both ADRs cite the dry-run evidence for *their* respective hops, not just the release notes.

> [!instructor-review]
> The official Spring Boot support matrix lists 3.5.x OSS support windows that the curriculum should re-verify against the live `spring.io/projects/spring-boot/support` page in the week immediately before the cohort writes its future-hop ADRs. If the 3.5.x OSS-support window has changed since `last_verified` (2026-05-26), the trade-off between 3.5 and 4.0 in the curriculum example may shift — surface to instructor for current-week verification.

## 4. Generic Implementation

The template below is a future-hop ADR skeleton with the evidence floor encoded as required sections. The names of the framework and the hop are intentionally generic.

```
# Future-hop ADR: <framework> <current-version> → <target-version>

## Status
Proposed | Accepted | Superseded by ADR-NNN

## Context
- Current state: we are on <framework> <current-version> as of <date>.
- Target state: <target-version>, released <date>, OSS support window <window>.
- Trigger: <why this ADR exists now — calendar pressure, security, feature need>

## Decision
We recommend <hop now / defer until X / skip to a later version>.

## Evidence (REQUIRED — the floor)

### 1. Dry-run output
- Tool: <e.g., OpenRewrite UpgradeFooBar_X_Y, or custom codemod>
- Module: <which module the dry-run was applied to>
- Command: <exact command>
- Output: see migration-evidence/<dir>/dry-run.patch (link)

### 2. Captured patch / recipe report
- File: migration-evidence/<dir>/recipe-report.json (or .patch)
- Size: <N> files touched, <M> total line changes
- Inspection notes: <one paragraph on what stood out>

### 3. Compile / build evidence
- Module: <name>
- Command: <e.g., mvn compile -pl <module>>
- Result: GREEN | RED | NOT-RUN (with reason)
- Captured log: migration-evidence/<dir>/build.log

### 4. Hand-edit residual
- At least ONE concrete hand edit the recipe does not handle.
- Why the recipe cannot handle it (intent inference, transitive dep, etc.).
- Estimated effort.

### 5. Cited primary source
- <Title> (URL) — retrieved <date>
- Section quoted or paraphrased: <section heading>

## Alternatives considered
- Defer (do nothing for one more quarter): <consequence>
- Skip to next major (jump two versions): <consequence>
- Stay on current (commercial support contract): <consequence>

## Consequences
- Required follow-up work after the hop lands.
- New constraints the hop introduces (e.g., new minimum runtime).
- Decisions this ADR enables (e.g., dependency upgrades blocked on this hop).
```

The template is deliberately heavy in the Evidence section. A future-hop ADR that has not filled in all five sub-sections has not yet earned the "Proposed" status — it is a Draft.

## 5. Real-world Patterns

**E-commerce platform — Node major upgrade ADR.** A retail engineering team published their internal future-hop ADR pattern for Node.js LTS upgrades. The ADR includes a `npm pack --dry-run` of dependency updates per service, a `node --use-strict --check` pass against the new runtime, and a list of breaking changes that touch the codebase. ADRs without the dry-pack output are returned to draft in code review ([architecture-decision-record GitHub collection](https://github.com/joelparkerhenderson/architecture-decision-record), retrieved 2026-05-26 — the convention is documented in the Henderson collection's case studies).

**Healthcare platform — FHIR resource upgrade ADR.** A US hospital network described its FHIR R4 → R5 future-hop ADRs as carrying a "structural diff" generated by a FHIR validator across a representative resource bundle. The ADR cites the validator output and lists at least one resource where intent-level reinterpretation is required (typically `Observation.component` mapping). Pure-paper ADRs are rejected ([Patterns of Legacy Displacement](https://martinfowler.com/articles/patterns-legacy-displacement/) — the pattern of evidence-backed proposals is the case-study takeaway from Thoughtworks' modernization work).

**Gaming infrastructure — Unity engine version ADR.** A studio engineering team built a Unity-engine-upgrade ADR template that requires an "API Updater" dry-run report per project, a list of deprecated APIs the project uses, and a build-evidence section showing the project still compiles in headless mode against the target engine. The ADRs that get accepted have all three; the ones that get returned cite only the engine release notes.

**Banking — payment SDK migration ADR with Advice Process.** A European bank's payments team wrote a future-hop ADR for swapping a card-network SDK. The ADR went through Andrew Harmel-Law's Advice Process style — the author consulted the security team and the network operations team before publishing — and the published ADR included evidence the dry-run had been applied to one representative transaction type, with the residual (3-D Secure flow integration) called out explicitly ([Scaling the Practice of Architecture, Conversationally](https://martinfowler.com/articles/scaling-architecture-conversationally.html), retrieved 2026-05-26).

## 6. Best Practices

- Write the ADR's evidence section *before* the prose; the prose is easier to write when the evidence is collected.
- Keep evidence artifacts in a `migration-evidence/<adr-id>/` directory checked into the same repo as the ADR — link to specific commits, not branches.
- For each future-hop ADR, run the codemod's dry-run against at least one representative module before claiming the hop's effort.
- Name at least one hand-edit residual; if you cannot, the hop is either trivial (and does not need an ADR) or you have not exercised it.
- Cite primary sources only — release notes, official migration guides, recipe definitions — and link with retrieval dates.
- When two future-hop ADRs cover related decisions (e.g., next-minor vs next-major), publish them together so the trade-off is visible.
- Review future-hop ADRs in pairs: one reviewer reads the prose, the other inspects the captured evidence — they should reach the same conclusion.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** Pick a framework or library your team depends on. Draft the Evidence section of a future-hop ADR for the next major version. You do not need to run the dry-run during this exercise — just specify:

1. The tool you would use (codemod, recipe, manual script).
2. The representative module you would apply it to first, and why.
3. The two or three failure signals you would expect to see in the build output.
4. The most likely hand-edit residual based on what you know about your code.
5. The primary source you would cite (URL).

**What good looks like:** the tool is specific (a recipe ID, a codemod name) not generic ("a refactoring tool"); the representative module is named (not "the smallest one"); the failure signals are concrete (specific error messages or deprecation warnings, not "things might break"); the residual names a specific pattern in your code (not "some refactoring will be needed"); the primary source has a URL and a section heading.

## 8. Key Takeaways

- *What does an ADR capture?* A single architectural decision and its rationale — not a design; not a plan; one decision per record.
- *What distinguishes a future-hop ADR from an ordinary ADR?* The future-hop ADR evaluates a state the project has not yet reached, which requires evidence the project *could* reach it.
- *What are the five points of the evidence floor?* Dry-run output, captured patch/report, compile/build evidence, named hand-edit residual, cited primary source.
- *Why is "looks straightforward" the most common failure mode?* Because release notes are reassuring; reading them is not equivalent to running the migration; reviewers must ask for the dry-run.
- *How does the evidence floor connect to the Advice Process?* The floor is what the author brings to the conversation; the conversation evaluates the trade-off; the evidence floor is necessary but not sufficient.

## Sources

1. [Architectural Decision Records (adr.github.io)](https://adr.github.io/) — retrieved 2026-05-26
2. [architecture-decision-record (Henderson collection)](https://github.com/joelparkerhenderson/architecture-decision-record) — retrieved 2026-05-26
3. [Scaling the Practice of Architecture, Conversationally](https://martinfowler.com/articles/scaling-architecture-conversationally.html) — retrieved 2026-05-26
4. [Spring Boot 3.5 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.5-Release-Notes) — retrieved 2026-05-26
5. [Spring Boot 4.0.0 available now](https://spring.io/blog/2025/11/20/spring-boot-4-0-0-available-now) — retrieved 2026-05-26

Last verified: 2026-05-26
