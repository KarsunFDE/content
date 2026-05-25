---
week: W02
title: "RAG Architecture (pure RAG — no agentic content)"
phase: "Phase 1 — AI Adoption into Brownfield"
gate: "—"
density_target: "8–12 topics/day (Foundation week)"
last_verified: 2026-05-23
hitl_touchpoint: "#2 — Thu (RAG fallback)"
corpus: "FAR + DFARS (~3,500 pages) per D-060 — dual-source citation + clause-precedence resolution is the W2 teaching artifact"
bedrock_authority: "Real Bedrock InvokeModel + token/cost accounting authorized W2 onward (D-060 exception to D-050). AWS *managed* RAG services (Knowledge Bases, Agents-for-Bedrock, OpenSearch Managed) STILL deferred to W5."
---

# W02 PLAN — Mon–Fri at a glance

> Master plan for Week 2 per `RevaturePro_FDE-Karsun_v2.pdf` + `pipeline/PIPELINE.md` §15 + `training-project/week-dependency-map.md` W2 section. **Pure RAG depth — no agentic content this week** (agentic lands W3 per D-032/D-033). Corpus = **FAR + DFARS** (~3,500 pages, dual-source citation + precedence resolution per D-060). Vector store = **MongoDB Atlas Vector Search** (already in stack — collocated with the document store per `pipeline/DECISIONS.md` D-031). LLM provider = **AWS Bedrock** via real `InvokeModel` (D-060 exception authorizing Bedrock from W2; AWS *managed* RAG services still deferred to W5 per D-050).

## Mon — Plan Day · RAG Architecture & Scenario Design Planning *(11 topics)*

*First formal Plan Day of the programme (per D-029). The pair authors a graded **Scenario Design Planning** artifact — no code lands today. §0 Plan retrospective on W1 Fri plan-spec is **deferred** (W1 Fri was Day 1 of Phase 1, not a multi-day plan worth retro-ing); the §0 retro pattern formally starts W3 Mon per D-036.*

**Morning (war-room, `war-room/Mon.md`)** — Federal-acquisitions RAG problem framed at the whiteboard. Instructor speaks as the Contracting Officer from W1 Thu/Fri: *"You told me grounding ships this week. Walk me through what that actually means before I authorise the agency CIO to commit budget."* Cohort surfaces the six RAG decision dimensions (corpus shape, embedding model, vector store, retrieval mode, citation grounding, evaluation pattern) against the FAR/DFARS corpus + OIG reproducibility NFR.

**Afternoon (practical, Plan Day)** — Each pair authors a **Scenario Design Planning** artifact (graded — substitutes for the lighter `week-plan-spec.md` template per `pipeline/PIPELINE.md` §4). Six required artifacts: requirements synthesis, 6R/system map, 2–3 ADRs (must commit chunking strategy + embedding model + vector store choice + LangChain v1.0 posture per D-033), estimate, risk register, open questions. Each pair commits ADRs in their pair-project repo + an identifying-marker stub PR against `acquire-gov` referencing the W2 work surface (`ClauseLibraryEntry`, `GET /api/clauses/search`, `POST /rag/clause-search` per `training-project/week-dependency-map.md`). **No code lands Monday.** Plan-spec integrity check by instructor at 17:00.

**Conceptual (`pre-session/Mon.md`)** — RAG architecture intro + vector DB landscape (pre-read **before** Mon morning). Tee-up for Tue retrieval engineering.

## Tue — Enterprise Retrieval Engineering *(11 topics)*

**Morning (war-room, `war-room/Tue.md`)** — Incident-mode war-room. Vendor question lands at the Q&A Triage queue (`/solicitations/:id/qa`) at 09:00: *"Section L.4 says proposals are due 30 days after solicitation posting; FAR 15.208(a) says 30 calendar days. DFARS 215.371-4 references a different timing. Which timing governs?"* Cohort discovers the dual-source FAR vs DFARS precedence question — and the platform's `POST /answer-qa` endpoint has no grounding yet. Pairs trace the failure path: naive retrieval over FAR alone misses the DFARS supplement; DFARS-precedence rule (per 48 CFR §201.104) isn't encoded anywhere.

