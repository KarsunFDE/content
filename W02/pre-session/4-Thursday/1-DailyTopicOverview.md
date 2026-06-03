---
week: W02
day: Thu
phase: Foundation
title: "Pre-session — Citation grounding deep-dive + HITL #2 (RAG fallback)"
audience: All cohort members
estimated_total_minutes: 65
last_verified: 2026-06-03
fde_situations: [1, 2, 3, 6, 7, 9, 11, 12]
tech: [runtime-faithfulness, hitl-rag-fallback, ragas, llm-as-judge, prompt-injection-via-retrieval, owasp-llm01, owasp-llm06, owasp-llm08, latency-tuning, audit-trail-for-retrieval, langchain-v1, bedrock-claude-haiku]
sources_research_briefs:
  - "research/langchain-v1-20260522.md"
  - "research/bedrock-claude-catalog-20260522.md"
  - "research/owasp-llm-top-10-20260522.md"
  - "research/mongodb-atlas-vector-search-20260522.md"
author: instructor
hitl_touchpoint: "#2 of 7 — RAG fallback (per D-043 + D-044)"
---

# W2 Thu Pre-Session — HITL #2 (RAG fallback) + the four runtime gates

> [!NOTE]
> **From Tue (D2):** the embedding-model misfire — "evaluation" in the query embedded near "evaluation of packaging" in a FAR 47 chunk. High similarity, wrong scope. That misfire is Thursday's incident.

## 1. What you'll learn today

By the end of war-room you'll be able to:

- Wire a same-status-different-body envelope on `/answer-qa` that distinguishes auto-publish from review-queue routing.
- Explain why faithfulness-only is insufficient and which RAGAS axis catches wrong-chunk retrieval.
- Defend a 0.85 conjunction threshold against both tighter and looser alternatives using cost-asymmetry framing.
- Apply three layered controls (sanitization, provenance scaffold, classifier pre-filter) against indirect prompt injection.
- Reconstruct a six-month-old RAG request from the audit trail using correlation ID, chunk content hash, and doc version.

## 2. The day at a glance

```mermaid
graph LR
    T2[Topic 2: Faithfulness] --> T3[Topic 3: HITL envelope]
    T3 --> T4[Topic 4: Security primer]
    T4 --> T5[Topic 5: Latency]
    T5 --> T6[Topic 6: Audit trail]
    T6 --> WR[War-room:<br/>HITL #2 wiring]
```

| Topic | Focus | Reading min | Why you'll need it |
|-------|-------|------------:|--------------------|
| 2. Runtime faithfulness | Three failure modes + RAGAS direction | ~11 min | Names what HITL #2 is gating against |
| 3. HITL #2 envelope | Same-status, different-body contract | ~11 min | The exact shape you wire on `/answer-qa` |
| 4. RAG security primer | LLM01 indirect + LLM06 + LLM08 | ~11 min | Retrieval is now part of the attack surface |
| 5. Latency tuning | Six hot-path calls + five levers | ~11 min | Two judges parallelized = ~400ms saved |
| 6. Audit trail | Correlation ID + Item 5 migration | ~11 min | EOD verification depends on reconstruction |

## 3. Threading

- **HITL programme thread:** #2 of 7 — RAG fallback envelope on `/answer-qa`.
- **Phase thread:** Phase 1 (AI Adoption) — grounding moves from "model retrieves chunks" to "model retrieves chunks AND gate decides whether to ship."
- **Pair-project:** the envelope shape and audit taxonomy ports verbatim into your pair-project Q&A endpoint.
- **Decision anchors:** D-033 (LangChain v1.0 posture), D-043 + D-044 (HITL 7-touchpoint thread), D-034 (Codex Adversarial at Ramping), D-060 (real `InvokeModel`, no mocks).

## 4. Why today matters

Yesterday's auto-published wrong-FAR-Part answer is real. The contracting officer reading it would have signed a transportation-packaging spec into a Section M evaluation matrix. The fix is not "better model" or "more retrieval" — both were nominally working. The fix is a **runtime gate** that refuses to auto-publish when the system's own self-assessment falls below threshold, plus an **envelope shape** that surfaces the gate's decision at the HTTP boundary, plus an **audit trail** that reconstructs the whole chain six months later when OIG opens an investigation.

