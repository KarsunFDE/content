---
week: W02
day: Thu
topic_slug: hitl-2-rag-fallback
topic_title: "HITL #2 — RAG fallback as a programme-thread touchpoint"
parent_overview: W02/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 15
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
last_verified: 2026-05-26
---

# HITL #2 — RAG fallback as a programme-thread touchpoint

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define **human-in-the-loop (HITL) fallback** in a RAG system as *conditional* escalation against a runtime threshold, not blanket review.
- Explain the response-envelope contract that makes the gate observable at the HTTP boundary (same status, different body shape).
- Articulate the cost-asymmetry argument for why HITL is not "approve every output."
- Describe the same pattern at two layers — HTTP-envelope (this week) and graph-`interrupt()` (LangGraph, next week) — and name when each is appropriate.
- Name the four downstream actions a reviewer takes on a queued draft and how each closes the loop in the audit trail.

## 2. Introduction

A common misreading of "human-in-the-loop" in AI systems is that humans review *every* model output. That works in a demo and the first ten production requests per day; it does not work at production throughput. The version that scales is **conditional HITL** — humans review only the outputs the platform itself flags as low-confidence — and the engineering challenge is the gate that decides what counts as low-confidence.

For grounded RAG, that gate is the runtime evaluation discussed in the companion topic on faithfulness and relevance. When the platform's self-assessment of an answer falls below threshold on either dimension, it does not ship the answer; it ships a structured draft to a review queue. The "fallback" name comes from this routing: the *default* path is automatic publish; the *fallback* path is human review.

This pattern is load-bearing for regulated systems. A confidently-wrong AI output that propagates to a regulator, auditor, or counterparty is more dangerous than no output at all. Conditional HITL is the technical mechanism that lets a team underwrite the system — "we ship answers when our own evaluation says they're trustworthy, and drafts to humans otherwise" is a defensible posture; "we ship every model output" is not.

The pattern shows up across the 6-week programme in **seven progressively-deeper touchpoints**. Today is touchpoint #2: the HTTP-envelope shape and the threshold gate. Next week (W3 Thu, touchpoint #5) the same pattern reappears as LangGraph's `interrupt()` primitive — a graph-execution pause with state persistence and resume-via-`Command`. Same shape, deeper implementation.

## 3. Core Concepts

### 3.1 Conditional escalation, not blanket approval

The defining property of HITL fallback is that the **default** is automatic. The platform proceeds without a human on the majority of requests; only requests that fail a quantitative gate route to review. The approval step is the *exception*, not the rule.

The misreading to pre-empt: "HITL = approve every LLM output." That posture buries reviewers in confident-correct drafts, generates fatigue, and degrades to rubber-stamping within weeks — at which point the safety property is silently gone.

### 3.2 The response-envelope contract

The gate is most cleanly expressed at the HTTP boundary as a **same-status, different-body** contract. Both auto-publish and review-queue cases return HTTP 200 — the request succeeded, the platform handled it. The difference is in the JSON body. Generically:

```jsonc
// Auto-publishable: gate passed
{
  "status": "published",
  "response_text": "...",
  "retrieved_chunks": [...],
  "faithfulness_score": 0.91,
  "relevance_score": 0.88,
  "audit_event_id": "..."
}

// Escalation: gate failed
{
  "status": "draft_for_review",
  "response_text": "...",
  "retrieved_chunks": [...],
  "faithfulness_score": 0.78,
  "relevance_score": 0.62,
  "failure_mode": "low_relevance",
  "review_queue_id": "..."
}
```

Why same-status, different-body: the calling client is always succeeding at the network layer. What changes is the semantic meaning of success. Some clients (a chatbot UI) render the published response directly; others (a workflow engine) check the `status` field and route. There is no third shape — every response either has `published` or `draft_for_review`. Anything else indicates a bug.

### 3.3 The four reviewer actions

A queued draft surfaces to a reviewer with the response text, the retrieved chunks, the two scores, and the failure mode. The reviewer has exactly four options, each of which closes the loop in the audit trail differently:

