# W02 · RAG Architecture (pure RAG — no agentic content)

*Phase:* Phase 1 — AI Adoption into Brownfield (pure RAG depth; agentic content lands W3 per D-032/D-033)
*Gate:* —
*Project phase:* Pair Project Phase 1 continues (W1 Thu → W3 Fri)
*HITL touchpoint:* **#2 — Thu (RAG fallback)** — 2nd of 7 programme HITL touchpoints per D-043 + D-044
*Corpus:* **FAR + DFARS** (~3,500 pages) per D-060 — dual-source citation + clause-precedence resolution is the W2 teaching artifact
*LLM provider:* **AWS Bedrock** via real `InvokeModel` (D-060 exception authorizing Bedrock from W2 onward; AWS *managed* RAG services — Knowledge Bases, Agents-for-Bedrock, OpenSearch Managed — STILL deferred to W5 per D-050)
*Vector store:* MongoDB Atlas Vector Search (already in stack per D-031)
*Weekly assessments:* W2 standard two-tier MCQ (first appearance of senior+entry format) + Scenario Design Planning (Mon Plan Day) + Live Defense (Fri — first of programme) + Codex on PRs (Ramping per D-034)

## v2 day-by-day (per `RevaturePro_FDE-Karsun_v2.pdf` + `pipeline/PIPELINE.md` + `training-project/week-dependency-map.md` W2 section)

| Day | Headline | Anchor topics |
|-----|----------|---------------|
| **Mon Plan Day** | RAG Architecture & Scenario Design | Plan Day framing per D-029; six RAG decision dimensions at the whiteboard; FAR + DFARS corpus + OIG citation-trail constraint; Scenario Design Planning artifact (graded, six required items); ADR Discipline; LangChain v1.0 posture (no Chain class, no LCEL pipes per D-033) |
| Tue | Enterprise Retrieval Engineering | Vendor Q&A drops in (clause precedence FAR 15.208(a) vs DFARS 215.371-4); FAR + DFARS ingestion from eCFR Title 48; chunking (sub-paragraph); Bedrock Titan Embeddings v2; Atlas Vector Search index with `agency_id` filter declared; hybrid retrieval via `$rankFusion`; flat-file eval scaffold; Item 7 (pinecone-client) close |
| **Wed** | Advanced RAG + DEDICATED RESEARCH SLOT (per D-040) | Ingestion regression dropped DFARS Parts >=212 + cross-tenant clause leak (Item 10 manifests on `/api/clauses/search`); parent-child indexing; contextual compression; reranker (Cohere Rerank 3.5 on Bedrock); pre-filter vs post-filter; **PM afternoon = scenario-alternatives `/web-research` only — NO implementation** |
| Thu | Citation Grounding Deep Dive + **HITL #2 (RAG fallback)** | Faithfulness regression in prod (FAR 47.305-2 wrong-chunk auto-published as Section M factor); 3 RAG failure modes (retrieval miss, faithfulness drop, wrong-chunk); HITL #2 `needs_human_review` envelope on `POST /answer-qa`; faithfulness + relevance LLM-as-judge at threshold 0.85; CO review queue; Item 5 (legacy `LLMChain.run()`) migration to plain Python composition |
| Fri | RAG Evaluation + Live Defense + MCQ | Build eval harness from scratch (4-dimension RAGAS-style: faithfulness, context recall, context precision, answer relevance) with LLM-as-judge (Claude Haiku); first piece of CI in the repo (partial close of Item 12 GHA lint disabled); PR #47 don't-ship decision (latency +30% but context recall -14%); OIG-style finding for Item 12 via `POST /api/findings`; **Live Defense 14:00 (first of programme)** + **W2 MCQ 16:30 (standard two-tier — first appearance)** |

## acquire-gov surface touched this week (per `training-project/week-dependency-map.md` W2 section)

