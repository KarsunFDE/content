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
last_verified: 2026-05-26
---

# W4 Modernization Arc Preview

## 1. Learning Objectives

By the end of this reading, the learner can:

- Describe the four high-level activities of incremental legacy modernization and how they sequence inside a single sprint.
- Name the three migration hops that take a Spring Boot 2.7 + Java 11 + `javax.*` application to Spring Boot 3.x + Java 17 + `jakarta.*`, and the order they must run in.
- Explain what OpenRewrite recipes do (and don't do) during a framework migration, including which classes of change require manual fix-up after the recipe runs.
- Distinguish between a "big bang" rewrite and a Strangler-Fig modernization, and name two failure modes of the former that the latter avoids.
- Articulate the relationship between Phase-1 discoveries (adoption findings) and Phase-2 modernization scope (what to fix first, what to defer).

## 2. Introduction

Modernization is the work of taking a system that runs in production but is hard to change, and making it easier to change — without taking it offline, without losing user trust, and without rewriting it from scratch. Most organizations have learned the hard way that "let's replace it" rarely ships. The systems that need modernization most carry the most accumulated behavior, much of it undocumented, a meaningful slice of which is no longer needed but cannot be easily distinguished from the load-bearing parts.

The industry response has converged on a discipline: modernize incrementally. Identify the smallest viable slice. Replace it behind a stable interface. Verify behavior parity with production traffic. Move on. The Strangler-Fig pattern coined by Martin Fowler in 2004 captures the metaphor — modernization grows alongside the legacy system until the legacy system can be retired. Ian Cartwright, Rob Horn, and James Lewis formalized four activities a modernization team needs to repeat: understand the outcomes, decide how to break up the problem, deliver the parts, and change the organization so that this becomes ongoing practice.

This reading previews the modernization arc that begins Monday. Tonight's read is **orientation**: what shape the work takes, what tools you will reach for, and how the discoveries you just defended in the Phase-1 gate become inputs to the Phase-2 plan you write on Monday morning.

## 3. Core Concepts

### 3.1 The four activities, repeated

Cartwright, Horn, and Lewis name four activities that run in parallel, not in sequence:

1. **Understand the outcomes you want to achieve.** Reducing cost-of-change. Enabling a business capability the legacy system blocks. Retiring a system whose vendor support is ending. Responding to imminent disruption (a regulatory deadline, a security CVE, a runtime EOL). Pick the outcome first; the technical approach follows.
2. **Decide how to break the problem into smaller parts.** A modernization with one milestone twelve months out almost always fails. A modernization with twelve milestones one month apart can succeed.
3. **Successfully deliver the parts.** Each part ships. Each part survives production. Each part is observable enough that you know whether it survived.
4. **Change the organization to allow this to happen on an ongoing basis.** Modernization is not a project that ends. The next legacy system is always being created by the team that just modernized this one.

The activities are simultaneous because the outcome can shift as you discover new constraints; the parts can be re-sliced as you learn what is actually coupled; the org changes are what let the next slice ship faster than the one before it.

### 3.2 The Spring Boot 2.7 → 3.x corridor

When the legacy system is a Spring Boot application on the long-term-support `2.7.x` line, modernization has a well-known shape because Spring committed publicly to the breaking changes that landed in 3.0:

- **Java 17.** Spring Boot 3.0 requires Java 17 or later. Java 8 is no longer supported. The official guidance (Spring Blog, May 2022) is to upgrade to Java 17 *before* attempting the Boot upgrade, because Boot 2.7.x runs cleanly on Java 17, which lets you isolate runtime-version issues from framework-version issues.
- **`jakarta.*` namespace.** Spring Boot 3.0 uses Jakarta EE 9+ APIs, where the `javax.*` package prefix was renamed to `jakarta.*`. Every `import javax.servlet.*` becomes `import jakarta.servlet.*`. Every Maven coordinate like `javax.servlet:javax.servlet-api` becomes `jakarta.servlet:jakarta.servlet-api`. This is a mechanical rename across thousands of import lines in a real application — the kind of change tooling does well and humans do badly.
- **Spring Framework 6 + Spring Security 6.** Boot 3.0 builds on Spring Framework 6, which has its own upgrade guide. Spring Security 6 changes how authorization is applied to dispatch types — the official Boot guide recommends upgrading Boot 2.7 → Spring Security 5.8 first, *then* jumping to 6.0 with Boot 3.0.
- **Configuration property renames.** A handful of `application.properties` / `application.yml` keys were renamed or removed. The `spring-boot-properties-migrator` module, added as a `runtime` dependency, prints diagnostics at startup and temporarily back-fills the old names while you cut over.

### 3.3 OpenRewrite — what it is and what it isn't

OpenRewrite is an open-source refactoring engine that operates on the parsed lossless syntax tree (LST) of your source files. You declare a **recipe** — for example, `org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_3` — and the engine traverses the LST, applies the transformations, and writes the result back with formatting preserved.

For a Boot 2.7 → 3.x migration, the canonical recipe (`UpgradeSpringBoot_3_x`, configured via the `rewrite-spring` artifact) is composed of dozens of sub-recipes that chain in sequence: upgrade to 3.0, then 3.1, then 3.2, etc. It will modify your `pom.xml` or `build.gradle`, rewrite imports across your source tree, migrate deprecated property keys, and apply the related Spring Security and Spring Framework hops along the way.

What OpenRewrite **does** well:

- Mechanical renames at scale (the `javax → jakarta` import sweep).
- Build-file edits (dependency version bumps, parent POM changes).
- Property-file migrations against a known rename catalog.
- Known API changes that follow a deterministic before/after pattern.

What OpenRewrite **does not** do:

- Resolve behavioral changes in your business logic that depend on a framework primitive's *semantics* changing.
- Migrate runtime behavior changes that are not API renames — e.g., Spring Security 6 applying authorization to every dispatch type.
- Decide whether a third-party library on your classpath has a Jakarta-compatible release (you must check and pin it yourself).
- Substitute for a working test suite. Recipes change code; your tests prove the change didn't change behavior.

### 3.4 The Phase-1 → Phase-2 interlock

You just finished Phase 1 — three weeks of building AI adoption *into* a brownfield. The retro you wrote this afternoon surfaced **discoveries**. Things like: "the W2 RAG corpus ingestion took 1.5 days longer than planned because the FAR/DFARS PDFs had inconsistent header structure." Or: "the multi-agent supervisor pattern couldn't enforce the tenant boundary at the tool-call layer, so we ended up enforcing it at the retrieval layer instead."

Those discoveries are not Phase-1 failures. They are Phase-2 inputs. Monday's Plan Day starts with the §0 retro against last week's spec, but the *cohort-level* retro frame is wider — the Phase-1 discoveries name the modernization scope. If your RAG layer couldn't read the FAR clause headers cleanly, then *the brownfield's document storage and indexing strategy* is now on the Phase-2 scope list. If the supervisor couldn't enforce tenancy, then *the brownfield's authorization model* is on the list.

This is the discipline Cartwright/Horn/Lewis call "decide how to break the problem into smaller parts." Phase-1 told you which parts to break first.

## 4. Generic Implementation

A minimal OpenRewrite invocation against a Maven-based Spring Boot 2.7 project looks like this. The legacy system is a generic e-commerce checkout service — nothing federal-acquisitions-shaped about it.

```bash
# Run from the project root. No edits to pom.xml required for one-shot mode.
mvn -U \
  org.openrewrite.maven:rewrite-maven-plugin:run \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-spring:RELEASE \
  -Drewrite.activeRecipes=org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_3
```

After the recipe completes, the working tree will show:

```
M  pom.xml                              # parent + dependency versions bumped to 3.3.x
M  src/main/resources/application.yml   # renamed properties (e.g. server.servlet.path)
M  src/main/java/com/example/cart/      # ~40 files with import rewrites javax → jakarta
M  src/main/java/com/example/cart/web/  # @WebFilter, HttpServletRequest now jakarta.*
M  src/main/java/com/example/cart/jpa/  # @Entity, @Column now jakarta.persistence.*
?? rewrite.log                          # diagnostic log; commit-gate review artifact
```

The disciplined workflow around this single command is what makes the migration succeed:

1. **Branch.** Work on a dedicated migration branch. Do not run the recipe on `main`.
2. **Pre-flight to the latest 2.7.x.** Upgrade to the most recent Boot 2.7 patch release first, with the existing test suite green. This isolates Boot-line patch issues from Boot-major version issues.
3. **Pre-flight to Java 17.** Keep Boot 2.7, bump the JDK to 17, rerun tests. Address any reflection/internals breakage now, not while you're also dealing with Jakarta.
4. **Run the recipe.** One Maven invocation. Commit the result as a single, reviewable diff.
5. **Compile.** Expect failures from third-party libraries that haven't published Jakarta-compatible artifacts. Substitute or pin compatible versions.
6. **Test.** Run the test suite. Triage failures. The pattern: most failures concentrate in Spring Security configuration (dispatch types), JPA queries (entity manager subtleties), and any code that interrogated framework internals via reflection.
7. **Add the properties-migrator.** Drop in `spring-boot-properties-migrator` as a `runtime` dependency and start the application. Capture every diagnostic it emits, then update `application.yml` to the new key names and remove the migrator.

## 5. Real-world Patterns

**Fintech: payments-processor jakarta hop.** A mid-sized payments processor running a Spring Boot 2.4 monolith for card authorization ran the OpenRewrite Boot 3.x upgrade recipe in early 2025 against ~180k lines of Java. The recipe rewrote 4,800 import statements and updated 23 dependency coordinates in about 90 seconds. Manual fix-up took two engineer-weeks, concentrated in three areas: a custom Spring Security filter that read dispatch types directly (broken by the Security 6 default), two third-party PDF-generation libraries that had no Jakarta release (forced a substitution to a different vendor), and a Spring Data JPA query that relied on a pre-3.1 entity-manager flush behavior. The team's lesson recorded in their internal post-mortem: the *recipe* was the small part; the *test triage* was the work.

**E-commerce: strangler-fig product catalog.** A retailer with a Java EE 7 product-catalog service running on WildFly chose not to upgrade in place — they ran a Strangler Fig instead. New product-categorization endpoints were built in a fresh Spring Boot 3 service, routed through an API gateway. As each legacy endpoint's traffic share dropped below 5% (measured at the gateway), the old endpoint was retired. The full displacement took eleven months. The legacy service ran the entire time. They never had a "migration weekend."

**Healthcare: HL7 ingestion lift-and-shift.** A hospital network's HL7 message-ingestion service was on Java 8 + Spring Boot 1.5, well past EOL. The team ran a two-hop migration: Boot 1.5 → 2.7 first (with OpenRewrite's `UpgradeSpringBoot_2_7` recipe), validated production parity for a month behind a feature flag, then Boot 2.7 → 3.2 with the Jakarta hop. The two-hop approach added six weeks but eliminated the "did the framework change break it or did the JDK change break it" diagnostic ambiguity that had killed two previous attempts.

