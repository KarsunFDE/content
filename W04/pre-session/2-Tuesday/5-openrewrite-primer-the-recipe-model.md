---
week: W04
day: Tue
topic_slug: openrewrite-primer-the-recipe-model
topic_title: "OpenRewrite primer — the recipe model"
parent_overview: W04/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://docs.openrewrite.org/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.openrewrite.org/concepts-and-explanations/recipes
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.openrewrite.org/running-recipes/running-rewrite-on-a-maven-project-without-modifying-the-build
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.openrewrite.org/recipes/java/migrate/jakarta/javaxmigrationtojakarta
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# OpenRewrite primer — the recipe model

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define an OpenRewrite *recipe*, *visitor*, and *Lossless Semantic Tree (LST)* and explain how they relate.
- Distinguish *imperative*, *declarative* (YAML), *Refaster template*, and *scanning* recipes by their use cases.
- Run a single OpenRewrite recipe against a Maven project from the command line without modifying the build.
- Use `dryRun` to preview changes and read the resulting patch before applying it.
- Estimate the proportion of a migration OpenRewrite can automate vs the proportion that requires hand edits.

## 2. Introduction

OpenRewrite is an open-source automated-refactoring engine for source code. Per the project's own one-line description (docs.openrewrite.org, retrieved 2026-05-26), it is "an auto-refactoring engine that runs prepackaged, open-source refactoring recipes for common framework migrations, security fixes, and stylistic consistency tasks — reducing your coding effort from hours or days to minutes." Its original focus was Java; community work is expanding it to other languages.

Why this matters for brownfield modernization: most of the cost of a framework upgrade is *not* the conceptual work (knowing that `javax.persistence` becomes `jakarta.persistence`). The cost is the *mechanical labor* of touching every import, every annotation, every XML descriptor, every dependency coordinate, in the correct order, without breaking anything. A 200-class service might have 1,500 individual edits in a Jakarta EE 9 migration. OpenRewrite turns that into one command.

The model is *recipes operating on Lossless Semantic Trees*. The LST is OpenRewrite's parsed representation of source code that preserves whitespace, comments, and formatting — so when a recipe prints the tree back to source code after a change, the unchanged parts of the file look identical to the original. This is the key property that makes OpenRewrite output reviewable as a normal PR diff: the changes are minimally invasive.

The reason a brownfield-modernization curriculum spends time on OpenRewrite specifically is that the alternative — sed / awk / hand-editing across hundreds of files — produces PRs that are unreviewable. A Jakarta EE 9 migration done by sed leaves a PR with thousands of irrelevant whitespace diffs; an OpenRewrite-generated migration leaves a PR where every edit is structural. The discipline is to let the tool do its 70% mechanically and reserve human attention for the 30% it cannot handle.

## 3. Core Concepts

### 3.1 Recipes, visitors, and the LST

Three OpenRewrite concepts compose:

- **The Lossless Semantic Tree (LST)** — a structured representation of source code preserving every byte of the original (whitespace, comments, formatting). The LST is what visitors traverse.
- **A visitor** — a piece of code that walks the LST and produces an updated LST. Visitors are written in Java (`org.openrewrite.java.JavaVisitor`) and have access to type information, not just syntax.
- **A recipe** — a packaged refactoring operation. A recipe delegates to one or more visitors. Recipes can be composed: a "Migrate to Jakarta EE 9" recipe is internally a list of ~30 smaller recipes (per the OpenRewrite recipe page for `JavaxMigrationToJakarta`, retrieved 2026-05-26).

The OpenRewrite documentation's canonical illustration is `UseStaticImport`: before the recipe, `Assert.assertTrue(condition);` references the class explicitly; after, `assertTrue(condition);` is called against a static import. The visitor traversed the LST, found the matching call, restructured it, and the printer emitted source code that differs from the original only in the changed area.

### 3.2 Four recipe types

The OpenRewrite docs describe four recipe types, each suited to a different job:

- **Imperative recipes** — Java code extending `org.openrewrite.Recipe`. The most expressive; used when the transformation requires custom logic. Example: `ChangeType`, which takes old and new fully-qualified type names and rewrites every reference.
- **Declarative recipes** — YAML files that compose existing recipes. No code. Example: a `JUnit5Migration` declarative recipe lists `ChangeType`, `AssertToAssertions`, and `RemovePublicTestModifiers` as its recipe list and runs them in order.
- **Refaster template recipes** — pattern-based replacements with full compiler and type support. Ideal for simple expression-to-expression rewrites like `StringUtils.equals(..)` → `Objects.equals(..)`.
- **Scanning recipes** — recipes that need to see all source files before making changes (for example, to generate a new file based on what they discovered). Have three phases: scan, generate, edit.

Declarative recipes are the most common entry point for migration users. They give you composition without requiring you to write Java. A typical declarative recipe for a Jakarta EE 9 migration is a 30-line YAML file that aggregates the package-by-package sub-migrations.

### 3.3 Running a recipe against a Maven project

The simplest way to invoke a recipe is the `rewrite-maven-plugin`'s `run` goal, which can be applied from the command line without modifying the project's `pom.xml`. From the OpenRewrite "Running recipes" guide (retrieved 2026-05-26):

```bash
# Run a single recipe with no configuration parameters
mvn -U org.openrewrite.maven:rewrite-maven-plugin:run \
  -Drewrite.activeRecipes=org.openrewrite.java.RemoveUnusedImports
```

For a recipe from a non-core library (such as the Jakarta migration), you also specify the recipe artifact coordinates:

```bash
mvn -U org.openrewrite.maven:rewrite-maven-plugin:run \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-migrate-java:RELEASE \
  -Drewrite.activeRecipes=org.openrewrite.java.migrate.jakarta.JavaxMigrationToJakarta
```

The plugin downloads the recipe library, parses the source code into LSTs, applies the recipe, and writes the modified source back to disk. Diff the result with git to see what changed.

### 3.4 Dry runs are the safety net

The Maven plugin's `dryRun` goal performs the entire transformation but writes the patch to a file (`target/rewrite/rewrite.patch`) instead of modifying source. The patch is git-style and human-reviewable. The discipline for any non-trivial recipe is to run `dryRun` first, review the patch in a normal code review tool, *then* run `run`.

```bash
mvn -U org.openrewrite.maven:rewrite-maven-plugin:dryRun \
  -Drewrite.activeRecipes=org.openrewrite.java.migrate.jakarta.JavaxMigrationToJakarta
```

For large codebases the dry-run output can be hundreds or thousands of edits; the discipline is not to read every line but to spot-check categories (do all the `javax.servlet` imports flip cleanly? are any annotation arguments mangled? are XML files updated correctly?). Anything that looks wrong in the dry run is *certain* to be wrong in the live run.

### 3.5 The 70/30 split — what tools cannot do

OpenRewrite handles a high but never-complete fraction of any migration. For Spring Boot 2→3 + Jakarta EE 9, the practical experience is that ~70% of edits are handled deterministically and ~30% require human attention. Categories OpenRewrite struggles with:

- **Custom subclasses of removed abstract classes** — for example, `WebSecurityConfigurerAdapter` is removed in Spring Security 6; a recipe can detect the subclass exists but cannot reliably rewrite custom override methods into the new `SecurityFilterChain` bean shape, because the override logic is arbitrary.
- **Reflection and dynamic class loading** — anywhere a class name appears as a string (`Class.forName("javax.persistence...")`), the recipe cannot follow.
- **Configuration files outside the recipe's domain** — recipes know `pom.xml`, `application.yml`, and `web.xml`; they may not know your custom Helm chart, Terraform module, or shell script that mentions an old package.
- **Behavior changes the recipe authors didn't anticipate** — a library may have changed a default value (e.g., a connection-pool size, a timeout) without changing its API. The recipe cannot detect this; only your test suite can.

The discipline is to enumerate the 30% before running the recipe (this is exactly what section 6 of the brownfield-analysis ADR is for). After the recipe runs, the team's job is to handle the named-in-advance categories and triage anything that surfaces unexpectedly.

## 4. Generic Implementation

A worked example: a generic `customer-portal` service is migrating from `javax.*` to `jakarta.*` using OpenRewrite's `JavaxMigrationToJakarta` recipe. The team has already written a brownfield-analysis ADR and is now executing the hop.

**Step 1 — Capture a baseline.** Before any change, record the current state:

