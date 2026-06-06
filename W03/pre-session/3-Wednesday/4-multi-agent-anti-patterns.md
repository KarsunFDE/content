---
week: W03
day: Wed
topic_slug: multi-agent-anti-patterns
topic_title: "Multi-agent anti-patterns — chatty handoffs, audit fan-out, lost context"
parent_overview: W03/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://www.augmentcode.com/guides/why-multi-agent-llm-systems-fail-and-how-to-fix-them
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://towardsdatascience.com/the-multi-agent-trap/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://xtrace.ai/blog/ai-agent-context-handoff
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://achan2013.medium.com/ai-agent-anti-patterns-part-1-architectural-pitfalls-that-break-enterprise-agents-before-they-32d211dded43
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://microsoft.github.io/code-with-engineering-playbook/observability/correlation-id/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-06-06
---

# Multi-agent anti-patterns — chatty handoffs, audit fan-out, lost context

> [!NOTE]
> **From earlier:** Tue's ReAct anti-patterns lived inside a single agent. Today's three are *new* — they emerge from boundaries that don't exist at single-agent scope. Recognise them as distinct.

## 1. Learning Objectives

- Name three multi-agent-specific anti-patterns and recognise their symptoms in a LangSmith trace.
- Apply the *single-audit-writer* pattern to break audit fan-out without losing observability.
- Use a *threaded correlation ID* to make lost-context failures diagnosable after the fact.
- Distinguish these from Tue's single-agent ReAct anti-patterns — the defences operate on topology shape, not on agent prompts.

## 2. Introduction

Multi-agent systems introduce failure modes that don't exist in single-agent designs. MAST 2026 lists "context collapse" and "inter-agent misalignment" as 36.9% of all production agent failures.

None of the three look like bugs when written: chatty handoffs look like *thorough* supervision; audit fan-out looks like *good* observability; lost context looks like *clean* delegation. Each becomes pathological at scale. Today's load surfaces all three — **Item 2** (audit-log race) and **Item 6** (correlation IDs) are live brownfield debt.

## 3. Core Concepts

### 3.1 Chatty handoffs

**Definition.** The supervisor re-invokes its routing LLM between every worker *step* — not every decision boundary. A five-call flow balloons to thirty-plus LLM calls.

**Symptoms:** supervisor tool-call count is 5×–20× the worker count; most invocations reach the same decision; latency dominated by supervisor calls, not workers.

**Standard defence.** Per-*decision-boundary* invocation — supervisor fires only after a worker completes a unit whose output changes the routing decision. Implement with explicit `goto` back to supervisor plus a hard iteration cap.

### 3.2 Audit fan-out

**Definition.** Every agent writes directly to a shared persistence layer. N agents × K events = N×K concurrent inserts against one row-locked table. Item 2's race: 3 evals × 6 evaluators × ~4 events = ~72 rows fanning into one Postgres write surface. Lock contention rises, rows are lost, duplicates appear on retry.

**Standard defence — single-audit-writer pattern.** Two shapes:

1. *Dedicated audit-writer node.* Workers append to `pending_audit_events` on shared state (list-concat reducer). A dedicated node drains the list in one batch insert per transition.
2. *Single utility.* All writes go through one function from one node; workers return events as output state; supervisor's next turn flushes the batch.

Pick one shape per system; mixing both creates ambiguity.

### 3.3 Lost context across the supervisor boundary

**Definition.** The supervisor passes a stripped-down state to a worker; the worker can't recover needed framing; results come back inconsistent. MAST calls this *context collapse* — 36.9% of failures.

**Symptoms:** two workers given the same task return different-shaped outputs; a worker references something the supervisor didn't pass; same input produces different results on rerun.

**Standard defence.** A *documented worker-input contract*: typed schema (Pydantic / TypedDict) naming every field + a correlation ID threaded through every node, tool call, and audit row. With one ID per logical task, you can reconstruct the full flow even after partial context losses.

### 3.4 What these are not

These require boundaries — they don't exist at single-agent scope. The defences change the *topology's invocation rhythm* or *data contract*, not the agent's prompt.

