---
week: W01
day: Thu
topic_slug: async-fastapi-integration-with-spring-boot
topic_title: "Async FastAPI integration with Spring Boot"
parent_overview: W01/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 8
sources:
  - url: https://fastapi.tiangolo.com/advanced/events/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.python-httpx.org/async/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://fastapi.tiangolo.com/async/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.spring.io/spring-framework/reference/web/webflux-webclient.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://asgi.readthedocs.io/en/latest/specs/lifespan.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Async FastAPI integration with Spring Boot

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish `async def` from `def` path operations in FastAPI and predict the runtime behaviour of each.
- Implement a FastAPI `lifespan` async context manager to manage long-lived clients (HTTP, database, model).
- Make non-blocking outbound HTTP calls from FastAPI using `httpx.AsyncClient` with a single shared instance.
- Configure a Spring `WebClient` bean and explain why it is non-blocking by default.
- Reason about which side of a Python-Python or Python-Java integration should be async-end-to-end and which can stay synchronous.

## 2. Introduction

Async Python and reactive Java are two solutions to the same problem: how to wait for many slow I/O operations at once without burning a thread per wait. The cohort touches both. FastAPI on the Python side runs on an asyncio event loop and exposes `async def` endpoints. Spring on the Java side offers `WebClient` (in the WebFlux module) as the non-blocking complement to the classic blocking `RestTemplate`. The two stacks are compatible when each is configured deliberately.

The reason this matters in W1 is small but load-bearing: Friday's PR work introduces streaming + retries to the AI orchestrator. Streaming responses to a slow LLM is the canonical use case for `async def` — an event loop holding many in-flight requests is dramatically cheaper than a threadpool with one blocked thread per request. Get the async shape wrong now and W3's multi-agent work magnifies the cost; get it right and the patterns extend cleanly.

This reading is a foundation-stable reference. The patterns described — lifespan handlers, shared `AsyncClient`, `WebClient` beans, the `async def`-vs-`def` rule — have been canonical since FastAPI 0.93 (mid-2023) and Spring Framework 5 (2017) respectively. They are not the bleeding edge; they are the practical baseline.

## 3. Core Concepts

### 3a. `async def` vs `def` in FastAPI — a real correctness issue

FastAPI accepts both `async def` and `def` for path-operation functions. The choice is *not* stylistic — it changes where the function runs:

- **`async def`** runs *on the event loop*. Any `await` inside it yields control so other requests can progress. Any *blocking* call inside it (a synchronous database query, a `time.sleep`, a CPU-bound computation) blocks the entire event loop and stalls every other in-flight request.
- **`def`** runs *on a threadpool* (default 40 workers). It can block freely without stalling other requests, but each request occupies a thread for its full duration. Throughput is bounded by the threadpool size.

The rule that flows from this: if your function's I/O has an async client, use `async def` and `await` the I/O. If your function's I/O is blocking-only and there's no async client available, use `def` and let FastAPI run it on the threadpool. The bug that occurs most often in cohort code is calling blocking I/O inside `async def` — the function looks async, the calls don't yield, the event loop stalls. The FastAPI docs are explicit about this and it remains the most common production fault in async Python code.

### 3b. The lifespan handler — long-lived resources

FastAPI exposes a `lifespan` async context manager for resources whose lifetime is the application's, not a single request's. The canonical examples: an HTTP client, a database connection pool, a Bedrock or other model client.

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
import httpx

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: build clients once
    app.state.http = httpx.AsyncClient(timeout=30.0, limits=httpx.Limits(max_connections=100))
    yield
    # Shutdown: release cleanly
    await app.state.http.aclose()

