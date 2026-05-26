---
week: W04
day: Thu
topic_slug: deployment-planning-for-incremental-rollouts
topic_title: "Deployment Planning for Incremental Rollouts"
parent_overview: W04/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://martinfowler.com/bliki/BlueGreenDeployment.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/bliki/CanaryRelease.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/bliki/ParallelChange.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/bliki/FeatureToggle.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://sre.google/sre-book/handling-overload/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Deployment Planning for Incremental Rollouts

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the four canonical incremental-rollout strategies (blue-green, canary, feature flag, expand-and-contract) and the failure profile each one is designed for.
- Identify which strategy fits a given change shape — runtime upgrade, schema change, new user-facing feature, internal refactor.
- Explain why a successful CI pipeline is necessary but not sufficient for a successful production rollout, and what observability is required at the rollout boundary.
- Sequence the artifacts of a single PR (commits, CI signals, codex/peer review, merge, deploy, canary, full rollout) into a defensible release plan.
- Recognise the difference between *deploy* (binary lives on the target) and *release* (users see the change), and use the distinction in planning.

## 2. Introduction

Incremental rollout is the practice of moving a change into production in small, observable, reversible steps. The big-bang alternative — push the change to all users at once and hope — is the historical baseline; the incremental discipline replaces hope with measurement.

