---
week: W03
day: Tue
topic_slug: idempotency-for-state-mutating-tools
topic_title: "Idempotency for state-mutating tools"
parent_overview: W03/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://stripe.com/blog/idempotency
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.stripe.com/api/idempotent_requests
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_prevent_interaction_failure_idempotent.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.langchain.com/oss/python/langgraph/durable-execution
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://aloknecessary.github.io/blogs/idempotency-distributed-systems/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Idempotency for state-mutating tools

## 1. Learning Objectives

- By the end of this reading, the learner can define idempotency in the context of a tool-using LLM agent and explain why it is non-optional for state-mutating tools.
- By the end of this reading, the learner can list three implementation shapes (unique-constraint column, conditional insert, Redis SETNX) and choose between them based on scope.
- By the end of this reading, the learner can explain why LangGraph's checkpoint/resume model makes idempotency a correctness requirement rather than a hygiene preference.
- By the end of this reading, the learner can construct an idempotency key from a meaningful business tuple and articulate why the key shape matters as much as the storage shape.
- By the end of this reading, the learner can identify three common failure modes — replay drift, key collision across tenants, and expiration mismatches — and the defences against each.

## 2. Introduction

"Idempotent" — from the Latin for "same power" — describes an operation whose effect is the same whether you run it once or a hundred times. `set_temperature(72)` is idempotent; `increment_temperature_by(2)` is not. In tool-using LLM agent systems, idempotency is the property that lets you retry a tool call after a network blip, a runtime crash, or a checkpoint replay without producing duplicate state.

The need is structural. Agents run inside non-deterministic runtimes — LangGraph can crash mid-graph and resume from the last checkpoint, an HTTP-level retry can re-fire a tool call after a timeout, a multi-agent supervisor can re-dispatch a worker that already completed. Without idempotency, any of these produces duplicate side effects: two payments, two notifications, two database rows, two audit entries. The fintech industry learned this lesson hard enough that Stripe built idempotency into the API surface as a first-class header and has published the canonical engineering write-up on the pattern.

This reading walks the contract, the three storage shapes, the key-construction question, and the LangGraph-specific reason this discipline applies at agent-tool boundaries even more than at conventional API boundaries.

## 3. Core Concepts

### 3.1 What "idempotent" means precisely

A tool call is idempotent if calling it twice with the same inputs (including the same idempotency key) produces:

- **The same observable side effect** — one row inserted, not two; one email sent, not two; one notification fired, not two.
- **The same response payload** — the second call returns the result of the first, not an error and not a fresh execution.
- **The same downstream effects** — if the tool emits an event, the event is emitted once, not twice.

Note "with the same inputs" is doing real work. Different inputs with the same key is a different problem (parameter mismatch — error) and different inputs with different keys is just two distinct operations.

### 3.2 Why agents make this harder than ordinary APIs

In a normal client-server API, retries happen at one layer: the client sees a timeout and retries. The client controls when to retry.

In an agent runtime, retries happen at three layers simultaneously:

1. **The model.** A ReAct loop that sees a `tool_result` error can decide to retry the call with the same arguments.
2. **The HTTP layer.** Bedrock, the FastAPI gateway, or an internal service mesh may retry on 5xx or timeouts.
3. **The graph runtime.** LangGraph (and similar durable runtimes) checkpoints state between supersteps; on resume, a node may re-execute from the beginning. Any tool call inside that node fires again unless the call itself is idempotent.

The LangGraph documentation makes this explicit: when a graph resumes from an `interrupt()` or a crash, **the entire node re-executes from the beginning**. Therefore, developers must ensure that any logic placed before the interrupt — including API requests that incur charges or database writes — is idempotent.

### 3.3 The three storage shapes

**Shape 1 — Idempotency-key column with a unique constraint.** The most common pattern. The tool accepts an explicit `idempotency_key` argument. The mutating table has a `UNIQUE` constraint on that column. A duplicate call fails the constraint; the application catches the violation, looks up the prior result, and returns it.

```sql
CREATE TABLE dispatch_log (
    id            BIGSERIAL PRIMARY KEY,
    vehicle_id    UUID NOT NULL,
    job_id        UUID NOT NULL,
    idempotency_key TEXT NOT NULL,
    result_json   JSONB NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (idempotency_key)
);
```

Strengths: durable, survives application restart, easy to audit. Weaknesses: requires a schema migration if you bolt it on later.

**Shape 2 — Conditional insert (no extra column).** Use a natural business tuple as the dedupe key. A `WHERE NOT EXISTS` clause prevents the second insert.

```sql
INSERT INTO dispatch_log (vehicle_id, job_id, dispatched_at)
SELECT $1, $2, now()
WHERE NOT EXISTS (
    SELECT 1 FROM dispatch_log WHERE vehicle_id = $1 AND job_id = $2
);
```

Strengths: no extra column. Weaknesses: the dedupe scope is the business tuple — you cannot retry the *same* (vehicle, job) on purpose after the first attempt succeeded.