- **Entity:** `ClauseLibraryEntry` (solicitation-service + MongoDB vector index) — corpus = FAR + DFARS
- **Endpoints:** `GET /api/clauses/search` (sol-svc hybrid), `POST /rag/clause-search` (ai-orch RAG benchmark surface), `POST /draft-solicitation` (RAG-grounded drafter — Item 5 migration target), `POST /answer-qa` (Q&A Triage — HITL #2 gate)
- **Admin surface:** `/admin/config` System Config (Item 7 lie removed)
- **Debt items closed/partially-closed this week:** Item 4 (Pydantic on AI endpoints), Item 5 (LLMChain.run() migration), Item 7 (pinecone-client + admin lie removed), Item 10 (RAG retrieval `agency_id` pre-filter at index level), Item 12 (partial — rag-eval CI re-enabled; broader lint stays for W4)

## Folder contents

| Subfolder / file | Artifact type | Files |
|------------------|---------------|-------|
| `PLAN.md` | Week-at-a-glance plan (Mon-Fri grid) | 1 |
| `pre-session/` | Pre-session readings (Mon-Fri, one per day) | 5 — Mon, Tue, Wed, Thu, Fri |
| `war-room/` | Morning war-room scenarios (Mon-Fri) | 5 — Mon (Plan Day), Tue, Wed, Thu (HITL #2), Fri (eval ship-gate) |
| `scenarios/` | Scenario-alternatives prompts | 3 — W02-SA-1 (vector store), W02-SA-2 (hybrid retrieval), W02-SA-3 (embedding model) + `_index.md` |
| `assessments/` | Live Defense rubric + W2 MCQ | 2 |
| `retros/` | Weekly retro template (4 copies — 3 pair + 1 cohort during cohort delivery) | 1 (`_template.md`) |
| `codex-reviews/` | Codex Adversarial Review outputs (populated during cohort delivery; Ramping strictness per D-034) | 1 (`_README.md` placeholder) |
| `README.md` | This file | 1 |

## Special notes for W2

- **First formal Plan Day Mon** (per D-029). Cohort authors graded Scenario Design Planning artifact. No code lands Monday.
- **First incident-mode war-room Tue** (W1 Mon/Tue were Tour mode; W2 Mon was Plan Day). Tue's vendor Q&A surfaces the clause-precedence problem from D-060 head-on on Day 2.
- **Wednesday afternoon = dedicated `/web-research` slot per D-040.** Pairs work the 3 scenario-alternatives; NO implementation in the afternoon.
- **HITL #2 anchors Thu** (per D-043 + D-044 — 2nd of 7 programme HITL touchpoints). Pre-session names the failure-mode taxonomy; war-room is the anchor scene.
- **First Live Defense Fri 14:00** (W1 deferred per `weeks/W01/scenarios/_index.md` — no scenarios with ADRs by Fri W1).
- **First standard two-tier MCQ Fri 16:30** (W1 was Light tier — calibration is lower because cohort was bedding into the programme).
- **Codex Adversarial Review at Ramping strictness** per D-034 — P1 architectural-drift findings now blocking (W1 was Light). Specifically: pre-v1.0 LangChain patterns are now blocking, not coaching.
- **First piece of CI in the repo lands Fri** (the eval harness GHA workflow). Partial close of debt Item 12. OIG-style finding opened for the broader close (W4 Tue).
- **W3 Mon = first formal §0 retro** per D-036 — these W2 retros are its inputs. Don't bury negative signals.
- **Topic-density: 8-12/day** (Foundation density). Wed sits at the 12-topic cap; if morning multi-tenant work runs over, defer caching-patterns to Thu pre-session (acceptable carry-over per PLAN.md).

## Cohort #1 calendar note

W2 runs Mon 1 Jun — Fri 5 Jun 2026 (no holiday clash; full 5-day week). First standard W1 Mon→Fri rhythm of the programme.

## Verification — artifacts produced vs PLAN.md

| Artifact (per PLAN.md "Assets in this folder") | Status |
|------------------------------------------------|--------|
| `PLAN.md` | Produced |
| `pre-session/Mon.md` through `Fri.md` | All 5 produced |
| `war-room/Mon.md` through `Fri.md` | All 5 produced |
| `scenarios/` (3 scenario-alternatives + index) | Produced (W02-SA-1, W02-SA-2, W02-SA-3, _index.md) |
| `assessments/` (Live Defense + W2 MCQ) | Produced (W02-Live-Defense-rubric.md, W02-MCQ.md) |
| `retros/` (template scaffold for 4 retros during cohort delivery) | Produced (_template.md) |
| `codex-reviews/` (placeholder for cohort delivery outputs) | Produced (_README.md) |
| `README.md` | This file |

Total artifact count: **18 files** (1 PLAN + 5 pre-session + 5 war-room + 4 scenarios + 2 assessments + 1 retro template + 1 codex placeholder + 1 README) = **18**.