**Afternoon (practical)** — Pairs implement Tue plan: ingestion pipeline for FAR + DFARS (~3,500 pages dual-source); chunking strategy from Mon ADR; embedding generation via Bedrock Titan Embeddings v2 (real `InvokeModel` calls per D-060 exception — token/cost accounting from request 1); Atlas Vector Search index build with **`agency_id` filter field** baked into the schema (closes **Item 10** at the index level, not at query time). Dense retrieval baseline + sparse (BM25) baseline + hybrid via Atlas `$vectorSearch` + `$search` with Reciprocal Rank Fusion in plain Python (NOT LCEL pipes — D-033). Flat-file RAG eval harness scaffold (LangSmith deferred to W5 per D-031).

**Conceptual (`pre-session/Tue.md`)** — Chunking strategies + embedding model selection (pre-read **before** Tue morning).

## Wed — Advanced RAG Patterns + **Dedicated Scenario Research Slot** *(12 topics, at cap)*

*Wednesday afternoon practical block is the **dedicated `/web-research` slot** (per D-040). Pairs work the 2–3 scenario-alternatives candidate techs against the constraint; ADRs due Thu EOD; Live Defense Fri.*

**Morning (war-room, `war-room/Wed.md`)** — Incident-mode. Cohort sees the first ingestion regression: re-indexing FAR Part 15 with the new chunking strategy quietly **lost the DFARS 215.3xx supplements** (folder-walk bug; DFARS sits one directory deeper). Multi-tenant boundary scare: a CS at agency `DLA` running `GET /api/clauses/search?q=trade-off` saw a draft clause from agency `GSA-FAS` (Item 10 surface). Wed morning is multi-tenant retrieval boundary work + parent-child chunking + contextual compression + reranking + retrieval anti-patterns.

**Afternoon (DEDICATED RESEARCH SLOT per D-040)** — Pairs work the 3 scenario-alternatives prompts (vector store choice, hybrid-retrieval strategy, embedding-model selection). 2–3 candidate techs each. ADR drafts due Thu EOD. Wed afternoon is NOT for implementation — it's for `/web-research`-grounded comparison.

**Conceptual (`pre-session/Wed.md`)** — Advanced RAG patterns + multi-tenant retrieval (pre-read **before** Wed morning).

## Thu — Citation Grounding Deep Dive + **HITL #2 (RAG fallback)** *(11 topics)*

