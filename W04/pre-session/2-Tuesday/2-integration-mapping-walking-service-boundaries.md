---
week: W04
day: Tue
topic_slug: integration-mapping-walking-service-boundaries
topic_title: "Integration Mapping — walking service boundaries"
parent_overview: W04/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://martinfowler.com/articles/microservices.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/bliki/BoundedContext.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/bliki/StranglerFigApplication.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://microservices.io/patterns/observability/distributed-tracing.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Integration Mapping — walking service boundaries

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define an *integration map* and distinguish it from an architecture diagram, a sequence diagram, and a deployment topology.
- Name the four call-direction categories every service has (inbound, outbound, datastore, async event) and apply them to an unknown service.
- Identify *seams* in a brownfield system — the places where the documented integration disagrees with the actual call graph.
- Choose between three evidence sources (static code grep, runtime traces, access logs) for confirming an integration edge.
- Produce a one-page integration map for a single service in under 30 minutes.

## 2. Introduction

Before you change a system, you have to know what crosses what. That sentence is older than microservices — it was the operating principle behind 1990s enterprise-integration diagrams, and behind the "context map" practice Eric Evans codified in *Domain-Driven Design* in 2003. What changed with microservices is the *number* of crossing points. A monolith has one inbound front door (the request handler), one outbound back door (the database driver), and a few side-doors (the cron job, the message queue). A microservices fleet has a graph: every service is a node, and every call is a directed edge — and the documentation almost never matches the runtime reality.

This mismatch is where brownfield modernization either succeeds or stalls. Martin Fowler's 2014 *Microservices* article noted that microservice teams treat the service boundary as "smart endpoints and dumb pipes" — and the corollary is that the boundary moves around as the team evolves. The integration map is the artifact that names *where the boundary is right now*, regardless of where the original architect said it would be.

An integration map is not a Visio diagram of boxes and arrows. It is a *table of evidence*: for one service, every inbound caller (with what payload, on what path, with what auth), every outbound dependency (with what client library, what timeout, what retry policy), every datastore touched (with what schema, what indexes, what tenancy filter), and every async event published or consumed (with what topic, what schema, what subscriber). Once you have that table, you can compare what the code does to what the spec says, and the disagreements are the seams you can modernize.

The connection between integration mapping and the Strangler Fig pattern is direct. Fowler's update on Strangler Fig (Aug 2024) lists "decide how to break the problem up into smaller parts" as the second of four activities for incremental modernization, and notes that this work involves "identifying *seams* that we can insert into the system to allow it to be split." An integration map IS the seam-discovery artifact. Without one, you are guessing about what's safe to extract.

## 3. Core Concepts

### 3.1 The four call-direction categories

Every service participates in four kinds of integrations. Walking a service systematically through all four is the discipline that prevents missed dependencies:

- **Inbound synchronous** — HTTP/gRPC requests this service handles. The evidence is in route registrations (`@RequestMapping`, FastAPI `@app.post`, Express `app.get`) and in the access log.
- **Outbound synchronous** — HTTP/gRPC requests this service initiates. The evidence is in client library calls (`RestTemplate`, `WebClient`, `requests.post`, `axios.get`) and in egress firewall logs if available.
- **Datastore access** — direct reads and writes against databases, caches, object stores, and search indexes. The evidence is in repository classes, ORM models, raw SQL/NoSQL queries, and the actual schema of each store.
- **Async events** — messages this service publishes or subscribes to. The evidence is in topic configuration, Kafka/SQS/SNS client setup, and the schema registry if one exists.

A common failure mode is to enumerate only the first two. Datastore access is where multi-tenancy bugs hide (a query without a tenant filter is a cross-tenant leak waiting to happen); async events are where the system's actual coupling lives (services with no synchronous edges can still be tightly coupled through a shared event schema).

### 3.2 Bounded contexts and context maps

The integration map is the operational artifact; the *context map* is the strategic artifact behind it. Evans' Domain-Driven Design defines a Bounded Context as the linguistic boundary inside which a domain model is internally consistent. Different teams build different models because they speak different vocabularies — Fowler's "meter" example (does it mean the physical device, the grid connection, or the billing account?) is the canonical illustration. A context map names the relationships between contexts: shared kernel, customer-supplier, conformist, anti-corruption layer, separate ways.

In a microservices brownfield, the integration map is the *observed* version of the context map. The original team may have intended one service per bounded context (the ideal), but as the system grew, services crossed boundaries — a single service started handling pieces of two contexts, or one context got fragmented across three services that all need to be redeployed together. The integration map exposes this: when you find a service whose inbound callers all want piece A and whose outbound callees all support piece B, you have a candidate split.

### 3.3 Seams and where to look for them

