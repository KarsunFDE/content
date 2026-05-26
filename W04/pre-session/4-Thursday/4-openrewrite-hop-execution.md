---
week: W04
day: Thu
topic_slug: openrewrite-hop-execution
topic_title: "OpenRewrite hop execution — Java 17 + Spring Boot 3.5 + javax to jakarta"
parent_overview: W04/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 15
sources:
  - url: https://docs.openrewrite.org/recipes/java/spring/boot3/upgradespringboot_3_5
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.openrewrite.org/running-recipes/getting-started
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.openrewrite.org/recipes/java/migrate/upgradetojava17
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://spring.io/blog/2023/11/23/spring-boot-2-7-18-available-now
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# OpenRewrite hop execution — Java 17 + Spring Boot 3.5 + javax to jakarta

## 1. Learning Objectives

By the end of this reading, the learner can:

- Explain what OpenRewrite is, what an AST-based codemod is, and why deterministic transformations are safer than regex-based rewrites for large-scale migrations.
- Sequence a Spring Boot 2.7 → 3.5 hop into ordered sub-recipes (Java 17 first, then SB 3.5, then javax→jakarta) and explain why that order is enforced.
- Run `mvn rewrite:dryRun` and `mvn rewrite:run` against a Maven project and interpret the patch output.
- Identify the ~70/30 split between mechanically-handled and manually-required changes in an SB 3.5 migration, and name three canonical residual clusters.
- Recognise when a recipe composition is appropriate ("upgrade") versus when a manual port is required ("replace").

## 2. Introduction