```bash
# Tag the legacy baseline so rollback is always one git command away.
git tag legacy-baseline-pre-jakarta
git push origin legacy-baseline-pre-jakarta

# Capture current test pass count and build time.
mvn -B clean verify | tee pre-migration.log
```

**Step 2 — Run a dry run.** Generate the patch without applying it:

```bash
mvn -U org.openrewrite.maven:rewrite-maven-plugin:dryRun \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-migrate-java:RELEASE \
  -Drewrite.activeRecipes=org.openrewrite.java.migrate.jakarta.JavaxMigrationToJakarta \
  -Drewrite.exportDatatables=true
```

The patch lands at `target/rewrite/rewrite.patch`. Open it in any review tool. Spot-check:

- Every `javax.persistence` import becomes `jakarta.persistence`.
- Every `javax.servlet` import becomes `jakarta.servlet`.
- `web.xml` schema version updated.
- `persistence.xml` xmlns updated.

Anything weird (a flipped import that does not exist, an annotation argument changed unexpectedly) — stop here and read the recipe source on GitHub.

**Step 3 — Run the recipe live.**

```bash
mvn -U org.openrewrite.maven:rewrite-maven-plugin:run \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-migrate-java:RELEASE \
  -Drewrite.activeRecipes=org.openrewrite.java.migrate.jakarta.JavaxMigrationToJakarta
```

This modifies source files in place. Diff with git to see the same changes the dry run showed.

**Step 4 — Build and test.**

```bash
mvn -B clean verify
```

Almost always there are compile errors at this point. They are the 30%. Categories the team named in advance — custom `javax.servlet.Filter` subclasses, reflection on package names, custom XML — get hand-edited. The build/test cycle runs until green.

**Step 5 — Commit and review.**

```bash
git checkout -b feat/jakarta-migration
git add .
git commit -m "Migrate javax.* to jakarta.* via OpenRewrite

Recipe: org.openrewrite.java.migrate.jakarta.JavaxMigrationToJakarta
Hand-edits applied: AuthFilter (servlet API change), 3 reflection
sites in CachedReflectionUtil, web.xml schema (manual update)."
git push origin feat/jakarta-migration
```

The commit message names the recipe and the hand-edit categories. This is what makes the PR adversarially reviewable: a reviewer can verify the recipe handled what it claims to have handled and the hand-edits cover what the ADR predicted.

## 5. Real-world Patterns

**Logistics platform — OpenRewrite shrank a quarter's modernization to a sprint.** A logistics company maintaining ~40 microservices needed to upgrade them all from Spring Boot 2.5 to 2.7 before Spring's OSS-support window closed. Initial estimate (hand-edits, traditional code review): one quarter, two engineers. They ran the OpenRewrite Spring Boot 2.7 recipe across all 40 services in a single sprint. The recipe handled the bulk of dependency-version bumps and minor API renames. Hand-edits totaled ~80 across all 40 services; most were related to a custom logging framework that referenced internal Spring classes by name. The savings were not the engineers' typing speed; they were the *reviewer's* ability to read a PR where every diff was structural rather than mechanical.

**Healthcare data warehouse — Refaster recipes caught security anti-patterns.** A healthcare data warehouse used Refaster template recipes to enforce coding standards across a 600-class data-pipeline codebase. Each rule was a template like "replace `String.matches(regex)` with `Pattern.compile(regex).matcher(s).matches()`" to avoid recompiling regular expressions on every call. The team layered 23 Refaster rules and ran them weekly. Over six months, the codebase eliminated ~3,000 instances of the slow pattern without anyone writing imperative migration code. The Refaster recipes also caught new occurrences in PRs through the same machinery, turning the migration into ongoing lint.

**E-commerce marketplace — scanning recipe generated module-specific configuration files.** An e-commerce marketplace had 90 microservices that needed identical health-check endpoint definitions added to their Helm charts. They wrote a custom scanning recipe: phase 1 (scan) detected each service's `pom.xml` and accumulated a list of service names; phase 2 (generate) created a `health-check.yaml` per service with the service name interpolated; phase 3 (edit) added a reference to the new file from each service's main Helm chart. One recipe run produced 270 file changes deterministically. Without scanning recipes, this would have been a python script with all the maintenance burdens of an internal tool.

