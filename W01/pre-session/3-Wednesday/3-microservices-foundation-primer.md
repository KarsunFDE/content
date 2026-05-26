---
week: W01
day: Wed
topic_slug: microservices-foundation-primer
topic_title: "Microservices Foundation primer — what you'll walk Wed PM"
parent_overview: W01/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 15
sources:
  - url: https://istio.io/latest/about/service-mesh/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/bliki/BoundedContext.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://datatracker.ietf.org/doc/html/rfc7519
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.w3.org/TR/trace-context/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.docker.com/build/building/best-practices/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.docker.com/reference/dockerfile/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Microservices Foundation primer — what you'll walk Wed PM

> The Wednesday afternoon `acquire-gov` walkthrough is the cohort's first deep contact with a four-service mesh. This generic deep-dive prepares you for that walk by laying out the substrate: how service boundaries are drawn, how identity propagates across them, how you observe what is happening inside the mesh, and how containerisation either supports or sabotages all three. The overview (`1-DailyTopicOverview.md` §3) already maps these concepts to `acquire-gov`'s specific debt items; this reading is the framework underneath.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define a **bounded context** and explain why service boundaries derived from business capabilities outperform boundaries derived from technical layers.
- Trace a **JWT through an API gateway** into downstream services, identifying the three places identity can be re-validated and the security failure mode that emerges when one of them is skipped.
- Explain the role of a **correlation ID** in distributed-systems debugging and describe the contract between correlation IDs and trace IDs.
- Audit a **Dockerfile** against four production-readiness criteria (multi-stage build, immutable tags, healthchecks, persistent volume strategy) and identify the antipatterns that cause cascading 5xxs.
- Distinguish **synchronous from asynchronous** service-to-service paths and articulate where retries, circuit breakers, and idempotency keys live in each pattern.

## 2. Introduction

The microservices style emerged because monolithic deployments became politically expensive: teams could not ship independently, tests took hours, and one team's slow regression blocked everyone else's releases. The architectural answer — small services, owned by small teams, deployed independently — solved the political problem and immediately created technical problems that had never been first-class concerns in a monolith: where do service boundaries actually go, how does a user identity travel across services without being forged in the middle, how do you debug a single request that touched seven services, and how do you make a fleet of containers start in the right order every time.

Two decades of practice (early SOA in the 2000s, microservices in the 2010s, service mesh in the late 2010s, post-mesh sidecar-free patterns now arriving) have converged on a small canon of foundational ideas [1][2]. Bounded contexts from Domain-Driven Design determine the lines you draw. JWTs and their relatives carry identity across the lines [3]. Correlation IDs and distributed tracing make request paths observable across the lines [4]. Multi-stage Dockerfiles, healthchecks, and persistent-volume strategies determine whether the lines hold up under restart, redeploy, or partial-failure conditions [5][6].

The forward-deployed engineer's job is rarely to *design* a fresh microservices architecture — that work is done by the client's architects. The FDE's job is to walk into an existing service mesh, recognise which of the foundational ideas were applied well, which were skipped, and which were applied half-way. The "half-way" applications are the most dangerous: a JWT that is validated at the gateway but trusted everywhere downstream looks correct from the outside until a malicious internal caller bypasses the gateway. A correlation ID that is generated at the frontend but dropped by the second service produces traces that mislead more than they help. A multi-stage Dockerfile that uses `:latest` for the base image looks slim until the night the upstream image changes and the production deploy stops being reproducible.

This reading walks the substrate. The afternoon walkthrough applies it to `acquire-gov`.

## 3. Core Concepts

### 3.1 Bounded contexts: where service boundaries actually go

A bounded context, in Domain-Driven Design terms, is a region of the business in which a particular model of the domain is internally consistent [2]. "Customer" in the billing context is not the same shape as "Customer" in the support context — billing cares about credit, payment history, and tax jurisdiction; support cares about open tickets, contract tier, and language preference. Forcing both into one shared model produces friction at every change.

Bounded contexts are the natural place to put service boundaries because they minimise cross-team coordination. A team that owns the billing context can ship without a meeting with the support team. The opposite frame — splitting services by technical layer (one service for "the database", one for "the API", one for "the cache") — produces services that cannot ship independently because every business change ripples across all three.

Three practical tests for whether a boundary is sound: (1) can the team owning the service deploy a change without coordinating with another team? (2) does the language used inside the service stay internally consistent (one definition of "Customer", not three)? (3) when a feature request arrives, does it land cleanly inside one service, or does it inherently span two? Boundaries that fail those tests are the ones that wear teams out.

