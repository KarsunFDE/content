---
week: W01
day: Tue
topic_slug: the-acquire-gov-training-repo
topic_title: "The training repo — microservices mesh, container-first, four services across three runtimes"
parent_overview: W01/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://konghq.com/blog/enterprise/the-difference-between-api-gateways-and-service-mesh
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://apisix.apache.org/learning-center/api-gateway-for-microservices/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.docker.com/compose/compose-file/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://istio.io/latest/about/service-mesh/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.solo.io/topics/istio/service-mesh-vs-api-gateway
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# The training repo — microservices mesh, container-first, four services across three runtimes

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish **north-south** (client-to-service) from **east-west** (service-to-service) traffic and name which layer (API gateway vs service mesh) handles each.
- List at least five capabilities an API gateway provides at the edge of a microservices system.
- Explain the trade-off between a **monolith** and a **microservices mesh** in the language of a federal modernization conversation (6R, replatform-one-service-at-a-time).
- Read a multi-service `docker-compose.yml` and name the four moving parts: service definitions, port mappings, depends-on ordering, container-internal DNS.

## 2. Introduction

Microservices is the architecture pattern of decomposing a single application into multiple independently-deployable services that communicate over the network. The Apache APISIX 2024 reference characterises the trade-off bluntly: "microservices simplify scaling individual components but introduce communication complexity that a monolith never had to handle." A monolith is one process, one deploy unit, one logging stream; a microservices mesh is N processes, N deploys, N logging streams, and a communication fabric between them.

That communication fabric splits into two layers in 2025 industry practice. The **API gateway** sits at the edge — every external client (browser, mobile app, partner integration) goes through it. The **service mesh** sits internal — every service-to-service call goes through it. The Kong 2024 article frames the distinction crisply: "An API gateway handles north-south traffic (client-to-service), while a service mesh manages east-west traffic (service-to-service)." Both can coexist; large systems run both; small systems often run only the gateway.

This reading walks the API-gateway-and-mesh conceptual model generically, so when you tour an unfamiliar microservices repo at a federal client (or in `acquire-gov` this Tuesday afternoon), you can immediately ask: where is the edge? what services are behind it? are they talking directly, or through a mesh layer? The architecture decision is also a **modernization** decision — federal procurements increasingly require systems that can be 6R-mapped at the service level, not the application level, which means a service-decomposed shape is the load-bearing precondition for any future "replatform one piece at a time" conversation.

## 3. Core Concepts

### 3.1 North-south vs east-west — the two traffic axes

The Solo.io 2024 reference draws the diagram every architecture conversation eventually returns to:

```
                  ┌─────────────┐
                  │   Client    │   (browser, mobile, partner)
                  └──────┬──────┘
                         │   north-south
                         ▼
                  ┌─────────────┐
                  │ API Gateway │   (auth, rate-limit, routing)
                  └──────┬──────┘
                         │
            ┌────────────┼────────────┐
            │            │            │
            ▼            ▼            ▼
       ┌────────┐   ┌────────┐   ┌────────┐
       │ Svc A  │◄──┤ Svc B  │──►│ Svc C  │     east-west
       └────────┘   └────────┘   └────────┘
              ▲                       ▲
              └───────── mesh ────────┘
```

The Kong 2024 piece quotes it directly: "**North-South (External):** flows between external clients and your services, crossing the network perimeter, including mobile apps, browsers, and third-party integrations. **East-West (Internal):** the communication occurring between internal services, staying entirely within the network boundary." The two axes need different controls because they have different threat models, different latency budgets, and different operators.

### 3.2 What an API gateway actually does

The Apache APISIX 2024 and Kong 2024 references converge on a stable list of gateway capabilities. At a federal-acquisitions client, expect every one of these:

| Capability | What it solves |
|------------|----------------|
| **Authentication** (JWT/OAuth2, mTLS, API keys) | Single place to validate every incoming request — back-end services trust the gateway's word |
| **Rate limiting** | Per-consumer, per-tenant, or per-endpoint quotas. Prevents a single bad actor from saturating the system. |
| **Routing & versioning** | `/v1/solicitations` vs `/v2/solicitations` map to different services or different versions |
| **Tenant resolution** | Multi-tenant systems use a JWT claim (or hostname) at the gateway to pick which tenant's data to expose |
| **Payload transformation** | Inbound XML → JSON for legacy clients, outbound field-filtering for compliance |
| **Observability** | Single point to emit request metrics, access logs, distributed-trace start spans |
| **Spike-arrest & circuit-breaking** | When a downstream service degrades, the gateway can shed load instead of cascading the failure |

The Kong piece states the consolidation principle plainly: "Authenticating at the gateway gives you one place to enforce token validation, scopes, rate limits, audit logging, and identity forwarding, which usually reduces duplicated security code across backend services." This is why federal architectures lean hard on gateways — auditors want one place to look.

### 3.3 What a service mesh adds beyond a gateway

