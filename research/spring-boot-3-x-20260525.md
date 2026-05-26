---
template: research-brief
tech: Spring Boot 3.x / 4.x (Java back-end framework)
version_pinned: "3.5.14 (current 3.x LTS-grade, OSS ends 30 Jun 2026); 4.0.6 (current major GA)"
last_verified: 2026-05-25
last_verified_via: web-research (Firecrawl self-hosted)
recency_window: foundation-stable 12mo
sources_count: 6
target_weeks: [W04, W05]
candidates_deferred: []
known_bad_patterns_flagged:
  - id: spring-boot-2x-javax
    note: "Any tutorial showing `import javax.persistence.*` / `import javax.servlet.*` is Spring Boot 2.x era (pre-3.0). Spring Boot 3.x mandates Jakarta EE 9+ (`jakarta.*`). Brief counter-teaches with explicit `jakarta.*` namespace."
  - id: spring-boot-2x-security-config
    note: "`WebSecurityConfigurerAdapter` (Spring Security 5.x) is removed in Spring Security 6.x (Spring Boot 3.x+). Brief shows the component-based `SecurityFilterChain` bean pattern."
  - id: spring-boot-2x-actuator-defaults
    note: "Spring Boot 3.5 tightened the `heapdump` actuator endpoint to `access=NONE` by default. Older tutorials that say 'just expose heapdump' are now misleading — must both expose AND configure access."
known_bad_patterns_checked: true
author: research-subagent
---

# Tech research brief — Spring Boot 3.x / 4.x

Last verified: 2026-05-25 · Recency window applied: foundation-stable 12mo · Pinned versions: 3.5.14 (current 3.x) + 4.0.6 (current 4.x major)

## 1. What it is (3–5 sentences, no jargon)

Spring Boot is the convention-over-configuration overlay on the Spring Framework that auto-wires sensible defaults for HTTP servers, persistence, security, observability, and packaging so a Java service can be `java -jar`-launched without XML. Spring Boot 3.x (GA November 2022) is the **Jakarta EE 9+ generation** — every `javax.*` package the cohort sees in the acquire-gov 2.7.18 baseline must become `jakarta.*` to compile against Spring Framework 6.x. Spring Boot 4.x (GA 20 Nov 2025, current 4.0.6) is the **modularization + Spring Framework 7 generation** — smaller jars, portfolio-wide null-safety via JSpecify, and first-class Java 25 support while retaining Java 17 compatibility. For the Karsun FDE auditor question, the load-bearing facts are: (a) acquire-gov's Spring Boot 2.7.18 has been OSS-unsupported since 30 Jun 2023 (commercial Tanzu-only until 2029), and (b) the next OSS-supported target is either 3.5.14 (OSS ends 30 Jun 2026) or 4.0.6 (OSS ends 31 Dec 2026).

## 2. Current stable state (as of `last_verified`)

- **Latest stable releases (endoflife.date, 2026-05-25):**
  - 4.0.6 — released 23 Apr 2026 (GA line opened 30 Nov 2025); OSS support ends 31 Dec 2026
  - 3.5.14 — released 23 Apr 2026 (GA line opened 31 May 2025); OSS support ends 30 Jun 2026; commercial support until 30 Jun 2032
  - 3.4.13 — released 18 Dec 2025; OSS support **already ended 31 Dec 2025**
  - 2.7.18 (acquire-gov baseline) — OSS support ended 30 Jun 2023; commercial Tanzu/VMware until 30 Jun 2029
- **4.0.0 GA was 20 Nov 2025** (Spring blog announcement). Headline changes:
  - Complete codebase modularization → smaller, more focused jars.
  - Portfolio-wide null-safety with JSpecify (`@Nullable` propagated everywhere).
  - First-class Java 25 support; retains Java 17 baseline compatibility.
  - HTTP Service Clients (auto-config + properties for `@HttpExchange` interfaces).
  - API Versioning (`spring.mvc.apiversion.*` / `spring.webflux.apiversion.*`).
  - New `JmsClient` API auto-configured alongside existing `JmsTemplate`.
  - `spring-boot-starter-opentelemetry` first-party starter (OTLP export of metrics + traces).
  - Gradle 9 supported; milestones now published to Maven Central in addition to repo.spring.io.