### 3.2 Identity propagation: the three-checkpoint model

In a typical JWT-based microservice mesh [3], identity propagation has three checkpoints:

1. **Issuance.** The user authenticates against an identity provider (Auth0, Okta, Keycloak, AWS Cognito); the IdP issues a signed JWT.
2. **Edge validation.** The API gateway verifies the JWT signature against the IdP's public keys (JWKS), checks expiration, audience, and issuer, and forwards the request inward.
3. **Service re-validation.** Each downstream service either (a) re-validates the JWT itself, (b) trusts the gateway's validation and reads claims from forwarded headers, or (c) accepts a gateway-issued internal token (token exchange).

The most common failure mode is the lazy version of option (b): the downstream service reads `X-User-Id` from a header without verifying that the request actually came through the gateway. Any internal caller (a misconfigured background job, a compromised sidecar, an attacker who gained access to the internal network) can spoof that header. The fix is either (a) full re-validation, (c) token exchange to a service-mesh-issued internal token, or a network-level guarantee (mTLS in a service mesh) that the request genuinely came from the gateway.

The federal context adds an extra dimension: claim re-validation matters not only for security but for **audit**. If a downstream service writes "user X performed action Y" to an append-only audit log on the strength of a header it never validated, the audit log is technically a forgery surface. This is the kind of finding that surfaces in an authorisation-to-operate review.

### 3.3 Correlation IDs and distributed tracing

A **correlation ID** is a single string that travels with a request end-to-end so that every log line emitted by any service touching that request can be filtered to a single timeline [4]. The mechanic is mundane: the first service to touch the request (typically the frontend or the gateway) generates a UUID, puts it in an HTTP header (commonly `X-Correlation-ID` or `X-Request-ID`), and every downstream service reads-and-propagates that header on every outbound call and includes it in every log line.

A **trace ID** is the OpenTelemetry-standard equivalent for distributed tracing — same idea (one ID across a request graph) plus per-service span IDs that form a tree. Modern systems carry both, because traces are sampled (only a small percentage are recorded) while logs are usually not. When a customer reports an issue, the correlation ID is what you grep production logs for; the trace ID is what you open in your APM tool to see the latency breakdown.

The contract is simple and brittle: *every* service must read-and-propagate the header on every outbound call. A single service that drops the header (or generates a fresh one for its outbound calls) breaks the chain for the rest of the request graph. The most common drop point is an asynchronous handoff: service A writes a message to a queue, service B consumes the message — if A did not put the correlation ID in the message envelope, B cannot recover it.