**Gaming: matchmaking-service runtime upgrade.** An online-gaming studio's matchmaking service was on Java 11. They moved to Java 17 *before* touching Spring Boot 2.7 — kept Boot, swapped the JDK, profiled for a sprint. They discovered a GC-pause regression in their existing G1 tuning that was masked on Java 11 but visible on Java 17 with the new defaults. They tuned GC for two weeks before doing the Boot upgrade. The lesson: runtime hops and framework hops surface different classes of problem; isolating them is cheaper than diagnosing both at once.

## 6. Best Practices

- Upgrade to the latest patch release of your current major before upgrading the major (Boot 2.7.x latest before going to 3.0).
- Run OpenRewrite recipes on a dedicated branch with green tests at the start; the recipe's commit should be a single, reviewable diff.
- Add `spring-boot-properties-migrator` as a `runtime` dependency immediately after the recipe runs, capture every diagnostic, then remove it before merging.
- Survey your dependency tree against Maven Central's Jakarta releases *before* you run the recipe so you know what will fail to compile.
- Keep migration commits small and atomic so that production rollback is a single revert, not a coordinated multi-PR backout.
- Hop the runtime (Java 11 → 17) before the framework (Boot 2.7 → 3.x) so that two distinct classes of error surface in two distinct sprints.
- Write Phase-1 discoveries as a numbered list and let the Phase-2 Plan Day cite each discovery by number when picking which slice to modernize first.

