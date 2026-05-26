---
template: pre-session-reading
week: W04
day: Tue
phase: SDLC
topic: "Requirement decomposition + Integration mapping + API Modernization Patterns + Brownfield-analysis ADR + Validation-as-spec"
estimated_total_minutes: 50
last_verified: 2026-05-26
fde_situations: [3, 5, 7, 9, 10]
tech: [spec-driven-dev, OpenRewrite, GitHub-Actions, Spring-Cloud-Gateway, integration-mapping, validation-as-spec]
sources_research_briefs: [research/spring-boot-2-7-to-3-x-20260525.md]
author: instructor
---

# W4 Tue Pre-Session — Requirement Decomposition + Brownfield Planning

> Read Mon night after Plan Day, before Tue's spec-driven workshop. ~50 min. You authored two ADRs Mon afternoon (W4 Modernization Scope + AI Security Threat Model). Tonight prepares you to (a) walk acquire-gov's integration seams as decomposed requirements, (b) plan Thu's OpenRewrite hop as a per-module brownfield-analysis ADR, and (c) discover Tuesday morning that your prior week's PRs were never actually linted.
>
> **Cohort-#1 calendar note:** Checkpoint 2 exam 09:00–10:30 in-person per D-061 — the workshop opens *after* the exam at 10:30. Walk in fresh; the discovery is the discipline.

## 1. Integration Mapping — walking acquire-gov's service boundaries (8 min)

PDF column lead: **Integration Mapping + System-Boundary Analysis.** Before you modernize anything, you map what crosses what. `acquire-gov` has four services plus a frontend (per D-054 + D-056, this is the legacy stack and main IS the modernization target):

- Angular 17+ SPA (Officer Dashboard, Vendor Portal)
- `api-gateway` (Spring Boot 2.7.18, Java 11, the auth + routing edge)
- `solicitation-service` (Spring Boot 2.7.18, Java 11, javax.*)
- `evaluation-service` (Spring Boot 2.7.18, Java 11, javax.*)
- `ai-orchestrator` (Python FastAPI, calls Bedrock)

The integration map is the *graph of who calls whom* — and where that graph has been bypassed. **Debt item 8** is the warm-up: the Angular Officer Dashboard hardcodes `http://localhost:8081/api/solicitations` and skips the gateway entirely. The map says "frontend → gateway → service." The code says "frontend → service direct." That seam is your Tuesday-PM target.

Map the integration on paper Mon night for your assigned service: every inbound call (who calls in), every outbound call (who you call), every datastore touched (Postgres or Atlas). Bring the map to the war-room workshop.

> **Question to bring to morning war-room:** *"Open your Officer Dashboard component (or equivalent). Trace one outbound HTTP call. Does the request show up in the api-gateway access log? If not — where is it going?"*

## 2. API Modernization Patterns — Spring Cloud Gateway, the SB 3.x edge (8 min)

PDF column lead: **API Modernization Patterns.** The Tue PM warm-up: fix debt item 8 via Spring Cloud Gateway predicates + filters, not by editing the Angular code. The pattern: the edge owns routing; the frontend never knows a service hostname.

Spring Cloud Gateway 4.x is the SB 3.x-compatible edge proxy. Its building blocks:

- **Predicates** — match-conditions on inbound requests (path, header, method, host). `Path=/api/solicitations/**` is the predicate that catches your debt-item-8 traffic.
- **Filters** — transformations applied per route (strip prefix, rewrite path, add header, retry, circuit-break). `StripPrefix=1` removes `/api/` before forwarding.
- **Routes** — predicate + filter chain + downstream URI.

For Tuesday's fix you write *one route* in `api-gateway`'s application config that catches Officer Dashboard traffic and routes it to `solicitation-service`. The frontend changes one line (the base URL). The gateway owns the routing forever after.

This is the broader API-modernization principle: **the edge is policy; the service is logic.** Auth, routing, retries, rate-limiting, observability — these live at the edge. Business rules live in the service. Brownfield debt accumulates when that line gets crossed (debt item 8 = service hostname leaked to client; debt item 1 = JWT signature validation skipped at edge for "public" paths).

[Spring Cloud Gateway — Predicates and Filters, retrieved 2026-05-23 via /web-research, https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway/gateway-request-predicates-factories.html]

## 3. Brownfield Planning + Complete Brownfield-analysis ADR for Single Module (10 min)

