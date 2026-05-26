---
week: W02
day: Thu
phase: Foundation
topic: "RAG fallback patterns + HITL #2 (RAG-fallback) framing"
estimated_total_minutes: 50
last_verified: 2026-05-23
fde_situations: [1, 2, 3, 6, 7, 9, 11, 12]
tech: [rag-fallback, hitl-rag-fallback, faithfulness-evaluation, ragas, citation-grounding, audit-trail]
sources_research_briefs: []
author: instructor
hitl_touchpoint: "#2 of 7 — RAG fallback (per D-043 + D-044)"
---

# Pre-session reading — RAG fallback patterns + HITL #2 (RAG fallback)

Week 2, Day 4 (Thu). Estimated total time on task: ~50 minutes. Last verified: 2026-05-23.

> Read **before** W2 Thu morning. **Thu is HITL #2** — the second of seven programme HITL touchpoints (per D-043 + D-044). The pattern: when retrieval misses, or when faithfulness drops below threshold, the platform escalates to a contracting officer rather than ship a guess. This pre-read names the pattern explicitly so Thu's war-room (a faithfulness regression in production) lands with context, not as a surprise.

## 1. Why this matters (~85 words)

A grounded LLM that surfaces wrong-but-plausible answers is more dangerous than an ungrounded one — federal acquisitions runs on citations and the OIG audits them. Retrieval failures are expected (rare-but-real query, corpus gap, embedding-model drift after Bedrock model deprecation, reranker promoting the wrong chunk). The question is not *can we eliminate them* — it's *what does the platform do when one happens?* HITL #2's answer: **escalate to a contracting officer, never ship a guess.** This is the federal-acquisitions analog of the "fail loud" discipline.

## 2. Core concept in 5 minutes

A retrieval-augmented endpoint has three failure modes worth naming:

**Failure mode 1: retrieval miss.** No retrieved chunk crosses the similarity threshold. The cohort knows the question is out-of-corpus. Easy case: return `needs_human_review` with a "no clauses retrieved above confidence X" message. CO sees the question in a review queue.

**Failure mode 2: faithfulness drop.** Retrieval surfaced chunks; the LLM composed a response; a downstream faithfulness check (LLM-as-judge or RAGAS-style scoring) finds the response makes claims not supported by the retrieved chunks. The model went off-grounding. This is the most dangerous case because the response *looks* fine. Pattern: a second LLM call after generation that scores faithfulness; if below threshold, the response goes to the CO review queue with a flagged annotation showing which sentences failed.

**Failure mode 3: wrong-chunk retrieval.** Retrieval surfaced a *related but wrong* chunk (the Thu war-room scenario — FAR 47.305-2 packaging clause surfaced for a Section M evaluation factor question because the embedding model mapped "evaluation" to the wrong clause). The model dutifully composed a response from the wrong chunk. Faithfulness check passes (response is faithful to the chunk!) but the chunk is wrong. The fix is a relevance check: does the retrieved chunk's metadata (FAR Part) match the query's expected scope?

HITL #2 is the system-design answer to all three: **build the escalation gate at the response-emission point**. Every response from `POST /answer-qa` either (a) ships with citations the CO can verify or (b) goes to the CO review queue. There is no third path. The CO has a hard latency budget — they can't be the bottleneck on 200 Q&As/day — so the platform must minimize false-positive escalations (don't escalate confident-correct responses) AND minimize false-negative auto-publish (never ship a wrong-but-confident response).

## 3. What to read or watch tonight