app = FastAPI(lifespan=lifespan)
```

Two properties matter. First, the `@app.on_event("startup")` / `@app.on_event("shutdown")` decorators are *deprecated* — they pre-date the ASGI lifespan protocol and Starlette has marked them for removal. Always use `lifespan` in new code. Second, building the `AsyncClient` once and sharing it across requests is required for HTTP connection pooling to actually pool — instantiating a new client per request defeats the entire benefit and is a common-enough anti-pattern that multiple 2026 tutorials open with it as the bug to avoid.

### 3c. `httpx.AsyncClient` for outbound HTTP

`httpx` is the async HTTP client FastAPI's documentation and the broader ecosystem standardise on. The usage pattern from inside an `async def` endpoint:

```python
from fastapi import FastAPI, Request

@app.post("/proxy-search")
async def proxy(request: Request, query: str):
    http: httpx.AsyncClient = request.app.state.http
    response = await http.get("https://upstream.example.com/search", params={"q": query})
    response.raise_for_status()
    return response.json()
```

Key configuration knobs:

- `timeout=` — set per-call or per-client; `httpx.Timeout(connect=5.0, read=30.0)` for fine-grained control.
- `limits=httpx.Limits(max_connections=N, max_keepalive_connections=K)` — pool sizing.
- `transport=httpx.AsyncHTTPTransport(retries=N)` — automatic connection-level retries (separate from application-level retries via Tenacity or similar).

The shared-client pattern interacts with FastAPI's dependency injection cleanly: build the client in `lifespan`, expose it via a `Depends(...)` getter, inject it into endpoints. This makes testing straightforward — swap the dependency for a mock in tests.

### 3d. Spring `WebClient` — the Java complement

Spring's `WebClient` is the non-blocking analogue. It lives in the `spring-webflux` module and is non-blocking by default; it runs on Reactor Netty's event loop. The Spring Framework reference docs converge on three configuration recommendations:

1. **Centralise the bean.** One `WebClient.Builder` configured with the base URL, default headers, and timeouts. Inject as a `WebClient` everywhere.
2. **Keep HTTP I/O on the event loop.** Do not call `.block()` from request-handling threads — that converts the non-blocking client into a blocking one and defeats the purpose.
3. **Layer timeouts.** A single `.timeout(...)` at the end of the chain is not enough; layer connect, read, response timeouts so each failure mode has a bounded recovery path.

```java
@Configuration
class HttpClientConfig {
    @Bean
    WebClient aiOrchestratorClient() {
        return WebClient.builder()
            .baseUrl("http://ai-orchestrator:8000")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }
}

@Service
class DraftService {
    private final WebClient http;
    DraftService(WebClient aiOrchestratorClient) {
        this.http = aiOrchestratorClient;
    }

    Mono<DraftResponse> requestDraft(DraftRequest req) {
        return http.post()
            .uri("/draft")
            .bodyValue(req)
            .retrieve()
            .bodyToMono(DraftResponse.class)
            .timeout(Duration.ofSeconds(30));
    }
}
```

### 3e. The shape of the integration — async where, blocking where?

A Python-Java integration has four nodes: client → Java service → Python service → upstream (model API or other). Each hop can be sync or async independently. A defensible W1 starting shape:

- **Client → Java (Spring controller).** Either; depends on the surrounding stack. A traditional Spring MVC `@RestController` blocks per request, which is fine for moderate-throughput business endpoints.
- **Java → Python (Spring `WebClient` → FastAPI).** Async on both sides if the Java caller needs concurrency (multiple parallel calls). For W1's single-call shape, the Java side may block on the response (`.block()`) deliberately, accepting the throughput limit to keep the Java code simple.
- **Python (FastAPI) → upstream.** Async end-to-end — `async def` endpoint, shared `httpx.AsyncClient`, awaited model SDK calls. The LLM latency makes this the most important hop to keep non-blocking; streaming responses from a model amplify the benefit further.

The W1 simplification is "Spring blocks; FastAPI is async-all-the-way." W3 multi-agent work introduces concurrent calls on the Spring side and moves the integration toward async-end-to-end. The simplification is deliberate: get the async discipline right on the side that matters most first, then extend.

## 4. Generic Implementation

A worked example outside federal acquisitions — a healthcare platform's Java patient-portal calling a Python clinical-summarisation service.

**FastAPI side (`summariser/main.py`):**

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, HTTPException, Request
from pydantic import BaseModel
import httpx

class SummaryRequest(BaseModel):
    patient_id: str
    note_text: str

class SummaryResponse(BaseModel):
    summary: str
    confidence: float

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.http = httpx.AsyncClient(
        timeout=httpx.Timeout(connect=5.0, read=30.0),
        limits=httpx.Limits(max_connections=50),
    )
    yield
    await app.state.http.aclose()

app = FastAPI(lifespan=lifespan)

@app.post("/summarise", response_model=SummaryResponse)
async def summarise(req: SummaryRequest, request: Request):
    http: httpx.AsyncClient = request.app.state.http
    try:
        # Outbound call to model provider — awaited, non-blocking
        r = await http.post(
            "https://model.example.com/v1/messages",
            json={"input": req.note_text},
        )
        r.raise_for_status()
        body = r.json()
    except httpx.HTTPError as exc:
        raise HTTPException(status_code=502, detail=str(exc))
    return SummaryResponse(summary=body["summary"], confidence=body["confidence"])
```

