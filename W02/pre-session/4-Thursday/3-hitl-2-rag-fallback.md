---
week: W02
day: Thu
topic_slug: hitl-2-rag-fallback
topic_title: "HITL #2 — RAG fallback as a programme-thread touchpoint"
parent_overview: W02/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 6
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
> **From topic 2:** three failure modes (retrieval miss / faithfulness drop / wrong-chunk) feed one queue. This topic is the envelope shape that does the routing.

## The envelope contract

Same HTTP status (200). Different body shape. The `status` field tells the caller which path the request took.

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

## Failure-mode → response-shape mapping

| Failure mode | `status` | `failure_mode` | Downstream |
|---|---|---|---|
| Both axes clear | `published` | (null) | UI renders; audit row `qa_published` |
| Faithfulness low, relevance ok | `draft_for_review` | `low_faithfulness` | CO queue (generation bug) |
| Faithfulness ok, relevance low | `draft_for_review` | `low_relevance` | CO queue (**retrieval bug** — yesterday's shape) |
| Both axes low | `draft_for_review` | `double_failure` | CO queue (escalate priority) |

## Four reviewer actions, no fifth

| Action | What ships | Audit event |
|---|---|---|
| **Approve** | Draft as-written | `qa_co_approved` |
| **Edit-and-publish** | Reviewer-modified text | `qa_co_edited` (diff preserved) |
| **Reject-with-redraft** | Nothing; re-prompt with feedback | `qa_co_rejected` + new cycle |
| **Reject-hard** | Nothing; request closed unanswered | `qa_co_rejected` (terminal) |

There is no fifth path. SLA breach is itself an audited event (`qa_co_sla_breach`) that escalates to a supervising reviewer — drafts cannot age out silently.

> [!IMPORTANT]
> **CO latency budget.** 4-hour business-hours SLA on the CO queue. Tune the threshold so queue volume stays below ~30 items/day per CO — otherwise the false-positive cost (CO swamped) destroys the workflow and reviewers degrade to rubber-stamping.

> [!WARNING]
> **Anti-pattern: "HITL = approve every LLM output."** Most internet HITL framings present blanket review as the safety story. Operationally infeasible at 200 Q&As/day — reviewers degrade to rubber-stamping within weeks and the safety property is silently gone. HITL #2 is **conditional** escalation against thresholds, not blanket approval. Per `known-bad-patterns.yml` `hitl-blanket-review`. Push back on any design proposing "HITL every response."

## Two layers — envelope today, `interrupt()` next week

| Layer | When | State surface |
|---|---|---|
| HTTP envelope (today) | Final-answer ship decision | Draft + scores in review-queue DB |
| LangGraph `interrupt()` (W3 Thu, HITL #5) | Mid-graph tool-call decision | Full graph state in checkpointer; resume via `Command(resume=...)` |

Same conceptual pattern — pause with structured state. Envelope is the right tool when the decision is "should this answer ship." `interrupt()` is the right tool when the decision is "should this tool call execute" (DB write, external API). Today's pattern; next week's framework primitive.

## Self-check

> [!NOTE]
> **Self-check** (30s)
>
> 1. Why is the `status` field on the envelope load-bearing rather than just using HTTP status codes?
> 2. What's the audit event when a CO edits a draft before publishing, and what payload does it carry?

<details>
<summary>Show answers</summary>

1. HTTP 200 means "request succeeded." Both auto-publish and review-queue are successful requests — the semantic difference is downstream routing. A single `status` field makes that decision legible, testable, and stable across UI / workflow / telemetry consumers.
2. `qa_co_edited` — payload includes the reviewer's identifier, the original draft, the edited final text, and the diff between them. The diff is preserved so the chain (model draft → reviewer edit → final publish) is reconstructible from the audit log alone.

</details>

<details>
<summary>The seven-touchpoint thread</summary>

| # | When | What |
|---|---|---|
| 1 | W1 Fri | LLM Essentials — reversibility, blast radius, audit demands |
| **2** | **W2 Thu (today)** | RAG fallback envelope on `/answer-qa` |
| 3 | W3 Mon | Plan Day ADR — HITL boundary committed in plan-spec |
| 4 | W3 Wed | HITL between multi-agent handoffs |
| 5 | W3 Thu | LangGraph `interrupt()` primitive |
| 6 | W4 Wed | OWASP LLM06 (Excessive Agency) re-assertion |
| 7 | W5 Wed | AIOps auto-remediation authority boundaries |

</details>

<details>
<summary>LangChain v1.0 hygiene (D-033)</summary>

Plain Python composition. `model_client.invoke(build_prompt(...))` + Pydantic validation. No `Chain` subclass. No LCEL `|` pipe. No `chain.run()`. Per `known-bad-patterns.yml` IDs `langchain-chain-class`, `langchain-lcel-pipe`, `langchain-chaining-verb` (last reviewed 2026-05-12). The agent-first v1.0 framing uses `create_agent` for agent-shaped problems and plain Python for sequential composition.

</details>

<details>
<summary>Sources (retrieved via /web-research per D-046)</summary>

1. LangChain — Add HITL with LangGraph: <https://docs.langchain.com/oss/python/langgraph/add-human-in-the-loop> — 2026-05-26
2. LangChain blog — interrupt: <https://www.langchain.com/blog/making-it-easier-to-build-human-in-the-loop-agents-with-interrupt> — 2026-05-26
3. LangChain — Interrupts: <https://docs.langchain.com/oss/python/langgraph/interrupts> — 2026-05-26
4. Abstract Algorithms — HITL with LangGraph: <https://www.abstractalgorithms.dev/langgraph-human-in-the-loop> — 2026-05-26
5. GrowwStacks — HITL agents 2026: <https://growwstacks.com/blog/human-in-the-loop-ai-agents-langgraph> — 2026-05-26

</details>

Last verified: 2026-06-03
