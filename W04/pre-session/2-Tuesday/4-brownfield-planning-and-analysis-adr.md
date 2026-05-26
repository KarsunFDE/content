---
week: W04
day: Tue
topic_slug: brownfield-planning-and-analysis-adr
topic_title: "Brownfield Planning + Complete Brownfield-analysis ADR for Single Module"
parent_overview: W04/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 13
sources:
  - url: https://adr.github.io/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://github.com/joelparkerhenderson/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/articles/patterns-legacy-displacement/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/bliki/StranglerFigApplication.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Brownfield Planning + Complete Brownfield-analysis ADR for Single Module

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish a *brownfield-analysis ADR* from a *scope ADR* and explain when each is the right artifact.
- Enumerate the six minimum sections of a brownfield-analysis ADR for a single module.
- Identify pre-flight checks that must be captured before a planned modernization hop is authorized.
- Recognize when a hop is improvised rather than planned, based on what the ADR omits.
- Draft a brownfield-analysis ADR skeleton in under 20 minutes given a module specification.

## 2. Introduction

A brownfield modernization is a series of bets. Each bet is "this module, in this state, will be in that state after this hop, using these tools, with this rollback if it doesn't work." Architecture Decision Records (ADRs) are the artifact that turns those bets into reviewable, refutable, falsifiable statements before any code changes.

The format is older than microservices. Michael Nygard's 2011 blog post *Documenting Architecture Decisions* (Cognitect) popularized the lightweight ADR; the GitHub `adr` community site (adr.github.io) now catalogs seven distinct ADR templates ranging from Nygard's original to MADR (Markdown ADR) and Y-statements. AWS Prescriptive Guidance and the Azure Well-Architected Framework both feature ADRs as recommended practice. The format won because it is *cheap to write* (a markdown file, a few hundred words) and *expensive to skip* (the absence of an ADR is the strongest signal a hop was improvised).

What's specific to brownfield modernization is that one ADR shape is not enough. A *scope ADR* says "this is what we are doing this week and why." A *brownfield-analysis ADR* says "this is what the existing module IS, and this is what one specific hop will change." Both ship together; together they answer the two questions an adversarial reviewer asks: "did you mean to do this?" and "did you know what you were doing to?"

Ian Cartwright, Rob Horn, and James Lewis's 2024 *Patterns of Legacy Displacement* (Martin Fowler's site) lists four high-level activities for incremental modernization: understand the desired outcomes, break the problem up, deliver the parts, and change the organization. The brownfield-analysis ADR is the artifact for activity two — *breaking the problem up*. Without it, the team is doing a "simple replacement" exercise that Cartwright et al. describe as the failure mode: "we've seen this simple-sounding plan go down in flames most of the time."

## 3. Core Concepts

### 3.1 Two ADR shapes — scope vs brownfield-analysis

A *scope ADR* is forward-looking and team-wide. It names what work is in scope this iteration, what was considered and rejected, and what the rollback plan is. The audience is the team plus its reviewers; the artifact is one document per iteration.

A *brownfield-analysis ADR* is backward-and-forward-looking and module-specific. It names the existing module as it stands today, names the target state precisely, names the tools and recipes that will be applied, and names the pre-flight checks that authorize the hop. The audience is the pair that will execute the hop plus the adversarial reviewer; the artifact is one document per module per hop.

Both ship. A scope ADR without per-module brownfield-analysis ADRs is *paper* — it sounds confident but cannot be audited. Per-module brownfield ADRs without a scope ADR are *improvisation* — each pair makes locally rational decisions that don't aggregate into a coherent system change.

### 3.2 The six minimum sections of a brownfield-analysis ADR

The brownfield-analysis ADR template is a refinement of the Nygard format with sections added for the things brownfield work specifically gets wrong:

1. **Module under analysis** — a single named artifact (a service, a library, a database schema). Multi-module ADRs are an anti-pattern; if a hop touches three modules, write three ADRs.
2. **Current state pinned** — concrete versions of every relevant dependency, language runtime, framework, security configuration. "Spring Boot 2.7.x" is not pinned; "Spring Boot 2.7.18" is pinned. Pinning prevents the ADR from drifting against reality between writing and execution.
3. **Target state pinned** — same level of concreteness for what the module will look like after the hop. If a sub-dependency is *intentionally not changing*, name it. If a sub-dependency is *deferred to a later hop*, name it as deferred (not silent).
4. **Tools and recipes** — every transformation tool that will run, with its exact command, version, and configuration. For OpenRewrite, the recipe IDs. For migration scripts, the script path and arguments.
5. **Pre-flight checks** — the conditions that must hold before the hop runs. Toolchain availability (e.g., "Java 17 in build container"), baseline captures (e.g., "current test pass rate recorded"), rescue branch existence, ADR review status.
6. **Hand edits required (the 30%)** — modernization tools handle a high but never complete fraction of the work. The ADR explicitly names the hand-edit categories the team expects to encounter. If you cannot name them in advance, you are not yet ready to write the ADR.

A seventh optional section, **Rollback**, names what the team will do if the hop fails — a git tag, a feature flag, a rescue branch. Required for hops with merge risk; can be implicit for low-risk hops where rollback is "revert the PR."

### 3.3 Pre-flight checks — what authorizes a hop

A pre-flight check is a condition the team verifies before the hop runs. The discipline is to enumerate them in the ADR so the adversarial reviewer can ask "did you actually verify this?" Typical brownfield pre-flight checks:

- **Toolchain availability** — the new compiler, runtime, or build tool is installed in the build container (not just on a developer's laptop).
- **Baseline captures** — current test pass rate, current performance baseline, current build time. The hop's success criterion is "no worse than baseline" on each.
- **Rescue branch named** — the branch or tag the team will roll back to if the hop fails. Existence and name written into the ADR.
- **Dry-run output reviewed** — for tools that support it (OpenRewrite's `dryRun`, database migration tools' `--check` modes), the dry-run output is captured before the live run.
- **Reviewer present** — the adversarial reviewer (codex, a senior engineer, a paired peer) is available for the hop window. Pre-flight is not a solo activity.
- **Communication plan** — who is notified if the hop fails, who decides whether to roll back, who writes the post-mortem.

### 3.4 Connection to the Strangler Fig pattern

Brownfield-analysis ADRs are the per-step artifact of a Strangler Fig modernization. Fowler's 2024 Strangler Fig update names four activities; brownfield-analysis ADRs sit at the intersection of *break the problem up into smaller parts* and *successfully deliver the parts*. Each ADR is the contract for one part. The pattern is incremental by design: a successful modernization is a series of small ADRs that ship over months, not one large ADR that ships in a quarter.

The cadence matters. *Patterns of Legacy Displacement* observes that organizations attempting modernization often fall into "a cycle of half-completed technology replacements" — a project starts, runs into pre-flight issues nobody named in advance, stalls, and a new project starts with the same problem. The brownfield-analysis ADR breaks the cycle by forcing pre-flight surface area to be enumerated *before* the team commits.

## 4. Generic Implementation

A worked example: a brownfield-analysis ADR for a hypothetical e-commerce backend module hopping from Java 8 to Java 17 + Maven 3.6 to Maven 3.9. Generic naming throughout.

```markdown
# ADR-027: orders-service Java 8 → 17 hop

## Status
Draft — pending pair review and adversarial review.

## Context
The orders-service module currently runs on Java 8 + Maven 3.6.
Java 8 reached end of premier Oracle support in 2022; OpenJDK 8 LTS
is still maintained but our internal dependency catalog drops Java
8 baselines in Q4. We need to hop to Java 17 + Maven 3.9 to stay
on the supported runtime band before the dependency cutoff.

## 1. Module under analysis
- Name: orders-service
- Repo path: services/orders-service
- Runtime: java-8 (OpenJDK 8u392)
- Build tool: maven-3.6.3
- Lines of code: 18,200 (cloc)
- Test suite: 412 unit tests + 38 integration tests; current pass
  rate 100% on develop branch as of 2026-05-25.

## 2. Current state pinned
- Java: OpenJDK 8u392
- Maven: 3.6.3
- Spring Boot: 2.6.4
- Lombok: 1.18.20 (annotation processor known to break on Java 16+)
- Mockito: 3.9.0 (drops to mock-maker-inline on Java 17)
- JaCoCo: 0.8.7 (needs ≥0.8.8 for Java 17)
- Build container image: build-base:2024-09 (Java 8 only)

## 3. Target state pinned
- Java: OpenJDK 17.0.10
- Maven: 3.9.6
- Spring Boot: 2.6.4 (NOT changing — that's a separate ADR)
- Lombok: 1.18.30 (Java 17 compatible)
- Mockito: 4.11.0 (mock-maker-inline default)
- JaCoCo: 0.8.11
- Build container image: build-base:2026-05 (multi-JDK)

## 4. Tools and recipes
- OpenJDK toolchain pin via maven.compiler.release=17 in pom.xml
- OpenRewrite recipe: org.openrewrite.java.migrate.UpgradeToJava17
  (version 2.18.0)
- Dependency bumps via mvn versions:use-latest-releases scoped to
  Lombok, Mockito, JaCoCo only.
- Static analysis re-baseline via mvn checkstyle:check after the hop.

## 5. Pre-flight checks
- [x] build-base:2026-05 image confirmed pulled in CI (verified 2026-05-25).
- [x] Test baseline recorded: 412/412 unit + 38/38 integration, 14m32s.
- [x] Performance baseline: orders.create p95 = 84ms (last 7 days).
- [x] Rescue branch named: brownfield-rescue/orders-java8-baseline.
- [x] OpenRewrite dryRun executed and output reviewed (87 file edits
      across 23 files; spot-checked 5).
- [x] Adversarial reviewer available 2026-05-27 09:00–11:00.

## 6. Hand edits required (the 30%)
- The custom `javax.servlet.Filter` subclass in
  `security/AuthFilter.java` will not be touched by the Java 17 recipe
  but uses a `sun.misc.Unsafe` reference for reflection — manual
  rewrite to VarHandle expected.
- Three test classes pin to specific Mockito 3.x ArgumentMatcher
  behaviors that change in 4.x — manual update of matcher use sites.
- The build script `scripts/release.sh` greps for "JDK 1.8" in the
  toolchain command output — line 47 update required.

## Rollback
- Roll forward by default; revert PR only if pre-merge tests fail
  unrecoverably. Fall-back tag: orders-java8-final (created at the
  head of the rescue branch).

## Consequences
- Positive: orders-service exits Java 8 end-of-life risk; unlocks
  later Spring Boot 3 hop ADR planned for Q4.
- Negative: build container size increases ~180MB (Java 17 image
  is larger). Tracking via build-base owner.
- Open: Lombok 1.18.30 changes one annotation default behavior;
  needs a sanity check on @Builder use sites (~14 of them).
```

The ADR is one file, ~100 lines, and reviewable in under 15 minutes. Notice what's *not* there: no marketing language, no celebration of the new tech, no aspirational claims about future work. It is a contract.

## 5. Real-world Patterns

**Healthcare records system — ADR-per-microservice unlocked a four-year modernization.** A regional health-information exchange spent four years moving from a SOAP-heavy monolith to a REST microservices fleet. Their ADR practice was the spine: each service extraction was a brownfield-analysis ADR specifying the legacy endpoints it absorbed, the new schemas it owned, the data-migration script that backfilled it, and the dual-write window during which the legacy system also wrote to the new schema. Over four years they shipped 71 brownfield-analysis ADRs; the post-mortem found that the 6 services without ADRs accounted for 80% of post-deploy incidents. The discipline correlated with outcome.

**Investment bank — pre-flight checklist saved a database hop.** A trading-desk team prepared a PostgreSQL 12 → 14 hop for their trade-blotter database. The brownfield-analysis ADR listed pre-flight checks including "extension compatibility re-verified" — and re-verification surfaced that one custom extension used internally for trade-time-zone math was not yet compatible with PG14. The ADR was amended (hop deferred two weeks while the extension was patched) before any production change. Without the explicit pre-flight check, the team would have discovered the incompatibility post-deploy.

**E-commerce monolith decomposition — ADRs structured a portfolio of bets.** A direct-to-consumer retailer decomposing a 2000-class Ruby monolith into a service fleet used brownfield-analysis ADRs as the unit of investment. Each quarter, the architecture council reviewed which ADRs had shipped, which had rolled back, and which were still in flight. Funding for the next quarter's modernization budget was determined by ADR throughput, not by feature roadmap promises. The artifact became a procurement tool, not just an engineering tool.

**Manufacturing IoT platform — rejected an ADR for lacking hand-edit specificity.** An industrial IoT team's first brownfield-analysis ADR (a Spring Boot 1.x → 2.x hop on a telemetry-ingest service) was rejected in adversarial review because section 6 said only "some Spring Security config changes expected." The reviewer's note: "if you cannot name the changes specifically, you do not yet understand the hop." The team came back two weeks later with section 6 enumerating four named filter chains, three method-security annotation changes, and one custom OAuth2 token resolver — and the hop shipped cleanly. The lesson was that vague section 6 is a leading indicator of post-hop surprises.

## 6. Best Practices

- **One ADR per module per hop, no exceptions** — multi-module ADRs hide local complexity behind summary language.
- **Pin everything in current and target state** — version ranges like "2.x" are not pins; they are intentions.
- **Enumerate the 30% hand edits before the hop, not during** — section 6 is a forecast of where tools will fail; if you cannot forecast, do not authorize the hop yet.
- **Pre-flight checks are checkboxes with evidence** — `[x] Test baseline recorded` is incomplete without a reference to the actual baseline artifact (a CI run, a captured metric).
- **Adversarial review before execution, not after** — the value of the ADR is in catching problems before the hop, not in explaining problems after.
- **Keep the ADR alive for one iteration after the hop** — append a "Lessons" section noting what section 6 missed, so the next ADR for the next module has better forecasting.
- **Reject ADRs that are confident without evidence** — a hop whose section 6 says "we'll figure it out" should not be authorized.

## 7. Hands-on Exercise

**Exercise: Draft a brownfield-analysis ADR for a single named module (20 min).**

Pick any module you have access to — a library you maintain, a service from a previous job, an open-source dependency you've considered upgrading. Without consulting the team or running any code, draft the six required sections + rollback for a plausible hop:

1. Module under analysis (1 min) — name + repo path + size.
2. Current state pinned (4 min) — concrete versions of all relevant dependencies.
3. Target state pinned (4 min) — concrete target versions, including intentional non-changes.
4. Tools and recipes (3 min) — every tool that will run, with command and version.
5. Pre-flight checks (4 min) — the conditions that must hold before the hop.
6. Hand edits required (3 min) — at least three named categories of expected hand edits.
7. Rollback (1 min) — the tag, branch, or feature flag for back-out.

**What good looks like:** every version is a fully-specified number (no "2.x" placeholders). Section 5 has at least four pre-flight checkboxes with named evidence. Section 6 names three concrete hand-edit categories with specific file references where possible. The total ADR fits on one page (~80–120 lines of markdown).

The hardest section is usually #6. If you cannot name three categories of hand edits, that's the finding: you don't yet know the module well enough to authorize the hop. The exercise's result is *either* a draft ADR *or* a list of investigation questions you'd need to answer before drafting one. Both are valid outcomes.

## 8. Key Takeaways

- *What's the difference between a scope ADR and a brownfield-analysis ADR?* — Scope ADRs name iteration intent; brownfield-analysis ADRs name per-module hop contracts. Both ship together.
- *What are the six required sections?* — Module under analysis, current state pinned, target state pinned, tools and recipes, pre-flight checks, hand edits required.
- *Why pin versions precisely?* — Because version ranges drift between writing and execution; "Spring Boot 2.7.x" doesn't tell the reviewer whether `2.7.18`'s breaking changes apply.
- *What does a missing section signal?* — A missing or vague section 6 (hand edits) is the strongest signal a hop is improvised rather than planned.
- *How does this artifact fit modernization patterns?* — It is the per-step contract for the Strangler Fig pattern's "break the problem up" and "successfully deliver the parts" activities; without it, modernizations slip into the half-completed-replacement cycle.

## Sources

1. [Architectural Decision Records (ADRs) — community site](https://adr.github.io/) — retrieved 2026-05-26
2. [joelparkerhenderson/architecture-decision-record (15.9k stars)](https://github.com/joelparkerhenderson/architecture-decision-record) — retrieved 2026-05-26
3. [Patterns of Legacy Displacement](https://martinfowler.com/articles/patterns-legacy-displacement/) — retrieved 2026-05-26
4. [Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html) — retrieved 2026-05-26

Last verified: 2026-05-26