**Spring side (`patient-portal/src/.../SummaryClient.java`):**

```java
@Service
public class SummaryClient {
    private final WebClient http;

    public SummaryClient(WebClient.Builder builder) {
        this.http = builder.baseUrl("http://summariser:8000").build();
    }

    public SummaryResponse fetchSummary(String patientId, String noteText) {
        return http.post()
            .uri("/summarise")
            .bodyValue(Map.of("patient_id", patientId, "note_text", noteText))
            .retrieve()
            .bodyToMono(SummaryResponse.class)
            .timeout(Duration.ofSeconds(30))
            .block();   // W1-shape: Spring blocks on the response
    }
}
```

The `.block()` call on the Spring side is the deliberate W1 simplification — it converts the async `Mono` into a synchronous return, accepting the per-request thread cost in exchange for a simpler caller shape. W3 introduces parallel calls (`Flux.merge`, `Mono.zip`) that make the non-blocking path worth its complexity.

## 5. Real-world Patterns

**E-commerce — checkout fraud-screening:** A retailer's Java checkout service calls a Python fraud-screening service that runs an ML model. The fraud service is async end-to-end with a shared `httpx.AsyncClient` for outbound calls to a feature store and a model endpoint. The Java side uses `WebClient` with a 200ms timeout — if the screening service is slow, checkout proceeds with a "review later" flag rather than blocking the user. The async shape is load-bearing here because checkout latency is a measured business metric.

**Healthcare — radiology-report drafting:** A clinical-portal in Java sends DICOM-metadata + image URLs to a Python service that orchestrates a vision-model call. The Python service's `async def` endpoint awaits the model call (~5–8s typical); the Java side uses `WebClient` and exposes a long-poll endpoint that the UI reconnects to on completion. The async event loop on the Python side serves dozens of in-flight requests concurrently with a small worker count.

**Logistics — route-optimisation API:** A Java dispatch system calls a Python optimisation service that runs both an LP solver and an ML-based ETA model. The Python service uses `lifespan` to manage a Redis cache client + a solver pool; outbound model calls go through a shared `httpx.AsyncClient`. The Java side uses `WebClient` with explicit retry-on-503 logic to handle the solver pool's queue-full responses.

**Gaming — live-service moderation:** A Java game-server calls a Python content-classification service for in-game chat moderation. The Python service is async end-to-end; the Java side uses `WebClient` with a tight 100ms timeout because moderation latency directly affects player experience. The shared-client pattern on the Python side is load-bearing — the per-request connection cost of instantiating a new client would dominate at the service's throughput.

## 6. Best Practices