**Morning (war-room, `war-room/Thu.md` — HITL #2 lands here)** — Incident-mode. Faithfulness regression hits production: cohort's W1 Thu hallucination ("48 CFR 47.305-2") re-surfaces, now with retrieval — the model cited FAR 47.305-2 (real clause about packaging) when asked about Section M evaluation factors (wrong FAR Part entirely). Retrieval pulled the wrong chunk; reranker promoted it. **This is the HITL #2 framing scene.** *When retrieval confidence drops below threshold OR when faithfulness check fails, what does the platform do?* The federal-acquisitions answer: **escalate to a contracting officer**, do NOT ship a guess. Cohort wires the HITL surface: `POST /answer-qa` returns a `needs_human_review` envelope when faithfulness < threshold; the CO sees a "review queue" in the Q&A Triage UI; only the CO publishes to all vendors.

**Afternoon (practical)** — End-to-end RAG integration on `POST /rag/clause-search` + `POST /answer-qa` + `POST /draft-solicitation`. Pair-project: same RAG layer on the pair's federal-acquisitions aspect (grants, FOIA, post-award). Citation grounding UI surface: every clause-quote in the drafter links back to the source chunk + clause-ID + last_revised date. Audit-trail expansion: every retrieval emits an `AuditEvent` with `correlation_id` (Item 6 — still inconsistent, fixed W5; but the retrieval-emit row gets in). **Item 5** (`legacy_chain.py` `LLMChain.run()` invocations) migrated to plain Python composition in the 3 entry points (Drafting Wizard + Amendment Editor + notification-copy generator) — D-033 LangChain v1.0 posture honored.

**Conceptual (`pre-session/Thu.md`)** — RAG fallback patterns + HITL #2 framing (pre-read **before** Thu morning — this is where HITL #2 is named in pre-reading so the war-room exercise lands with context).

## Fri — RAG Evaluation + Live Defense + W2 MCQ *(11 topics)*

**Morning (war-room, `war-room/Fri.md`)** — Incident-mode → **Eval-driven framing**. Scenario: cohort's PR-Fri-W2 RAG change improved drafter latency 30% but the eval harness shows a 12% drop in faithfulness on the FAR Part 15 held-out QA set. *"Do we ship?"* Cohort wires the RAG eval harness from scratch: flat-file QA pairs in `tests/rag-eval/qa.jsonl`, 4 RAGAS-style dimensions (faithfulness, context recall, context precision, answer relevance), LLM-as-judge (Bedrock Claude — real `InvokeModel` per D-060 exception), eval-as-test-fixture in CI (debt **Item 12** — GHA lint disabled — gets re-enabled here, scope: rag-eval suite only). Regression framing: any PR dropping faithfulness > 5% blocks. Security-eval extension: prompt-injection-via-clause-text probes (these formally land W4 Wed under OWASP LLM01).

**Afternoon (practical + assessment)** — **Live Defense Fri** (first of the programme — W1 deferred because no scenarios with ADRs yet). One learner per pair defends one W02 scenario-alternative for 10 minutes. **W2 MCQ** released 16:30 (standard two-tier — senior + entry — first appearance of the standard format; W1 was Light). Codex Adversarial Review (**Ramping** strictness per D-034 — P1 architectural-drift findings now blocking, not coaching).

**Conceptual (`pre-session/Fri.md`)** — RAG eval dimensions + W3 Mon Plan Day prep (agentic systems preview).

## Cohort #1 calendar note

W2 runs Mon 1 Jun – Fri 5 Jun 2026 (no holiday clash; full 5-day week). First standard W1 Mon→Fri rhythm of the programme.

## acquire-gov surface this week touches (from `training-project/week-dependency-map.md` W2 section)

- **Entity:** `ClauseLibraryEntry` (solicitation-service + MongoDB vector index) — FAR + DFARS clause body, `far_part`, `last_revised`. Corpus populated from real FAR/DFARS text retrieved via `/web-research`.
- **Endpoints:** `GET /api/clauses/search` (solicitation-service hybrid lexical+vector); `POST /rag/clause-search` (ai-orchestrator hybrid RAG benchmark surface); `POST /draft-solicitation` (RAG-grounded Section C SOW + Section L drafter — **Item 5** legacy chain migrated here); `POST /answer-qa` (Q&A Triage RAG — HITL #2 gate at "CO publishes answer" step).
- **Admin surface:** `/admin/config` System Config screen — "available vector stores: pinecone, atlas" (the **Item 7** lie made visible; cohort removes the unused `pinecone-client==5.0.0` dep + the config-screen line).
- **Debt items exposed this week:** **Item 4** (no Pydantic schema on the 4 AI endpoints), **Item 5** (legacy `LLMChain.run()` in 3 entry points — D-033 migration), **Item 7** (pinecone listed unused), **Item 10** (RAG retrieval must filter by `agency_id` — Wed work).

## Density risk — Wed at cap (12 topics)

If Wed morning multi-tenant work runs over, defer **caching patterns for hot queries** to Thu pre-session (acceptable carry-over — caching is foundation-stable). Do NOT defer the multi-tenant boundary work — Item 10 is load-bearing for W3 multi-agent + W4 AI Security; it needs to land in W2 or the W3 scenarios are unsafe to run.

## Assets in this folder

- `PLAN.md` — this file.
- `pre-session/Mon.md` through `Fri.md` — pre-session reading (one per day; pre-read before that day's war-room).
- `war-room/Mon.md` through `Fri.md` — morning war-room scenario (one per day; all incident-mode from Mon onward since cohort now has architectural vocabulary).
- `scenarios/` — scenario-alternatives prompts released this week (3 — at the upper end of D-040's 2–3 because Wed has dedicated research slot).
- `assessments/` — W02 standard two-tier MCQ + Live Defense rubric for Fri.
- `retros/` — End-of-week retros (3 pair + 1 cohort).
- `codex-reviews/` — Codex Adversarial Review on PRs (Ramping strictness per D-034).
- `README.md` — week overview.
