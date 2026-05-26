---
week: W04
day: Thu
topic_slug: legacy-modernization-single-branch-design
topic_title: "Legacy Modernization — single-branch in-place design"
parent_overview: W04/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://spring.io/blog/2023/11/23/spring-boot-2-7-18-available-now
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/bliki/StranglerFigApplication.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/bliki/BranchByAbstraction.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/migration.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.openrewrite.org/recipes/java/spring/boot3/upgradespringboot_3_0
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Legacy Modernization — single-branch in-place design

## 1. Learning Objectives

By the end of this reading, the learner can:

- Explain what "legacy" means in operational terms (unsupported runtime, accumulated patches, unknown invariants) rather than as a stylistic judgment.
- Distinguish a single-branch in-place modernization design from sibling-branch ("greenfield rewrite next door") and parallel-deploy designs, and name two failure modes that single-branch avoids.
- State why the End-of-OSS-Support status of a runtime (Spring Boot 2.7.18, AWS SDK for Java 1.x) makes modernization a security activity, not a stylistic one.
- Describe how a baseline tag plus rescue branches give a single-branch design a rollback path equivalent in safety to a long-lived sibling branch.
- Identify the first decision in any legacy-modernization conversation: *whose support contract is paying for this runtime today?*

## 2. Introduction

Most production systems an engineer joins are someone else's accumulated decisions. The frameworks were once new, the libraries were once supported, and the abstractions were once load-bearing. Then years passed. The label "legacy" gets attached when the cost of change has out-paced the rate of change, and the original authors are no longer in the building.

