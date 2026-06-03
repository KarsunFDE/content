---
week: W02
day: Thu
topic_slug: hitl-2-rag-fallback
topic_title: "HITL #2 — RAG fallback as a programme-thread touchpoint"
parent_overview: W02/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://docs.langchain.com/oss/python/langgraph/add-human-in-the-loop
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.langchain.com/blog/making-it-easier-to-build-human-in-the-loop-agents-with-interrupt
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://docs.langchain.com/oss/python/langgraph/interrupts
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.abstractalgorithms.dev/langgraph-human-in-the-loop
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://growwstacks.com/blog/human-in-the-loop-ai-agents-langgraph
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-06-03
---

# HITL #2 — RAG fallback as a programme-thread touchpoint

> [!NOTE]
> **From topic 2:** three failure modes (retrieval miss / faithfulness drop / wrong-chunk) feed one queue. This topic is the envelope shape that does the routing — same HTTP status, different body shape, four reviewer actions, no fifth path.

## 1. Learning Objectives

- Define HITL fallback as *conditional* escalation against a runtime threshold, not blanket review.
- Sketch the same-status-different-body envelope contract on `/answer-qa`.
- Name the four reviewer actions and the audit event each emits.
- Compare HTTP-envelope HITL (today) vs LangGraph `interrupt()` HITL (W3 Thu) and when to use each.

## 2. Introduction

A common misreading of "human-in-the-loop" is that humans review *every* model output. That works in a demo and the first ten production requests per day; it does not scale. The version that scales is **conditional HITL** — humans review only what the platform itself flags as low-confidence — and the engineering challenge is the gate that decides what counts as low-confidence. For grounded RAG that gate is the two-axis runtime evaluation from topic 2. When either axis falls below threshold the platform does not ship; it routes a structured draft to a review queue. This is load-bearing for regulated systems: "we ship answers when our own evaluation says they're trustworthy, drafts to humans otherwise" is a defensible posture; "we ship every model output" is not.

## 3. Core Concepts

### 3.1 The envelope contract — same status, different body

```jsonc
// Auto-publish: both axes ≥ 0.85
{ "status": "published",
  "response_text": "...",
  "retrieved_chunks": [...],
  "faithfulness_score": 0.91,
  "relevance_score": 0.88,
  "audit_event_id": "..." }

// Escalation: either axis < 0.85
{ "status": "draft_for_review",
  "response_text": "...",
  "retrieved_chunks": [...],
  "faithfulness_score": 0.78,
  "relevance_score": 0.62,
  "failure_mode": "low_relevance",
  "review_queue_id": "..." }
```

Both return HTTP 200 — the request succeeded. What changes is semantic success. The `status` field makes the gate decision legible at the API boundary so calling clients (chatbot UI vs workflow engine) route accordingly. Implicit signals (HTTP code games, response-length heuristics) are an anti-pattern — testability dies.

### 3.2 Failure-mode → response-shape mapping

