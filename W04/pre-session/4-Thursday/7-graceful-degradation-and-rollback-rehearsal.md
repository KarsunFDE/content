---
week: W04
day: Thu
topic_slug: graceful-degradation-and-rollback-rehearsal
topic_title: "Graceful Degradation + Rollback Rehearsal"
parent_overview: W04/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 13
sources:
  - url: https://sre.google/sre-book/handling-overload/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/bliki/CircuitBreaker.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://principlesofchaos.org/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/bliki/FeatureToggle.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/bliki/StranglerFigApplication.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Graceful Degradation + Rollback Rehearsal

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define graceful degradation as a system property and distinguish it from "the system stays up" or "the system fails fast."
- Identify three common degradation patterns (circuit breaker, fallback path, load shedding) and the failure modes each one addresses.
- Define a rollback rehearsal and explain why the rehearsal — not the rollback documentation — is what makes a rollback reliable.
- Apply a five-element rollback rehearsal checklist to a real change, and identify which elements are most often skipped.
- Recognise the operational link between graceful degradation and modernization: during a migration, both the old and the new path coexist, and degradation behavior must be defined for the boundary.

## 2. Introduction

Graceful degradation is the property that, when something inside a system fails, the system continues to deliver a reduced but useful version of its function rather than failing entirely. "Continues to deliver something" is the operational definition; the user does not always need the full feature set, and they almost always need *something*.

