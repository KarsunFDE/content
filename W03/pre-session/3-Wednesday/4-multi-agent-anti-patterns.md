---
week: W03
day: Wed
topic_slug: multi-agent-anti-patterns
topic_title: "Multi-agent anti-patterns — chatty handoffs, audit fan-out, lost context"
parent_overview: W03/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 12
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
last_verified: 2026-05-26
---

# Multi-agent anti-patterns — chatty handoffs, audit fan-out, lost context

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name three multi-agent-specific anti-patterns (chatty handoffs, audit fan-out, lost context across the supervisor boundary) and recognize their symptoms in a trace.
- Diagnose chatty handoffs as a latency-and-cost amplifier even when individual agent calls look reasonable.
- Apply the *single-audit-writer* pattern to break the audit-fan-out failure mode without losing observability.
- Use a *threaded correlation ID* to make the lost-context failure mode visible and reconstructable.
- Distinguish multi-agent anti-patterns from single-agent ReAct anti-patterns (they are not the same — the new ones do not exist at single-agent scope).

## 2. Introduction

Multi-agent systems introduce failure modes that simply do not exist in single-agent designs. A solo agent can be slow, can hallucinate, can fail to call a tool — but it cannot have a *chatty handoff* with itself, cannot fan out audit writes against its own database, and cannot lose its own context across a supervisor boundary that does not exist. These three anti-patterns appear only when work crosses agent boundaries, and they appear consistently enough that the 2026 multi-agent-failure taxonomy (MAST) lists "context collapse" and "inter-agent misalignment" as 36.9% of all observed failures in production agent systems [1].

The problem is that none of the three look like bugs at the moment they are written. Chatty handoffs look like *thorough* supervisors checking on workers. Audit fan-out looks like *good* observability — every agent dutifully logs its own actions. Lost context looks like *clean* delegation — the supervisor passes only what the worker needs. Each is a defensible choice in isolation; each becomes pathological at scale.

The work this reading does is to give each anti-pattern a name, a symptom set, and a standard defense, so that when you spot one in a trace you can route around it with a known fix rather than rediscovering the solution. The defenses themselves are not exotic — they are distributed-systems patterns (single-writer, correlation IDs, contract enforcement) applied to agent topologies.

## 3. Core Concepts

### 3.1 Chatty handoffs

**Definition.** The supervisor re-invokes its routing LLM between every worker step — sometimes between every *thought* — "to check progress" or "to see if anything has changed." A nominally five-call flow balloons into thirty-plus LLM calls with no observability win [1][2].

**Symptoms in a trace:**

- The supervisor's tool-call count is a small-integer multiple of the worker count (5×, 10×, 20×).
- Most supervisor invocations conclude with the same decision they made on the prior turn.
- Total latency is dominated by supervisor calls, not worker calls.
- LangSmith / OpenTelemetry traces look like dense thickets of short LLM calls interleaved with each worker step.

**Why it happens.** The supervisor's prompt usually says something like "after each worker step, review the state and decide what to do next." This sounds prudent but means the supervisor's policy is invoked once per *step* instead of once per *decision boundary*. A decision boundary is "all the data needed to choose the next worker is now available." A step is "any change to state." The two are not the same.

**Standard defense.** Move from per-step re-invocation to per-decision-boundary re-invocation. The supervisor only fires after a worker (or batch of workers) completes a unit of work whose output changes the routing decision. Implement by: (a) explicit `goto` from workers back to the supervisor instead of after every state mutation; (b) supervisor's prompt explicitly says "only re-plan when X, Y, or Z conditions are met"; (c) a per-run iteration cap as a hard backstop [3].

### 3.2 Audit fan-out

**Definition.** Every agent writes its own audit log row to a shared persistence layer. With *N* agents producing *K* events each over a single user request, you generate *N × K* writes against one row-locked table — concurrent inserts collide, lock contention rises, individual rows are lost, and the audit log becomes unreliable [4][5].

**Symptoms in operation:**