PDF column lead: **Brownfield Planning + Complete Brownfield-analysis ADR for Single Module.** Tuesday afternoon, each pair plans the Thu OpenRewrite hop *for one specific service* (yours) and writes the brownfield-analysis ADR that authorizes the hop.

A brownfield-analysis ADR is a different shape from Mon's scope ADR. It names *the existing system as it stands*:

1. **Module under analysis** — `solicitation-service` (or `evaluation-service`, or the shared `javax.*` modules — assigned at war-room).
2. **Current state pinned** — Spring Boot 2.7.18, Java 11, javax.persistence + javax.servlet + javax.validation, Spring Security 5.x (`WebSecurityConfigurerAdapter`), AWS SDK v1 (`com.amazonaws.*`).
3. **Target state pinned** — Spring Boot 3.5.x, Java 17, `jakarta.*`, Spring Security 6.x (`SecurityFilterChain` bean). AWS SDK v1→v2 hop is **W5 work per D-050** — flag it, don't execute it Thu.
4. **OpenRewrite recipes applied** — `org.openrewrite.java.migrate.UpgradeToJava17` + `org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_5`. (NOT `UpgradeSpringBoot_3_0` — pin 3.5 per the SB 3.x research brief: 3.4.x OSS support already ended 31 Dec 2025; 3.5 is the current OSS-supported target.)
5. **Pre-flight checks named** — Java 17 toolchain in container (no host JDK pinning), `mvn rewrite:dryRun` baseline captured, rescue branch name committed.
6. **Hand edits required (the 30%)** — D-054 Pass 3 evidence floor says you must name ≥1 hand edit OpenRewrite won't handle. Likely candidates: custom `javax.servlet.Filter` subclasses, `WebSecurityConfigurerAdapter` subclasses, Spring Cloud Sleuth → Micrometer Tracing.
7. **Rollback** — the `v0.1-legacy-baseline` git tag (per D-056). No sibling "modern" branch exists — main IS the legacy stack being modernized forward.

This ADR is the artifact that lets Codex Full (lands Thu per D-034) say *"yes, this hop was planned, not improvised."*

## 4. OpenRewrite primer — the recipe model (8 min)

OpenRewrite is a refactoring engine that applies *recipes* (declarative AST transforms) to a codebase. For Spring Boot 2.7→3.5 + Java 11→17 + `javax.*` → `jakarta.*`, OpenRewrite composes ~30 sub-recipes — one command, hundreds of edits, deterministic output. It does roughly 70% of the work; you hand-edit the remaining 30%.

The Thu hop's 5-checkpoint shape (instructor's private safety-net naming, per W04 PLAN §Thu — NOT cohort-target branches):

```
legacy-baseline → stage-J17 → stage-3.0-rewrite → stage-3.0 → final-expected-PR
```

You work the J17 + 3.0-rewrite + 3.0 stages on a per-pair feature branch into `main` (per D-056). Each stage gets a rescue branch. Updates land in `planning/W04/known-failures.md` as failures appear.

**`dryRun` is your friend.** Run `mvn rewrite:dryRun` first to see the patch without applying it. D-054 Pass 3 evidence floor requires `dryRun` output in your future-hop ADRs (SB 3.5 → 4.0 land Thu as evidence-backed paper ADRs).

