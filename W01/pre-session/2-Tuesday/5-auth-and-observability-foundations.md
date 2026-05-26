---
week: W01
day: Tue
topic_slug: auth-and-observability-foundations
topic_title: "Auth + observability foundations — OAuth2/JWT + structured logging + correlation IDs"
parent_overview: W01/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-jwt-authorizer.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://datatracker.ietf.org/doc/html/rfc7519
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://opentelemetry.io/docs/concepts/context-propagation/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.w3.org/TR/trace-context/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://microsoft.github.io/code-with-engineering-playbook/observability/correlation-id/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Auth + observability foundations — OAuth2/JWT + structured logging + correlation IDs

> This is a **primer-depth** reading. The Microservices Foundation deep-dive on Wednesday afternoon goes deeper on each of these topics — JWT internals, MDC vs `contextvars` mechanics, OpenTelemetry W3C trace-context propagation. Use this reading to internalise the vocabulary and the request-flow story; defer the deep mechanics to Wed PM.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Trace an authenticated request through a microservices system end-to-end: SPA → JWT issuance → gateway validation → downstream service → response.
- Name the load-bearing JWT claims an API gateway checks (signature, `iss`, `aud`, `exp`) and explain why each matters in one sentence.
- Explain why structured (JSON) logging plus a correlation ID is the only practical way to debug a distributed system.
- Recognise that the W3C `traceparent` header is the standard carrier for distributed trace context.

> **Deferred to Wed PM Microservices Foundation:** JWKS key-rotation mechanics, gateway-internal validation depth, language-level propagation implementation (Java MDC, Python `contextvars`). Today's reading is the request-flow narrative; tomorrow afternoon walks the implementation.

## 2. Introduction

Every authenticated request through a microservices system is a small choreography between a token issuer, an API gateway, and a backend service. The dance is the same across industries: **the user authenticates once, gets a token, presents that token on every subsequent request, and the gateway validates it on every call**. AWS API Gateway docs: "If you configure a JWT authorizer for a route of your API, API Gateway validates the JWTs that clients submit with API requests." That sentence is the entire pattern.

The IETF JWT spec (RFC 7519) names why this matters in distributed systems: JWTs are **self-contained** — the signed token carries the claims it asserts, so the gateway does not have to call the identity provider on every request. That property is what makes JWT validation tolerable at scale; the alternative ("session lookup per request") doesn't survive a 100-microservice mesh.

Once auth is solved, observability is next. A request touching six services produces six log streams; `grep` across them fails because each service logs at its own cadence with no shared identifier. The fix is universal: **assign every request a correlation ID at the gateway, propagate it via headers and language-level context, emit it on every log line in JSON form**. The Microsoft Engineering Fundamentals Playbook puts it plainly: "A correlation ID is a unique identifier that is added to logs as well as headers in HTTP calls so that it is easier to correlate logs across multiple components."

## 3. Core Concepts

### 3.1 The request-flow story

```
   ┌────────────┐
   │   User     │
   │   (SPA)    │
   └─────┬──────┘
         │ 1. login
         ▼
   ┌────────────┐         ┌────────────┐
   │   IdP /    │─────────▶│  user has  │
   │   Auth0    │         │   JWT      │
   └────────────┘         └─────┬──────┘
                                │ 2. every request: Authorization: Bearer <JWT>
                                ▼
                         ┌─────────────────┐
                         │   API Gateway   │
                         │  - sig check    │
                         │  - claims check │
                         │  - issue X-Corr │
                         └────────┬────────┘
                                  │ 3. forward with JWT + X-Correlation-ID
                                  ▼
                         ┌─────────────────┐
                         │ Domain service  │
                         │ logs every line │
                         │ with X-Corr ID  │
                         └─────────────────┘
```

Three load-bearing moves:

1. **Login produces a JWT.** The identity provider (Auth0, Cognito, Okta, Keycloak, internal IdP) authenticates the user once and issues a signed JWT containing claims about who they are, what they can do, and when the token expires.
2. **Every subsequent request carries the JWT.** The SPA stores it (memory or secure storage), attaches it as `Authorization: Bearer <token>`. No more username/password traverses the wire.
3. **The gateway validates the JWT and stamps a correlation ID** before forwarding. Downstream services inherit the correlation ID for log correlation — but they do **not** blindly trust the gateway's word on identity. Defence-in-depth requires the downstream to *also* validate (or at minimum verify `iss` + `aud` claims) so that a compromised gateway is not equivalent to full system compromise. Wed PM's Microservices Foundation deep-dive walks the three-checkpoint model and the lazy-trust failure mode that emerges when this is skipped.