> [!TIP]
> **AutoGen and CrewAI** default to conversational loops where every inter-agent message is a handoff. In fed-acq work, every handoff is a candidate audit-log row and HITL gate candidate — named slugs `autogen-unbounded-chatter` and `crewai-no-handoff-gate`.

## 4. Generic Implementation

Single-audit-writer + correlation-ID skeleton (e-commerce checkout pipeline, not federal-acquisitions):

```python
import uuid
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command
import operator

class CheckoutState(TypedDict):
    order: dict
    correlation_id: str
    pending_audit_events: Annotated[list[dict], operator.add]  # reducer!
    findings: Annotated[list[dict], operator.add]
    done: bool

def entry_node(state: CheckoutState) -> Command:
    cid = str(uuid.uuid4())
    return Command(update={
        "correlation_id": cid,
        "pending_audit_events": [{"event": "FLOW_STARTED", "correlation_id": cid}],
    })

def worker_enrich(state: CheckoutState) -> Command:
    result = enrich(state["order"]["shipping_address"])
    return Command(update={
        "findings": [{"step": "address", "result": result}],
        "pending_audit_events": [{
            "event": "ADDRESS_ENRICHED",
            "correlation_id": state["correlation_id"],
            "before": state["order"]["shipping_address"],
            "after": result,
        }],
    })

def audit_writer(state: CheckoutState) -> Command:
    if state["pending_audit_events"]:
        batch_insert_audit_logs(state["pending_audit_events"])
    return Command(update={"pending_audit_events": []})

def supervisor(state: CheckoutState) -> Command:
    # Per-decision-boundary only — not after every state mutation
    decision = call_router_llm(state["findings"], state["order"])
    if decision == "done":
        return Command(update={"done": True}, goto="audit_writer")
    return Command(goto=decision)
```

`correlation_id` is minted once and threaded through every event. `pending_audit_events` uses `operator.add` so parallel workers append safely. `audit_writer` is the sole DB caller. `supervisor` is on the path only between completed work units.

> [!NOTE]
> **Items 2 + 6 are live today.** Item 2 (audit-log race) and Item 6 (missing correlation IDs) are not hypothetical — they surface in the morning's 18-parallel evaluator dispatch. The single-audit-writer and correlation ID patterns are the direct fixes.

## 5. Real-world Patterns

**Fintech — loan-decisioning audit collisions.** A neobank's four-agent loan system hit lock contention under 10× load: decisions applied with no audit row, triggering a regulatory inquiry. Remediation: single-audit-writer batching all four agents' events.

**E-commerce — chatty catalog enrichment.** Supervisor re-invoked routing LLM after every step. Wall-clock: 24s per product → 8s after moving to per-decision-boundary invocation; no quality change.

**Healthcare — lost context in clinical handoffs.** Three workers each received a "patient packet." Labs worker assumed clinical context; history worker assumed administrative. Typed `PatientPacket` Pydantic model with explicit fields fixed inconsistency.

## 6. Best Practices

- **Detect chatty handoffs by counting supervisor invocations per logical task.** Over 3× the worker count is suspect; over 10× is pathological.
- **Adopt single-audit-writer from the first multi-agent deployment.** Retrofitting later requires touching every worker.
- **Thread a correlation ID through every state object, tool-call metadata, and audit row.** Cheap to add; impossible to retrofit cleanly.
- **Use typed schemas for worker input contracts.** Catches lost-context bugs at definition time.

> [!WARNING]
> **Anti-patterns: `autogen-unbounded-chatter` + `crewai-no-handoff-gate`.** Both frameworks default to unbounded conversational loops. In fed-acq work every handoff is a candidate audit-log row and HITL gate — an unbounded loop means unbounded un-audited decisions. Wire a handoff gate at every audit-relevant supervisor → worker boundary.

## 7. Hands-on Exercise

Below is a broken three-agent travel-booking sketch. Identify which anti-patterns it exhibits, name the symptom you'd expect in production, and describe the rewrite.