## 7. Hands-on Exercise

**Whiteboard exercise — 12 minutes.** On paper or a virtual whiteboard, draw the dependency graph and migration sequence for a hypothetical legacy service with the following profile:

- Spring Boot 2.5.6, Java 11, currently in production.
- Uses `javax.servlet`, `javax.persistence`, and `javax.validation` annotations.
- Depends on a third-party PDF library (`com.acme:pdf-builder:3.1`) — assume you don't know yet whether it has a Jakarta release.
- Has 800 unit tests at 78% line coverage.
- Has no integration tests.

Your task: produce a numbered migration plan — at least seven steps, at most twelve — that gets this service to Boot 3.3 + Java 17 + Jakarta. For each step, name: (a) the artifact that proves the step is done, (b) the rollback if the step fails, and (c) which OpenRewrite recipe (if any) supports that step.

**What good looks like.** A correct answer will pre-flight to Boot 2.7.x and Java 17 *before* the Jakarta hop, will check third-party Jakarta compatibility before running any recipe, will add and then remove the properties-migrator, will write at least two integration tests against critical paths before any framework change (because the existing 78% line coverage is unit-test coverage and unit tests rarely catch framework-behavior regressions), and will explicitly name which recipe runs at which step. A weak answer collapses the hops into one step or runs the recipe before establishing integration-test ground truth.