### 3.2 What the gateway actually checks — four claims

Every JWT-validating gateway (AWS API Gateway, Kong, Apigee, Spring Cloud Gateway, Envoy with a JWT filter) checks the same four things. Memorise these four; they cover ~95% of the security story:

- **Signature** — token was signed by the trusted IdP and not tampered with. The gateway fetches the IdP's public key (via JWKS — covered Wed PM) and verifies. AWS docs: *"Check the token's algorithm and signature by using the public key that is fetched from the issuer's jwks_uri."*
- **`iss` (issuer)** — token came from the IdP we trust, not some other IdP.
- **`aud` (audience)** — token was issued *for this API*, not some sibling API run by the same IdP.
- **`exp` (expiration)** — token is still fresh; *"Must be after the current time in UTC."*

If any fails, the gateway returns 401. After validating, the gateway forwards the request — typically with the raw JWT passed through unchanged. **Defence-in-depth requires the downstream to re-validate (or at minimum re-check `iss` + `aud`)** — trusting a forwarded header as identity is the classic lazy-trust failure mode. The Wed PM Microservices Foundation deep-dive walks the three-checkpoint model (issuance → edge validation → service re-validation) and the failure modes when one is skipped.

### 3.3 The structured-logging discipline

A monolith logs `INFO User 1234 created order 5678` and you can grep `order 5678` to find every related line. A microservices system logs that line in service A, then `INFO Inventory reserved for order 5678` in service B — with no obvious connection between them in the interleaved log output. The fix has two parts:

**Part 1 — structured (JSON) logs.** Each log line is a JSON object, not a free-text string:

```json
{"ts":"2026-05-26T13:42:11Z","level":"INFO","service":"orders","correlation_id":"5e4b…","user_id":"1234","msg":"created order","order_id":"5678"}
```

JSON lets log aggregators (CloudWatch Logs Insights, Datadog, Grafana Loki, Splunk) query by field. You can write `correlation_id="5e4b…"` and get every line from every service for that one request.

**Part 2 — correlation ID at the edge.** The gateway mints a UUID per request (or accepts a valid upstream `X-Correlation-ID`), attaches it to the request headers, and emits its own log lines with it. Downstream services read the header and emit it on every log line during the request.

**The W3C `traceparent` header is the OpenTelemetry-standard carrier** for trace ID + span ID across services — pair it with `X-Correlation-ID` and you get tooling-vendor portability. The OpenTelemetry docs state: "Context propagation … allows trace context, including span and trace IDs, to be passed across distributed system boundaries."

> **Deferred to Wed PM:** the *how* of language-level propagation (Java SLF4J MDC, Python `contextvars`, OpenTelemetry SDK auto-injection). For today, internalise the *what* — every log line carries the correlation ID; somewhere a per-language mechanism makes that automatic.

## 4. Generic Implementation

The shape of a gateway's request handler at primer-depth (full mechanics in Wed PM):

```
on every inbound request:
  1. read Authorization: Bearer <jwt>
  2. verify signature (using IdP's public key from JWKS)
  3. check iss, aud, exp claims
  4. on any failure → 401
  5. mint or read X-Correlation-ID; emit a log line in JSON with it
  6. forward to the downstream service (raw JWT passed through)
```

The downstream service, on receiving the request:

```
1. (defence-in-depth) re-validate the JWT, or at minimum re-check iss + aud
2. read X-Correlation-ID from the headers
3. ensure every subsequent log line (including from library code) carries it
4. propagate X-Correlation-ID on any outbound call to other services
```

The implementation knobs for step 3 (Java MDC, Python `contextvars`, OpenTelemetry auto-injection) are Wed PM's material. For today, what matters is the *contract*: the gateway mints once, every service propagates, every log line carries it.

## 5. Real-world Patterns

**Fintech (Stripe API).** Stripe issues opaque secret keys (`sk_live_...`) for API auth, but every webhook delivery carries a `Stripe-Signature` header that's effectively a per-event signed assertion. The pattern: **stamp every cross-boundary message with a signed claim so the receiver verifies origin without a callback**.