A *seam* is a place in the code or architecture where you can change behavior on one side of a line without changing the other side. Fowler's bliki entry on Legacy Seams (referenced from the Strangler Fig article) emphasizes that well-designed systems have seams already; legacy systems generally do not. The integration map's job is to surface candidate seams. The richest seam locations are:

- **Process boundaries** — every network hop is already a seam, because the wire protocol enforces a contract.
- **Module/package boundaries** — within a service, the import graph may reveal a sub-component that only the rest of the service calls *into*, never *out of*.
- **Datastore boundaries** — a table that only one service writes to is a candidate extraction point.
- **Configuration boundaries** — a feature flag, a per-tenant configuration value, or an environment-conditional code path is a runtime seam.

### 3.4 Evidence sources: static grep vs runtime trace vs access log

The integration map is only as good as the evidence under it. Three evidence sources, in increasing order of authority:

- **Static analysis (grep / AST)** — fast, cheap, but reports *everything the code could call*, not what it actually does. Good for outbound dependency enumeration; weak on conditional or reflective calls.
- **Distributed traces (OpenTelemetry, Zipkin, Jaeger)** — show what was actually called in production. Authoritative on synchronous edges. Microservices.io's *Distributed Tracing* pattern (Chris Richardson) names this as the canonical observability tool: instrument every service to propagate a request ID, then aggregate the traces to see the whole call tree.
- **Access logs and database query logs** — last-line ground truth. If something appears in the access log, it actually happened. Weak point: high volume, often retained only briefly.

The discipline: start with static analysis to get a *hypothesis* map; confirm or refute each edge using runtime traces; back-stop the highest-risk edges (auth, money, PII) with access-log diffs.

## 4. Generic Implementation

Below is a one-page integration-map template a learner can fill in for any service in under 30 minutes. The example is a generic `orders-service` in an e-commerce backend — no domain ties to any specific industry.

```yaml
# integration-map.orders-service.yml
# Service under analysis: orders-service
# Date: 2026-05-26
# Author: <pair name>

service:
  name: orders-service
  language: java-17
  framework: spring-boot-3.2
  deployed_at: prod-eu-west-1, prod-us-east-1

inbound_sync:
  # Every route handler this service exposes.
  - path: POST /orders
    callers_observed:  [web-storefront, mobile-bff]
    auth:              jwt-bearer (validated locally)
    rate_limit:        100/min/tenant
    evidence:          access-log + OTEL span name=orders.create

  - path: GET /orders/{id}
    callers_observed:  [web-storefront, mobile-bff, ops-console]
    auth:              jwt-bearer (validated locally)
    evidence:          access-log

outbound_sync:
  # Every service this one calls. Each row is a coupling.
  - target: inventory-service
    path:        POST /reservations
    client_lib:  WebClient (Reactor Netty)
    timeout:     2000ms
    retries:     2 (exponential, jittered)
    evidence:    grep RestTemplate + OTEL parent span

  - target: payment-service
    path:        POST /authorizations
    client_lib:  Feign
    timeout:     5000ms
    retries:     0  # idempotency concerns
    evidence:    grep @FeignClient + OTEL parent span

datastore_access:
  - store:       postgres (orders schema)
    tables_read:  [orders, order_items, addresses]
    tables_write: [orders, order_items]
    tenancy_filter_observed: WHERE tenant_id = ? (every query)
    evidence:    Spring Data repository methods + pg_stat_statements

  - store:       redis (idempotency cache)
    keys:         "order:idem:<key>"
    ttl:          24h
    evidence:    RedisTemplate usage

async_events:
  publishes:
    - topic:  orders.created.v1
      schema: avro://schema-registry/orders.created.v1
      consumers_observed: [analytics-pipeline, fulfillment-service]
      evidence: Kafka producer config + consumer-group list

  subscribes:
    - topic:  payments.captured.v1
      handler: OrderPaymentHandler#onCaptured
      evidence: @KafkaListener annotation + consumer-group lag metric

seams_identified:
  - id: SEAM-1
    description: "Idempotency cache is a redis-only side door; could be extracted to a shared idempotency-service."
    risk: medium
  - id: SEAM-2
    description: "GET /orders/{id} has three independent callers with different field-set needs; candidate for BFF split."
    risk: low
```

The annotations matter. Each row carries an `evidence` field that says *how we know* this edge exists. A row without evidence is a hypothesis, not a fact, and the integration-mapping discipline is to chase down evidence for every row before signing the map.

## 5. Real-world Patterns

**Streaming media platform — context map drove the recommendation-service extraction.** A large video-on-demand company found their monolithic `personalization-service` was being called by both the homepage (which wanted "shows for you") and the player (which wanted "what to watch next"). The integration map showed two distinct inbound caller groups with non-overlapping output field sets. The team extracted the player path into a new `next-episode-service` over six months, using the integration map as both the discovery artifact and the acceptance test — the new service had to reproduce every observed edge before the old one was deprecated. The pattern follows the *bounded-context-per-service* ideal Evans described, applied retroactively.

