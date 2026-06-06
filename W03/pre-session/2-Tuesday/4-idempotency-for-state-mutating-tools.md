---
week: W03
day: Tue
topic_slug: idempotency-for-state-mutating-tools
topic_title: "Idempotency for state-mutating tools"
parent_overview: W03/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 12
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
last_verified: 2026-06-06
---

# Idempotency for state-mutating tools

> [!NOTE]
> **From earlier:** Topic 3 established that write tools carry an `idempotency_key` argument and document the contract in their description. This topic explains *why* that matters and *how* to implement it correctly.

## 1. Learning Objectives

- Define idempotency in the context of a tool-using LLM agent and explain why it is non-optional for state-mutating tools
- List three implementation shapes (unique-constraint column, conditional insert, Redis SETNX) and choose between them based on scope
- Explain why LangGraph's checkpoint/resume model makes idempotency a correctness requirement rather than a hygiene preference
- Construct an idempotency key from a meaningful business tuple and articulate why key shape matters as much as storage shape

## 2. Introduction

"Idempotent" — same power — describes an operation whose effect is identical whether you run it once or a hundred times. `set_temperature(72)` is idempotent; `increment_temperature_by(2)` is not. In tool-using agent systems, idempotency lets you retry a tool call after a network blip, a runtime crash, or a checkpoint replay without duplicate side-effects.

The need is structural: LangGraph can crash mid-graph and resume from checkpoint; HTTP retries re-fire on timeout; a multi-agent supervisor can re-dispatch a worker that already completed. Without idempotency each scenario produces duplicate state — two routing rows, two CO notifications, two audit entries.

In `acquire-gov` intake-triage, `route_to_evaluators` and `escalate_to_co` are both state-mutating. A duplicate `route_to_evaluators` row tells the audit trail two evaluator assignments happened — per FAR record-keeping, the audit trail is *lying*, a bigger violation than the original double-write.

## 3. Core Concepts

### 3.1 What "idempotent" means precisely