```python
def supervisor(state):
    decision = call_router_llm(state)
    log_audit_event("supervisor_decided", state, decision)   # supervisor writes
    return Command(goto=decision)

def worker_flight(state):
    flights = search_flights(state["request"])
    log_audit_event("flight_searched", state, flights)       # worker writes
    return Command(update={"flights": flights}, goto="supervisor")

def worker_hotel(state):
    hotels = search_hotels(state["request"])
    log_audit_event("hotel_searched", state, hotels)         # worker writes
    return Command(update={"hotels": hotels}, goto="supervisor")
```

> [!NOTE]
> **Self-check** (30s — answer mentally before expanding)
>
> 1. `log_audit_event` is called directly in each node. Which anti-pattern is this, and what's the failure mode under 500 concurrent searches?
> 2. Workers always return `goto="supervisor"` after every step. Which anti-pattern is this?

<details>
<summary>Show answers</summary>

1. Audit fan-out — four nodes writing directly to the audit table concurrently. Under 500 concurrent searches: lock contention on `audit_logs`, missing rows, duplicates on retry. Fix: add `pending_audit_events: Annotated[list[dict], operator.add]` to state; workers append to it; a dedicated `audit_writer` node drains it once per transition.
2. Chatty handoffs — the supervisor fires between every worker step, not at decision boundaries. Under N workers the supervisor invokes N+1 times per request. Fix: fan out workers in parallel (or sequence them directly to a synthesizer) and let the supervisor re-plan only after all workers for one user request complete.

</details>

> [!IMPORTANT]
> **Typed schemas are the cheapest prevention.** A `TypedDict` or Pydantic model per worker input costs minutes to add and prevents the lost-context failure class entirely at definition time — versus days of debugging after a production incident.

## 8. Key Takeaways

- Three multi-agent anti-patterns: chatty handoffs (supervisor invocation rhythm), audit fan-out (single-writer not adopted), lost context (missing worker-input contract).
- Single-audit-writer separates event emission from persistence — same primitive as CQRS / event sourcing.
- Correlation ID is the load-bearing thread that makes post-hoc diagnosis tractable.
- These are topology-level defences, not prompt-level — don't conflate with single-agent ReAct fixes.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://www.augmentcode.com/guides/why-multi-agent-llm-systems-fail-and-how-to-fix-them — retrieved 2026-05-26 — hot-tech
- https://towardsdatascience.com/the-multi-agent-trap/ — retrieved 2026-05-26 — hot-tech
- https://xtrace.ai/blog/ai-agent-context-handoff — retrieved 2026-05-26 — hot-tech
- https://achan2013.medium.com/ai-agent-anti-patterns-part-1-architectural-pitfalls-that-break-enterprise-agents-before-they-32d211dded43 — retrieved 2026-05-26 — hot-tech
- https://microsoft.github.io/code-with-engineering-playbook/observability/correlation-id/ — retrieved 2026-05-26 — foundation-stable

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

**MAST taxonomy in depth.** The 2026 multi-agent failure taxonomy classifies failures into three clusters: context collapse (36.9%), inter-agent misalignment (includes chatty handoffs and audit fan-out), and capability gaps. The context-collapse cluster is the largest because it manifests silently — the system produces outputs but they are inconsistent or wrong, with no obvious error signal. The intervention is the typed contract + correlation ID pattern described in §3.3.

**Audit as a domain object.** Treating the audit log as a domain object (schema-versioned, tested, the subject of an ADR) rather than a logging concern pays off at the W5 OTel anchor week. The `audit_correlation_id` threaded through `EvaluationState` today becomes the W5C `traceparent` trace ID propagated through OpenTelemetry — same primitive, two levels of the stack. Senior FDEs should anticipate this interlock when designing today's audit schema.

**AutoGen vs CrewAI vs LangGraph handoff discipline.** AutoGen's `GroupChat` and CrewAI's `Crew` both produce human-readable agent-to-agent dialogue, which is appealing for demos. Production deployments at scale consistently report the unbounded-chatter failure: agents negotiate over multiple turns before each action, making traces unreadable and costs unpredictable. LangGraph's explicit `goto` + typed state schema enforces per-decision-boundary discipline by construction. When evaluating framework choices in SA-2, weight handoff explicitness as a first-class criterion.

</details>

Last verified: 2026-06-06