**Healthcare claims processor — async event archaeology found unowned coupling.** A US healthcare claims processor discovered during a modernization that one service was emitting an event topic (`claims.adjudicated.v3`) that *no documented subscriber* read. Static analysis missed it because subscribers were in a different repo. Runtime trace data, however, showed the topic had three active consumer groups — two were lambdas owned by a department that had reorganized away, and one was a billing reconciliation job nobody had updated since 2019. The integration map made the orphan ownership visible; the team froze the topic schema and instrumented the lambdas to identify their actual purpose before either modernizing or retiring them.

**Logistics last-mile platform — seam discovery via access-log diff.** A logistics company's `route-optimizer` service had grown to handle three workflows (commercial routes, residential routes, returns routes). Their integration map looked uniform — same inbound paths, same datastores. But an access-log diff over a week showed that 92% of `POST /route-optimize` calls with `?mode=returns` had distinct request shapes and never touched the commercial-route tables. The seam was real but invisible at the architecture level. They extracted a `returns-router` service in three weeks because the integration map had already proven the edges were clean.

**Gaming backend — distributed tracing exposed accidental fan-out.** An online game's matchmaking service appeared simple in the static integration map — three outbound calls (player profile, skill rating, region pool). Production traces showed every match request was *actually* fanning out to seven services because of a logging library that pulled session context from a tracing-sidecar service synchronously on every span emit. The integration map was wrong because static analysis missed the implicit dependency. The fix was straightforward (move the lookup to async); the lesson was that the integration map must include implicit transitive dependencies surfaced by tracing.

## 6. Best Practices

- **One service, one page, one author** — integration maps that span multiple services lose their authoritative voice. Each service gets its own map, signed by its owning team.
- **Evidence per row, no exceptions** — every edge cites a source (static analysis, trace, log). Edges without evidence are hypotheses, not facts.
- **Update the map at every modernization checkpoint** — a stale integration map is worse than no map because it gives false confidence. Re-derive it at the start of every modernization sprint.
- **Include async edges, not just synchronous calls** — most missed coupling lives in event topics, not HTTP routes.
- **Record what's not there** — if a documented integration doesn't appear in the runtime evidence, that's the most interesting kind of seam (either dead code or hidden behavior).
- **Name seams explicitly** — give every candidate extraction or split a SEAM-ID and a risk rating so the modernization backlog has names to point at.
- **Re-verify monthly in active systems** — service-to-service graphs drift; treat the map as living documentation, not a one-time artifact.

## 7. Hands-on Exercise

**Exercise: Map a service in 25 minutes.**

Pick any one service in a system you know (your current job, an open-source repo, or a previous project). Without using any auto-generated diagrams, fill in the four-section integration-map template above:

1. **Inbound sync** (5 min) — find every route handler. Note caller, auth, rate limit.
2. **Outbound sync** (5 min) — grep for HTTP-client library calls. Note target, timeout, retry.
3. **Datastore access** (5 min) — enumerate the stores. For each, note tenancy filter discipline.
4. **Async events** (5 min) — find every producer and consumer registration.
5. **Seams (5 min)** — name at least two candidate seams with risk ratings.

**What good looks like:** every row has an `evidence` field that names the file or log it came from. The seam list contains at least one seam ranked `low` (safe to act on now) and one ranked `medium` or `high` (needs more evidence). The total artifact fits on one page (~60 lines of YAML).

If you cannot find two seams in 25 minutes, that's an interesting finding — either the service is very well-bounded already, or your evidence sources are incomplete. Both are useful data.

## 8. Key Takeaways

- *What is an integration map?* — A per-service table of inbound, outbound, datastore, and async edges, with evidence for each row.
- *How do you find seams?* — Look for places where the observed graph disagrees with the documented architecture, and where datastore tenancy filters or feature flags hint at a runtime split.
- *Which evidence source is authoritative?* — Distributed traces and access logs beat static analysis; the discipline is to confirm hypotheses from grep using runtime data.
- *Why include async events?* — Because the deepest coupling in microservices systems usually hides in event topics, not in HTTP routes.
- *When do you re-verify the map?* — At every modernization checkpoint, and monthly in active systems, because service graphs drift faster than documentation.

## Sources

1. [Microservices — a definition of this new architectural term](https://martinfowler.com/articles/microservices.html) — retrieved 2026-05-26
2. [Bounded Context](https://martinfowler.com/bliki/BoundedContext.html) — retrieved 2026-05-26
3. [Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html) — retrieved 2026-05-26
4. [Microservices Pattern: Distributed Tracing](https://microservices.io/patterns/observability/distributed-tracing.html) — retrieved 2026-05-26

Last verified: 2026-05-26