- **3.5.x notable tightenings** (released May 2025, still in OSS support 1 more month):
  - `heapdump` actuator endpoint default changed to `access=NONE` (security hardening; must both `include: heapdump` AND set `endpoint.heapdump.access: unrestricted` to use it).
  - `.enabled` boolean properties now strictly require `true`/`false` (previously anything ≠ `false` was treated as enabled).
  - Profile-name validation: only `-`, `_`, letters, digits (relaxed in 3.5.1 to also allow `.`, `+`, `@`).
  - Auto-configured `TaskExecutor` bean is now only `applicationTaskExecutor`; the `taskExecutor` alias was removed.
- **3.x foundational features (still load-bearing for 4.x):** Jakarta EE 9+ namespace migration (`javax.*` → `jakarta.*`), Java 17 baseline, native-image AOT support via Spring AOT + GraalVM, virtual-thread enablement via `spring.threads.virtual.enabled=true` (Java 21+), observability via Micrometer 1.x + Micrometer Tracing, Spring Security 6 (drops `WebSecurityConfigurerAdapter` in favour of `SecurityFilterChain` bean).
- **In-flight / preview:** 4.1.0-RC1 dropped 23 Apr 2026 with API versioning enhancements, gRPC `@GrpcAdvice` exception handling, OpenTelemetry SDK env-var support, ANSI-by-default on Windows 11+.
- Known-bad-pattern check: **flagged** — see frontmatter. `javax.*` imports, `WebSecurityConfigurerAdapter`, and "just expose heapdump" all appear in Spring Boot 2.x tutorials that surface in Stack Overflow / Baeldung searches; the brief counter-teaches.

## 3. What we teach (and what we deliberately don't)

- **In scope (W4 brownfield modernization → W5 AIOps observability):** The 2.7.18 → 3.5.x upgrade path is the W4 modernization exercise on acquire-gov. Cohort touches: (a) the `javax.* → jakarta.*` mechanical sweep (OpenRewrite recipe `org.openrewrite.java.migrate.UpgradeToJava17` + `org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_5`); (b) `WebSecurityConfigurerAdapter` removal → `SecurityFilterChain` bean; (c) the Micrometer + Micrometer Tracing observability stack feeding OpenTelemetry collectors (W5 AIOps anchor); (d) Java 17 LTS as the minimum runtime; (e) virtual threads as the Project Loom integration for high-concurrency I/O (`spring.threads.virtual.enabled=true`).
- **Out of scope (deliberately):** AOT native-image compilation (GraalVM toolchain learning is too deep for 1 week — flagged as W6 "what we'd do with more time"); the 3.x → 4.x jump (acquire-gov stops at 3.5.x for the cohort; 4.0 is named in the auditor question bank as the "what would come next" answer); Spring Cloud (out of scope for the monolith-to-microservices arc — federal modernization in acquire-gov stays on Spring Boot core).
- **Misconceptions to pre-empt:** (a) "Spring Boot 2.x is still safe because Tanzu provides commercial support." — true commercially, but the cohort's audience (federal modernization) expects an OSS-supported posture; commercial-only is a procurement red flag. (b) "We can upgrade in place from 2.7 → 4.0 in one PR." — false; Spring's guidance is explicit: upgrade to 3.5 first, then to 4.0. (c) "Spring Boot 3.x means we automatically get virtual threads." — false; Java 21+ runtime + `spring.threads.virtual.enabled=true` are both required.

## 4. Recommended primary sources

- [Spring Boot project page](https://spring.io/projects/spring-boot) — accessed 2026-05-25. Pins the current version family (4.0.6 at time of verification).
- [Spring Boot 4.0 Release Notes (wiki)](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Release-Notes) — last edited 23 Jan 2026, accessed 2026-05-25. Canonical 4.0 feature list + migration entry point.
- [Spring Boot 4.0.0 available now (Spring blog)](https://spring.io/blog/2025/11/20/spring-boot-4-0-0-available-now) — published 20 Nov 2025, accessed 2026-05-25. Authoritative GA date and headline-feature framing.
- [Spring Boot 3.5 Release Notes (wiki)](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.5-Release-Notes) — last edited 13 Mar 2026, accessed 2026-05-25. Authoritative on `heapdump`/`.enabled`/profile-name tightenings.
- [Spring Boot endoflife.date](https://endoflife.date/spring-boot) — updated 1 May 2026, accessed 2026-05-25. Canonical OSS + commercial support windows for every Spring Boot generation, including the 2.7.18 acquire-gov baseline.
- [GitHub releases · spring-projects/spring-boot](https://github.com/spring-projects/spring-boot/releases) — accessed 2026-05-25. Source for current 4.0.6 + 4.1.0-RC1 + 3.5.14 patch dates.

## 5. Code reference snippets (idiomatic, current API)

**Spring Security 6.x — `SecurityFilterChain` bean (Spring Boot 3.x+):**

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> {}));
        return http.build();
    }
}
```

**Virtual threads (Spring Boot 3.2+ on Java 21+) — opt-in via property:**

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true
```