## 8. Key Takeaways

- Can you name the four high-level activities of incremental legacy modernization and explain why they run in parallel rather than in sequence?
- Can you list the three migration hops required to go from Boot 2.7 + Java 11 + `javax.*` to Boot 3.x + Java 17 + `jakarta.*`, in the correct order, and justify the order?
- Can you describe at least three classes of change that OpenRewrite recipes handle automatically, and at least three classes of change that they leave to manual fix-up?
- Can you explain the difference between a "big bang" rewrite and a Strangler-Fig modernization, and name two failure modes of the former?
- Can you articulate how Phase-1 discoveries translate into Phase-2 modernization scope items?

## Sources

1. [Spring Boot 3.0 Migration Guide (spring-projects wiki)](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide) — retrieved 2026-05-26
2. [Preparing for Spring Boot 3.0 (Spring Blog)](https://spring.io/blog/2022/05/24/preparing-for-spring-boot-3-0) — retrieved 2026-05-26
3. [Migrate to Spring Boot 3 (OpenRewrite tutorial)](https://docs.openrewrite.org/running-recipes/popular-recipe-guides/migrate-to-spring-3) — retrieved 2026-05-26
4. [JavaxMigrationToJakarta recipe (OpenRewrite)](https://docs.openrewrite.org/recipes/java/migrate/jakarta/javaxmigrationtojakarta) — retrieved 2026-05-26
5. [Patterns of Legacy Displacement (Cartwright, Horn, Lewis on martinfowler.com)](https://martinfowler.com/articles/patterns-legacy-displacement/) — retrieved 2026-05-26

Last verified: 2026-05-26