[OpenRewrite Spring Boot 3.0 Migration Recipe, retrieved 2026-05-23 via /web-research, https://docs.openrewrite.org/recipes/java/spring/boot3/upgradespringboot_3_0]
[Spring Boot 3.5 Release Notes, retrieved 2026-05-25 via /web-research (Firecrawl), https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.5-Release-Notes] — focus on the breaking-changes section + the `heapdump` actuator + `.enabled` tightenings.

**`jakarta.*` migration vocabulary** (terms you'll hear Tue and Thu): every `javax.persistence`, `javax.servlet`, `javax.validation` import flips. `WebSecurityConfigurerAdapter` (Spring Security 5) is removed in 6 → replaced by `SecurityFilterChain` bean. The full 2.7.18 baseline is 3 generations behind OSS support (per `research/spring-boot-2-7-to-3-x-20260525.md` — 2.7 OSS support ended 30 Jun 2023). That's the framing for the agency CIO's Friday exec brief: "we are on unsupported software."

## 5. Validation-as-spec Discipline — the meta-joke (8 min)

PDF column lead: **Validation-as-spec Discipline + End-to-end Validation Across Microservices + Domain-specific Validation Surface.** This is the Tue AM workshop opener post-Checkpoint-2-exam, and it lands as guided self-discovery.

Three weeks of merged PRs across `acquire-gov` + your pair-project repos all "passed CI." Tomorrow you walk those PRs back and confirm whether your CI actually checks what you think it checks. You open the lint output locally — `ruff check API/`, `eslint UI/`, `mvn checkstyle:check` — and the volume shocks you. The question becomes: **why didn't CI catch any of this?**

What you'll find: `.github/workflows/ci.yml` has the lint job gated `if: false` — **debt item 12 from W1 Tue's inventory.** Your PRs were merged green because lint never ran. The point isn't blame — the point is the OIG-Findings-Tracker meta-joke (per `feature-inventory-target.md` line 387): *the first finding you open this week is against your own CI*, via `POST /api/findings` on `evaluation-service`. Finding format per line 237: `finding_type`, `severity`, `evidence_requests[]`, `remediation_due`.

This is **validation-as-spec discipline** — the spec said clean CI; we proved it without checking that "clean CI" meant "lint ran." Spec-driven dev fails when nobody checks the spec is actually enforced.

The **end-to-end** and **domain-specific** layers expand the question: a spec says "no PII in audit logs." Is that validated end-to-end across the 4 services? A spec says "every solicitation amendment cites a FAR clause." Is that validated on the domain surface (the `POST /draft-amendment` endpoint), or just at the database write? Tuesday's workshop seeds the question; Wednesday's AI Security day exercises the answers.

## 6. ADR Writing Discipline — Brownfield ADR shape vs Scope ADR shape (6 min)

PDF column lead: **ADR Writing Discipline** (Tue's appearance — Mon also had ADR writing as a topic).

Two ADR shapes show up in W4:

- **Scope ADR** (Mon's shape) — *what we're doing this week and why.* Names debt items in scope, decisions, alternatives considered, rollback. Codex Full strictness applies — P0/P1 findings block merge per D-034.
- **Brownfield-analysis ADR** (Tue's shape — see topic §3) — *what the existing module IS and what one specific hop will change.* Pins current/target state, OpenRewrite recipes applied, hand-edit floor, rollback tag.

A scope ADR without a per-module brownfield ADR is paper. A per-module brownfield ADR without a scope ADR is improvisation. Both ship Mon–Tue; both gate Thu's hop.

A third ADR shape lands Thursday: **future-hop ADR** (SB 3.5 → 4.0, Java 17 → 21). D-054 Pass 3 floor: ≥1 OpenRewrite `dryRun` per future hop + ≥1 representative-module compile attempt + named breaking changes + ≥1 hand edit identified. **Future-hop ADRs are evidence-backed paper exercises — they ship in Fri Live Defense and get cut**, per D-054. Not executed this week.

## 7. Further reading (optional) + tomorrow's questions

- *(optional)* [Spring Security 5 → 6 migration](https://docs.spring.io/spring-security/reference/5.8/migration/index.html) (~20 min), retrieved 2026-05-23 via /web-research. Useful background; per D-056 you do execute `WebSecurityConfigurerAdapter` → `SecurityFilterChain` on `solicitation-service` + `evaluation-service` because they're hopping to SB 3.5. Read this if your assigned service has a custom security config.
- *(optional)* [OpenRewrite — Authoring a custom recipe](https://docs.openrewrite.org/authoring-recipes/recipe-development-environment) (~10 min). Skip unless you intend to ship a custom recipe for a per-pair-unique debt item (per D-059).
- *(optional)* [GitHub Actions — Conditional job execution (`if:`)](https://docs.github.com/en/actions/using-jobs/using-conditions-to-control-job-execution) (~5 min), retrieved 2026-05-23 via /web-research. So you recognise the `if: false` pattern when you see it Tuesday morning.
- Reference brief: `research/spring-boot-2-7-to-3-x-20260525.md` (3.5.14 current + 2.7.18 OSS-EOL + `jakarta.*` migration vocabulary).

**Two questions to bring to Tuesday war-room** (workshop opens with these):

1. *"Open your three most-recent merged PRs and the GitHub Actions runs for each. Did lint actually run? If not — when did it stop, and who would have noticed?"*
2. *"Frontend currently hits `http://localhost:8081/api/solicitations` directly from the Officer Dashboard (debt item 8). What's the simplest change that routes it through the gateway without changing the Angular code's mental model?"*

Last verified: 2026-05-26
