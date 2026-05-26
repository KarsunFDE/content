---
template: pre-session-reading
week: W04
day: Thu
phase: SDLC
topic: "Brownfield modernization execution — OpenRewrite hop + ADR review + graceful degradation + rollback rehearsal"
estimated_total_minutes: 55
last_verified: 2026-05-26
fde_situations: [3, 5, 7, 9, 10, 11]
tech: [OpenRewrite, Spring-Boot-4.0, Java-21, Spring-Framework-7.0, Spring-Security-7.0, jakarta-EE-11, javax-migration]
sources_research_briefs:
  - research/spring-boot-2-7-to-4-x-20260525.md
  - research/aws-sdk-v1-to-v2-migration-20260525.md
author: instructor
---

# W4 Thu Pre-Session — Brownfield Modernization Execution

> Read Wed night, before Thu Modernization Execution Day. ~55 min. Tomorrow is the largest execution day of W4. Per D-056, `acquire-gov` main IS the legacy stack (SB 2.7.18 + Java 11 + `javax.*` + Spring Security 5 + AWS SDK v1) — your pair modernizes it forward, in-place. The PDF column for Thu is *Brownfield Modernization*; these seven topics map 1:1 to the PDF canonical sub-topics (Legacy Modernization, Incremental Migration Approaches, Workflow Augmentation vs Replacement, Deployment Planning for Incremental Rollouts, ADR Review, Graceful Degradation, Rollback Rehearsal).

## 1. Legacy Modernization — what acquire-gov main IS (per D-056) (8 min)

The single-branch design decision matters more than any recipe you'll run tomorrow. Per D-056, there is **no sibling "modern" branch** for `acquire-gov`. Main IS the legacy stack: Spring Boot 2.7.18, Java 11, `javax.*`, Spring Security 5, AWS SDK v1. The `v0.1-legacy-baseline` git tag preserves pre-modernization state for rollback; your pair's feature branch is the working surface; PRs land into `main`.

Why this is "security work, not aesthetic": Spring Boot 2.7.18 has been **OSS-unsupported since 30 Jun 2023** (commercial Tanzu support only) per `research/spring-boot-2-7-to-4-x-20260525.md`. AWS SDK for Java 1.x reached **end-of-support 31 Dec 2025** per `research/aws-sdk-v1-to-v2-migration-20260525.md`. Acquire-gov is shipping today on two unsupported runtimes. The agency CIO's "Spring Boot 4.x posture statement by Friday" ask isn't cosmetic; it's the posture statement the OIG would ask for. **Why 4.x and not 3.5:** SB 3.5 OSS-EOLs 30 Jun 2026 — within days of Cohort #1's W4 finish; landing on a runtime that flips to commercial-only at W5 is a federal-compliance posture failure. SB 4.0 OSS-supported through 31 Dec 2026 (~6 months runway). Java 21 because SF7's recommended runtime is Java 21+.

**Question to bring to morning war-room:** *which of acquire-gov's two unsupported runtimes (SB 2.7.18 OSS-EOL or AWS SDK v1 EoS) does your service touch most invasively?* The AWS SDK v1→v2 hop is **out of scope** for tomorrow per D-054 — but it's the cross-cutting context your future-hop ADRs reference.

## 2. Incremental Migration Approaches — the 5-stage checkpoint shape (10 min)

The hop is sequenced in **2 real migration stages** plus 3 instructor-staged safety checkpoints. Java 21 lands first because Spring Boot 4.x recommends Java 21 (Java 17 floor); SB 4.0 + `javax.*`→`jakarta.*` lands second. After each stage, a **rescue branch** is created so a failed stage doesn't bury Wed's clean state. The composed `UpgradeSpringBoot_4_0` recipe walks through SB 3.x internally (it composes `UpgradeSpringBoot_3_5` + `UpgradeSpringFramework_7_0` + `UpgradeSpringSecurity_7_0`); the intermediate `stage-3.0-rewrite` checkpoint is preserved as an inspectable mid-state.

The 5-checkpoint shape (instructor-private naming; your pair works the middle three):