Federal acquisitions sits at the intersection of high false-negative cost (wrong answer creates legal exposure) and an OIG audit framework that treats incomplete audit trails as no audit trail. Today's five topics are the surface of that intersection. The morning war-room is implementation — none of the design decisions are new at war-room time because tonight's reading made them explicit.

> [!IMPORTANT]
> **War-room scene:** Yesterday at 14:47 the SSA shows a screenshot of an auto-published `/answer-qa` response citing FAR 47.305-2 ("Loading, blocking, and bracing of freight in railcars") as authority for a Section M evaluation factor. Faithfulness was 0.92. Relevance was 0.61. The platform shipped it anyway because only faithfulness was gating. By 17:00 today, the platform stops auto-publishing unreviewed answers and the failing query lands in `qa.jsonl` as a regression fixture.

## 5. How to read this

- Read topics 2-6 in order — each builds on the prior. Topic 2 names the failure modes, topic 3 routes them, topic 4 hardens the retrieval surface, topic 5 tunes the hot path, topic 6 makes the whole chain reconstructible.
- Self-checks at the end of each topic — take 30s to answer mentally before expanding.
- Deeper-dives optional but recommended for senior FDEs (judge selection, KV-cache, tamper-evidence layers).
- Hands-on exercises feed directly into the morning war-room.
- Total expected time: **~65 min at 100 wpm**.

> [!CAUTION]
> **Cross-topic anti-pattern:** Internet "RAG-in-90-min" tutorials treat RAGAS `faithfulness` as the sole runtime gate AND treat retrieved chunks as trusted context AND treat partial audit logs as acceptable. All three are anti-patterns today's curriculum explicitly contradicts. Per `known-bad-patterns.yml` slugs `ragas-faithfulness-only`, `retrieval-as-trusted-input`, `partial-audit-log`.

## 6. Two questions to walk in with tomorrow

1. Why is the `status` field on the envelope load-bearing rather than just using HTTP status codes — and what fails if you remove it?
2. Yesterday's incident scored 0.92 faithfulness. Which failure mode was it, and which RAGAS axis catches it that faithfulness misses?

<details>
<summary>Topic-to-war-room map</summary>

- Topic 2 → War-room block A (~25 min): wire faithfulness + relevance judges against `/answer-qa`; failing fixture from yesterday's incident.
- Topic 3 → War-room block B (~30 min): envelope shape + four reviewer actions + CO queue.
- Topic 4 → War-room block C (~20 min): three-layer defense applied to the vendor-amendment ingestion path (Item 5 corpus).
- Topic 5 → War-room block D (~15 min): parallel judges via `asyncio.gather` + p95 measurement.
- Topic 6 → War-room block E (~25 min): correlation ID propagation + Item 5 `LLMChain.run()` migration + EOD reconstruction test.

</details>

<details>
<summary>Tonight's reading map + thread context</summary>

- Pre-read budget Thu: ~65 min total at 100 wpm across 5 topic files + this overview.
- HITL thread: W1 Fri (#1) → **W2 Thu (#2, today)** → W3 Mon (#3) → W3 Wed (#4) → W3 Thu (#5, LangGraph `interrupt()`) → W4 Wed (#6, LLM06) → W5 Wed (#7).
- Tomorrow (Fri): RAG eval-harness build AM, first Live Defense + first two-tier MCQ PM.
- W3 Mon §0 retro consumes today's HITL #2 threshold landing as evidence.

</details>

<details>
<summary>Consolidated sources (all retrieved via /web-research per D-046)</summary>

- LangChain v1.0 — `create_agent` + HITL middleware + `interrupt()`: <https://docs.langchain.com/oss/python/langgraph/add-human-in-the-loop> — 2026-05-22
- AWS Bedrock Claude catalog: <https://docs.aws.amazon.com/bedrock/latest/userguide/> — 2026-05-22
- OWASP GenAI LLM Top 10 2025: <https://genai.owasp.org/llm-top-10/> — 2026-05-22
- MongoDB Atlas Vector Search: <https://www.mongodb.com/docs/atlas/atlas-vector-search/> — 2026-05-22
- RAGAS faithfulness: <https://docs.ragas.io/en/latest/concepts/metrics/available_metrics/faithfulness/> — 2026-05-26

Research briefs: `research/langchain-v1-20260522.md`, `research/bedrock-claude-catalog-20260522.md`, `research/owasp-llm-top-10-20260522.md`, `research/mongodb-atlas-vector-search-20260522.md`.

</details>

Last verified: 2026-06-03