- Missing rows under load — the LangSmith trace shows an action that has no corresponding `audit_logs` row.
- Postgres `pg_stat_activity` shows wait events on `audit_logs` during peak agent runs.
- Duplicate rows when retries kick in (because the original write didn't return cleanly and the agent retried).
- Audit-log query latency rises in proportion to agent fan-out width.

**Why it happens.** "Every agent should write its own audit event" sounds correct — it satisfies the principle that the actor logs its action. But "actor logs its action" does not have to mean "actor *writes* its action." The actor can *emit* the action, and a single writer can persist it. This separation is the same primitive that CQRS, event sourcing, and the write-audit-publish pattern have used for decades: the producer of events is not the same component as the persister of events [5].

**Standard defense — the single-audit-writer pattern.** Two viable shapes:

1. *Dedicated audit-writer node.* Each worker appends to a `pending_audit_events` list on shared state (with a list-concat reducer, so parallel writes are safe). A dedicated `audit_writer` node drains this list on every supervisor → worker transition and writes one batch insert per drain. Reduces *N × K* writes to ~*K* batches per transition [4].
2. *Shared utility in a single place.* All audit writes go through one function called from one node (typically the supervisor). Workers return their events as part of their output state; the supervisor's next invocation flushes the batch. Simpler than (1), works well when the supervisor is already in the critical path.

Pick one shape per system. Mixing both leads to ambiguity about which path actually persists which event.

### 3.3 Lost context across the supervisor boundary

**Definition.** The supervisor passes a stripped-down state to a worker; the worker can't recover the framing it needs; results come back inconsistent because each worker reconstructs the missing framing differently [3]. The MAST taxonomy calls this *context collapse*; 36.9% of multi-agent failures fall into this class [1].

**Symptoms:**

- Two workers given the same task return different-shaped outputs.
- A worker's response references something the supervisor "should know" but didn't pass.
- The same input produces different results on rerun because the worker is filling the gap differently each time.
- Workers occasionally produce confident, well-formed answers that are simply wrong because they hallucinated the framing.

**Why it happens.** "Pass only what the worker needs" is good advice for prompt engineering. But what the worker *needs* is often more than what the supervisor *thinks* it needs. The supervisor compresses state implicitly (via its prompt's mental model of the worker's task); the worker decompresses it implicitly (via the worker prompt's mental model). The two models drift apart.

**Standard defense.** A *documented worker-input contract* per worker type. Concretely:

- A typed schema (Pydantic model, TypedDict, JSON Schema) declaring exactly what the worker receives.
- A docstring on the worker function naming every field and its purpose.
- A unit test for each worker that asserts the supervisor's emission matches the schema.
- A *correlation ID* threaded through the input so the worker's outputs trace back to the originating user request [6].

The correlation ID is the load-bearing piece. With one ID per logical task threaded through every node's state, every tool call's metadata, and every audit-log row, you can reconstruct the entire flow after the fact even when individual context losses occur [6].

### 3.4 What these are not — distinguishing from single-agent anti-patterns

Single-agent ReAct anti-patterns (the kind that surfaced on Tue: tool-call loops, premature termination, prompt drift between turns) are about *one* agent's internal discipline. The Wed anti-patterns are *new concepts that do not exist at single-agent scope*:

- Chatty handoffs requires two agents (supervisor + worker).
- Audit fan-out requires multiple actors writing to the same audit surface.
- Lost context across the supervisor boundary requires a boundary to lose context across.

This matters because the defenses are different. A ReAct agent's tool-loop is fixed by tuning the agent's prompt or adding a tool-call counter. Chatty handoffs are fixed by changing the *topology's invocation rhythm*. Mixing the two diagnoses leads to fixes that don't work.

## 4. Generic Implementation

A generic single-audit-writer + correlation-ID skeleton, in a non-federal domain (think e-commerce checkout pipeline with multiple agents enriching the order):

```python
import uuid
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command
import operator

class CheckoutState(TypedDict):
    order: dict
    correlation_id: str                                # one ID per request
    pending_audit_events: Annotated[list[dict], operator.add]  # reducer!
    findings: Annotated[list[dict], operator.add]
    done: bool

def entry_node(state: CheckoutState) -> Command:
    # Mint the correlation ID once at the start of the flow
    cid = str(uuid.uuid4())
    return Command(update={
        "correlation_id": cid,
        "pending_audit_events": [{
            "event": "FLOW_STARTED",
            "actor": "entry",
            "correlation_id": cid,
        }],
    })

def worker_enrich_address(state: CheckoutState) -> Command:
    # Worker contract: reads order, emits findings + audit event.
    # Note: worker does NOT write to audit_logs directly.
    result = enrich(state["order"]["shipping_address"])
    return Command(update={
        "findings": [{"step": "address", "result": result}],
        "pending_audit_events": [{
            "event": "ADDRESS_ENRICHED",
            "actor": "worker_enrich_address",
            "correlation_id": state["correlation_id"],
            "before": state["order"]["shipping_address"],
            "after": result,
        }],
    })

def audit_writer(state: CheckoutState) -> Command:
    # Single writer — drains pending events in one batch insert.
    if not state["pending_audit_events"]:
        return Command()
    batch_insert_audit_logs(state["pending_audit_events"])
    return Command(update={"pending_audit_events": []})

def supervisor(state: CheckoutState) -> Command:
    # Per-decision-boundary invocation: only re-plan when work has
    # produced new findings to act on, not after every state mutation.
    decision = call_router_llm(state["findings"], state["order"])
    if decision == "done":
        return Command(update={"done": True}, goto="audit_writer")
    return Command(goto=decision)

graph = StateGraph(CheckoutState)
graph.add_node("entry", entry_node)
graph.add_node("supervisor", supervisor)
graph.add_node("worker_enrich_address", worker_enrich_address)
graph.add_node("audit_writer", audit_writer)

graph.add_edge(START, "entry")
graph.add_edge("entry", "supervisor")
graph.add_edge("worker_enrich_address", "audit_writer")  # drain after worker
graph.add_edge("audit_writer", "supervisor")             # then re-route
graph.compile()
```

What each piece does:

- **`correlation_id`** is minted once at entry and threaded through every audit event — the lost-context defense.
- **`pending_audit_events`** uses `operator.add` so parallel workers can safely append; the `audit_writer` is the sole component that ever calls the database — the audit-fan-out defense.
- **`supervisor`** is on the path only between completed work units (after `audit_writer` drains), not after every state mutation — the chatty-handoff defense.
- **Worker contract** is visible: `worker_enrich_address` declares exactly what it reads and writes, and emits a structured audit event the writer can persist atomically.

## 5. Real-world Patterns

**Fintech — multi-agent loan-decisioning collisions (case study from a 2026 production retro).** A neobank's loan-decisioning system used four agents writing to one `decision_audit_log` Postgres table. Under a marketing campaign that drove 10× normal application volume, audit-log writes started failing under lock contention; some decisions were applied to customers' accounts with no corresponding audit row, triggering a regulatory inquiry. The remediation was a single-audit-writer service that batched events from all four agents — the same pattern as event sourcing, applied to an agent graph [1].

**E-commerce — chatty supervisor in catalog enrichment.** A mid-sized marketplace deployed a catalog-enrichment pipeline where the supervisor re-invoked its routing LLM after every worker step. Reported wall-clock latency was 24s per product enrichment; profiling showed 19s of that was supervisor calls. After moving to per-decision-boundary invocation (supervisor only fires after all enrichment workers for one product complete), latency dropped to 8s with no change in output quality [2].

**Healthcare — lost context in clinical handoffs.** A patient-summary system had three workers (labs, imaging, history) each receiving a "patient packet" from the supervisor. The labs worker assumed `patient_packet.context` referred to clinical context; the history worker assumed it referred to administrative context. Results were inconsistent across runs. The fix was a typed `PatientPacket` Pydantic model with explicit fields (`clinical_context`, `admin_context`, `correlation_id`) and a contract test that the supervisor's emission must validate [3].

**Logistics — correlation IDs as the load-bearing thread.** A last-mile-delivery operator routes packages through ~12 agents (routing, capacity, traffic, weather, customer-window, …). Each agent emits structured events to a central event store with the same `delivery_id` correlation ID threaded through every event. When a customer complains about a missed window three weeks later, ops can reconstruct the full decision chain in 90 seconds — the correlation ID is what makes this tractable [6].

## 6. Best Practices

- **Detect chatty handoffs by counting supervisor invocations per logical task. Anything over 3× the worker count is suspect; over 10× is pathological.**
- **Adopt a single-audit-writer from the first multi-agent deployment.** Retrofitting later is harder because every worker has assumed it owns its writes.
- **Thread a correlation ID through every state object, every tool-call metadata, every audit row, every external service call.** It is cheap to add and impossible to retrofit cleanly.
- **Use typed schemas (Pydantic / TypedDict) for every worker input contract.** Catches lost-context bugs at definition time instead of in production.
- **Profile your supervisor's invocation rhythm separately from its routing accuracy.** A 90%-accurate supervisor that fires 10× too often is worse than an 80%-accurate one that fires at the right rhythm.
- **Treat the audit log as a domain object, not a logging concern.** Schema it, version it, test it; do not let it accrete via `print` statements that someone later wires to a database.
- **Resist the urge to "let every agent log its own actions." That is the audit-fan-out anti-pattern.** Emit-and-persist is the right separation.

## 7. Hands-on Exercise

**Code-reading + diagnosis prompt (15 min).** Below is a (deliberately broken) sketch of a three-agent travel-booking flow. Identify which of the three anti-patterns it exhibits, name the symptom you'd expect to see in production, and rewrite the relevant section to fix it.

```python
# Anti-pattern sketch — DO NOT use this in production
def supervisor(state):
    decision = call_router_llm(state)
    log_audit_event("supervisor_decided", state, decision)  # supervisor writes
    return Command(goto=decision)

def worker_flight(state):
    flights = search_flights(state["request"])
    log_audit_event("flight_searched", state, flights)  # worker writes
    return Command(update={"flights": flights}, goto="supervisor")  # back to supervisor

def worker_hotel(state):
    hotels = search_hotels(state["request"])
    log_audit_event("hotel_searched", state, hotels)  # worker writes
    return Command(update={"hotels": hotels}, goto="supervisor")  # back to supervisor

def worker_car(state):
    cars = search_cars(state["request"])
    log_audit_event("car_searched", state, cars)  # worker writes
    return Command(update={"cars": cars}, goto="supervisor")  # back to supervisor
```

**What good looks like:** You identify (a) **audit fan-out** — four nodes all writing audit rows; under 500 concurrent travel searches this will contend on the audit table. You identify (b) **chatty handoffs** — the supervisor is on the critical path between every worker; with N workers the supervisor fires N+1 times for one user request. Your rewrite introduces a `pending_audit_events` state key with a list-concat reducer; workers append to it; an `audit_writer` node drains it once before the synthesizer; and workers `goto` the synthesizer directly (or fan out in parallel) rather than ping-ponging through the supervisor. Bonus: you add a `correlation_id` field threaded through every emitted event.

## 8. Key Takeaways

- *What are the three named multi-agent anti-patterns and how do you spot each in a trace?* (Chatty handoffs → supervisor invocation count >> worker count; audit fan-out → missing/duplicate audit rows under load; lost context → inconsistent worker outputs across runs.)
- *Why is the single-audit-writer pattern the standard defense against audit fan-out?* (Separates event emission from event persistence — the same primitive CQRS, event sourcing, and write-audit-publish use; reduces N×K writes to ~K batches.)
- *What does threading a correlation ID through state buy you?* (Reconstructable traces even when context partially collapses; the load-bearing thread that makes lost-context bugs diagnosable post-hoc.)
- *Why are these different from single-agent ReAct anti-patterns?* (They are emergent from the topology — they require boundaries to exist; the defenses operate on topology shape, not on agent prompts.)
- *What is the cost of pre-emptively defending against these anti-patterns from day one?* (Low — typed contracts, a list-reducer on a pending-events key, and a correlation-ID minter — versus retrofitting after a production incident, which is high.)

## Sources

1. [Multi-Agent AI Systems: Why They Fail and How to Fix Coordination Issues 2026 (Augment Code)](https://www.augmentcode.com/guides/why-multi-agent-llm-systems-fail-and-how-to-fix-them) — retrieved 2026-05-26
2. [The Multi-Agent Trap (Towards Data Science)](https://towardsdatascience.com/the-multi-agent-trap/) — retrieved 2026-05-26
3. [AI Agent Handoff: Why Context Breaks & How to Fix It (XTrace)](https://xtrace.ai/blog/ai-agent-context-handoff) — retrieved 2026-05-26
4. [AI Agent Anti-Patterns Part 1: Architectural Pitfalls (Medium / Allen Chan)](https://achan2013.medium.com/ai-agent-anti-patterns-part-1-architectural-pitfalls-that-break-enterprise-agents-before-they-32d211dded43) — retrieved 2026-05-26
5. [Correlation IDs — Engineering Fundamentals Playbook (Microsoft)](https://microsoft.github.io/code-with-engineering-playbook/observability/correlation-id/) — retrieved 2026-05-26

Last verified: 2026-05-26