The Kong and Solo.io articles describe the service-mesh layer as an *infrastructure* concern, deployed alongside services (typically as sidecar proxies, or in newer "ambient mode" via shared node agents). Capabilities:

| Capability | What it solves |
|------------|----------------|
| **mTLS automatically** | Every east-west call encrypted, both ends authenticated, without app code changes |
| **Load balancing** between service replicas | Round-robin / least-request / locality-aware |
| **Service discovery** | Services find each other by logical name (not IP) — replicas come and go |
| **Distributed tracing** | Span propagation across service calls without app code changes |
| **Resilience policies** | Retries with backoff, timeouts, circuit breakers — declared as policy, not coded into every service |

Not every microservices system runs a mesh. The CloudNativeNow 2025 piece notes a trend back toward simpler architectures for systems with fewer than ~15 services — the mesh's operational cost (Istio control-plane management, sidecar overhead) does not pay off until scale demands it. For a four-service training system, **a service mesh would be overengineering**; for a 200-service production system at a federal agency, the mesh's policy-declaration leverage becomes essential.

### 3.4 Container-first as the run-time discipline

The 2024-2025 industry consensus, visible in every microservices reference cited here, is that microservices run in containers. The implications for an engineer landing in an unfamiliar microservices repo:

- **Each service has its own Dockerfile and pinned runtime.** You don't install Java/Python/Node on your host machine to run the system; you `docker compose up`.
- **Services talk to each other by container-DNS name**, not `localhost`. Inside the compose network, `api-gateway` resolves to the gateway container's IP; `localhost:8080` from inside a container is the container itself.
- **`docker-compose.yml` is the canonical "what runs" doc.** Reading it tells you every service, every port, every volume, every environment variable wired in.
- **Health endpoints are mandatory.** Each service exposes `/actuator/health` (Spring) or `/health` (FastAPI/Express). `docker compose ps` shows the green/red state at a glance.
- **`depends_on` waits for container start, not service-ready.** A common bug: Postgres takes 5–10 seconds to accept connections; Spring Boot tries to connect at second 2 and crashes. Application-level retry loops are the standard fix.

### 3.5 Why microservices at all — the modernization angle

The Apache APISIX reference frames this around modernization economics: "A monolith makes 'replatform one piece' a year-long migration; a microservices mesh makes it a two-week pull request." Federal procurement language increasingly demands **6R-mappable** systems: Rehost / Replatform / Refactor / Repurchase / Retain / Retire, where each R can be applied to a *service*, not the whole *application*. If the application is a monolith, every 6R decision is an all-or-nothing bet. If it's decomposed into services, you can mix — "we're Replatforming the evaluation service to ECS Fargate, Retaining the gateway on EC2, Refactoring the AI orchestrator to Lambda." That conversation is the one a contracting officer can defend in a CIO review.

## 4. Generic Implementation

A minimal four-service `docker-compose.yml` for a generic e-commerce mesh (Angular SPA → Spring gateway → two Spring services + a Python AI service) — naming intentionally generic, no Karsun domain terms:

```yaml
version: "3.9"
services:
  frontend:
    build: ./frontend
    ports: ["4200:4200"]
    depends_on: [api-gateway]
    # Browser hits this on host:4200; it then calls api-gateway:8080 via the
    # docker network. NOT localhost:8080 — that would be inside this container.

  api-gateway:
    build: ./services/api-gateway
    ports: ["8080:8080"]
    environment:
      JWT_ISSUER_URI: http://idp.example.com
      DOWNSTREAM_ORDERS: http://orders-service:8081
      DOWNSTREAM_CATALOG: http://catalog-service:8082
    depends_on: [orders-service, catalog-service, ai-orchestrator]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 10s

  orders-service:
    build: ./services/orders-service
    expose: ["8081"]                # internal only — no host port mapping
    environment:
      DB_URL: jdbc:postgresql://postgres:5432/orders
    depends_on: [postgres]

  catalog-service:
    build: ./services/catalog-service
    expose: ["8082"]
    environment:
      DB_URL: jdbc:postgresql://postgres:5432/catalog
    depends_on: [postgres]

  ai-orchestrator:
    build: ./services/ai-orchestrator
    expose: ["8000"]
    environment:
      LLM_ENDPOINT: ${LLM_ENDPOINT}
      LOG_LEVEL: INFO

  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s

volumes:
  pgdata:
```

Reading this file end-to-end takes ~2 minutes once you know the dialect. You learn: what services exist (5), how the frontend reaches the gateway (browser → host:4200 → SPA → gateway:8080), how the gateway reaches downstream services (compose-DNS, internal-only ports), where data lives (a single Postgres container with named volume), and where the boot-order dependencies are.

## 5. Real-world Patterns