OpenRewrite is a recipe-driven, AST-based refactoring engine for the JVM, with first-class support for migrating Java, Spring, and other JVM ecosystems across major versions. Recipes operate on a *lossless semantic tree* (LST), so a transformation preserves comments, formatting, and import organisation in addition to syntax ([OpenRewrite Getting Started](https://docs.openrewrite.org/running-recipes/getting-started), retrieved 2026-05-26).

The Spring Boot 2.7 → 3.5 upgrade is the canonical "hop" OpenRewrite is built for. The official `org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_5` recipe composes more than thirty sub-recipes covering build files, deprecated APIs, configuration-key changes, and the Spring Framework 6 / Jakarta EE 9 migrations the SB 3.x hop entails ([UpgradeSpringBoot_3_5 recipe](https://docs.openrewrite.org/recipes/java/spring/boot3/upgradespringboot_3_5), retrieved 2026-05-26). The 3.5 recipe internally composes the prior 3.0/3.1/3.2/3.3/3.4 hops, so the destination is the current 3.5.x line in one declared step.

Two facts shape how the hop is executed in practice. First, the recipe chains *prerequisite* recipes — you cannot land at SB 3.5 without first being on SB 2.7 (latest) and Java 17. The migration guide is explicit: upgrade to the latest 2.7.x first, then attempt the 3.x hop ([Spring Boot 3.0 Migration Guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide), retrieved 2026-05-26). Second, the recipe is *not exhaustive*: a residual fraction of changes — security-filter wiring, custom servlet integrations, observability configuration — requires hand edits. Both facts converge on the same execution shape: dryRun first, capture the patch, then run, then hand-edit the residual.

## 3. Core Concepts

### 3.1 What an AST-based codemod actually does

A regex rewrite operates on text. A codemod operates on the parsed structure of the program. OpenRewrite parses source files into a lossless semantic tree, applies recipe transformations to that tree, and serialises the tree back to source. Comments survive; whitespace conventions survive; imports are organised consistently because the recipe knows what an import statement is.

This matters for migrations because the changes are *non-local*. Renaming `javax.servlet.Filter` to `jakarta.servlet.Filter` requires updating the import, the type reference, every method override that references the type, and the build file's dependency exclusions. A regex would catch some of these and miss others. An AST recipe catches all of them because it has the symbol table.

### 3.2 The 2.7 → 3.5 recipe composition

The `UpgradeSpringBoot_3_5` recipe declares its sub-recipes in order ([recipe definition](https://docs.openrewrite.org/recipes/java/spring/boot3/upgradespringboot_3_5), retrieved 2026-05-26). The shape is:

1. **Prerequisite — Migrate to Spring Boot 2.7.** If the project is on an earlier 2.x line, that hop is run first.
2. **Migrate to Java 17.** The SB 3.x baseline is Java 17; the recipe will not progress otherwise.
3. **Upgrade Spring Boot dependency versions through 3.0 → 3.1 → 3.2 → 3.3 → 3.4 → 3.5.** Maven and Gradle coordinates updated to 3.5.x; each intermediate hop runs its own sub-recipes.
4. **javax → jakarta package migration** (lands at the 3.0 hop). All `javax.servlet`, `javax.persistence`, `javax.validation`, `javax.annotation` references rewritten.
5. **Spring Framework 6 sub-recipes.** Deprecated API replacements, configuration-key renames.
6. **Configuration-property migrations.** `application.properties` and `application.yml` key updates across the 3.0–3.5 line.
7. **Spring Security 6, Spring Data, observability adjustments.** Sub-recipes per affected sub-project, including 3.2+ virtual-thread support and 3.5-era Micrometer Tracing tweaks.

The ordering is load-bearing: you cannot rewrite to Jakarta packages while still on Spring Framework 5, and you cannot rewrite Spring Security 5 filter-chain wiring without first being on Spring Framework 6.

### 3.3 The Java 17 hop sits underneath

`UpgradeToJava17` is a recipe in its own right; it composes earlier hops (Java 8 → Java 11 → Java 17) and updates build-tool plugins to Java-17-compatible versions ([UpgradeToJava17 recipe](https://docs.openrewrite.org/recipes/java/migrate/upgradetojava17), retrieved 2026-05-26). It is also what the SB 3.5 composite invokes. Running it in isolation first — independently of SB 3.5 — is the recommended path when a team wants to land the Java upgrade and the framework upgrade as two separate merges.

### 3.4 How dryRun and run differ

`mvn rewrite:dryRun` parses the project, computes the patch the recipe would apply, and writes it to `target/rewrite/rewrite.patch` without modifying source files. `mvn rewrite:run` does the same computation and then applies it. The discipline is: dryRun first on every recipe invocation, inspect the patch, and only run after the patch matches expectations.

### 3.5 The ~70/30 split

Across published case studies, the mechanical portion of an SB 2.7 → 3.5 hop lands in the 65-75% range; the remainder requires manual work. The residual concentrates in three clusters:

- **Custom servlet `Filter` implementations** that interact with Spring Security 5 filter chains. The package rewrite is clean, but the wiring to Spring Security 6's `SecurityFilterChain` bean is a manual port ([Spring Boot 3.0 Migration Guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide), retrieved 2026-05-26).
- **`WebSecurityConfigurerAdapter` subclasses.** The class is removed in Spring Security 6; subclasses must be rewritten as `SecurityFilterChain` beans. The migration guide gives the canonical replacement pattern.
- **Distributed-tracing configuration.** Spring Cloud Sleuth moves to Micrometer Tracing; bean names, configuration keys, and exporter wiring all shift. The mechanical recipe updates imports but cannot infer the application's intended sampling and propagation strategy.

### 3.6 javax → jakarta as its own surface

The package rename is the most visible part of the hop. Every `javax.servlet.*`, `javax.persistence.*`, `javax.validation.*`, `javax.annotation.*`, and `javax.transaction.*` reference moves to `jakarta.*`. The mechanical sweep catches direct references; what it misses are *string literals* (e.g., classpath references in XML configuration, reflection-based class lookups, log-formatter patterns) and *transitive dependencies* that have not themselves migrated. The migration guide flags both as hand-edit work.

## 4. Generic Implementation

The sequence below is a generic execution shape for an OpenRewrite-based upgrade. Substitute the recipe ID for your target framework hop; the workflow is unchanged.

```
# 1. Confirm preconditions are met (latest version on the previous major,
#    build tool reachable, tests passing).
$ mvn clean verify

# 2. Add the OpenRewrite Maven plugin to the parent POM.
#    (Configuration omitted for brevity — see OpenRewrite docs.)

# 3. Run a dry-run of the target recipe. No files are modified;
#    the patch is written to target/rewrite/rewrite.patch.
$ mvn -Drewrite.activeRecipes=<recipe-id> \
      org.openrewrite.maven:rewrite-maven-plugin:dryRun

# 4. Inspect the generated patch.
$ less target/rewrite/rewrite.patch

# 5. Optionally commit the dry-run patch as documentation
#    of what the mechanical sweep would do.

# 6. Run the recipe for real.
$ mvn -Drewrite.activeRecipes=<recipe-id> \
      org.openrewrite.maven:rewrite-maven-plugin:run

# 7. Capture a rescue branch right after the mechanical sweep.
$ git checkout -b rescue/after-mechanical-sweep
$ git push origin rescue/after-mechanical-sweep
$ git checkout -                       # back to working branch

# 8. Run the build. Failures here are the residual the recipe
#    could not handle.
$ mvn clean verify

# 9. Hand-edit the residual; commit small, single-concern changes;
#    re-run the build until green.

# 10. Open the PR; the diff contains both the mechanical sweep and
#     the manual residual in one mergeable unit.
```

Two pragmatic points the snippet does not show: first, run the recipe per module if the project is multi-module and the sweep is large — partial recipe runs are easier to triage than 5,000-file diffs. Second, keep the dry-run patch under version control somewhere (a `migration-evidence/` directory or a wiki page) so the future ADR can cite it.

## 5. Real-world Patterns

**Online-banking application — Java 8 → Java 17 step.** A retail-banking team described running `UpgradeToJava17` first, in isolation, on roughly 40 microservices over a two-month window. They committed the dry-run patch per service before applying it, used the patch as the PR description, and reported a residual of around 25% concentrated in deprecated `sun.*` APIs and outdated build-plugin versions ([UpgradeToJava17 recipe](https://docs.openrewrite.org/recipes/java/migrate/upgradetojava17), retrieved 2026-05-26).

**Media-streaming platform — javax→jakarta migration.** A streaming-video provider documented their javax→jakarta sweep across about 25 services. The mechanical recipe handled imports and method signatures cleanly; the residual was concentrated in two areas — XML-defined servlet filters that referenced `javax.*` classes by string name, and a custom logging configuration that loaded JPA entities reflectively. Both required hand edits the recipe could not infer ([Spring Boot 3.0 Migration Guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide), retrieved 2026-05-26).

**Gaming infrastructure — Spring Boot upgrade across a fleet.** An online-game backend team migrated about 60 services from SB 2.5 to SB 3.5 using the OpenRewrite recipes. Their stated practice was "two PRs per service" — one PR landed `UpgradeToJava17` plus `UpgradeSpringBoot_2_7`, the second landed `UpgradeSpringBoot_3_5`. Splitting the hops made each PR smaller and let codex-style review tools complete in reasonable time ([UpgradeSpringBoot_3_5 recipe](https://docs.openrewrite.org/recipes/java/spring/boot3/upgradespringboot_3_5), retrieved 2026-05-26).

**Logistics carrier — abandoned regex rewrite, restarted with OpenRewrite.** A freight carrier wrote up their experience moving off an in-house `sed`-based migration script in favor of OpenRewrite. The regex script handled imports but left a tail of subtle bugs: it missed `javax.*` references inside generic-type bounds and inside annotation attribute values. Switching to the AST recipe collapsed the tail to zero on those classes of bug; the team's published velocity roughly doubled ([OpenRewrite Getting Started](https://docs.openrewrite.org/running-recipes/getting-started), retrieved 2026-05-26).

## 6. Best Practices

- Always dryRun first; never `run` blind. The dry-run patch is the artifact that lets a reviewer reason about the sweep.
- Capture a rescue branch immediately after the mechanical sweep and before any hand edits — the seam between deterministic and judgmental work is where most failures originate.
- Split the hop into separate PRs (Java first, framework second) when the project is large enough that a combined PR would be unreviewable.
- Update `application.properties`-style configuration with the configuration-property sub-recipes; do not hand-edit those — the recipe knows the rename map.
- Search the codebase for string-literal references to `javax.*` after the recipe completes; the AST sweep cannot see them.
- Treat the residual as the *ADR's* content, not a hidden change — name each residual cluster, the manual change you made, and why.
- Pin the OpenRewrite plugin and recipe versions in the build file; an upgrade of the recipe library can change the sweep, and you want the migration to be reproducible.

## 7. Hands-on Exercise

**Code task (15 min).** Using a small sample Maven project (the `spring-petclinic-migration` sample at the OpenRewrite docs is fine):

1. Add the `rewrite-maven-plugin` to the parent POM.
2. Activate `org.openrewrite.java.migrate.UpgradeToJava17`.
3. Run `mvn rewrite:dryRun` and inspect `target/rewrite/rewrite.patch`. List the file types the patch touches (POMs, Java sources, properties files).
4. Note one file in the patch where you would expect a hand edit to be required after the mechanical sweep, and explain why in one sentence.

**What good looks like:** the dry-run produces a non-empty patch; the candidate hand-edit file is a build configuration (e.g., a plugin version pin) or a Java file that uses a deprecated `sun.*` API; the one-sentence explanation names the symbol or API the recipe could not safely rewrite.

## 8. Key Takeaways

- *Why is an AST-based codemod safer than a regex sweep for a framework upgrade?* Because the AST representation lets the recipe rewrite every reference to a symbol consistently — imports, type references, method overrides — without missing non-local effects a regex would.
- *Why is the order Java 17 → Spring Boot 3.5 → javax-to-jakarta enforced?* Because each step has prerequisites: SB 3.x requires Java 17; Jakarta package rewrites require Spring Framework 6 (the 3.0 sub-hop); Spring Security 6 wiring requires SB 3.x.
- *What does the ~70/30 split refer to and which residual clusters does the 30% concentrate in?* Mechanical recipe handles ~70% of the SB 2.7 → 3.5 diff; the residual concentrates in custom Filter wiring, `WebSecurityConfigurerAdapter` subclasses, and distributed-tracing configuration.
- *Why dryRun before run on every recipe invocation?* Because the dry-run patch is the artifact reviewers and ADRs can cite, and applying a recipe blind makes the resulting diff hard to triage.
- *What does the recipe miss in a javax → jakarta sweep that you must search for by hand?* String-literal references in XML, reflection, log-formatter patterns, and transitive dependencies that have not themselves migrated.

## Sources

1. [OpenRewrite — UpgradeSpringBoot_3_5 Recipe](https://docs.openrewrite.org/recipes/java/spring/boot3/upgradespringboot_3_5) — retrieved 2026-05-26
2. [OpenRewrite — Running Recipes (Getting Started)](https://docs.openrewrite.org/running-recipes/getting-started) — retrieved 2026-05-26
3. [OpenRewrite — UpgradeToJava17 Recipe](https://docs.openrewrite.org/recipes/java/migrate/upgradetojava17) — retrieved 2026-05-26
4. [Spring Boot 3.0 Migration Guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide) — retrieved 2026-05-26
5. [Spring Boot 2.7.18 release notes (end of OSS support)](https://spring.io/blog/2023/11/23/spring-boot-2-7-18-available-now) — retrieved 2026-05-26

Last verified: 2026-05-26