- [LangChain v1.0 — Human-in-the-Loop with interrupts](https://docs.langchain.com/oss/python/langgraph/add-human-in-the-loop) (~12 min read), retrieved 2026-05-23 via /web-research. The LangGraph `interrupt()` primitive — formal anchor of HITL #5 (W3 Thu). For W2 Thu, focus on the framing: when an LLM-driven flow needs human approval, the framework's affordance is a hard pause that resumes with the human's input. The W2 pattern is simpler (no LangGraph yet) but conceptually equivalent: `POST /answer-qa` returns a `needs_human_review` envelope with the same shape an interrupt would carry.
- [RAGAS — Faithfulness metric](https://docs.ragas.io/en/latest/concepts/metrics/available_metrics/faithfulness/) (~8 min read), retrieved 2026-05-23 via /web-research. The canonical faithfulness check: extract claims from the answer; for each claim, check whether it's supported by the retrieved context. Returns a 0–1 score. The W2 Thu HITL gate uses faithfulness as the threshold function. Note: faithfulness alone is insufficient — see `ragas-faithfulness-only` known-bad-pattern (also use context-recall, context-precision, answer-relevance per Fri's harness).
- [AWS Bedrock — Guardrails for content filtering](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html) (~10 min skim), retrieved 2026-05-23 via /web-research. Not the primary HITL #2 mechanism (Guardrails are about input/output content policies, not faithfulness) — but worth knowing as a defense-in-depth layer. The W2 Thu HITL gate sits **after** Guardrails (Guardrails block obvious problems; the HITL gate catches the subtle ones).
- [OWASP — LLM06: Excessive Agency](https://genai.owasp.org/llmrisk/llm062025-excessive-agency/) (~8 min read), retrieved 2026-05-23 via /web-research. The OWASP LLM Top 10 (2025 list) framing of "AI can take consequential action without sufficient human oversight." W2 Thu is the early-anchor for LLM06; W4 Wed (HITL #6) is the formal deep-dive. Pre-reads cross-reference each other.
- *(optional)* [Anthropic — How we built Claude Code](https://www.anthropic.com/news/how-we-built-claude-code) (~15 min read), retrieved 2026-05-23 via /web-research. The "permission system" section is a real-world HITL design at production scale. Useful framing for the W2 Thu envelope schema.

## 4. Two questions to come in with tomorrow

1. *"What's the false-positive cost of HITL #2 (escalating something the CO would have published anyway) vs the false-negative cost (auto-publishing a wrong-but-confident response)? Which threshold are we tuning toward?"* (Answer: false-negative cost dominates in federal acq — one wrong-FAR-citation auto-published into a vendor-facing Q&A is a finding the OIG can open against the agency. Tune for low false-negative. Accept high false-positive in early weeks; tighten later as the eval harness gives us calibration data — Fri builds it.)
2. *"The CO review queue itself becomes a UX problem — what's the lightest-weight surface that doesn't bury her? Email? In-app banner? Slack? A daily digest?"* (W6 client deliverability deep-dives the operational angle; W2 Thu just names the question.)

## 5. Glossary refresh (terms you'll hear tomorrow)

- **HITL #2** — Human-in-the-Loop touchpoint #2 of 7 (per D-043 + D-044). The RAG fallback pattern.
- **`needs_human_review` envelope** — Response shape from `POST /answer-qa` when the platform refuses to auto-publish. Contains: the question, the retrieved chunks, the generated draft answer, the faithfulness score, the specific failure mode (`retrieval_miss` / `faithfulness_drop` / `wrong_chunk`), a CO-actionable annotation.
- **Faithfulness** — RAGAS metric. 0–1. Are the answer's claims supported by the retrieved context?
- **Context recall** — RAGAS metric. Of the ground-truth supporting facts, how many appear in the retrieved chunks? (Fri's harness builds this.)
- **Context precision** — RAGAS metric. Of the retrieved chunks, how many are actually relevant? (Fri's harness builds this.)
- **Answer relevance** — RAGAS metric. Does the answer address the question (irrespective of correctness)? (Fri's harness builds this.)
- **LLM-as-judge** — Pattern of using a second LLM call to score the first LLM's output on a dimension. Common implementation for faithfulness scoring at runtime.
- **AWS Bedrock Guardrails** — Input/output content-policy filter. Defense-in-depth layer. NOT the primary HITL mechanism for faithfulness.
- **LangGraph `interrupt()`** — Framework primitive for pausing graph execution pending human input. Anchored W3 Thu (HITL #5); W2 Thu uses the simpler envelope pattern.

## 6. Optional deep-dive (not required to participate)

- [Anthropic — Claude system card § Refusals & Safe Completions](https://www.anthropic.com/research/safe-completions) (~10 min skim), retrieved 2026-05-23 via /web-research. The model's own refusal behavior. Worth knowing as the lowest-layer signal: when Claude itself says "I don't know," the platform should NEVER override the refusal — escalate to HITL.
- [NIST AI Risk Management Framework — Govern function](https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF) (~12 min skim), retrieved 2026-05-23 via /web-research. The federal-context framing of "AI governance includes named human-decision-makers at named boundaries." W2 Thu's HITL gate is a NIST AI RMF Govern-function affordance in code form.

---

## Sources

All citations retrieved 2026-05-23 via /web-research per `pipeline/RESEARCH-PROTOCOL.md`. Recency windows applied: hot-tech 3-month (LangChain v1.0 interrupts, Bedrock Guardrails, OWASP LLM 2025 list), reference-material (RAGAS docs, NIST AI RMF — always-current via cited authority).

## Known-bad-pattern warnings against the reading

- Multiple "HITL = approve every LLM output" patterns floating in the wild. That's NOT what HITL #2 is — escalating *every* response is operationally infeasible at 200 Q&As/day. HITL #2 is conditional escalation against a faithfulness threshold + retrieval-confidence threshold. If a pair argues "we should HITL every response," push back.
- The `ragas-faithfulness-only` blocklist entry applies here: faithfulness alone misses wrong-chunk retrieval (failure mode 3 above). Fri's harness uses all 4 RAGAS dimensions.
- LangChain v1.0 HITL examples occasionally still show `interrupt(...)` returning the human input directly via `.run()` — verify against the v1.0 docs that the resume pattern uses `Command(resume=...)` or `graph.invoke(Command(resume=...))`, not legacy `.run()` semantics.