Logs that carry correlation IDs are most useful when they are also **structured** — JSON output, one field per log line, machine-grepable. Plain-text logs containing `[corr=abc123]` somewhere in the middle of a free-text string are workable but waste analyst time. Logs in inconsistent formats across services (each team's preferred logger, each with different field names) waste analyst time at scale.

### 3.4 Synchronous vs asynchronous paths

A microservice mesh almost always contains both shapes:

- **Synchronous (HTTP/gRPC):** Service A calls Service B and waits for the response. Latency is bounded by the longest path. Failure handling lives in A: retry on transient errors, circuit-break on persistent failure, time-bound the wait. Idempotency matters: if A retries B and B already succeeded, the second call must not double-charge.
- **Asynchronous (message queue, event stream):** Service A puts a message on a queue and continues; Service B consumes the message when it can. Latency is decoupled. Failure handling lives in B: poison-message handling, dead-letter queues, retry-with-backoff. Idempotency matters even more: queue redelivery is the default, not the exception.

The architectural decision of which to use is rarely simple. Synchronous paths are easier to reason about and easier to debug; asynchronous paths absorb load spikes and survive downstream failures. Production systems mix both; the FDE's job is to recognise which is which when reading a service mesh.

### 3.5 Containerisation: the production-readiness checklist

Four container-level patterns separate a demo Dockerfile from a production one [5][6]:

- **Multi-stage builds.** A first stage compiles or builds; a second, smaller stage copies only the artifacts and runs them. The final image is the runtime, not the build-toolchain. Smaller images, fewer vulnerabilities, faster pull-and-start.
- **Immutable base tags.** `FROM python:3.11.7-slim` is reproducible; `FROM python:latest` is a time bomb. The first time the upstream image changes (a security patch, a minor version bump), your build is silently different. Many production incidents trace to this exact line.
- **Healthchecks.** A `HEALTHCHECK` instruction tells the orchestrator (Docker, Compose, Kubernetes) whether the container is genuinely ready to receive traffic. Without one, the orchestrator routes traffic the instant the process starts — before databases are connected, before initial caches are warm — and the first 30 seconds of traffic returns 5xx. Define one healthcheck per stage; multiple ambiguous healthchecks are an antipattern.
- **Persistent-volume strategy.** Data services (Postgres, MongoDB, Redis-with-AOF) need persistent volumes. A docker-compose file with no volume mounted on the data service loses all data on `docker-compose down -v`. In production this means: an accidental `docker compose down -v` during incident response wipes the dataset. Volume strategy is part of the production-readiness review, not an afterthought.

## 4. Generic Implementation

This section shows three small, generic excerpts — none of them tied to any federal-acquisitions domain — that illustrate the patterns above. The exercises live in the next section; here we just look at what good looks like.

**Correlation-ID propagation in a generic Express.js (Node) service:**

```javascript
// Middleware: read or create a correlation ID, attach to req + res, and log it.
function correlationIdMiddleware(req, res, next) {
  const id = req.headers['x-correlation-id'] || crypto.randomUUID();
  req.correlationId = id;
  res.setHeader('x-correlation-id', id);
  // Make available to any logger called inside this request.
  asyncLocalStorage.run({ correlationId: id }, () => next());
}

// Every outbound HTTP call must propagate it.
async function callPaymentService(payload) {
  return await fetch(PAYMENT_SVC_URL, {
    method: 'POST',
    headers: {
      'content-type': 'application/json',
      'x-correlation-id': asyncLocalStorage.getStore().correlationId,
    },
    body: JSON.stringify(payload),
  });
}

// Every log line includes it (structured JSON).
logger.info({
  correlationId: asyncLocalStorage.getStore().correlationId,
  event: 'payment.requested',
  amount_cents: payload.amount_cents,
});
```

**A production-shaped multi-stage Dockerfile for a generic Python service:**

```dockerfile
# Stage 1: build wheels with toolchain available.
FROM python:3.11.7-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip wheel --no-cache-dir --wheel-dir /wheels -r requirements.txt

# Stage 2: minimal runtime image — no toolchain in the final layer.
FROM python:3.11.7-slim AS runtime
RUN useradd --create-home --shell /bin/bash app
WORKDIR /home/app
COPY --from=builder /wheels /wheels
COPY --chown=app:app . .
RUN pip install --no-cache-dir --no-index --find-links=/wheels -r requirements.txt \
 && rm -rf /wheels
USER app
EXPOSE 8000
# One healthcheck per stage. Adjust path + interval to the app's reality.
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -fsS http://localhost:8000/health/ready || exit 1
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Three points to notice: (1) the base tag is `python:3.11.7-slim`, not `python:latest` or `python:3.11`; (2) the build stage stays out of the runtime image; (3) the `HEALTHCHECK` hits a `/health/ready` endpoint that the application returns 200 from only after database connections and warm caches are ready — not a `/health/live` endpoint that returns 200 the moment the process starts.

**Sketch of JWT validation in a generic downstream service (pseudocode, framework-agnostic):**

```
on every inbound request:
  1. assert request arrived on mTLS-protected internal port (network-level trust)
  2. read Authorization: Bearer <jwt>
  3. fetch JWKS from IdP (cached, refresh on cache miss)
  4. verify signature, expiration, audience, issuer
  5. extract user_id, roles, scopes from validated claims
  6. authorize the request against the service's local policy
  7. on every log line and on every outbound call, propagate the correlation ID
```

The order matters: network-level trust *and* full JWT re-validation. The pattern of "skip re-validation because the gateway already did it" is the antipattern; it is the failure mode the afternoon walkthrough surfaces in `acquire-gov`.

## 5. Real-world Patterns

**E-commerce — Netflix's bounded contexts.** Netflix has documented its decomposition of the video-streaming experience into bounded contexts (recommendations, encoding, billing, identity, playback) repeatedly in conference talks and tech-blog posts. The pattern that emerged: each context owns its own data, its own deploy cadence, and its own on-call rotation. Cross-context coordination happens through well-defined event contracts on the data bus, not through synchronous request graphs. The cost: every cross-context query becomes an integration project. The benefit: any team can deploy at any time without coordinating with any other team [2].

**Fintech — Monzo's JWT plus mTLS posture.** Monzo, a UK challenger bank, has talked publicly about its dual-trust posture: JWTs carry user identity end-to-end, but every internal service-to-service call also requires mTLS, meaning a compromised inbound credential alone is insufficient to forge a request. Each downstream service re-validates the JWT signature on every call rather than trusting the edge. The cost is non-trivial — JWKS fetches add latency — but the audit posture is the point [3].

**Healthcare — Cerner / Epic correlation IDs in clinical event flows.** Both major EHR vendors propagate a correlation identifier through clinical-event pipelines so that a single patient encounter — admission, lab order, lab result, clinician note, discharge — can be reconstructed from logs across a dozen services. The discipline came from the regulatory requirement to reproduce a clinical record on demand; the engineering benefit was incidental [4].

**Gaming — Riot Games' container reproducibility.** Riot has discussed in engineering posts the cost of `:latest` antipatterns in their game-server fleet: when upstream Ubuntu base images shipped a small kernel change, Riot's game servers started failing in subtle ways on a Tuesday morning. The post-incident response standardised pinned, hash-locked base images across the fleet [5][6].

## 6. Best Practices

- **Draw service boundaries on business capabilities, not on technical layers** — if two services must always deploy together, they are one service.
- **Re-validate JWT signatures at every service entry point** unless mTLS-plus-token-exchange explicitly removes the need; never trust a forwarded header as identity alone.
- **Generate the correlation ID at the request entry point and propagate it on every outbound call** — synchronous or asynchronous, log-line or queue-envelope, no exceptions.
- **Use structured (JSON) logging with consistent field names across all services** — the cost of standardisation is one-time; the cost of inconsistency compounds every incident.
- **Pin Docker base images to a specific tag, ideally with a digest** — `FROM python:3.11.7-slim@sha256:...` is the production-grade form; `FROM python:latest` is the time-bomb form.
- **Define one HEALTHCHECK per service stage and point it at a `/ready` endpoint that genuinely reflects readiness** — not at `/live` which returns 200 too early.
- **Mount persistent volumes on every data service and document the backup strategy as part of the deploy** — `docker-compose down -v` should not be the threat model.

## 7. Hands-on Exercise

**(Whiteboarding, 12 minutes.)** Sketch a 4-service architecture for a generic ride-hailing application: a mobile app, an API gateway, a `trip-service` (creates and tracks trips), a `pricing-service` (calculates fares), and a `notification-service` (sends push notifications). On the sketch, mark:

1. Where the JWT is issued, where it is validated, and where it is re-validated.
2. The correlation-ID hop points (where it is generated, where it is propagated, where it lands in a log).
3. Which inter-service calls are synchronous and which are asynchronous, with one-sentence justifications.
4. The `HEALTHCHECK` endpoint each service exposes and what it actually checks.

**What good looks like.** A finished sketch shows the JWT issued at the IdP, validated at the gateway, re-validated at each service entry point (or shown going through token exchange to an internal token). The correlation ID is generated at the gateway, propagated in every HTTP header, and included in every log line. `trip-service → pricing-service` is synchronous (the trip cannot start without a price). `trip-service → notification-service` is asynchronous via a queue (a delayed notification is acceptable; a failed notification must not fail the trip). Each `HEALTHCHECK` checks both process liveness *and* downstream connectivity (e.g., the trip-service health endpoint verifies the database connection, not just that the HTTP server is responding).

## 8. Key Takeaways

- Can you describe what makes a service boundary "sound" and name the test that distinguishes business-capability boundaries from technical-layer boundaries?
- Can you trace a JWT through a four-service mesh and identify the lazy-trust failure mode that emerges when downstream services skip re-validation?
- Can you explain the correlation-ID contract — generate once, propagate everywhere, on every hop — and why a single drop breaks the whole chain?
- Can you audit a Dockerfile against the four production-readiness criteria (multi-stage, immutable tag, healthcheck, volume strategy) and name the cascading-5xx failure mode that missing healthchecks produce?
- Can you decide whether an inter-service call should be synchronous or asynchronous, and place the retry/circuit-break/idempotency responsibilities accordingly?

## Sources

1. [What is a service mesh? (Istio docs)](https://istio.io/latest/about/service-mesh/) — retrieved 2026-05-26
2. [Bounded Context (Martin Fowler bliki)](https://martinfowler.com/bliki/BoundedContext.html) — retrieved 2026-05-26
3. [IETF RFC 7519 — JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519) — retrieved 2026-05-26
4. [W3C Trace Context — W3C Recommendation](https://www.w3.org/TR/trace-context/) — retrieved 2026-05-26
5. [Building best practices (Docker docs)](https://docs.docker.com/build/building/best-practices/) — retrieved 2026-05-26
6. [Dockerfile reference (Docker docs)](https://docs.docker.com/reference/dockerfile/) — retrieved 2026-05-26

Last verified: 2026-05-26
