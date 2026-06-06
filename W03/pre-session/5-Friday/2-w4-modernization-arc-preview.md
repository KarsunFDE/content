---
week: W03
day: 5-Friday
topic_slug: w4-modernization-arc-preview
topic_title: W4 Modernization Arc Preview
parent_overview: W03/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://spring.io/blog/2022/05/24/preparing-for-spring-boot-3-0
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.openrewrite.org/running-recipes/popular-recipe-guides/migrate-to-spring-3
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.openrewrite.org/recipes/java/migrate/jakarta/javaxmigrationtojakarta
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://martinfowler.com/articles/patterns-legacy-displacement/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-06-06
---

# W4 Modernization Arc Preview

> [!NOTE]
> **From earlier:** Wed's multi-agent topology exposed handoff failure modes — supervisor couldn't enforce the tenant boundary at the tool-call layer, so enforcement moved to the retrieval layer. Phase-1 discoveries like that one feed directly into W4 modernization scope.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the four incremental-modernization activities and explain why they run in parallel.
- List the three migration hops for `acquire-gov` (Boot 2.7→3.x→4.0 + J11→J21 + javax→jakarta) and justify their order.
- Explain what OpenRewrite handles automatically vs. what it leaves to manual fix-up.
- Distinguish a big-bang rewrite from a Strangler-Fig and name two failure modes of the former.
- Articulate how Phase-1 discoveries become Phase-2 scope via the D-029/D-036 §0 retro interlock.

## 2. Introduction

Modernization means making a production system easier to change without taking it offline or rewriting from scratch. Cartwright, Horn, and Lewis (Patterns of Legacy Displacement) formalized four repeating activities: understand the outcomes, break up the problem, deliver the parts, change the organization so this is ongoing practice.

`acquire-gov` arrives in W4 as deliberate legacy: Boot 2.7.18 + J11 + `javax.*` + AWS SDK v1 + Spring Security 5 (D-056). Monday's §0 retro (D-029, D-036) opens with the Phase-1 discoveries you author today.

> [!IMPORTANT]
> **Phase-1 discoveries ARE Phase-2 scope.** RAG layer couldn't read FAR clause headers cleanly → document storage strategy is on the modernization list. Supervisor couldn't enforce tenancy at the tool-call layer → authorization model is on the list. Monday's plan comes from today's retro, not from scratch.

## 3. Core Concepts

### 3.1 The four activities, running in parallel

Four activities run **simultaneously**, not sequentially:

1. **Understand the outcomes.** Boot 2.7 OSS support ended; SB 4.0 is current stable. Pick the outcome; the technical approach follows.
2. **Break into smaller parts.** One milestone twelve months out fails; twelve one month apart succeed. Each `acquire-gov` debt item is a candidate boundary.
3. **Deliver the parts.** Each part ships and is observable — the trace shows it survived.
4. **Change the organization.** W4 embeds the discipline: Plan Day → spec-driven sprint → retro → next slice.

> [!NOTE]
> Activity 4 is why W4 starts with §0 Plan-Day retro (D-029) — the org change is the discipline.

### 3.2 The Spring Boot 2.7 → 4.0 migration corridor

`acquire-gov` is SB 2.7.18 + J11 + `javax.*` + Spring Security 5. The W4 target is SB 4.0.6 + J21 + `jakarta.*` + Spring Security 6 (D-056, W4 retarget 2026-05-26). Three hops, run in this order:

| Hop | What changes | Key risk | Isolation trick |
|-----|-------------|----------|-----------------|
| 1. J11 → J21 (keep Boot 2.7) | Runtime only | GC tuning, reflection internals | Run tests on J21 with Boot untouched; profile for a sprint |
| 2. Boot 2.7 → 3.x + javax → jakarta | Namespace sweep, Security 5 → 6, property renames | Third-party libs without Jakarta release | Run OpenRewrite recipe; compile; triage missing Jakarta coords |
| 3. Boot 3.x → 4.0 + J21 | API removals, further Security hardening | Smaller surface than 2→3 | OpenRewrite `UpgradeSpringBoot_4_0` recipe; then test triage |