**Shape 3 — Redis SETNX with TTL.** When the write spans multiple tables, services, or systems that cannot share a SQL constraint, use Redis as a deduplication store.

```
SETNX idempotency:{key} 1 EX 86400
```

If the SET succeeded the key was new; proceed. If it failed the key was seen; look up the stored result. Strengths: cross-service, no migration. Weaknesses: TTL-bounded (typical 24 hours, matching Stripe's window); requires the Redis instance to be highly available.

### 3.4 Key construction — the underrated half

The storage shape is mechanical; the key construction is where most idempotency systems break. Three rules:

- **Use a meaningful business tuple, not a random UUID.** A UUID supplied by the model is just a token — if the model retries with a *new* UUID, you get duplicates. A key constructed from `(operation, primary_entity_id, secondary_entity_id, intent_hash)` is reproducible across retries: the model regenerates the same key from the same context.
- **Hash variable-shape inputs.** If the tool input includes a list (e.g., `evaluator_ids: [a, b, c]`), sort and hash before keying. Otherwise `[a, b, c]` and `[b, a, c]` are different keys for the same logical operation.
- **Namespace by tenant.** In a multi-tenant system, prefix the key with `tenant_id`. A collision across tenants is the worst kind of bug — it both leaks data and lies on the audit trail.

A good key looks like: `dispatch:fleet_42:vehicle_9f0d:job_42a1:v1` — readable, reproducible, tenant-scoped, and versioned so you can rotate the keying scheme without breaking existing entries.

### 3.5 What the tool should return on a replay

When a duplicate call arrives, the tool should return the **same response** the original call returned — same status, same payload, same headers. Stripe's API surfaces this with an `Idempotent-Replayed: true` header so clients can detect replays. Agents do not strictly need the flag (the model rarely cares whether it caused the effect or just observed it), but logging the replay is essential for debugging.

### 3.6 Parameter-mismatch errors

A subtle case: the second call uses the same idempotency key but **different parameters**. Stripe treats this as an error and returns 400 — the assumption is that the key was reused by mistake and silently succeeding would corrupt state. Most production idempotency layers do the same. The exception is when the key is constructed deterministically from the parameters, in which case parameter mismatch implies key mismatch — they cannot both happen.

## 4. Generic Implementation

A generic idempotent write tool for a logistics dispatch system, using Shape 1 (unique-constraint column) and a deterministically constructed key.

```python
import hashlib
from pydantic import BaseModel, Field
from uuid import UUID
from typing import Optional
import psycopg

class DispatchVehicleInput(BaseModel):
    vehicle_id: UUID
    job_id: UUID
    eta_minutes: int = Field(ge=0, le=720)
    # The key may be passed by the model; if absent, derived deterministically below.
    idempotency_key: Optional[str] = None

def build_key(inp: DispatchVehicleInput, *, tenant_id: str, version: str = "v1") -> str:
    """
    Construct a deterministic, tenant-scoped, versioned idempotency key.
    Same inputs → same key, every time.
    """
    base = f"{tenant_id}:dispatch:{inp.vehicle_id}:{inp.job_id}:{version}"
    return base  # short enough to use directly; hash if you need fixed length

def dispatch_vehicle(inp: DispatchVehicleInput, *, conn: psycopg.Connection, tenant_id: str):
    key = inp.idempotency_key or build_key(inp, tenant_id=tenant_id)

    with conn.transaction():
        # Try the insert; UNIQUE constraint on idempotency_key catches duplicates.
        try:
            row = conn.execute(
                """
                INSERT INTO dispatch_log
                  (tenant_id, vehicle_id, job_id, eta_minutes, idempotency_key, result_json)
                VALUES (%s, %s, %s, %s, %s, %s)
                RETURNING id, result_json
                """,
                (tenant_id, inp.vehicle_id, inp.job_id, inp.eta_minutes, key, '{"status":"dispatched"}'),
            ).fetchone()
            return {"status": "ok", "id": row[0], "replayed": False, **row[1]}
        except psycopg.errors.UniqueViolation:
            # Second call with the same key — look up and return the original result.
            row = conn.execute(
                "SELECT id, result_json FROM dispatch_log WHERE idempotency_key = %s",
                (key,),
            ).fetchone()
            return {"status": "ok", "id": row[0], "replayed": True, **row[1]}
```

Three things worth noticing:

1. **The key is derived deterministically from inputs**, not generated as a UUID. A model that retries with the same arguments produces the same key automatically — no special prompt needed.
2. **The catch-and-lookup branch returns `replayed: True`**. The tool's caller (the host runtime; the LangSmith trace; the audit log) sees the replay explicitly.
3. **The transaction wraps both branches**. If the lookup fails for any reason after the constraint violation (e.g., row was deleted in the meantime), the whole thing rolls back.

## 5. Real-world Patterns

### 5.1 Fintech — Stripe's reference implementation

Stripe pioneered the idempotency-key pattern for payment APIs. The client supplies an `Idempotency-Key` header on POST requests; Stripe stores the request payload and the eventual response keyed by that header for 24 hours; any retry within the window returns the stored response with `Idempotent-Replayed: true`. The 24-hour window matches their retry-policy guidance — beyond that, the failure mode is operator-decided, not client-decided. The blog post "Designing robust and predictable APIs with idempotency" remains the canonical write-up.

### 5.2 E-commerce — Shopify's order-create idempotency

Shopify's order-create endpoint accepts an idempotency key to prevent duplicate orders during the checkout retry storm that happens whenever a CDN edge hiccups. Their published guidance: the key should be derived from `(cart_id, payment_session_id)` — both stable across retries — rather than from a client-generated UUID that the merchant's frontend might regenerate on a page refresh.

### 5.3 Healthcare — pharmacy fill-request systems

US pharmacy fulfillment networks (Surescripts, CoverMyMeds) treat duplicate fill requests as a controlled-substances compliance issue, not just a hygiene one. Their idempotency keys are constructed from `(prescription_id, fill_sequence_number, signature_hash)` — three fields that are stable across retries but unique across distinct fills. Duplicate suppression is a regulatory requirement under DEA EPCS rules.

### 5.4 Cloud infrastructure — AWS SDKs

Most AWS write APIs accept a `ClientRequestToken` (or equivalent name — `ClientToken`, `IdempotencyToken`) that serves the same role. The AWS Well-Architected Framework's REL04-BP04 lists "Make mutating operations idempotent" as a top-line reliability practice, on the grounds that distributed systems will retry, and the only defence is to make each retry safe by design.

## 6. Best Practices

- **Construct the key deterministically from inputs.** A model-generated UUID is not a key; it is a token, and tokens drift on retry.
- **Namespace keys by tenant.** Cross-tenant collisions are the worst-case bug — they both leak data and falsify the audit trail.
- **Wrap the insert and the lookup in one transaction.** Otherwise a race condition between the constraint violation and the lookup can return inconsistent results.
- **Match TTL to retry policy.** 24 hours is the Stripe default; choose the shortest window that covers your longest realistic retry storm.
- **Log replays explicitly.** The replayed call is informationally different from the original — your trace store should be able to tell them apart, even if the response payload is identical.
- **Reject parameter mismatches.** A second call with the same key and different parameters is a bug, not a retry. Return an error rather than silently succeeding.

## 7. Hands-on Exercise

**Code exercise (15 min).** You are implementing the `send_invoice_email` tool for a SaaS billing system. The model can call this tool to send an invoice to a customer. Requirements:

1. The tool must be idempotent — calling it twice with the same `(invoice_id, customer_email)` pair sends exactly one email.
2. The idempotency window must be 7 days (invoices can be retried up to a week later).
3. The tool returns `{"status": "sent" | "replayed", "message_id": str}`.
4. The tenant boundary must be enforced — the same `invoice_id` belonging to different tenants must not collide.

Write the Pydantic input model, the `build_key()` function, and the tool body using a Postgres unique-constraint table. Add a one-line comment explaining how a model-generated retry with the same arguments produces the same key.

**What good looks like.** A solution constructs the key as `{tenant_id}:invoice_email:{invoice_id}:{sha256(customer_email)}` (note the email hash — emails can be PII), uses a `UNIQUE` constraint on the key column, and returns `replayed: True` on the duplicate path. Bonus: the solution sets a 7-day TTL via a periodic cleanup job on the table rather than relying on Redis TTLs (which fight with database-of-record auditing).

## 8. Key Takeaways

- **Why is idempotency non-optional for state-mutating tools called by LLM agents?** (Three retry layers stack — model, HTTP, graph runtime — and any one of them can fire the same call twice.)
- **What are the three storage shapes for idempotency, and how do you choose?** (Unique-constraint column for the common case; conditional insert when you cannot add a column; Redis SETNX for cross-service or cross-table scope.)
- **Why does deterministic key construction matter more than the storage shape?** (Because a non-deterministic key — like a model-generated UUID — drifts on retry and lets duplicates through.)
- **What does the tool return on a replay?** (The same response as the original call, ideally with a `replayed: True` flag so observability can tell them apart.)
- **What makes LangGraph specifically require this discipline?** (Checkpoint/resume re-executes the entire failing node from the start, including any tool calls before the interrupt.)

## Sources

1. [Designing robust and predictable APIs with idempotency (Stripe)](https://stripe.com/blog/idempotency) — retrieved 2026-05-26
2. [Idempotent requests — Stripe API Reference](https://docs.stripe.com/api/idempotent_requests) — retrieved 2026-05-26
3. [REL04-BP04 Make mutating operations idempotent — AWS Well-Architected Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_prevent_interaction_failure_idempotent.html) — retrieved 2026-05-26
4. [Durable execution — LangGraph docs](https://docs.langchain.com/oss/python/langgraph/durable-execution) — retrieved 2026-05-26
5. [Idempotency in Distributed Systems: Design Patterns Beyond 'Retry Safely'](https://aloknecessary.github.io/blogs/idempotency-distributed-systems/) — retrieved 2026-05-26

Last verified: 2026-05-26