A tool call is idempotent if calling it twice with the same inputs produces: the same observable side-effect (one row, not two), the same response payload (second call returns first call's result), and the same downstream effects (event fires once). "With the same inputs" is doing real work — different inputs with the same key is a parameter mismatch error, not a replay.

### 3.2 Why agents make this harder than ordinary APIs

In a normal client-server API, retries happen at one layer: the client sees a timeout and retries. In an agent runtime, retries stack at three layers simultaneously:

| Layer | Mechanism | Example |
|-------|-----------|---------|
| Model | ReAct loop sees a `tool_result` error, decides to retry | Same `route_to_evaluators` call with same args |
| HTTP | Service mesh or Bedrock retries on 5xx / timeout | Tool endpoint called twice |
| Graph runtime | LangGraph resumes from checkpoint, re-executes the node | Entire node re-runs including all tool calls |

The LangGraph documentation is explicit: when a graph resumes from an `interrupt()` or a crash, **the entire node re-executes from the beginning**. Any tool call inside that node fires again unless the call itself is idempotent.

> [!IMPORTANT]
> **LangGraph checkpoint/resume is a correctness requirement, not a hygiene preference.** Thu's deep-dive wires `interrupt()` and `Command(resume=...)`. If your write tools are not idempotent before Thu, graph resumption will create duplicate side-effects. Fix it today.

### 3.3 The three storage shapes

**Shape 1 — Unique-constraint column.** Most common. Tool accepts `idempotency_key`; table has `UNIQUE` on it. Duplicate call fails the constraint; app catches, looks up prior result, returns it. Durable, auditable; requires schema migration if added later.

**Shape 2 — Conditional insert.** `WHERE NOT EXISTS` on a natural business tuple. No extra column, but intentional re-submission after the first attempt requires a new key.

**Shape 3 — Redis SETNX with TTL.** `SETNX idempotency:{key} 1 EX 86400`. Cross-service, no migration — but TTL-bounded (24 h) and requires Redis HA.

### 3.4 Key construction — the underrated half

Storage shape is mechanical; key construction is where most systems break. Three rules: use a **meaningful business tuple** (not a model-generated UUID — a new UUID on retry defeats the check); **hash variable-shape inputs** (sort `evaluator_ids` before hashing); **namespace by tenant** (`agency_id` prefix — cross-tenant collisions leak data and falsify the audit trail). Example: `agency_42:route:proposal_9f0d:evals_sha256abc:v1`.

## 4. Generic Implementation

Shape 1 (unique-constraint column) with deterministic key construction:

```python
import hashlib
from pydantic import BaseModel, Field
from uuid import UUID
from typing import Optional
import psycopg

class RouteToEvaluatorsInput(BaseModel):
    proposal_id: UUID
    evaluator_ids: list[UUID]
    idempotency_key: Optional[str] = None

def build_key(inp: RouteToEvaluatorsInput, *, agency_id: str, version: str = "v1") -> str:
    # Sort evaluator list so order doesn't affect the key
    sorted_evals = sorted(str(e) for e in inp.evaluator_ids)
    evals_hash = hashlib.sha256("|".join(sorted_evals).encode()).hexdigest()[:16]
    return f"{agency_id}:route:{inp.proposal_id}:{evals_hash}:{version}"

def route_to_evaluators(inp: RouteToEvaluatorsInput, *, conn: psycopg.Connection, agency_id: str):
    key = inp.idempotency_key or build_key(inp, agency_id=agency_id)
    with conn.transaction():
        try:
            row = conn.execute(
                """
                INSERT INTO routing_log
                  (agency_id, proposal_id, evaluator_ids, idempotency_key, result_json)
                VALUES (%s, %s, %s, %s, %s)
                RETURNING id, result_json
                """,
                (agency_id, str(inp.proposal_id),
                 [str(e) for e in inp.evaluator_ids], key, '{"status":"routed"}'),
            ).fetchone()
            return {"status": "ok", "id": row[0], "replayed": False}
        except psycopg.errors.UniqueViolation:
            row = conn.execute(
                "SELECT id, result_json FROM routing_log WHERE idempotency_key = %s",
                (key,),
            ).fetchone()
            return {"status": "ok", "id": row[0], "replayed": True}
```

Key is derived deterministically from inputs. The catch-and-lookup branch returns `replayed: True`. The transaction wraps both branches.

## 5. Real-world Patterns

> [!TIP]
> **Match TTL to your retry storm window.** Stripe's 24-hour window covers even slow CDN retry queues. For `acquire-gov` write tools, 24 hours is the safe default.

**Fintech (Stripe).** Clients supply `Idempotency-Key`; Stripe stores request + response for 24 hours; retries return the stored response with `Idempotent-Replayed: true`. The canonical reference implementation.

**E-commerce (Shopify order-create).** Key derived from `(cart_id, payment_session_id)` — stable across retries — not a frontend-regenerated UUID. Guards against the checkout retry storm on CDN edge hiccups.

**Healthcare (Surescripts, CoverMyMeds).** Duplicate fill requests are a DEA EPCS regulatory issue. Keys are `(prescription_id, fill_sequence_number, signature_hash)` — stable across retries, unique across fills.

> [!NOTE]
> **Cross-domain signal.** The `acquire-gov` federal-audit-trail requirement maps directly to the DEA EPCS story: in both cases, a duplicate write falsifies the official record of what occurred.

## 6. Best Practices

- Construct the key deterministically from inputs — model-generated UUIDs drift on retry
- Namespace by tenant — cross-tenant collisions leak data and falsify the audit trail
- Wrap insert and lookup in one transaction — race between constraint violation and lookup returns inconsistent results
- Match TTL to retry policy — 24 h is Stripe's default; choose the shortest window that covers your longest retry storm
- Log replays explicitly — a replayed call is informationally different from the original

> [!WARNING]
> **Anti-pattern: `state-mutation-without-idempotency-key`.** A write tool that accepts no idempotency key cannot be safely retried under any of the three retry layers (model, HTTP, graph runtime). When LangGraph's checkpoint/resume re-executes a node containing such a tool, the result is guaranteed duplicate side-effects. Every write tool in the intake-triage flow must have an idempotency key before Thursday's LangGraph deep-dive.

## 7. Hands-on Exercise

You are implementing `send_invoice_email` for a SaaS billing system. Requirements: (1) calling it twice with the same `(invoice_id, customer_email)` sends exactly one email; (2) the idempotency window is 7 days; (3) the tool returns `{"status": "sent" | "replayed", "message_id": str}`; (4) the same `invoice_id` for different tenants must not collide. Write the Pydantic input model, the `build_key()` function, and the tool body using a Postgres unique-constraint table.

> [!NOTE]
> **Self-check** (30 s — answer mentally before expanding)
>
> 1. Why is a model-generated UUID a poor idempotency key?
> 2. What does the tool return when the same key is submitted a second time?

<details>
<summary>Show answers</summary>

1. A UUID generated by the model is a random token. On retry the model may generate a *new* UUID — different key, different constraint check — so the duplicate insert succeeds and produces two side-effects. A key derived deterministically from the business tuple (invoice_id, tenant_id) is the same on every retry regardless of what the model generates.
2. The same response payload as the original call, ideally with a `replayed: True` flag so observability tooling (LangSmith, audit log) can distinguish the replay from the original. The underlying operation does not execute again.

</details>

## 8. Key Takeaways

- **Three retry layers stack** — model, HTTP, graph runtime — any one can fire the same write tool twice without idempotency
- **Three storage shapes:** unique-constraint column (most common), conditional insert (no extra column), Redis SETNX (cross-service)
- **Deterministic key construction matters more than storage shape** — a non-deterministic key lets duplicates through regardless of storage
- **Replay returns the original result** with a `replayed` flag — the tool consumer gets consistency, the trace gets visibility
- **LangGraph checkpoint/resume specifically requires this** — the entire failing node re-executes from the start on resume

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://stripe.com/blog/idempotency — Designing robust and predictable APIs with idempotency (Stripe) — retrieved 2026-05-26 — foundation-stable
- https://docs.stripe.com/api/idempotent_requests — Idempotent requests (Stripe API Reference) — retrieved 2026-05-26 — foundation-stable
- https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_prevent_interaction_failure_idempotent.html — REL04-BP04 Make mutating operations idempotent (AWS Well-Architected) — retrieved 2026-05-26 — foundation-stable
- https://docs.langchain.com/oss/python/langgraph/durable-execution — Durable execution (LangGraph docs) — retrieved 2026-05-26 — hot-tech
- https://aloknecessary.github.io/blogs/idempotency-distributed-systems/ — Idempotency in Distributed Systems: Design Patterns Beyond 'Retry Safely' — retrieved 2026-05-26 — foundation-stable

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

Parameter-mismatch errors are a subtle edge case: the second call uses the same idempotency key but **different parameters**. Stripe treats this as a 400 error — the assumption is that the key was reused by mistake and silently succeeding would corrupt state. Most production idempotency layers do the same. The exception: when the key is constructed deterministically from the parameters, parameter mismatch implies key mismatch — they cannot both happen simultaneously.

For Shape 3 (Redis SETNX), the atomicity guarantee depends on a single Redis instance or Redis Cluster with appropriate quorum. In a distributed Redis setup, the SETNX can race: two concurrent calls may both observe the key as absent before either writes it. Redlock (the Redis distributed lock algorithm) addresses this but adds complexity. For most `acquire-gov` write tools, Shape 1 (Postgres unique constraint) is the correct choice — it inherits Postgres's ACID guarantees and survives application restarts.

For the `evaluate` or `score` operations (Wed's multi-agent flow), consider whether the key should include a `run_id` or `attempt_number` to allow intentional re-scoring of the same proposal. If the business rule is "one score per proposal per evaluator," the key is `(agency_id, proposal_id, evaluator_id)`; if re-scoring on new information is allowed, the key must include a version or timestamp component.

</details>

Last verified: 2026-06-06