- Use `async def` for path operations whose I/O has an async client; use `def` for path operations whose I/O is blocking-only.
- Never call blocking I/O inside `async def` — it stalls the event loop and degrades throughput for every concurrent request.
- Build long-lived clients (HTTP, DB, model) once in the `lifespan` handler; share them across requests via `app.state` or dependency injection.
- Replace deprecated `@app.on_event` handlers with `lifespan` in any new code; the deprecated form is on the Starlette removal list.
- On the Spring side, centralise the `WebClient` bean configuration and inject it everywhere; avoid creating clients on-the-fly.
- Layer timeouts at the connect, read, and overall-response levels — a single end-of-chain timeout does not cover all failure modes.
- Make the async-vs-blocking decision deliberate at each hop in a polyglot integration — async-end-to-end is best for high-concurrency or LLM-streaming paths; selective blocking is acceptable elsewhere.

## 7. Hands-on Exercise

**Task (10–15 min):** Build a minimal FastAPI service with these properties:

1. A `lifespan` handler that constructs a shared `httpx.AsyncClient` with `timeout=30.0` and closes it on shutdown.
2. An `async def` endpoint `/echo-upstream` that takes a query string `target_url`, makes a GET request to that URL using the shared client, and returns the upstream response's status code and first 200 chars of body.
3. A second `async def` endpoint `/version` that returns a hardcoded version string — no I/O. (This exists to demonstrate that pure-async endpoints don't *need* an awaited call.)
4. A `def` endpoint `/cpu-task` that does ~500ms of CPU-bound work (e.g., a tight loop summing integers). (This exists to demonstrate when `def` is the right choice.)

Start the service with `uvicorn`. Send 20 concurrent requests to `/echo-upstream` and confirm none block one another. Send 20 concurrent requests to `/cpu-task` and observe that they queue through the threadpool.

**What good looks like:** The lifespan handler logs "startup complete" and "shutdown complete." The `/echo-upstream` endpoint reuses a single client across all 20 concurrent requests (you can verify by checking `httpx`'s connection-pool metrics or by adding a connection-event log). The `/cpu-task` endpoint runs without any `await` keyword in sight — using `async def` for CPU work would be the bug. If your `/echo-upstream` opens a new client per request, redo it; that's the most common variant of the anti-pattern.

## 8. Key Takeaways

- *When do I use `async def` vs `def`?* `async def` for code whose I/O is awaitable; `def` for blocking or CPU-bound code that should run on the threadpool.
- *What is `lifespan` for?* Long-lived resources (HTTP clients, DB pools, model clients) whose lifetime is the application, not a single request — and it replaces the deprecated `@app.on_event` decorators.
- *Why share an `httpx.AsyncClient`?* Connection pooling, TLS reuse, and HTTP/2 multiplexing — instantiating per request defeats all three.
- *What's the Spring complement to FastAPI async?* `WebClient`, non-blocking by default, configured as a centralised bean with layered timeouts.
- *When is "Spring blocks; FastAPI is async" defensible?* When throughput on the Java side is moderate and the simpler code is worth more than the throughput ceiling — the W1 starting shape, extended as W3 introduces concurrency.

## Sources

1. [FastAPI Lifespan Events (FastAPI docs)](https://fastapi.tiangolo.com/advanced/events/) — retrieved 2026-05-26
2. [HTTPX Async Support (HTTPX docs)](https://www.python-httpx.org/async/) — retrieved 2026-05-26
3. [Concurrency and async / await (FastAPI docs)](https://fastapi.tiangolo.com/async/) — retrieved 2026-05-26
4. [WebClient (Spring Framework reference docs)](https://docs.spring.io/spring-framework/reference/web/webflux-webclient.html) — retrieved 2026-05-26
5. [Lifespan Protocol (ASGI specification)](https://asgi.readthedocs.io/en/latest/specs/lifespan.html) — retrieved 2026-05-26

Last verified: 2026-05-26
