---
template: pre-session-reading
week: W03
day: Fri
phase: Agentic (Gate)
topic: "Phase 1 Gate prep + W4 modernization arc preview"
estimated_total_minutes: 35
last_verified: 2026-05-23
fde_situations: [1, 2, 3, 4, 5, 6, 7, 9, 10, 11, 12]
tech: [cumulative — LLM essentials + RAG + agentic]
sources_research_briefs: [research/phase-1-defense-shape.md]
author: instructor
---

# Pre-session reading — Phase 1 Gate prep + W4 modernization arc preview

Week 3, Day Fri (Gate Day). Estimated total time on task: ~35 minutes. **Short on purpose** — Friday is gate day; the cohort has been reading + building all week. Tonight is for closing the loop, not opening new threads. Last verified: 2026-05-23.

## 1. Why this matters

Friday morning is the **Mid-Programme Gate** — Phase 1 (W1 Thu – W3 Fri) concludes. Each pair gets 45 minutes to demo + defend the pair-project as built across three weeks of AI Adoption. Friday afternoon is the Mid-Program Retro that drives W4 Mon's §0 plan retrospective. By Monday morning your pair will be planning Phase 2 (Modernization) against a brownfield that the *whole programme* now uses (acquire-gov) — not just your pair-project.

Tonight's read is short and prescriptive. Do the gate prep. Skim the W4 framing. Sleep.

## 2. Gate format — what to bring + what to expect (10 min)

**Per pair: 45 minutes total.**

- **15-min demo.** Live walkthrough of the pair-project as it stands. Show:
  - W1 LLM endpoint with structured output + HITL surface + retry/backoff.
  - W2 RAG layer with FAR/DFARS corpus + multi-tenant boundary + citation grounding + Thu W2 HITL-as-RAG-fallback.
  - W3 agentic flow — at least the intake-triage (Mon-Tue) AND a partial multi-agent evaluator flow (Wed-Fri) with at least one hard-interrupt resume captured live. LangSmith trace screenshots ready.
- **10-min Agency CIO Q&A.** Instructor or external observer plays the Agency CIO. Questions favor: cost shape, FedRAMP boundary, vendor migration story, "what if we add a second agency tenant tomorrow."
- **10-min OIG Q&A.** Instructor or observer plays the OIG auditor. Questions favor: audit completeness (where do AuditEvents live, what's the gap surface), HITL authority defensibility (FAR-citation backing each gate), retrieval reproducibility (can you replay last week's draft).
- **10-min per-individual architecture defense.** Tier-aware per CLAUDE.md memory `entry-tier-applied-recognition.md`. Senior candidates get production-shape questions ("at 10× volume, where breaks first?"); Entry candidates get applied-recognition questions ("name the three failure modes the LangGraph `interrupt_before` protects against").

**Tier mapping:**

| Tier | Defense shape | What the rubric is testing |
|------|---------------|------------------------------|
| **Senior FDE candidate** | Production-shape defense — defend cost, on-call, FedRAMP, scale, observability completeness | Can the candidate own a Phase 2 modernization decision in a real engagement? |
| **Entry FDE candidate** | Applied-recognition defense — name the patterns used + recognize the failure modes + explain trade-offs in their own words | Can the candidate *apply* the patterns under supervision + escalate appropriately when out of depth? |

## 3. The §0 retro framing (5 min) — what you'll do this afternoon

After the demos + defenses, the cohort retros Phase 1 as a whole. The §0-retro shape (per D-036) extends to a *phase-level* retro this week:

- What did the **W1 Thu** plan get right about LLM essentials? Where did reality diverge?
- What did the **W2 Mon** plan get right about RAG? Where did reality diverge? (You already did this retro last Monday — pull from your `W02-pair-N-retro.md`.)
- What did the **W3 Mon** plan get right about agentic systems? Where did reality diverge? (You'll author this retro today.)
- **Cohort-level synthesis:** what did Phase 1 *reveal* about the brownfield (acquire-gov) that modernization needs to address? This is the input to W4 Mon's §0 — the discoveries name themselves today.

Be specific. "We underestimated complexity" is not useful. "The W2 plan didn't budget time for the multi-tenant filtering on the RAG endpoint and we shipped it 1.5 days late" is useful.

## 4. W4 modernization arc preview (10 min) — don't dive in tonight

W4 starts Phase 2: **Modernization driven by Phase-1 discoveries** (per D-039). Three things shift Monday:

1. **acquire-gov becomes shared scope again.** Your pair-project continues but the whole cohort also now works against the shared brownfield. The 12 named brownfield-debt items are W4 modernization targets (Items 1 + 4 + 5 + 9 + 10 + 11 are the AI-security + supply-chain anchors; Item 3 + Item 2 are the Mid-Sprint Surprise anchors per D-060).
2. **The stack is LEGACY by design.** Per D-056, `acquire-gov` main is Spring Boot 2.7.18 + Java 11 + javax + AWS SDK v1 + Spring Security 5. W4 Thu is the OpenRewrite hop to SB 3.0 + J17 + jakarta. You'll feel the version-era debt before you fix it.
3. **HITL #6 lands W4 Wed** (AI Security / OWASP LLM06 Excessive Agency). HITL is now a programme thread you've seen explicitly four times (W1 Fri, W2 Thu, W3 Mon/Wed/Thu — wait, that's six). Tally going into W4: **5 of 7 touchpoints landed**. Two left: #6 W4 Wed, #7 W5 Wed.

Don't pre-plan W4 tonight. Monday's the Plan Day. Tonight is gate prep.

## 5. Defense anti-patterns to avoid (5 min — read aloud, memorize)

- **Defending decisions you don't own.** If your pair-partner made the Wed multi-agent framework decision, you don't defend it — they do. Per-individual architecture defense is *yours*; trade-off honesty matters more than knowing everything.
- **Citing sources you didn't read.** The OIG Q&A round has a question about FAR 15.308 — if you cite it, be ready for "what specifically does §15.308 say about delegation?" Read the text before citing.
- **Buried-the-lede answers.** When asked "what's your biggest risk in production?", don't open with framing. Lead with the risk. Then explain.
- **Pretending Phase 1 was clean.** The retro that surfaces the *discoveries* is the most valuable artifact you produce Friday afternoon. Pretending the W2 RAG corpus worked perfectly on the first ingestion means W4 Mon will start with a fake plan.
- **Skipping the LangSmith trace screenshots.** They're your evidence. The OIG plays an audit role; auditors love evidence.

## 6. EOD Friday — what gets committed

- 3 pair-retro files (`retros/W03-pair-N-retro.md`).
- 1 cohort-retro file (`retros/W03-cohort-retro.md`) naming Phase-1 discoveries that drive W4 Mon §0 retro.
- Phase 1 freeze tag on each pair-project repo (`phase1-freeze-pair-N-20260612` per D-056).
- Weekly MCQ submitted (mid-tier, 30 min, run after the retros).
- W4 Mon briefing distributed via the cohort channel for weekend skim.

## 7. Glossary refresh (cumulative — terms across W1–W3)

- **Hallucination failure modes** (W1) — confabulation, stale knowledge, schema drift.
- **Hybrid retrieval** (W2) — dense + sparse (BM25) + reranker. Default for FAR/DFARS clauses.
- **Citation grounding** (W2 Thu) — every claim traceable to a chunk; faithfulness threshold below which HITL #2 fires.
- **ReAct loop** (W3 Tue) — Reason + Act + Observe.
- **Supervisor-worker** (W3 Wed) — multi-agent default shape; supervisor delegates to workers.
- **Soft vs hard interrupt** (W3 Thu) — soft suggests, hard blocks. FAR-anchored where applicable.
- **Checkpointing** (W3 Thu) — `PostgresSaver` for production-shape; survives the SSA-shows-up-tomorrow gap.
- **HITL touchpoints landed so far:** #1 W1 Fri LLM essentials, #2 W2 Thu RAG fallback, #3 W3 Mon Plan Day ADR, #4 W3 Wed multi-agent handoffs, #5 W3 Thu LangGraph deep-dive.

## 8. Optional deep-dive (not required to participate)

- *(nothing tonight)* — Friday is gate day. The optional read is on hold until W4 Mon's pre-session.