**Spring Boot 4.0 — OpenTelemetry starter (drop-in OTLP export):**

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-opentelemetry</artifactId>
</dependency>
```

```yaml
# application.yml — exports metrics + traces over OTLP to the collector
management:
  otlp:
    metrics:
      export:
        endpoint: http://otel-collector:4318/v1/metrics
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
```

**Spring Boot 4.0 — `@HttpExchange` declarative HTTP client (auto-configured):**

```java
@HttpExchange(url = "https://api.example.gov")
public interface VendorLookupService {
    @GetExchange("/vendors/{uei}")
    Vendor lookup(@PathVariable String uei);
}
```

Anti-pattern (do NOT teach as current — appears in Spring Boot 2.x tutorials):

```java
// Spring Security 5.x — REMOVED in 6.x (Spring Boot 3.x+)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests().anyRequest().authenticated();
    }
}

// javax.* imports — Jakarta EE 8 era, must become jakarta.*
import javax.persistence.Entity;       // → import jakarta.persistence.Entity;
import javax.servlet.http.HttpServletRequest; // → jakarta.servlet.http.HttpServletRequest;
```

## 6. Risks and watch-items

- **OSS-support cliff for 3.5.x is 30 Jun 2026 — 36 days from `last_verified`.** Any cohort that lands at 3.5.x in W4 will face a re-upgrade-to-4.0 ask within weeks of programme end. Auditor question bank should flag this as a "you'd recommend 4.0 directly if the team has bandwidth for one bigger jump" answer.
- **4.x is on a 6-month minor cadence.** 4.1.0 GA is likely Aug/Sep 2026; brief pins 4.0.6 today but assumes the major-version commitments hold.
- **Native-image / AOT** is real but cohort-deep — flag as a W6 "what we'd do next" but do not teach as the W4 default.
- **OpenRewrite recipes for Spring Boot 4.0** exist but are still settling; W4 demos should pin a known-good recipe version.

## 7. Alternatives the cohort should be aware of

- **Quarkus** — Red Hat's Jakarta-EE-native, GraalVM-first competitor; faster cold-start, smaller memory footprint. Surfaced as W4 scenario-alternative ("why not Quarkus?") but acquire-gov is Spring shop.
- **Micronaut** — compile-time DI competitor with similar cold-start advantages; same scenario-alternative framing.
- **Helidon (Oracle)** — Jakarta-EE-aligned, niche federal/Oracle-shop relevance.

## 8. Brief sign-off

- Drafted by: research-subagent (2026-05-25)
- Reviewed against `known-bad-patterns.yml`: **flagged** — three new patterns surfaced (javax-namespace, WebSecurityConfigurerAdapter, heapdump-defaults); brief counter-teaches.
- 1-month-release check: triggered (4.0.6 + 3.5.14 both dropped 23 Apr 2026 = 32 days ago; 4.1.0-RC1 in pre-release). Resolved by anchoring brief to 4.0.x (GA line) as primary, with 4.1 named only as in-flight.
- Approved for downstream artifact authoring: 2026-05-25

## Sources

- Spring Boot project page, spring.io, retrieved 2026-05-25 via web-research (Firecrawl). <https://spring.io/projects/spring-boot>
- Spring Boot 4.0 Release Notes, github.com/spring-projects, retrieved 2026-05-25 via web-research (Firecrawl). <https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Release-Notes>
- Spring Boot 4.0.0 available now, spring.io blog, published 2025-11-20, retrieved 2026-05-25 via web-research (Firecrawl). <https://spring.io/blog/2025/11/20/spring-boot-4-0-0-available-now>
- Spring Boot 3.5 Release Notes, github.com/spring-projects, retrieved 2026-05-25 via web-research (Firecrawl). <https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.5-Release-Notes>
- Spring Boot endoflife.date, endoflife.date, updated 2026-05-01, retrieved 2026-05-25 via web-research (Firecrawl). <https://endoflife.date/spring-boot>
- GitHub releases · spring-projects/spring-boot, github.com, retrieved 2026-05-25 via web-research (Firecrawl). <https://github.com/spring-projects/spring-boot/releases>