**Gaming backend — OpenRewrite was rejected for a UI codebase, retained for backend.** A multiplayer game's backend team adopted OpenRewrite enthusiastically; their frontend team tried it for an Angular upgrade and found the LST coverage for TypeScript was much thinner than for Java. The frontend team retained OpenRewrite for their Java-based BFFs but used a different tool (jscodeshift) for the Angular code. The lesson: OpenRewrite's maturity varies sharply by language; the JVM ecosystem is its strongest area. Other languages may or may not justify the investment.

## 6. Best Practices

- **Always `dryRun` before `run`** — non-trivial recipes can have surprising scope; the dry-run patch is your last review before disk is modified.
- **Tag a baseline commit before any migration recipe** — `git tag legacy-baseline-<purpose>` is two seconds of insurance against a runaway recipe.
- **Run one recipe at a time when learning** — declarative composition is powerful but masks which sub-recipe touched what; compose only after you understand the building blocks.
- **Read the recipe's source for anything mission-critical** — recipes are open source on GitHub; for a recipe touching auth or money, read it before you trust it.
- **Pin recipe versions in CI** — `RELEASE` is fine for exploration; production CI should pin to a known recipe version so reruns are deterministic.
- **Enumerate the 30% in the ADR before running the recipe** — if section 6 is empty, you do not yet know enough to authorize the run.
- **Commit the recipe-generated changes separately from the hand edits** — two commits ("recipe run" + "hand edits for the 30%") make the PR vastly easier to review than one mega-commit.

## 7. Hands-on Exercise

**Exercise: Dry-run a recipe against an unfamiliar project (15 min).**

Find any open-source Java + Maven project with at least one `javax.persistence` or `javax.servlet` import (most pre-2024 Spring projects qualify). Clone it locally. Without modifying anything:

1. Run `mvn -U org.openrewrite.maven:rewrite-maven-plugin:dryRun -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-migrate-java:RELEASE -Drewrite.activeRecipes=org.openrewrite.java.migrate.jakarta.JavaxMigrationToJakarta`.
2. Open `target/rewrite/rewrite.patch`.
3. Count the number of edited files and the total number of line changes.
4. Identify three sub-categories of edits (e.g., import flips, annotation changes, descriptor file updates).
5. Identify at least one edit that looks risky or that would benefit from human review.

**What good looks like:** the dry-run completes without errors, the patch is non-empty, you can name three sub-categories of edits with examples, and you can point to one edit that you would want to read carefully before committing. If the patch is empty, the project is already migrated — pick another.

A common surprise the first time: the patch is much larger than expected. A Spring Boot 2.5 → 2.7 + Jakarta EE 9 composite migration on a 50-class service can produce 800-line patches. The exercise teaches calibration — *how much change OpenRewrite represents per command* — which is the intuition you need to estimate hop scope on your own systems.

## 8. Key Takeaways

- *What is OpenRewrite?* — An automated refactoring engine that runs prepackaged recipes against Lossless Semantic Trees to produce reviewable, structural source code changes.
- *What are the four recipe types?* — Imperative (Java code), declarative (YAML composition), Refaster template (pattern replacements), and scanning (cross-file aware).
- *How do you run a recipe safely?* — Tag a baseline, `dryRun` first, review the patch, then `run`, then build/test.
- *What is the 70/30 split?* — Recipes handle ~70% of a typical migration deterministically; the remaining 30% requires hand edits the team should name in advance.
- *When does OpenRewrite not fit?* — When the codebase is in a language with thin LST coverage, when transformations require runtime context the LST cannot see, or when the migration is so small that hand-editing is faster than configuring a recipe.

## Sources

1. [OpenRewrite Docs — Home](https://docs.openrewrite.org/) — retrieved 2026-05-26
2. [OpenRewrite Docs — Recipes (concepts)](https://docs.openrewrite.org/concepts-and-explanations/recipes) — retrieved 2026-05-26
3. [OpenRewrite Docs — Running Rewrite on a Maven project without modifying the build](https://docs.openrewrite.org/running-recipes/running-rewrite-on-a-maven-project-without-modifying-the-build) — retrieved 2026-05-26
4. [OpenRewrite Recipe — Migrate to Jakarta EE 9 (`JavaxMigrationToJakarta`)](https://docs.openrewrite.org/recipes/java/migrate/jakarta/javaxmigrationtojakarta) — retrieved 2026-05-26

Last verified: 2026-05-26