The order matters: hop 1 isolates runtime-era issues from framework-era issues so failures are attributable.

> [!TIP]
> Boot 2.7.x runs cleanly on Java 21 — do the runtime hop first with Boot unchanged.

### 3.3 OpenRewrite — what it does and what it leaves

OpenRewrite traverses the lossless syntax tree (LST), applies a declared recipe (`UpgradeSpringBoot_3_3` or `UpgradeSpringBoot_4_0`), and writes back with formatting preserved.

Handles well: `javax → jakarta` import sweep, build-file version bumps, property-file key renames, deterministic API changes.

Leaves to manual fix-up: behavioral semantic changes (Spring Security 6 dispatch-type authorization), third-party Jakarta compatibility (you survey and pin), and anything only a test suite catches.

## 4. Generic Implementation

A one-shot OpenRewrite migration against a Maven-based Spring Boot 2.7 project. The target project is a generic e-commerce checkout service — nothing federal-acquisitions-shaped — to prove the pattern generalizes.

```bash
# Run from the project root on a dedicated migration branch.
# Pre-flight: tests green on Boot 2.7.x latest + Java 21 before running this.
mvn -U \
  org.openrewrite.maven:rewrite-maven-plugin:run \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-spring:RELEASE \
  -Drewrite.activeRecipes=org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_3

# After recipe: add properties-migrator as runtime dep, start app, capture diagnostics.
# Then remove migrator before merging.

# For the 3.x → 4.0 hop (separate branch, separate PR):
mvn -U \
  org.openrewrite.maven:rewrite-maven-plugin:run \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-spring:RELEASE \
  -Drewrite.activeRecipes=org.openrewrite.java.spring.boot3.UpgradeSpringBoot_4_0
```

After the recipe runs: `pom.xml` version bumps, `application.yml` property renames, ~40+ files with import rewrites. Disciplined workflow: branch → pre-flight J21 (Boot unchanged) → pre-flight 2.7→3.x → compile → triage Jakarta coords → test → add/remove properties-migrator → repeat for 3.x→4.0. Each hop = one reviewable diff.

## 5. Real-world Patterns

**Fintech: payments-processor jakarta hop.** A mid-sized processor ran OpenRewrite Boot 3.x against ~180k lines in early 2025 — 4,800 import rewrites in ~90s. Manual fix-up: two engineer-weeks on a Security 6 dispatch-type filter, two PDF libs without Jakarta releases, and a JPA flush regression. Recipe was small; triage was the work.

**E-commerce: strangler-fig catalog.** A Java EE 7 retailer built new catalog endpoints in a fresh Boot 3 service behind a gateway; retired each legacy endpoint when traffic dropped below 5%. Full displacement: 11 months, no migration weekend.

## 6. Best Practices

- Hop the runtime (J11→J21) before the framework (Boot 2.7→3.x) — two distinct error classes, two distinct sprints.
- Run OpenRewrite on a dedicated branch with tests green; the recipe's commit is a single, reviewable diff.
- Add `spring-boot-properties-migrator` as a `runtime` dep immediately after the recipe; capture diagnostics, then remove before merging.
- Write Phase-1 discoveries as a numbered list; W4 §0 Plan Day retro (D-029) cites each by number when picking the first modernization slice.

> [!WARNING]
> **Anti-pattern: `modernization-without-discovery`.** Internet advice says run OpenRewrite immediately and ship. Without Phase-1 discovery first you don't know which behavioral gaps — multi-tenant enforcement, auth model shape, document indexing — the recipe *cannot fix*. The result is a Jakarta-namespaced codebase with the same architectural debt, now harder to see because the imports look modern.

## 7. Hands-on Exercise

**Whiteboard exercise — 10 minutes.** Draw the Phase-1→Phase-2 interlock for your pair-project:

1. Name two Phase-1 discoveries — things that took longer or forced an unplanned design change. Write them as numbered scope items.
2. For each, identify which of `acquire-gov`'s 12 brownfield-debt items it points to. Which argues for W4 week 1?
3. Sketch a three-step sequence for one debt item: pre-flight validation, recipe or manual change, test that proves behavior parity.

> [!NOTE]
> **Self-check** (30s — answer mentally before expanding)
>
> 1. What is the correct order of the three migration hops (runtime upgrade, Boot 2→3 + jakarta, Boot 3→4), and why does order matter?
> 2. Name one class of change that OpenRewrite handles automatically and one class it leaves to manual fix-up in a Spring Security 6 migration.

<details>
<summary>Show answers</summary>

1. Runtime first (J11→J21 with Boot 2.7 unchanged), then Boot 2.7→3.x + javax→jakarta, then Boot 3.x→4.0. Order matters because mixing runtime-era and framework-era issues in one hop makes failures unattributable — you can't tell whether a test failure is a GC regression or a Security 6 dispatch-type change.

2. Automatic: `javax→jakarta` import sweep across all source files (deterministic rename). Manual: Spring Security 6 applying authorization to every dispatch type — this is a behavioral change, not an API rename, so OpenRewrite can't infer the correct new configuration.

</details>

## 8. Key Takeaways

- Modernization runs four activities at once: outcomes, parts, delivery, org change.
- Correct hop order for `acquire-gov`: J21 first, then Boot 2.7→3.x + jakarta, then 3.x→4.0.
- OpenRewrite handles mechanical renames; behavioral semantic changes and third-party gaps stay manual.
- Strangler-Fig avoids migration weekends; big-bang rewrites fail on undocumented behavior.
- Phase-1 discoveries = Phase-2 scope items; Monday §0 retro (D-029, D-036) cites them by number.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide — retrieved 2026-05-26 — foundation-stable
- https://spring.io/blog/2022/05/24/preparing-for-spring-boot-3-0 — retrieved 2026-05-26 — foundation-stable
- https://docs.openrewrite.org/running-recipes/popular-recipe-guides/migrate-to-spring-3 — retrieved 2026-05-26 — hot-tech
- https://docs.openrewrite.org/recipes/java/migrate/jakarta/javaxmigrationtojakarta — retrieved 2026-05-26 — hot-tech
- https://martinfowler.com/articles/patterns-legacy-displacement/ — retrieved 2026-05-26 — foundation-stable

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

**Strangler-Fig routing specifics.** The gateway layer in a Strangler-Fig migration is the critical seam. Traffic routing decisions — by path, by tenant, by feature flag, by content-type — need to be observable from day one. A common senior-tier error is treating the gateway as a temporary shim and under-investing in its metrics. In federal-acq deployments, the gateway is also an audit boundary: route changes need to appear in the audit log so that the OIG can reconstruct which version of a service handled a given contract-award transaction.

**OpenRewrite LST model.** The lossless syntax tree model means that OpenRewrite preserves comments, whitespace, and import ordering after a recipe runs. This matters for code review velocity: a diff that changes only what it was supposed to change is reviewable in hours; a diff that also reformats the entire file is reviewable in days. If your OpenRewrite diff includes reformatting you didn't expect, check whether a formatting recipe was activated transitively by the `UpgradeSpringBoot_*` composite recipe.

**SB 4.0 vs 3.x surface area.** The 3→4 hop is smaller than the 2→3 hop because the `javax→jakarta` rename was the dominant change in 2→3. SB 4.0's breaking changes cluster in: removed deprecated APIs from SB 3.x, further Spring Security hardening, and some Actuator endpoint changes. For `acquire-gov` specifically, the larger risk in the 3→4 hop is the AWS SDK v1 dependency (also a W4 debt item) — AWS SDK v1 has no Jakarta-compatible release; the migration to SDK v2 (also a W4 target) should be colocated with or immediately follow the Boot 3→4 hop.

</details>

Last verified: 2026-06-06