| Action | What ships | Audit event |
|---|---|---|
| **Approve** | The draft as-written | `qa_co_approved` |
| **Edit and publish** | A reviewer-modified version | `qa_co_edited` (diff preserved) |
| **Reject with redraft** | Nothing; the model is re-prompted with reviewer feedback | `qa_co_rejected` + new draft cycle |
| **Reject hard** | Nothing; the request is closed unanswered | `qa_co_rejected` (terminal) |

There is **no fifth path**. A draft cannot expire silently — if it ages out of SLA, that itself is a logged event (`qa_co_sla_breach`) that triggers escalation to a supervising reviewer. The audit trail is the load-bearing artifact for regulated environments; a six-month-old request needs to be traceable from "user query" through "platform-generated draft" through "reviewer action" with no gaps.

### 3.4 SLA shape

The review queue is a workflow primitive, not just a list. It has:

- **Per-item SLA**, set by the failure-mode classification and the request's downstream consumer (e.g., 1-hour for a customer-facing escalation, 4-hour for an internal audit-prep query, 24-hour for archive enrichment).
- **Reviewer routing** by domain — drafts touching one specialty go to that specialty's queue, not the global pool.
- **Escalation policy** — if a reviewer flags a draft as out-of-scope or contested, it moves to a senior reviewer's queue rather than bouncing.

### 3.5 The two-layer pattern: envelope and graph-interrupt

The HTTP-envelope shape is the simpler implementation: the platform makes its evaluation decision after the model has responded, and the route is a fork on the response body. State is small — a draft plus its scores — and lives in the review-queue database.

LangGraph's `interrupt()` primitive ([LangChain docs](https://docs.langchain.com/oss/python/langgraph/interrupts), retrieved 2026-05-26) is the same pattern at a different scope. Where the envelope pauses *between* requests, `interrupt()` pauses *within* an agent's graph execution at a node. Graph state — variables, message history, intermediate tool calls — is persisted to a checkpointer; the reviewer responds via `Command(resume=...)` and execution continues from that node.

The two coexist. The envelope is appropriate when the decision is "should this *final answer* ship." The graph-interrupt is appropriate when the decision is "should this *tool call* execute" — a database write, an external API call — because the state needed to resume includes the agent's intermediate reasoning, not just a final draft. Today's pattern is the envelope; next week's is the graph-interrupt.

### 3.6 The seven-touchpoint thread, in context

| # | When | What |
|---|---|---|
| 1 | W1 Fri | LLM Essentials — reversibility, blast radius, audit demands |
| **2** | **W2 Thu (today)** | **RAG fallback envelope on the answer endpoint** |
| 3 | W3 Mon | Plan Day ADR — HITL boundary committed in plan-spec |
| 4 | W3 Wed | HITL between multi-agent handoffs |
| 5 | W3 Thu | LangGraph `interrupt()` primitive (technical anchor) |
| 6 | W4 Wed | OWASP LLM06 (Excessive Agency) re-assertion |
| 7 | W5 Wed | AIOps auto-remediation authority boundaries |

The thread accumulates: today's gate is the simplest expression of the pattern, W3 Thu is the framework-level version, W4 Wed re-asserts the design constraint as a security property, W5 Wed extends it to ops automation. Naming the connections at each step is what keeps the curriculum from feeling like seven unrelated topics.

## 4. Generic Implementation

A generic FastAPI handler for the envelope shape, using a non-Karsun domain (a customer-support assistant for a SaaS product):