**Fintech (Stripe's internal architecture).** Stripe famously runs a Ruby/Scala-heavy microservices mesh with hundreds of services. Their Envoy-based gateway handles every external API call; mTLS via internal mesh handles east-west. The pattern: **gateway terminates client TLS, mesh handles internal TLS** — a security defence-in-depth model the Istio reference documents as the modern default for regulated industries.

**Healthcare IT (Epic's Care Everywhere integration).** Epic's hospital-to-hospital data exchange runs over an API-gateway layer that handles authentication, FHIR-payload validation, and per-organization rate limiting. There is no service mesh in the traditional sense — the gateway is the only network-policy enforcement point, because the architecture is gateway-and-direct-service rather than mesh-of-services. This is a reminder: **not every microservices system needs a mesh**.

**E-commerce (Shopify's storefront platform).** Shopify's gateway handles per-merchant tenant routing — the hostname `mystore.myshopify.com` is resolved at the gateway to determine which merchant's data to load, which feature flags apply, and which rate-limit tier to enforce. The pattern: **tenant resolution at the gateway, not in every service** — services trust the gateway's tenant claim and operate on the data the gateway tells them is in-scope.

**Gaming (Riot Games' live-service backends).** Riot has discussed publicly using a service mesh (Envoy + custom control plane) for east-west traffic between match-server orchestration, ranked-match-making, and player-profile services. The mesh's selling point for Riot is the *resilience policy*: a misbehaving downstream service can be circuit-broken at the mesh layer without every upstream service implementing its own retry logic. The pattern: **mesh as resilience-policy substrate at scale.**

## 6. Best Practices

- **Always read the `docker-compose.yml` (or k8s manifests) first when landing in an unfamiliar microservices repo.** It tells you what runs and how it's wired.
- **Default to "gateway handles north-south, services handle east-west directly" until you have ≥ 15 services or hard mTLS-everywhere requirements** — only then add a mesh.
- **Put authentication, tenant resolution, and rate-limiting at the gateway, not in each service.** Duplicating this logic across services is how it drifts out of sync.
- **Expose services to the docker network with `expose:` (internal-only) instead of `ports:` (host-mapped) unless you need direct access.** Reduces accidental gateway-bypass.
- **Always include a healthcheck on every service.** `docker compose ps` is your sanity check; without healthchecks it's useless.
- **Use container-DNS names (`api-gateway:8080`), never `localhost`, for service-to-service calls inside compose.** `localhost` inside a container is the container itself.
- **Treat the gateway's bypass routes as a red flag.** If a frontend service calls a domain service directly, you've lost auth/rate-limit/tenant resolution in one stroke.

## 7. Hands-on Exercise

**Whiteboard / 15-min exercise (no code):**

Sketch a four-service microservices system for a hypothetical mid-size **logistics platform** (truck-routing SaaS for regional carriers). Required:

1. Identify the four services. Suggested decomposition: `route-planner-service`, `dispatch-service`, `carrier-portal-frontend`, `ai-eta-predictor`.
2. Place the API gateway. Show what hits it (north-south clients) and what it routes to.
3. Mark each port: which is exposed to the host (browser-reachable), which is internal-only.
4. Annotate three gateway-layer concerns: (a) where JWT validation happens, (b) where per-carrier rate-limiting happens, (c) where tenant resolution happens.
5. Indicate the database(s) and how services reach them.

**What good looks like:** a diagram where the browser hits `:4200` (frontend) or `:8080` (gateway), every backend service is internal-only (`expose:`, no `ports:`), the gateway is the single front door, and the three gateway concerns are explicit. The exercise mirrors what you'll do at a real client: read the compose file, sketch the actual mesh, name the gaps. If you can sketch this for an industry you've never worked in, you can sketch it for any federal client.

## 8. Key Takeaways

- *Can I name the difference between north-south and east-west traffic and say which is gateway vs mesh?* (Maps to LO 1.)
- *Can I list five capabilities an API gateway provides?* (Maps to LO 2.)
- *Can I explain monolith-vs-mesh in 6R modernization terms?* (Maps to LO 3.)
- *Can I read a four-service `docker-compose.yml` and name service-defs, ports, depends_on, internal DNS?* (Maps to LO 4.)

## Sources

1. [Kong — The Difference Between API Gateways and Service Mesh](https://konghq.com/blog/enterprise/the-difference-between-api-gateways-and-service-mesh) — retrieved 2026-05-26
2. [Apache APISIX — API Gateway for Microservices: Architecture, Patterns & Best Practices](https://apisix.apache.org/learning-center/api-gateway-for-microservices/) — retrieved 2026-05-26
3. [Compose file reference (Docker docs)](https://docs.docker.com/compose/compose-file/) — retrieved 2026-05-26
4. [What is a service mesh? (Istio docs)](https://istio.io/latest/about/service-mesh/) — retrieved 2026-05-26
5. [Solo.io — Service Mesh vs API Gateway](https://www.solo.io/topics/istio/service-mesh-vs-api-gateway) — retrieved 2026-05-26

Last verified: 2026-05-26