Four strategies dominate the published practice. **Blue-Green Deployment** keeps two identical production environments and switches traffic between them at the router, providing an instant rollback path ([Blue Green Deployment](https://martinfowler.com/bliki/BlueGreenDeployment.html), retrieved 2026-05-26). **Canary Release** rolls the new version out to a small subset of users first, monitors metrics, and ramps if the metrics look healthy ([Canary Release](https://martinfowler.com/bliki/CanaryRelease.html), retrieved 2026-05-26). **Feature Flag** rollouts decouple deploy from release entirely — the new code ships to production but is gated by configuration ([Feature Flag](https://martinfowler.com/bliki/FeatureToggle.html), retrieved 2026-05-26). **Expand-and-Contract** (also called Parallel Change) handles backward-incompatible interface changes by supporting both old and new APIs simultaneously while consumers migrate ([Parallel Change](https://martinfowler.com/bliki/ParallelChange.html), retrieved 2026-05-26).

The strategies are not mutually exclusive — many real rollouts combine two or three. What they share is the underlying premise: a deploy that is observable and reversible is qualitatively safer than one that is fast.

## 3. Core Concepts

### 3.1 Deploy vs release — the load-bearing distinction

A **deploy** is the act of putting a new binary, container image, or function bundle onto the target infrastructure. A **release** is the act of making that new code visible to users. In a feature-flag-driven workflow the two are separated by hours, days, or weeks. The deploy is low-risk because the new code is dormant; the release is the high-risk moment, and the team controls when it happens.

In a blue-green deployment the two are separated by seconds — the deploy lands the new version on the idle environment, and the release happens when the router switches. In a classic single-step deploy, the two happen at the same instant. The more sophisticated the rollout strategy, the further apart deploy and release move.

### 3.2 Blue-Green Deployment

Two production environments are kept as identical as possible. One serves traffic ("blue"); the other is staged with the new version ("green"). Final pre-release validation happens against green. When the team is satisfied, the router switches green to live and blue becomes the standby. Rollback is the same router switch in reverse ([Blue Green Deployment](https://martinfowler.com/bliki/BlueGreenDeployment.html), retrieved 2026-05-26).

Blue-green is appropriate when:
- The cost of running two production environments is acceptable.
- The application's data layer can tolerate the cutover (often via expand-and-contract).
- Rollback speed matters and "switch the router back" is the operational primitive.

### 3.3 Canary Release

A small subset of users (often by random assignment, by geographic region, or by internal-employee cohort first) is routed to the new version while the rest stay on the old. If metrics — error rate, latency, business KPIs — degrade, the canary is halted. If they hold, the canary ramps in stages: 5%, 25%, 50%, 100% ([Canary Release](https://martinfowler.com/bliki/CanaryRelease.html), retrieved 2026-05-26).

Canary is appropriate when:
- The change affects user-visible behavior and you want production-traffic validation before full release.
- The team has the observability to distinguish canary metrics from baseline metrics at the cohort level.
- The change is large enough that the risk of a full-population rollout outweighs the operational cost of running both versions.

### 3.4 Feature Flag rollouts

The new code ships to production but is gated by a runtime configuration. The flag may be off in production for days while telemetry confirms the new code path is not exercised; then the flag flips on for a small cohort, then ramps. The flag remains in the codebase until the new behavior is established and the old code path is removed ([Feature Flag](https://martinfowler.com/bliki/FeatureToggle.html), retrieved 2026-05-26).

Feature flags are appropriate when:
- The change is conditional on user, tenant, region, or experimental cohort.
- The team needs the ability to flip behavior in seconds without a redeploy.
- The deploy and release schedules need to be decoupled (e.g., the engineering team ships when ready; the product team releases when marketing is ready).

### 3.5 Expand-and-Contract for schema and API changes

When the change is a backward-incompatible interface modification, the rollout shape is necessarily different — both versions of the interface must coexist long enough for consumers to migrate. The pattern is three phases: **expand** (introduce the new interface alongside the old), **migrate** (move consumers to the new interface), **contract** (remove the old interface once no consumers remain) ([Parallel Change](https://martinfowler.com/bliki/ParallelChange.html), retrieved 2026-05-26).

The pattern applies to API versions, database schemas, message-bus message formats, and library-public interfaces. Skipping the expand phase forces all consumers to upgrade simultaneously, which is the failure mode the pattern is designed to avoid.

### 3.6 Combining strategies for a runtime upgrade

A framework or runtime upgrade (Spring Boot 2.7 → 3.0, Node 18 → 20, Python 3.11 → 3.12) often combines several strategies:

- The PR that ships the new runtime lands on main and deploys to a canary instance.
- Behind a feature flag at the gateway, a small percentage of traffic routes to the new-runtime instance.
- If error rates and latency look healthy, the flag ramps; if not, the flag flips back and the canary is recycled.
- After full rollout, the flag is removed and the old-runtime instances are decommissioned.

The combined pattern reuses the artifacts of each strategy. The PR is the artifact a code review evaluates; the canary is the artifact production observability evaluates; the flag is the artifact the on-call engineer can flip during an incident.

### 3.7 Why CI is necessary but not sufficient

A green CI pipeline tells the team that the code compiles, the unit tests pass, and the integration tests pass in the CI environment. It does *not* tell the team that the change will behave correctly under production load, with production data shapes, alongside production dependencies. The canary and the flag-ramp are how the team learns the second category.

The Google SRE book chapter on Handling Overload describes the load-test/canary boundary explicitly: capacity behavior under real production traffic is rarely the same as under synthetic load, because real traffic includes the distribution of edge cases the synthetic load did not anticipate ([Handling Overload](https://sre.google/sre-book/handling-overload/), retrieved 2026-05-26). The incremental rollout is how the team gets that information without paying for it in a customer-visible incident.

## 4. Generic Implementation

The sequence below is a generic deployment plan for a single PR that introduces a runtime or framework upgrade. The names of stages are intentionally generic.

```
DEPLOYMENT PLAN — PR #<n>

1. Pre-deploy
   - CI: green on the PR branch (unit + integration tests).
   - Code review: at least one human reviewer + automated review (codex, eng-bot).
   - ADR: linked from the PR description if architecturally significant.
   - Rollback rehearsed in staging: yes / no (yes is required).

2. Deploy (binary on target, no user-visible change)
   - Build artifact: <image tag or release version>.
   - Target: canary instance only (1 of N pods/instances).
   - Flag state: feature.runtime_v3 = OFF for 100% of traffic.

3. Smoke (synthetic confirmation on canary)
   - Health endpoints respond on the canary instance.
   - Synthetic transaction against the canary returns expected shape.
   - No new ERROR-level log lines in 5 minutes.

4. Release wave 1 (5% of traffic)
   - Flag flipped on for 5% of traffic, weighted to canary instance.
   - Observability gate: p50 latency within ±10% of baseline;
     p99 within ±20%; error rate within +0.1pp of baseline.
   - Wait window: 30 minutes.

5. Release wave 2 (25%)
   - Same gate; same wait window.

6. Release wave 3 (100%)
   - Flag at 100%; canary instance is now a normal production
     instance running the new runtime.

7. Post-release
   - Old runtime instances decommissioned over the next deploy.
   - Flag scheduled for deletion in two weeks.
   - Rollback procedure documented as a runbook,
     with the on-call engineer named.

ROLLBACK PROCEDURE (at any wave)
- Flag flipped to OFF for 100% of traffic — < 60 seconds.
- Canary instance recycled to old runtime image.
- Incident ticket opened; postmortem scheduled.
```

The plan is deliberately concrete. Numbers are placeholders, but the *shape* — observability gates, named waves, explicit rollback — is the load-bearing content.

## 5. Real-world Patterns

**Social platform — mobile app rollout via feature-flag-driven canary.** A major social platform documented a release process in which the new app code ships to TestFlight, then to a 1% production cohort, then ramps in 5/25/50/100 stages over a week. Engagement-metric regressions are detected at the 5% stage and remediated before broader release. The pattern combines canary release with feature-flag gating at the API layer ([Canary Release](https://martinfowler.com/bliki/CanaryRelease.html); [Feature Flag](https://martinfowler.com/bliki/FeatureToggle.html), retrieved 2026-05-26).

**Cloud storage — blue-green deployment for the metadata service.** A managed cloud-storage provider described its metadata-service rollout as blue-green with database expand-and-contract. The new metadata-service version deploys to the green environment alongside an expanded schema; consumers continue reading from both old and new columns; after the router switches green to live, the old columns are dropped in a follow-up release ([Blue Green Deployment](https://martinfowler.com/bliki/BlueGreenDeployment.html); [Parallel Change](https://martinfowler.com/bliki/ParallelChange.html), retrieved 2026-05-26).

**E-commerce — checkout-service runtime upgrade.** A retail engineering team described upgrading the checkout service's runtime (a major Java version hop) using the combined pattern from Section 3.6: PR landed, canary instance deployed, flag ramped from 1% to 100% over 24 hours, with rollback rehearsed against staging the day before. The 25% wave caught a connection-pool tuning regression that did not surface in CI; flag flipped back, fix shipped, ramp resumed.

**Healthcare — clinical-notes service API expand-and-contract.** A health-tech provider modified the schema of its clinical-notes API to add structured fields. The expand phase introduced the new fields as optional; the migrate phase moved each consumer service over time; the contract phase deprecated the un-structured field nine months after expand. The deprecation window was set by the longest-tail consumer's release calendar, not by the API team's preference ([Parallel Change](https://martinfowler.com/bliki/ParallelChange.html), retrieved 2026-05-26).

## 6. Best Practices

- Separate deploy from release in the plan, even when the gap is small; the discipline saves you when the gap needs to grow during an incident.
- Define the observability gates between waves before the deploy starts; "we'll watch the dashboard" is not a gate.
- Rehearse the rollback before the deploy, not after the deploy fails; a rehearsed rollback is an asset, an unrehearsed one is a hope.
- Use expand-and-contract for any backward-incompatible interface change; never skip the expand phase to "save time" on schema or API changes.
- Treat the canary instance as production — not as "almost production" — and instrument it identically to baseline.
- Name the wait window between waves explicitly; the window is what gives observability time to surface a regression.
- Schedule the feature flag's deletion at the time you create the flag; flags that outlive their purpose become technical debt.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** Pick one change you have shipped (or expect to ship) in the next quarter. Draft a deployment plan using the seven-section structure from Section 4. Be specific about:

1. The artifact (image tag, version, package name).
2. The wave percentages and wait windows.
3. The observability gates — exact metrics, exact thresholds.
4. The rollback procedure — exact command or flag flip.
5. The deletion schedule for any flag introduced.

When you finish, swap with a partner. The reviewer's job is to find the *missing* number — the wait window left as "TBD," the threshold expressed as "looks fine," the rollback step that ends with "and then…"

**What good looks like:** the plan reads like a runbook a stranger could execute; every threshold is a number; every wait window is in minutes; the rollback step ends at a known-green state, not at "we figure it out."

## 8. Key Takeaways

- *What is the difference between deploy and release, and why does the distinction matter?* Deploy puts the binary on the target; release makes it visible to users; separating them is how the team controls risk during a rollout.
- *Which strategy fits a major framework or runtime upgrade?* Most often a combined pattern — canary instance + feature-flag-gated traffic ramp + observability gates + named rollback.
- *Why is a green CI pipeline necessary but not sufficient?* Because production traffic includes edge cases CI did not anticipate; the canary and the ramp are how the team learns those edge cases without paying for them in an incident.
- *When is expand-and-contract the right pattern?* For any backward-incompatible interface change — APIs, schemas, message formats, library interfaces — because consumers cannot migrate atomically.
- *What is the load-bearing artifact of a good deployment plan?* The numbers: wave percentages, wait windows, observability thresholds, rollback time bound. Plans without numbers are wishes; plans with numbers are runbooks.

## Sources

1. [Blue Green Deployment (Martin Fowler)](https://martinfowler.com/bliki/BlueGreenDeployment.html) — retrieved 2026-05-26
2. [Canary Release (Danilo Sato / Martin Fowler)](https://martinfowler.com/bliki/CanaryRelease.html) — retrieved 2026-05-26
3. [Parallel Change / Expand and Contract (Danilo Sato / Martin Fowler)](https://martinfowler.com/bliki/ParallelChange.html) — retrieved 2026-05-26
4. [Feature Flag (Martin Fowler)](https://martinfowler.com/bliki/FeatureToggle.html) — retrieved 2026-05-26
5. [Handling Overload — Google SRE Book Chapter 21](https://sre.google/sre-book/handling-overload/) — retrieved 2026-05-26

Last verified: 2026-05-26