```python
# Generic HITL #2 envelope. Same HTTP status, different body shape.
# No Karsun-specific names — this is a SaaS support-ticket assistant.

from fastapi import FastAPI
from pydantic import BaseModel
from enum import Enum

app = FastAPI()

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

FAITHFULNESS_THRESHOLD = 0.85
RELEVANCE_THRESHOLD = 0.85

@app.post("/answer", response_model=GatedAnswer)
async def answer(query: str) -> GatedAnswer:
    # 1. Retrieve chunks from the doc corpus.
    chunks = await retrieve(query, top_k=5)

    # 2. Generate a grounded response with the primary model.
    response_text = await generate(query, chunks)

    # 3. Score both axes in parallel.
    f_score, r_score = await asyncio.gather(
        score_faithfulness(response_text, chunks),
        score_relevance(query, chunks),
    )

    # 4. Gate: both must clear thresholds to auto-publish.
    if f_score >= FAITHFULNESS_THRESHOLD and r_score >= RELEVANCE_THRESHOLD:
        audit_id = await audit.emit("answer_published", query, response_text,
                                    chunks, f_score, r_score)
        return GatedAnswer(
            status=Status.PUBLISHED,
            response_text=response_text,
            retrieved_chunks=chunks,
            faithfulness_score=f_score,
            relevance_score=r_score,
            audit_event_id=audit_id,
        )

    # 5. Otherwise, route to review queue and return draft.
    failure = (
        "low_faithfulness" if f_score < FAITHFULNESS_THRESHOLD and r_score >= RELEVANCE_THRESHOLD
        else "low_relevance" if r_score < RELEVANCE_THRESHOLD and f_score >= FAITHFULNESS_THRESHOLD
        else "double_failure"
    )
    review_id = await review_queue.enqueue(
        query=query, draft=response_text, chunks=chunks,
        f_score=f_score, r_score=r_score, failure_mode=failure,
    )
    return GatedAnswer(
        status=Status.DRAFT_FOR_REVIEW,
        response_text=response_text,
        retrieved_chunks=chunks,
        faithfulness_score=f_score,
        relevance_score=r_score,
        failure_mode=failure,
        review_queue_id=review_id,
    )
```

The structure: retrieve, generate, score (parallel), branch. The two branches return the same shape with different field populations. The calling client checks `status` and renders accordingly.

> [!instructor-review]
> **LangChain v1.0 hygiene.** This example uses plain Python composition with `await` and explicit function calls. There is no `Chain` class, no LCEL `|` pipe operator, no `chain.run()`. Per the known-bad-patterns blocklist (`langchain-chain-class`, `langchain-lcel-pipe`, `langchain-chaining-verb`, last reviewed 2026-05-12), sequential composition in LangChain v1.0+ is normal Python — no framework abstraction.

## 5. Real-world Patterns

**Healthcare — diagnostic-imaging triage.** A radiology-AI vendor processes scans through a model that outputs preliminary findings. Findings with a model-reported confidence below a tuned threshold route to a radiologist for review before the report goes to the ordering clinician; findings above publish directly with an "AI-preliminary" attribution and a separate human-confirmation step. The envelope pattern and the four reviewer actions map directly; reviewer fatigue is managed by tuning the threshold against inter-rater agreement telemetry.

**Fintech — fraud-alert explanation.** A bank's fraud-investigation platform uses an LLM to draft customer-facing explanations of transaction holds. The runtime gate evaluates drafts against compliance-language requirements and tone rules; failing drafts route to a senior analyst's queue with the failure mode tagged ("compliance phrasing" vs. "tone" vs. "factual"). The audit trail traces a customer dispute back through draft → analyst action → final message via a single correlation ID.