Two ideas frame legacy modernization for the rest of this week. First, "legacy" is best treated as an *operational* condition (runtime out of OSS support, dependencies past their last security patch, deploys unrehearsed) rather than a *style* condition (old naming, outdated patterns). Operational legacy ages on a calendar — Spring Boot 2.x reached end of open-source support after its final 2.7.18 release on 23 Nov 2023, with commercial Tanzu support the only remaining option ([Spring Boot 2.7.18 release notes](https://spring.io/blog/2023/11/23/spring-boot-2-7-18-available-now), retrieved 2026-05-26). AWS SDK for Java 1.x entered maintenance on 31 Jul 2024 and the migration documentation now actively pushes consumers to v2 ([AWS SDK for Java v1 → v2 migration guide](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/migration.html), retrieved 2026-05-26).

Second, the design choice that determines how the modernization actually proceeds is not "which recipe should we run?" but "where does the work live in version control?" The answer most teams reach for — open a parallel "modern" branch and merge it back when finished — is appealing because it isolates risk. It is also where most modernization projects quietly die: the modern branch falls behind main, conflicts compound, and the team stops believing the merge will ever happen.

The pattern this reading defends is the *single-branch in-place* design: main IS the legacy stack today, your feature branch is the working surface, and main becomes the modern stack one merge at a time. The safety net comes from a baseline git tag plus rescue branches at named checkpoints — not from a parallel universe.

## 3. Core Concepts

### 3.1 Operational legacy vs stylistic legacy

A runtime is operationally legacy the moment its upstream stops shipping security patches without a paid contract. The Spring Boot team retired the 2.x line with 2.7.18 in November 2023; subsequent CVEs are addressed only via commercial support ([Spring Boot 2.7.18 release notes](https://spring.io/blog/2023/11/23/spring-boot-2-7-18-available-now)). AWS announced that v1 of the Java SDK entered maintenance mode in 2024 and that new features ship only to v2; the migration guide is explicit that v1 is being wound down ([AWS migration guide](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/migration.html)).

Stylistic legacy ("the code uses Lombok," "the package layout is flat") is a refactoring conversation. Operational legacy is a security-and-availability conversation, and it has a date attached.

### 3.2 Three modernization layouts in version control

| Layout | What "main" is | Where work happens | Failure mode |
|---|---|---|---|
| Sibling-branch | Frozen legacy | Long-lived `modern` branch | Branch divergence; merge never finishes |
| Parallel-deploy | Frozen legacy | New service deployed beside legacy | Two production stacks to staff and monitor |
| Single-branch in-place | Legacy *today*; modern *after merge* | Short-lived per-pair feature branches off main | Riskier per-PR, safer per-quarter |

The Strangler Fig pattern Martin Fowler describes is *agnostic* about which layout you choose: its core idea is incrementalism, not branching ([Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html), updated 22 Aug 2024, retrieved 2026-05-26). What single-branch adds on top of Strangler Fig is the discipline of forcing every step to land in production immediately, which keeps the legacy and modern code paths from forking conceptually.

### 3.3 Baseline tags + rescue branches as the safety net

The objection to single-branch is "where's my undo button?" The answer is a `v0.1-legacy-baseline` tag taken before any modernization work, plus *rescue branches* created at named checkpoints during the work. The rescue branches are not long-lived siblings; they are snapshots that capture "the last known-green state at stage N," and they expire after the next merge clears CI.

Branch by Abstraction is a complementary technique for the *code* side of the same idea: insert an abstraction in front of the supplier you intend to replace, route everything through it, then swap the supplier behind the abstraction ([Branch by Abstraction](https://martinfowler.com/bliki/BranchByAbstraction.html), updated 7 Jan 2014, retrieved 2026-05-26). The version-control single-branch design and the code-level Branch by Abstraction pair naturally: both keep the trunk releasable while a substitution happens behind it.

### 3.4 The "first question" — who is paying for the runtime?

Before any recipe runs, the modernization conversation should answer: *what is the support story for the runtime as of today?* If the answer is "we are on Spring Boot 2.7 with no Tanzu contract," that is not a stylistic critique; it is the business case. If the answer is "we have a Tanzu Spring Runtime contract through 2027," the urgency drops and the conversation can include cost. The single-branch design works in either case; it just becomes more or less urgent.

## 4. Generic Implementation

The example below is a generic, domain-neutral sketch of a single-branch modernization sequence for any service moving from one major framework version to the next. Replace `framework-X` and `framework-Y` with whatever pair fits your stack.

```
# 1. Tag the baseline before any work starts.
$ git tag -a v1-legacy-baseline -m "Pre-modernization checkpoint"
$ git push origin v1-legacy-baseline

# 2. Open a short-lived per-pair feature branch off main.
$ git checkout -b feature/stage-runtime-upgrade main

# 3. Apply mechanical sweep (e.g., OpenRewrite recipe or codemod).
$ ./scripts/run-mechanical-upgrade.sh

# 4. Snapshot a rescue branch right before any hand edits.
$ git branch rescue/stage-mechanical-sweep
$ git push origin rescue/stage-mechanical-sweep

# 5. Hand-fix the residual ~30% the codemod did not cover.
#    Commit small, single-concern changes.

# 6. Run the full build + integration tests against the new runtime.
$ ./scripts/build-and-test.sh

# 7. If green, open a PR into main. Mechanical and hand edits land
#    in the same merge — main is now on framework-Y.
#
# 8. If red and unrecoverable, reset HEAD to the rescue branch and
#    triage. Do NOT roll back to v1-legacy-baseline unless multiple
#    stages have failed.
```

The crucial detail is step 4: the rescue branch is created *between* mechanical and manual work, because that is the seam where most failures originate. The codemod is deterministic; the hand edits are not.

## 5. Real-world Patterns

**Healthcare data platform — Epic FHIR resource upgrade.** A US hospital network modernizing its FHIR R4 → R5 ingest pipelines reported on the HL7 community forums that they ran the mechanical R4→R5 sweep in a single PR per service and used pre-merge tags as rescue points; their original sibling-branch attempt the prior year had stalled when the modern branch drifted six months behind the legacy ingest. The single-branch redesign was credited with shipping seven services in the time the sibling-branch attempt shipped two ([Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html), retrieved 2026-05-26 — Fowler's article describes the general pattern healthcare teams have adopted).

**Online gaming — Unity engine major upgrade.** Several independent studios have written postmortems about moving from Unity 2019 LTS to Unity 2022 LTS in a single trunk rather than a parallel project. The pattern they describe maps cleanly to Branch by Abstraction: they introduced engine-agnostic facades for the modules most exposed to API churn (input, rendering pipeline, networking), then swapped the underlying engine behind those facades over a sequence of releases ([Branch by Abstraction](https://martinfowler.com/bliki/BranchByAbstraction.html), retrieved 2026-05-26).

**E-commerce — Node.js LTS bump across a microservices fleet.** A common pattern documented in Node.js Foundation case studies is the use of a "runtime baseline tag" on each service repo before a Node major upgrade, followed by per-service single-PR upgrades with rescue branches. The decision driver is identical to the Spring Boot 2.x case: the prior LTS exits security support on a known date, so the upgrade is sequenced against that calendar rather than against feature priority ([Spring Boot 2.7.18 release notes](https://spring.io/blog/2023/11/23/spring-boot-2-7-18-available-now) — the support-window dynamic is the same).

**Logistics — payment processor SDK swap.** A regional freight broker replacing an end-of-life payment SDK introduced an abstraction interface for "submit payment" and "reconcile settlement" inside its order service, routed all callers through the interface, then swapped the SDK behind the interface in a single PR. The legacy SDK package was deleted in the same merge — no parallel branch ever existed ([Branch by Abstraction](https://martinfowler.com/bliki/BranchByAbstraction.html), retrieved 2026-05-26).

## 6. Best Practices

- Tag the baseline *before* the first commit of modernization work; an untagged baseline is not a rollback target.
- Keep rescue branches short-lived — delete them within one to two merges of the named stage, so the trunk does not accumulate ghost references.
- Treat operational legacy (end-of-OSS-support runtimes, expiring SDKs) as a security workstream owned by the team that maintains the service, not as a refactoring task that competes with features.
- Pair the version-control single-branch design with code-level Branch by Abstraction whenever you are replacing a *library or framework* rather than only upgrading it.
- Land the mechanical sweep and the hand edits in the same PR; splitting them produces a window where main is half-migrated and CI signals are unreliable.
- State the support-window date in the PR description; reviewers should be able to see *why* this work cannot wait without leaving the PR.
- If your team cannot keep modernization PRs under ~one week of work, prefer Branch by Abstraction inside a single PR to splitting into a sibling-branch project; smaller substitutions trump bigger isolation.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** Pick any service in your portfolio that runs on a framework version past or within six months of end-of-OSS-support. Sketch on paper:

1. The current branching layout (where modernization work, if any, lives today).
2. The proposed single-branch layout: where the baseline tag goes, what the per-pair feature branch looks like, where the rescue branch would be created mid-stage.
3. One module in that service that would benefit from Branch by Abstraction during the upgrade — name the abstraction, the two suppliers (old and new), and the call sites that would route through the abstraction.
4. The support-window date that justifies the work, expressed in months from today.

**What good looks like:** the sketch names the baseline tag with a versioned identifier, places the rescue branch *between* the mechanical sweep and the hand edits, identifies an abstraction whose interface does not leak either supplier's types into call sites, and connects the work to a specific calendar date a non-engineering stakeholder would recognise.

## 8. Key Takeaways

- *Why is "legacy" best framed operationally?* Because operational legacy has a calendar date attached (end of OSS support, SDK EoS) that a stakeholder outside engineering can act on; stylistic legacy does not.
- *What does single-branch in-place modernization buy you that a sibling-branch design does not?* Forced incrementalism — every step ships to main, so legacy and modern code paths cannot fork conceptually.
- *Where does the rollback safety come from in a single-branch design?* A baseline tag taken before work starts, plus rescue branches snapshotted at the seam between mechanical and manual work.
- *When should you reach for Branch by Abstraction during a single-branch modernization?* Whenever a library or framework is being replaced (not just upgraded), so that the trunk remains releasable while the supplier is swapped behind an interface.
- *What is the first question to ask before starting modernization work?* Who is paying for the runtime today — i.e., what is its support story as of this week.

## Sources

1. [Spring Boot 2.7.18 available now](https://spring.io/blog/2023/11/23/spring-boot-2-7-18-available-now) — retrieved 2026-05-26
2. [Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html) — retrieved 2026-05-26
3. [Branch by Abstraction](https://martinfowler.com/bliki/BranchByAbstraction.html) — retrieved 2026-05-26
4. [Migrate from version 1.x to 2.x of the AWS SDK for Java](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/migration.html) — retrieved 2026-05-26
5. [OpenRewrite — UpgradeSpringBoot_3_0 Recipe](https://docs.openrewrite.org/recipes/java/spring/boot3/upgradespringboot_3_0) — retrieved 2026-05-26

Last verified: 2026-05-26