Two patterns dominate the literature. Michael Nygard's *Release It* introduced the Circuit Breaker — a wrapper around a remote call that "trips" and short-circuits when failures cross a threshold, preventing cascading collapse ([Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html), retrieved 2026-05-26). Google's SRE book chapter on Handling Overload describes load shedding — when a server is overwhelmed, it returns errors *fast* for some fraction of requests to keep latency healthy for the rest ([Handling Overload](https://sre.google/sre-book/handling-overload/), retrieved 2026-05-26). Both are degradation patterns: rather than failing the whole request, the system reshapes the failure into something bounded and observable.

Rollback rehearsal is the sibling discipline. A change that has "a rollback plan" but no rehearsal of that plan is, at best, a hopeful change. The team that rehearses its rollback discovers the missing step (the database migration that does not have a `--down`, the feature flag that does not actually gate the new code path) *before* the change is in production, not during the incident.

This reading treats the two together because in modernization work they show up at the same surface: during a single-branch hop, the old and new code paths coexist, the boundary between them must degrade gracefully, and the rollback path must have been walked at least once before it is needed.

## 3. Core Concepts

### 3.1 Three canonical degradation patterns

**Circuit Breaker.** A wrapper around a remote call that monitors failure rate and trips when a threshold is exceeded. When tripped, subsequent calls fail fast without invoking the supplier. After a cooldown, the breaker enters a half-open state and allows a probe call; if the probe succeeds, the breaker closes and normal operation resumes ([Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html), retrieved 2026-05-26). The pattern is appropriate when a downstream supplier is intermittently unavailable and the caller has critical-resource pressure (threads, connections) that would otherwise build up.

**Fallback Path.** The caller has a defined alternative when the primary path fails: a cached response, a degraded calculation, a default value, or a routing to a backup supplier. Fallback paths are appropriate when the consumer can tolerate stale or approximate data and the system value of "something now" exceeds the value of "the precise answer eventually."

**Load Shedding.** Under overload, the server returns errors fast for a fraction of requests to protect tail latency for the rest. The shedding policy may be random, priority-based (drop low-priority requests first), or quota-based (drop requests over a per-client cap). Load shedding is appropriate when the failure mode is *too much traffic*, not *broken downstream* ([Handling Overload](https://sre.google/sre-book/handling-overload/), retrieved 2026-05-26).

The three patterns are not exclusive; a mature service often uses all three. The decision for each surface is: which failure mode does this surface most need to defend against?

### 3.2 Graceful degradation during a migration

When a system is mid-migration — some services on the old runtime, others on the new — the inter-service boundaries are where degradation most often shows up first. Examples:

- A service on the new runtime expects a header the old service does not yet send. Without a default, requests fail.
- The old service serializes a payload in a format the new service no longer accepts. Without a translator, requests fail.
- A new service uses a connection pool whose semantics differ from the old service's pool. Under load, the new service exhibits a different tail-latency profile.

A migration that defines its degradation behavior at each cross-version boundary survives the in-between state. A migration that leaves degradation undefined hopes the migration finishes before the boundary is tested. Hope is not a strategy.

### 3.3 Rollback rehearsal — the five elements

A rollback rehearsal is a *walk-through* of the rollback plan against a real (or staging) system, performed before the change ships. The five elements:

1. **Identified rollback target.** The exact commit, tag, image, or release the system reverts to. Named precisely, not "the previous version."
2. **Executable rollback procedure.** The exact commands or pipeline invocations. If the procedure is "click the button," the rehearsal includes clicking the button.
3. **Data-state plan.** What happens to data written during the change window. If the schema changed, the rollback plan addresses the schema in both directions.
4. **Observability of the rolled-back state.** After the rollback, the team can confirm — within minutes — that the system is on the rollback target. Specific metrics or logs, not "things look fine."
5. **Time bound.** The estimated time to complete the rollback. The team has rehearsed within that bound, not "should be quick."

A rollback that has been rehearsed at least once — even imperfectly — beats a documented rollback that has never been run.

### 3.4 The relationship to chaos engineering

The Principles of Chaos Engineering frame degradation and rollback as part of the same discipline: rather than waiting for production to surface weaknesses, run controlled experiments that reveal them in advance. The first principle is to define *steady state* as a measurable output of normal behavior, then hypothesize that steady state will persist under perturbation ([Principles of Chaos Engineering](https://principlesofchaos.org/), retrieved 2026-05-26).

A rollback rehearsal is a small, scoped chaos experiment: the perturbation is "we revert the most recent change," and the hypothesis is "the system returns to steady state within N minutes." The team that runs the experiment before the change ships has the discipline; the team that runs it only during a real incident is doing chaos engineering involuntarily.

### 3.5 Feature flags as a degradation lever

Feature flags belong in the degradation toolbox. A flag that gates a new code path can be flipped off in seconds, which is faster than redeploying. The feature-flag literature treats this as decoupling deploy from release ([Feature Flag](https://martinfowler.com/bliki/FeatureToggle.html), retrieved 2026-05-26), and the operational consequence is that a regression detected in production can often be remediated with a flag flip rather than a full rollback. The flag does not eliminate the need for rollback — it is one tier above rollback in the response hierarchy.

### 3.6 Degradation, rollback, and the cost of doing nothing

There is a temptation, especially during modernization, to skip degradation and rollback prep because "we'll be fast enough that we don't need them." This usually reflects an under-estimation of how often migrations partially fail. The Strangler Fig literature is explicit about this: half-completed migrations are the default outcome of legacy modernization, not the exception ([Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html), updated 22 Aug 2024, retrieved 2026-05-26). Degradation and rollback are insurance against the *most likely* outcome, not the worst case.

## 4. Generic Implementation

The example below shows a circuit breaker plus a fallback path in front of a remote service call. The naming is intentionally generic.

```java
// Generic circuit breaker + fallback wrapping a remote call.
//
// Pseudocode — production implementations should use a
// well-maintained library (e.g., Resilience4j) and add metrics.

public final class PriceService {
    private final CircuitBreaker breaker;
    private final RemotePriceClient remote;
    private final LocalPriceCache cache;

    public Price fetchPrice(String sku) {
        if (breaker.isOpen()) {
            // Fast-fail path — no remote call attempted.
            return fallback(sku, "circuit-open");
        }
        try {
            Price p = breaker.execute(() -> remote.getPrice(sku));
            cache.put(sku, p);
            return p;
        } catch (RemoteUnavailableException e) {
            // Breaker counted the failure; return cached or default.
            return fallback(sku, "remote-unavailable");
        }
    }

    private Price fallback(String sku, String reason) {
        Price cached = cache.get(sku);
        if (cached != null) {
            telemetry.recordDegradedResponse(sku, reason, "cached");
            return cached;
        }
        telemetry.recordDegradedResponse(sku, reason, "default");
        return Price.unavailableDefault(sku);
    }
}
```

The pattern produces three observable outcomes: success (normal path), cached fallback (degraded with stale data), and default fallback (degraded with no data). The telemetry call is load-bearing — degradation that is not observable is degradation that cannot be reasoned about during an incident.

## 5. Real-world Patterns

**Streaming media — playback service.** Major video-streaming platforms publish post-mortems describing degradation tiers: when the metadata service is unavailable, players fall back to a cached manifest and resume playback with no recommendation row; when the recommendation engine is slow, the homepage renders without personalised rows; when CDN latency spikes, players reduce bitrate before pausing. Each tier is named, instrumented, and rehearsed against fault-injection drills ([Principles of Chaos Engineering](https://principlesofchaos.org/), retrieved 2026-05-26).

**Fintech — payments processor with circuit-breakers per acquirer.** A merchant payments platform described its multi-acquirer architecture as a circuit-breaker per acquirer, each with its own failure threshold and cooldown. When one acquirer trips, traffic shifts to the next-preferred acquirer; the failed acquirer is probed periodically. The pattern absorbed an acquirer outage during a Black Friday peak with no customer-visible impact in the published post-mortem ([Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html), retrieved 2026-05-26).

**E-commerce — load shedding under flash-sale traffic.** A retail engineering team described its load-shedding configuration during product launches: requests over a per-tier quota return HTTP 503 with a `Retry-After` header; carts and checkouts get priority over browse traffic; the policy is tuned per launch based on the previous launch's tail-latency profile ([Handling Overload](https://sre.google/sre-book/handling-overload/), retrieved 2026-05-26).

**Gaming — multiplayer matchmaking with rollback rehearsals.** A studio described its release process as a "rollback rehearsal before every deploy" — the on-call engineer literally runs `./rollback.sh` against a canary environment as part of the pre-deploy checklist. Defects in the rollback script (a missing migration step, a stale image tag) have been caught this way an average of once per quarter, well before the rollback was needed in anger.

**Logistics — warehouse cutover with rehearsed rollback.** A freight operator's warehouse-management cutover used a flag-driven ramp from old to new system. Before the first ramp, the team ran a rollback rehearsal in staging: flipped the flag back, confirmed the old system resumed processing, and verified no orders were stuck in the boundary state. The rehearsal exposed a missing index on the old system's queue table that would have caused a slow rollback in production — fixed before the real cutover ([Feature Flag](https://martinfowler.com/bliki/FeatureToggle.html), retrieved 2026-05-26).

## 6. Best Practices

- Name the degradation behavior for every remote call your service makes; "we'll handle it when it fails" is not a behavior.
- Instrument degraded responses separately from successes — a service that silently degrades is a service whose health you cannot reason about.
- Rehearse the rollback at least once per significant change; "we documented it" is not equivalent to "we ran it."
- Time-box the rollback — the rehearsal produces a real number, not "should be quick."
- Pair circuit breakers with cached fallbacks; a breaker that trips with no fallback is just a fancy error.
- Treat feature flags as the first response tier and rollback as the second; a flag flip is faster and reversible.
- After any production incident, ask: would the degradation pattern we have today have prevented this? If not, the post-mortem includes a degradation-pattern change.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** Pick one external dependency in a service you understand (a third-party API, a database, a cache, a message broker). Sketch on paper:

1. The degradation pattern you would apply at this dependency — circuit breaker, fallback path, or load shedding.
2. The specific failure threshold (e.g., "five consecutive failures within 30 seconds") or shedding policy.
3. The fallback behavior — what does the caller receive when the breaker is open or the fallback path activates?
4. The telemetry signal that distinguishes a degraded response from a normal one.
5. A five-element rollback rehearsal for the most recent change to this service: target, procedure, data-state plan, observability, time bound.

**What good looks like:** the failure threshold is a specific number (not "after a few failures"); the fallback returns a defined value (cached, default, or error code with `Retry-After`); the telemetry signal is a named metric (`degraded_response_total{reason="circuit-open"}`, not "some logging"); the rollback target is a specific commit, tag, or release; the time bound is in minutes.

## 8. Key Takeaways

- *What is graceful degradation, operationally?* The property that a system delivers a reduced-but-useful response when an internal dependency fails, rather than failing the whole request.
- *What three canonical patterns implement graceful degradation, and which failure mode does each address?* Circuit breaker (intermittent downstream failure), fallback path (caller-tolerant of stale or approximate data), load shedding (overload).
- *What distinguishes a rehearsed rollback from a documented rollback?* The rehearsal exposes the missing steps — the migration that does not have a `--down`, the flag that does not actually gate the new path, the data-state assumption that does not hold.
- *Why is the link between degradation and modernization specifically operational?* Because mid-migration boundaries (old and new code paths in production simultaneously) are exactly where degradation behavior is tested first.
- *What is the relationship to chaos engineering?* A rollback rehearsal is a small, scoped chaos experiment — define steady state, perturb the system, confirm the system returns to steady state within the expected bound.

## Sources

1. [Handling Overload — Google SRE Book Chapter 21](https://sre.google/sre-book/handling-overload/) — retrieved 2026-05-26
2. [Circuit Breaker (Martin Fowler)](https://martinfowler.com/bliki/CircuitBreaker.html) — retrieved 2026-05-26
3. [Principles of Chaos Engineering](https://principlesofchaos.org/) — retrieved 2026-05-26
4. [Feature Flag (Martin Fowler)](https://martinfowler.com/bliki/FeatureToggle.html) — retrieved 2026-05-26
5. [Strangler Fig Application (Martin Fowler)](https://martinfowler.com/bliki/StranglerFigApplication.html) — retrieved 2026-05-26

Last verified: 2026-05-26