1. `legacy-baseline` — = `v0.1-legacy-baseline` tag, = current `main`. Your rollback anchor.
2. `stage-J21` — Java 21 only, CI green. Java first because SB 4.x recommends Java 21. Isolates Java-vs-Spring failures.
3. `stage-3.0-rewrite` — composed OpenRewrite recipe applied, paused at the SB 3.x intermediate. May have known failures. **The point of this stage is to capture what OpenRewrite did at the SB 3.x mid-state before the SF7 / SS7 sub-recipes drive forward — which is also the natural mid-rollback target.**
4. `stage-4.0` — manually fixed after the composed recipe completes through SB 4.0 + SF7 + SS7. CI green. This is the cohort's target.
5. `final-expected-PR` — instructor-private. The target diff your pair converges toward. **NOT visible until W4 Fri post-incident retro** per D-054.

Each stage gets a rescue branch named in your Tue preflight (`planning/W04/03-thu-hop-preflight.md`). If `stage-4.0` blows up, you reset to the prior rescue branch — not all the way to `legacy-baseline`. Codex Full strictness applies (D-034): unknown + uncaught failure = P0; known + documented failure with rollback = P2.

## 3. OpenRewrite hop execution — Java 21 + SB 4.0 + javax→jakarta (10 min)

OpenRewrite is a declarative AST-transform engine; recipes compose. The SB 4.0 upgrade is a composite of 30+ sub-recipes that fire from the `org.openrewrite.java.spring.boot4.UpgradeSpringBoot_4_0` entry point (Community Edition). It internally composes `UpgradeSpringBoot_3_5` + `UpgradeSpringFramework_7_0` + `UpgradeSpringSecurity_7_0` — so the SB 2.7 → 3.5 path happens inside one recipe invocation, not as a separate hop. Tomorrow's PM practical: per-pair service ownership.

- **Pair 1:** `solicitation-service` — full hop.
- **Pair 2:** `evaluation-service` — full hop.
- **Pair 3:** cross-cutting `javax.*` → `jakarta.*` (Jakarta EE 11 / Servlet 6.1 namespace) in shared modules.

Tooling pattern: `mvn rewrite:dryRun` first to capture the patch (your Tue preflight already has this baseline); then `mvn rewrite:run` to apply. Update `known-failures.md` as failures land — codex Full reads it tomorrow night. Expect the hand-edit residual to lean heavier than a SB 3.5 hop would because SS6 → SS7 + SF6 → SF7 stack on top of the javax → jakarta sweep.

Primary reading tonight:

- [Spring Boot 4.0 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Release-Notes) (~15 min read), retrieved 2026-05-26 via /web-research. Focus on Java 21 baseline (Java 17 floor), Jakarta EE 11 / Servlet 6.1, Spring Framework 7.0, JSpecify null-safety, OpenTelemetry first-party starter, modularization.
- [OpenRewrite — UpgradeSpringBoot_4_0 Recipe (Community Edition)](https://docs.openrewrite.org/recipes/java/spring/boot4/upgradespringboot_4_0-community-edition) (~10 min), retrieved 2026-05-26 via /web-research. Skim the *Definition* tree — that's every sub-recipe that fires (including the composed `UpgradeSpringBoot_3_5` + `UpgradeSpringFramework_7_0` + `UpgradeSpringSecurity_7_0`).
- [OpenRewrite — UpgradeToJava21 Recipe](https://docs.openrewrite.org/recipes/java/migrate/upgradetojava21) (~5 min), retrieved 2026-05-26 via /web-research. The Java-runtime recipe that lands `stage-J21` independently of the framework hop.
- [OpenRewrite — Running Recipes with Maven](https://docs.openrewrite.org/running-recipes/getting-started) (~10 min), retrieved 2026-05-23 via /web-research. The `mvn rewrite:run` and `mvn rewrite:dryRun` invocations.
- [Jakarta EE 9 — javax to jakarta namespace](https://jakarta.ee/specifications/platform/9/jakarta-platform-spec-9.html#_renaming) (~10 min), retrieved 2026-05-23 via /web-research. Why every `javax.servlet`, `javax.persistence`, `javax.validation` flips. (Jakarta EE 11 / Servlet 6.1 is the SB 4.0 baseline — the namespace lands at the 3.0 sub-hop, version bumps land at the 4.0 destination.)

**Question to bring to morning war-room:** *for your assigned service or shared-module work, what's the smallest representative module you can `mvn compile` against `stage-4.0` to satisfy the D-054 Pass 3 evidence floor?* (Drives Thu AM scoping.)

## 4. Workflow Augmentation vs Replacement — what OpenRewrite catches vs what you hand-edit (8 min)

The ~70/30 split is the load-bearing number: OpenRewrite handles **~60-70%** of the SB 2.7→4.0 hop deterministically per `research/spring-boot-2-7-to-4-x-20260525.md` (slightly lower than the SB 3.5 hop's ~70% because SS6 → SS7 + SF6 → SF7 add residual on top of the javax → jakarta sweep); **~30-40%** requires hand edits. The principle the cohort exercises tomorrow is **augmentation, not replacement** — the human owns the judgment call on every hand edit; the tool does the mechanical sweep.

Canonical hand-edit clusters to expect from your `dryRun` patch:

- Custom `javax.servlet.Filter` implementations — the namespace rewrite is clean, but interactions with Spring Security 5 filter chains often need manual wiring against the `SecurityFilterChain` bean pattern (introduced in SS6, unchanged in SS7 — only the version pin moves).
- Custom `WebSecurityConfigurerAdapter` subclasses — **removed in Spring Security 6.x, remains absent in SS7.** Hand-rewrite as a `SecurityFilterChain` bean (snippet in `research/spring-boot-2-7-to-4-x-20260525.md` §5).
- Spring Cloud Sleuth → Micrometer Tracing — distributed-tracing config keys + bean wiring shift. SB 4.0 adds a first-party `spring-boot-starter-opentelemetry` that further alters defaults — expect to revisit observability config.
- `application.properties` boolean-property strictness (per SB 3.5 tightening, carries into 4.0) — `.enabled` properties now require strict `true`/`false`; anything ≠ `false` no longer means "enabled."
- SF6 → SF7 deprecated-API residuals + JSpecify null-safety annotation propagation — new in 4.0; expect hand work on any custom Spring beans that referenced now-removed APIs.

The augmentation/replacement framing is **also the W5 AIOps preview**: tomorrow's hand edits are decisions a human owns; Wed's HITL #6 authority-boundary table named which kinds of decisions humans always own. Same principle, different surface.

## 5. ADR Review — future-hop ADRs Java 25 + SF7-minor with D-054 Pass 3 evidence floor (8 min)

Two future-hop ADRs land tomorrow as evidence-backed paper (NOT executed):

- `planning/W04/adrs/03-java-25-future-hop.md` — what changes if you continued the Java hop from 21 to 25 next quarter (virtual-thread optimisation continuing the LTS line).
- `planning/W04/adrs/04-sf7-minor-future-hop.md` — what changes if you continued forward through SB 4.x minor releases (e.g., SF7.0 → SF7.1) as they GA. SB 4.x is on a ~6-month minor cadence.

**The Pass 3 evidence floor** (revised after codex Pass 3 called pure-paper ADRs P1):

1. Run OpenRewrite `dryRun` from `stage-4.0` toward the next hop (`UpgradeToJava25` for the Java ADR; SB 4.x minor recipes as they land for the SF7 ADR).
2. Capture the generated patch or recipe report.
3. Run at least `mvn compile` on one representative module — or record expected blockers from the dry run.
4. ADR cites tool output, not just docs.
5. **≥1 required hand edit identified** (prevents pure tool-demo).

If your ADR cites only the Spring Boot 4.0 release notes and says "looks straightforward," codex Full will flag it P1.

Additional reading for the future-hop ADRs:

- [Spring Boot 4.0 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Release-Notes) (~10 min), retrieved 2026-05-26 via /web-research.
- [OpenRewrite — UpgradeToJava25 Recipe](https://docs.openrewrite.org/recipes/java/migrate/upgradetojava25) (~5 min), retrieved 2026-05-26 via /web-research. The Java 21 → 25 hop recipe (composed: chains UpgradeToJava21 + UpgradeToJava17 internally).
- *(reference)* [Spring Boot 4.0.0 available now](https://spring.io/blog/2025/11/20/spring-boot-4-0-0-available-now), published 2025-11-20, retrieved 2026-05-25 via /web-research. Confirms the ~6-month minor cadence + Java 25 first-class support that anchors both future-hop ADRs.

[!instructor-review] *4.0.x OSS support runs through 31 Dec 2026 per `spring-boot-2-7-to-4-x-20260525.md`. If a 4.1.x line is now GA by the cohort delivery date, the SF7-minor future-hop ADR may need to reframe as "4.0 → 4.1" rather than "SF7.0 → SF7.1 minor." Surface to instructor for current-week verification.*

## 6. Graceful Degradation + Rollback Rehearsal (8 min)

The Thu PM final block before EOD PR. Two PDF topics merged because they're operationally one drill: **prove you can fail back cleanly** before you ship.

**Graceful degradation rehearsal** — your pair forces a partial-hop state and watches the gateway. `solicitation-service` on SB 3.0, `evaluation-service` still SB 2.7. Does `api-gateway` route degrade cleanly when one downstream is on a newer Spring Security 6 / Jakarta runtime and the other is not? Per D-054, `api-gateway` stays on SB 2.7 this week — your cross-version routing IS the rehearsal.

**Rollback rehearsal** — your pair walks the steps to restore the `v0.1-legacy-baseline` git tag (or the most recent rescue branch), and confirms: (a) `mvn compile` still succeeds against the rollback target, (b) the audit-log writer still emits, (c) the SB 2.7 actuator endpoints still respond. The rehearsal **is the safety net** for Friday — whatever surprises tomorrow, your pair has walked the rollback path once today.

Codex Full strictness reads your rollback documentation. Known + documented failure with rollback path = P2; unknown + uncaught failure = P0. Document the rollback you rehearsed.

## 7. Deployment Planning for Incremental Rollouts (5 min)

Brief because **today's hop doesn't deploy to a real environment** — it lands as a merged PR with green CI on `stage-4.0`. Real AWS deploy (OIDC → ECS) comes in W5 per D-050; Bedrock InvokeModel is authorized in W2-onward per D-060, but managed services (Knowledge Bases, Agents-for-Bedrock, OpenSearch Managed) and Bedrock Guardrails stay deferred to W5.

The incremental-rollout sequence the PR represents:

1. PR opens from per-pair feature branch into `main`.
2. Codex Full review fires; P0/P1 findings block merge.
3. Instructor sign-off after codex clears.
4. Merge to `main`.
5. (W5) Canary deploy via GitHub Actions OIDC → AWS.

The W5 anchor week brings the deploy surface; today's PR is the **artifact** the deploy will consume. Phrase your `known-failures.md` and your ADR rollback plan as if a stranger picks up the deploy next month — because in production, they will.

## 8. Further reading (optional) + tomorrow's questions

Optional deep-dive (NOT required to participate):

- [Spring Framework 7 — What's New](https://docs.spring.io/spring-framework/reference/7.0/spring-framework-overview.html) (~15 min), retrieved 2026-05-26 via /web-research. Java 21+ recommended runtime, portfolio-wide JSpecify null-safety, and the modularization headline that SB 4.0 inherits — that's the SB 4.x posture statement Karsun delivery would lean on.
- [Micrometer + Micrometer Tracing migration from Spring Cloud Sleuth](https://docs.micrometer.io/tracing/reference/migration.html) (~15 min), retrieved 2026-05-23 via /web-research. Background for the W5 OpenTelemetry work; tomorrow's hop touches the Sleuth → Micrometer Tracing flip. Note: SB 4.0 ships a first-party `spring-boot-starter-opentelemetry` that further alters defaults.

**Two questions to come in with tomorrow:**

1. *For your assigned service (`solicitation-service` or `evaluation-service`) or shared-module work, what's the smallest representative module you can `mvn compile` against `stage-4.0` to satisfy the D-054 Pass 3 evidence floor?* (Drives Thu AM scoping.)
2. *OpenRewrite handles ~60-70% of the SB 2.7→4.0 hop deterministically. Which 30-40% requires hand edits — and how do you catch it in the `stage-3.0-rewrite` → `stage-4.0` transition?* (Drives Thu PM execution.)

Last verified: 2026-05-26