**E-commerce — product-recommendation rationale.** A marketplace generates natural-language rationales for "why are we showing you this?" disclosures (required by emerging consumer-protection regulation). The rationale generator runs through a HITL gate that catches drafts citing product attributes the product does not actually have. Failing drafts route to a content-ops queue with a 30-minute SLA; passing drafts ship in real time. ([Abstract Algorithms 2026](https://www.abstractalgorithms.dev/langgraph-human-in-the-loop))

**Legal-tech — contract-clause Q&A.** A contract-analytics vendor's published 2026 evaluation noted that queue volume — not retrieval quality — was the binding capacity constraint, which forced a second-tier "auto-resolved with reviewer audit-sample" path: drafts in a middle confidence band (0.70–0.85) auto-publish but sample 10% for periodic reviewer audit, with telemetry feeding back into threshold tuning. ([GrowwStacks 2026](https://growwstacks.com/blog/human-in-the-loop-ai-agents-langgraph))

## 6. Best Practices

- **Make the gate's decision criterion explicit at the API boundary.** A single `status` field — "published" or "draft_for_review" — keeps downstream routing legible and testable. Avoid implicit signals like HTTP status codes or response-length heuristics.
- **Tune the threshold against the cost-asymmetry of *your* domain, not a default.** The right faithfulness/relevance threshold for a customer-support assistant is not the right threshold for a clinical-imaging triage system.
- **Preserve the `failure_mode` field through to the review UI and the audit log.** Reviewers benefit from knowing whether the draft is retrieval-bug-shaped or generation-bug-shaped; downstream telemetry slices on the same field.
- **Enforce SLAs on the queue, with a logged escalation when breached.** A draft that ages out silently destroys the audit guarantee. SLA breach is itself an audited event.
- **Resist the urge to add a fifth reviewer action.** Approve / edit-and-publish / reject-with-redraft / reject-hard covers the workflow; additional options dilute the audit-trail clarity.
- **Sample some auto-published responses for offline review.** Threshold drift over time is real; a periodic re-evaluation against a held-out eval set catches it before customers do.
- **Document the gate's logic in an ADR with the threshold values.** When the threshold is later tuned, the ADR becomes the changelog and the audit-defensibility artifact.

## 7. Hands-on Exercise

**Task (whiteboard, 10–15 min):** You are designing the HITL fallback for a *non-federal-acquisitions* domain of your choice (e.g., a healthcare-imaging triage system, a fintech fraud-alert explainer, a legal-tech contract Q&A, a SaaS support assistant). Sketch:

1. The two scores your runtime gate computes, with one-sentence definitions tied to your domain.
2. The threshold values for each score, with a one-sentence cost-asymmetry justification.
3. The full envelope shape (JSON) returned on auto-publish and on review-queue routing — make sure both shapes are emitted with the same HTTP status.
4. The SLA on the review queue, with one sentence on why that SLA matches the domain's downstream consumer.
5. One audit event your design emits at each of the four reviewer actions.

**What good looks like.** The two scores are independent dimensions (faithfulness-shape and relevance-shape), not two re-phrasings of the same check. The thresholds are *explicit numbers* with a *cost-asymmetry rationale* — not "0.85 because the textbook said so." Both envelope shapes share the request envelope but differ on the `status` and the downstream-ID fields. The SLA reflects who the consumer is (customer-facing → tight SLA; internal-archive → loose SLA). The audit events have meaningful action names (`*_approved`, `*_edited`, `*_rejected`) and carry the correlation ID forward. Bonus credit for naming what happens on SLA breach.

## 8. Key Takeaways

- **What is conditional HITL?** Human review of the outputs the platform *itself* flags as low-confidence, not blanket review of every output.
- **What is the envelope contract?** A same-HTTP-status, different-body shape that surfaces the gate decision (`published` vs `draft_for_review`) at the API boundary.
- **What four actions can a reviewer take?** Approve, edit-and-publish, reject-with-redraft, reject-hard — each with a distinct audit event closing the loop.
- **When does the same pattern appear at framework level?** Next week's LangGraph `interrupt()` primitive applies the same pattern within an agent's graph execution, with state persistence and `Command(resume=...)` to continue.
- **Why is the threshold a domain-specific design decision?** Because the false-positive cost (reviewer fatigue) and the false-negative cost (wrong answer shipped) are domain-specific; the right threshold is the one that balances them for *your* environment.

## Sources

1. [LangChain docs — Add human-in-the-loop with LangGraph](https://docs.langchain.com/oss/python/langgraph/add-human-in-the-loop) — retrieved 2026-05-26
2. [LangChain blog — Making it easier to build human-in-the-loop agents with interrupt](https://www.langchain.com/blog/making-it-easier-to-build-human-in-the-loop-agents-with-interrupt) — retrieved 2026-05-26
3. [LangChain docs — Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts) — retrieved 2026-05-26
4. [Abstract Algorithms — Human-in-the-loop workflows with LangGraph: Interrupts, approvals, and async execution](https://www.abstractalgorithms.dev/langgraph-human-in-the-loop) — retrieved 2026-05-26
5. [GrowwStacks Blog — Human-in-the-loop AI agents in LangGraph: The 2026 production-ready approach](https://growwstacks.com/blog/human-in-the-loop-ai-agents-langgraph) — retrieved 2026-05-26

Last verified: 2026-05-26