**Healthcare IT (SMART on FHIR).** The SMART-on-FHIR standard for clinical-app authentication is OAuth 2.0 + JWT. Every clinician app gets a JWT from the EHR's IdP; every FHIR API call carries it; the FHIR server validates locally. The pattern: **a regulated industry standardised on OAuth2+JWT because per-call signature verification beats per-call session lookup at scale**.

**E-commerce (Shopify storefront tokens).** Shopify issues per-merchant API tokens that look like JWTs (signed, time-bounded) for the Storefront API. Edge validation, no callback. Pattern: **JWT-style validation enables sub-100ms response times at the edge** for high-traffic storefronts.

**Logistics (Uber's internal microservices).** Uber pairs internal mTLS with per-service JWTs for service-to-service auth — both transport and application layer assert identity. Distributed-tracing context propagates via their Jaeger fork. Pattern: **defence-in-depth identity — even if the mesh is compromised, the JWT layer still proves service identity**.

## 6. Best Practices

- **Validate the JWT at the gateway AND re-validate (or at minimum check `iss` + `aud`) at every downstream service.** Defence-in-depth: gateway compromise must not equal full-system compromise. The "trust the gateway, skip downstream validation" shortcut is the most common authorisation flaw auditors flag.
- **Always check `aud`, not just `iss` and `exp`.** A token issued by your trusted IdP for a *different* application of yours is not valid for this API.
- **Mint a correlation ID at the gateway if the request didn't come with one.** Accept an upstream-supplied ID only if it matches a strict format (UUID v4) — otherwise mint your own.
- **Emit logs as JSON, not text.** Pick a logger config (Logback `JsonEncoder`, Python `python-json-logger`, structlog) and standardise it across services.
- **Pair `X-Correlation-ID` with the W3C `traceparent` header** — gives you OpenTelemetry-tooling portability for free.

> Best-practice items on JWKS caching, key rotation, and language-level propagation (MDC / `contextvars`) live in Wed PM's deep-dive — they are the implementation discipline behind these principles.

## 7. Hands-on Exercise

**Walking the request (5–8 min, no code):**

For a hypothetical **e-commerce checkout** flow, write out the request-flow story in sequence:

1. User clicks "Checkout." SPA calls `POST /api/orders` with `Authorization: Bearer <jwt>`. **Q:** which four JWT claims does the gateway check, and what does each prove?
2. Gateway forwards to `orders-service`. **Q:** what header carries the correlation ID? Does `orders-service` re-validate the JWT, and why?
3. `orders-service` calls `inventory-service` to reserve stock. **Q:** how does `inventory-service` end up with the same correlation ID on its log lines?
4. `inventory-service` logs a stock-reservation event. **Q:** what fields are on that log line? (Hint: at minimum: `ts`, `level`, `service`, `correlation_id`, `msg`.)

**What good looks like:** a short narrative where each JWT claim has a named purpose (sig = anti-tampering, exp = freshness, aud = anti-replay across APIs, iss = anti-impersonation), defence-in-depth is explicit (downstream re-validates), the correlation ID is minted once at the gateway, and the log-line schema is identical across all services.

## 8. Key Takeaways

- *Can I trace an authenticated request through the SPA → IdP → gateway → service → response path?* (Maps to LO 1.)
- *Can I name the four JWT checks (signature, `iss`, `aud`, `exp`) and why each matters in one sentence?* (Maps to LO 2.)
- *Can I explain why structured (JSON) logs plus a correlation ID is the only practical way to debug a distributed system?* (Maps to LO 3.)
- *Can I name the W3C `traceparent` header as the standard distributed-trace carrier?* (Maps to LO 4.)

> JWKS rotation depth, language-level propagation mechanics (MDC / contextvars), and gateway internals are Wed PM's territory — Tuesday is the request-flow narrative, Wednesday is the implementation.

## Sources

1. [AWS API Gateway — Control access to HTTP APIs with JWT authorizers](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-jwt-authorizer.html) — retrieved 2026-05-26
2. [IETF RFC 7519 — JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519) — retrieved 2026-05-26
3. [OpenTelemetry — Context Propagation](https://opentelemetry.io/docs/concepts/context-propagation/) — retrieved 2026-05-26
4. [W3C Trace Context — W3C Recommendation](https://www.w3.org/TR/trace-context/) — retrieved 2026-05-26
5. [Microsoft Engineering Fundamentals Playbook — Correlation IDs](https://microsoft.github.io/code-with-engineering-playbook/observability/correlation-id/) — retrieved 2026-05-26

Last verified: 2026-05-26