| Failure mode | `status` | `failure_mode` | Downstream |
|---|---|---|---|
| Both axes clear | `published` | (null) | UI renders; audit row `qa_published` |
| Faithfulness low, relevance ok | `draft_for_review` | `low_faithfulness` | CO queue (generation bug) |
| Faithfulness ok, relevance low | `draft_for_review` | `low_relevance` | CO queue (**retrieval bug** — yesterday's shape) |
| Both axes low | `draft_for_review` | `double_failure` | CO queue (escalate priority) |

### 3.3 Four reviewer actions, no fifth

| Action | What ships | Audit event |
|---|---|---|
| **Approve** | Draft as-written | `qa_co_approved` |
| **Edit-and-publish** | Reviewer-modified text | `qa_co_edited` (diff preserved) |
| **Reject-with-redraft** | Nothing; re-prompt with feedback | `qa_co_rejected` + new cycle |
| **Reject-hard** | Nothing; request closed unanswered | `qa_co_rejected` (terminal) |

SLA breach is itself an audited event (`qa_co_sla_breach`) — drafts cannot age out silently. The four-action ceiling is deliberate; a fifth path dilutes audit-trail clarity.

> [!IMPORTANT]
> **CO latency budget.** 4-hour business-hours SLA on the CO queue. Tune the conjunction threshold so queue volume stays under ~30 items/day per CO. Above that, reviewers swamp and degrade to rubber-stamping within weeks — the safety property is silently gone.

## 4. Generic Implementation

```python
# Generic HITL #2 envelope. Same HTTP status, different body shape.
# Lives in acquire-gov at services/ai-orchestrator/routes/answer_qa.py
from enum import Enum
from pydantic import BaseModel
import asyncio

class Status(str, Enum):
    PUBLISHED = "published"
    DRAFT_FOR_REVIEW = "draft_for_review"

class GatedAnswer(BaseModel):
    status: Status
    response_text: str
    retrieved_chunks: list[dict]
    faithfulness_score: float
    relevance_score: float
    failure_mode: str | None = None
    audit_event_id: str | None = None
    review_queue_id: str | None = None

async def answer(query: str, tenant_id: str) -> GatedAnswer:
    chunks   = await retrieve(query, tenant_id=tenant_id, top_k=5)
    response = await generate(query, chunks)
    f, r     = await asyncio.gather(score_faithfulness(response, chunks),
                                    score_relevance(query, chunks))
    if f >= 0.85 and r >= 0.85:
        evt = await audit.emit("qa_published", query, response, chunks, f, r)
        return GatedAnswer(status=Status.PUBLISHED, response_text=response,
                           retrieved_chunks=chunks, faithfulness_score=f,
                           relevance_score=r, audit_event_id=evt)
    failure = ("low_faithfulness" if f < 0.85 and r >= 0.85
               else "low_relevance" if r < 0.85 and f >= 0.85
               else "double_failure")
    qid = await review_queue.enqueue(query, response, chunks, f, r, failure)
    return GatedAnswer(status=Status.DRAFT_FOR_REVIEW, response_text=response,
                       retrieved_chunks=chunks, faithfulness_score=f,
                       relevance_score=r, failure_mode=failure,
                       review_queue_id=qid)
```

Retrieve, generate, score (parallel), branch. Both branches return the same shape with different field populations. Plain Python composition — no `Chain` subclass, no LCEL `|` pipe, no `chain.run()` (D-033).

## 5. Real-world Patterns

**Healthcare — diagnostic-imaging triage.** A radiology-AI vendor's model emits preliminary findings; findings below a tuned confidence threshold route to a radiologist before the report goes to the ordering clinician. The envelope pattern and four reviewer actions map directly. Reviewer fatigue is managed by tuning the threshold against inter-rater agreement telemetry — when human radiologists disagree above 12% on a slice, the model's threshold for that slice tightens.

**Fintech — fraud-alert explanation.** A bank's fraud platform uses an LLM to draft customer-facing hold explanations. The runtime gate scores drafts against compliance-language requirements + tone rules; failing drafts route to a senior analyst with the failure mode tagged (`compliance_phrasing` vs `tone` vs `factual`). The audit trail traces a customer dispute back through draft → analyst action → final message via correlation ID.

**Legal-tech — contract-clause Q&A.** A 2026 vendor write-up noted queue volume — not retrieval quality — was their binding constraint. They added a middle confidence band (0.70–0.85) where drafts auto-publish but sample 10% for periodic reviewer audit, with telemetry feeding back into threshold tuning. Two-tier escalation is a real production pattern once raw volume saturates the queue.

## 6. Best Practices

- **Make the gate decision explicit at the API boundary** via a `status` field — keeps downstream routing legible and testable.
- **Tune threshold against domain cost-asymmetry, not defaults.** Healthcare/legal/federal-acq tune high; high-volume support tunes lower with sampling.
- **Preserve `failure_mode` through to review UI and audit log.** Reviewers benefit from knowing retrieval-bug vs generation-bug; telemetry slices on the same field.
- **Enforce SLA with a logged escalation on breach.** Silent expiry destroys the audit guarantee.
- **Resist a fifth reviewer action.** Approve/edit/reject-with-redraft/reject-hard covers it; additions dilute audit clarity.

> [!WARNING]
> **Anti-pattern: HITL-blanket-review.** Most internet HITL framings present "approve every LLM output" as the safety story. Operationally infeasible at 200 Q&As/day — reviewers degrade to rubber-stamping within weeks and the safety property is silently gone. HITL #2 is **conditional** escalation against thresholds, not blanket approval. Per `known-bad-patterns.yml` `hitl-blanket-review`. Push back on any design proposing "HITL every response."

## 7. Hands-on Exercise

Wire `POST /answer-qa` in your pair-project repo with the envelope shape from §3.1 + the four-action review queue from §3.3. Use yesterday's failing FAR-47 query as the regression fixture — it must route to `draft_for_review` with `failure_mode: "low_relevance"`. Verify the four reviewer-action audit events fire correctly when you simulate each action. The 4-hour SLA timer + `qa_co_sla_breach` escalation is the stretch goal for the war-room.

> [!NOTE]
> **Self-check** (30s)
>
> 1. Why is the `status` field on the envelope load-bearing rather than just using HTTP status codes?
> 2. What's the audit event when a CO edits a draft before publishing, and what payload does it carry?

<details>
<summary>Show answers</summary>

1. HTTP 200 means "request succeeded." Both auto-publish and review-queue are successful requests — the semantic difference is downstream routing. A single `status` field makes that decision legible, testable, and stable across UI / workflow / telemetry consumers. Using HTTP codes (e.g., 202 for queue) overloads the network-layer semantics with application-layer routing — testability and client compatibility both suffer.
2. `qa_co_edited` — payload includes the reviewer's identifier, the original draft, the edited final text, and the diff between them. The diff is preserved so the chain (model draft → reviewer edit → final publish) is reconstructible from the audit log alone, which is the property OIG investigations exercise.

</details>

## 8. Key Takeaways

- HITL fallback is **conditional escalation** — default auto-publish; humans handle only flagged drafts.
- The envelope contract is same-status, different-body; one `status` field carries the routing decision.
- Four reviewer actions, each with a distinct audit event; no fifth path; SLA breach is its own logged event.
- Today's pattern (HTTP envelope) and next week's pattern (LangGraph `interrupt()`) are the same shape at different scopes — envelope pauses between requests, `interrupt()` pauses within graph execution.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- <https://docs.langchain.com/oss/python/langgraph/add-human-in-the-loop> — retrieved 2026-05-26 — hot-tech-3mo
- <https://www.langchain.com/blog/making-it-easier-to-build-human-in-the-loop-agents-with-interrupt> — retrieved 2026-05-26
- <https://docs.langchain.com/oss/python/langgraph/interrupts> — retrieved 2026-05-26
- <https://www.abstractalgorithms.dev/langgraph-human-in-the-loop> — retrieved 2026-05-26
- <https://growwstacks.com/blog/human-in-the-loop-ai-agents-langgraph> — retrieved 2026-05-26

</details>

<details>
<summary>Deeper dive — two-layer HITL pattern + the seven-touchpoint thread</summary>

**Two layers — envelope today, `interrupt()` next week:**

| Layer | When | State surface |
|---|---|---|
| HTTP envelope (today) | Final-answer ship decision | Draft + scores in review-queue DB |
| LangGraph `interrupt()` (W3 Thu, HITL #5) | Mid-graph tool-call decision | Full graph state in checkpointer; resume via `Command(resume=...)` |

Same conceptual pattern — pause with structured state. Envelope is the right tool when the decision is "should this answer ship." `interrupt()` is the right tool when the decision is "should this tool call execute" (DB write, external API). LangGraph persists graph state — variables, message history, intermediate tool calls — to a checkpointer; the reviewer responds via `Command(resume=...)` and execution continues from that node. State requirements drive the choice: small state (draft + scores) → envelope; large state (intermediate reasoning) → graph-interrupt.

**The seven-touchpoint thread, in context:**

| # | When | What |
|---|---|---|
| 1 | W1 Fri | LLM Essentials — reversibility, blast radius, audit demands |
| **2** | **W2 Thu (today)** | RAG fallback envelope on `/answer-qa` |
| 3 | W3 Mon | Plan Day ADR — HITL boundary committed in plan-spec |
| 4 | W3 Wed | HITL between multi-agent handoffs |
| 5 | W3 Thu | LangGraph `interrupt()` primitive |
| 6 | W4 Wed | OWASP LLM06 (Excessive Agency) re-assertion |
| 7 | W5 Wed | AIOps auto-remediation authority boundaries |

The thread accumulates: today's gate is the simplest expression of the pattern, W3 Thu is the framework-level version, W4 Wed re-asserts the design constraint as a security property, W5 Wed extends it to ops automation. Naming the connections at each step is what keeps the curriculum from feeling like seven unrelated topics.

**LangChain v1.0 hygiene (D-033):** Plain Python composition. `model_client.invoke(build_prompt(...))` + Pydantic validation. No `Chain` subclass. No LCEL `|` pipe. No `chain.run()`. Per `known-bad-patterns.yml` IDs `langchain-chain-class`, `langchain-lcel-pipe`, `langchain-chaining-verb`. v1.0 framing uses `create_agent` for agent-shaped problems and plain Python for sequential composition.

</details>

Last verified: 2026-06-03
