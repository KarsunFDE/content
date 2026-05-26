# W04 · AI-Native SDLC + AI Security + Brownfield Modernization

*Phase:* Phase 2 — Modernization driven by Phase-1 discoveries (per D-038 + D-039 + D-044). **Phase 2 begins Monday.** W4 Mon §0 plan retrospective on the W3 Phase 1 Defense findings names which Phase 1 modernization needs drive W4 scope. The modernization isn't arbitrary — it's the answer to W3 Fri's discoveries.
*Gate:* — (Mid-Sprint Surprise Fri per D-049 + D-060 — not a programme gate, but a high-stakes live exercise.)
*Project phase:* **Pair Project Phase 2 begins.**
*Weekly assessments:* Scenario Design Planning (Mon, graded) + MCQ (Fri) + Live Defense (Fri PM, scoped to amended plan-spec) + Codex on PRs (Full strictness per D-034) + Adversarial PR Rubric (modernization PRs).

## Week shape — three interlocked themes

| Theme | Where it lands |
|-------|---------------|
| **Spec-driven dev as living discipline** | Mon §0 retro (3rd practice — W2/W3/W4 Mon) + Tue workshop (cohort discovers debt item 12: their own PRs aren't actually linted) |
| **AI Security (OWASP LLM Top 10:2025)** | Wed — full-day deep-dive + HITL #6 (LLM06 Excessive Agency framing of authority boundaries) + dedicated `/web-research` slot per D-040 |
| **Modernization execution** | Thu — OpenRewrite hop SB 2.7→4.0 + J11→21 + `javax.*`→`jakarta.*` (Jakarta EE 11, SF7, SS7) per D-054 + D-056 single-branch design; retargeted from SB 3.5 because SB 3.5 OSS-EOLs 30 Jun 2026 |
| **Production-incident reality** | Fri — Mid-Sprint Surprise = Workflow 4 + Item 3 load incident (per D-049 + D-060) |

## Day-by-day

| Day | Headline | Anchor topics (density 5–8) |
|-----|----------|-----------------------------|
| **Mon Plan Day** | Phase 2 framing + W4 Modernization ADR + AI Security Threat Model ADR | §0 retro on W3 Phase 1 Defense; Scenario Design Planning brief (graded); two ADR drafts |
| **Tue** | Spec-driven dev workshop + Brownfield Modernization Planning | Cohort discovers debt item 12 (lint disabled in GHA); fixes debt item 8 (frontend hardcoded URL); OpenRewrite pre-flight |
| **Wed AI Security** | OWASP LLM Top 10 deep-dive + **HITL #6** + dedicated /web-research slot | Item 1 JWT-skip exploit demo (LLM07/LLM08); Item 9 prompt-injection-via-stored-content fix (LLM01); HITL #6 authority-boundary table (LLM06 framing) |
| **Thu** | Modernization Execution — OpenRewrite hop | SB 2.7→4.0 + J11→21 + javax→jakarta (Jakarta EE 11 / SF7 / SS7); Java 25 + SF7 minor-release future-hop ADRs (evidence-backed per D-054 Pass 3) |
| **Fri MID-SPRINT SURPRISE** | Workflow 4 + Item 3 load incident | Incident at 09:00 (no prior warning); detect → respond → RCA → fix-or-stage-for-W5; amended plan-spec; Live Defense on amendments |

## D-049 + D-060 framing — the Fri surprise

Per **D-049** (cohort #1 W4 surprise = acquire-gov prod incident) and **D-060** (Phase 1b resolution: Mid-Sprint Surprise shape = Workflow 4 + Item 3 load incident):

- **Incident shape (locked):** During GSA cloud-modernization SSDD signing, multiple evaluators concurrently call `solicitation-service` for proposal text via the evaluation-service → solicitation-service path (debt item 3 — no circuit breaker). Threads pile up. Audit-log race (debt item 2) leaves the SSDD-sign event un-logged. OIG observer is in the next office and would flag this.
- **Why this shape:** maps cleanly to W5 AIOps Resilience4j anchor (Agent F territory) — cohort sees the gap Fri, builds the circuit Mon W5. Maps cleanly to the W5 audit-log race detector. Maps cleanly to the FedRAMP AU-2 + OIG audit-finding mental model the cohort has been building since W1 Tue.
- **What the surprise is NOT:** Item 1 JWT-skip exploit stays on W4 Wed AI Security day as the OWASP LLM07/LLM08 demo. It is not the Fri surprise. Curriculum delta-check: if a cohort author confuses the two, the W5 hand-off breaks.
- **What HITL #6 is NOT:** HITL #6 is Wed (LLM06 Excessive Agency authority-boundary table). It is NOT the Fri-surprise touchpoint. The Fri surprise exercises HITL discipline implicitly (does the pair escalate? to whom? at what authority?) but does not introduce a new touchpoint.

## OWASP LLM Top 10:2025 coverage matrix

Per `research/owasp-llm-top-10-20260522.md`. Note: **2025 (v2.0)** is the canonical revision as of 2026-05-22 (verified via `/web-research`). User-supplied earlier scope referenced 2023 LLM07 "Insecure Plugin Design" — the 2025 LLM07 is **System Prompt Leakage**; LLM08 is **Vector and Embedding Weaknesses**.

| OWASP ID:2025 | W4 surface | acquire-gov debt item |
|---------------|------------|------------------------|
| LLM01 Prompt Injection (indirect/RAG-borne) | Wed AM war-room + PM Exercise 2 | Item 9 (4 surfaces feed `/draft-amendment` + `/answer-qa`) |
| LLM02 Sensitive Information Disclosure | Wed PM (cross-tenant data) | Item 10 (4 list endpoints reproduce the leak) |
| LLM03 Supply Chain | Wed conceptual + W04-SA-2 | Item 11 (4 Dockerfiles `:latest` + ai-orch hand-pin), Item 7 (pinecone listed unused) |
| LLM04 Data and Model Poisoning | Wed conceptual (lighter) | RAG corpus poisoning hand-off to W5 |
| LLM05 Improper Output Handling | Wed PM (output validation) | Item 4 (4 AI endpoints share schema drift) |
| LLM06 **Excessive Agency** — **HITL #6 framing** | Wed PM Exercise 3 (authority-boundary table) | All 8 ai-orchestrator endpoints |
| LLM07 System Prompt Leakage *(NOT "Insecure Plugin Design" — that was 2023)* | Wed PM Exercise 1 (JWT-skip exploit) — tool-misuse path | Item 1 (`/api/public/**` JWT-skip filter) |
| LLM08 Vector and Embedding Weaknesses | Wed PM Exercise 1 cross-ref | Item 10 multi-tenant boundary on Atlas Vector Search |
| LLM09 Misinformation | Wed AM war-room (the hallucinated-FAR-clause framing) | Item 4 + Item 5 |
| LLM10 Unbounded Consumption | Wed conceptual + ties to Fri surprise | Item 3 (no circuit breaker = no consumption bound) |

Coverage check: **10 of 10** entries land in W4 with a concrete acquire-gov surface. Six get hands-on practical work; four get conceptual treatment (LLM02, LLM03 partial, LLM04, LLM09, LLM10 — the last three carry forward to W5 AIOps governance and W6 security attestation).

## Folder shape

Daily pre-session reading lives at `pre-session/<N-DayName>/1-DailyTopicOverview.md` — one per teaching day, 5–8 topics/day cap. Authored by `pre-session-author`. See `weeks-overview.md` for which days each week has.

| Subfolder | Artifact type | Skill that authors it |
|-----------|---------------|------------------------|
| `war-room/D1..D4.md` | Morning war-room scenario | `war-room-scenario` |
| `war-room/Fri-mid-sprint-surprise.md` | **Mid-Sprint Surprise instructor script + incident shape + scoring rubric** | hand-authored per D-049 |
| `scenarios/W04-SA-1..3.md` | Alternative-tech scenario (3 per week per D-040, 2–3 candidate techs each) | `scenario-alternatives` |
| `assessments/` | Scenario Design Planning (Mon), MCQ (Fri), Live Defense (Fri PM scoped to amended plan-spec), Adversarial PR Rubric | `scenario-design-planning`, `mcq-generator`, `live-defense-rubric`, hand-authored |
| `codex-reviews/W04-modernization-pr-template.md` | Codex Adversarial Review template for modernization PRs (Full strictness) | `codex-adversarial-review` |
| `retros/Fri-post-incident-retro.md` | Post-incident retro (replaces standard weekly retro since the incident IS the retro shape) | `weekly-retro` |
| `PLAN.md` | One-page week plan | Instructor (hand-authored, reviewed weekly) |

## Special notes for Week 4

- **All facts cite `/web-research` retrieval dates.** Per `pipeline/RESEARCH-PROTOCOL.md` (D-046), no fact in this folder is model-internal. OWASP LLM Top 10:2025, OpenRewrite, Spring Boot 3.x migration, AWS Bedrock Guardrails, Resilience4j, GitHub Actions OIDC — all cite dated retrieval.
- **Codex Adversarial Review at Full strictness per D-034.** W4 is the first full-strictness week. P0 + P1 findings block PR merge.
- **`final-expected-PR` is instructor-private.** Per D-054, the cohort sees `legacy-baseline` + rescue branches + `known-failures.md` per stage — they do NOT see the instructor's target diff until the W4 Fri post-incident retro.
- **The Mid-Sprint Surprise must be reproducible.** Per D-049 instructor authoring, the incident is feature-flag-gated chaos injection inside `acquire-gov` (concurrent-evaluator-load simulator). Same incident, every cohort delivery. Scoring rubric in `war-room/Fri-mid-sprint-surprise.md` §6.
- **HITL #6 placement verified Wed.** Not Mon, not Fri. Wed PM Exercise 3 is the authority-boundary table — that IS HITL #6.
- **No new Live Defense scenario this week.** Live Defense slot is repurposed for the amended-plan-spec defense post-Fri-surprise. W04-SA-1/2/3 ADRs are due EOD Thu (per D-040 cadence) but their Live Defenses run W5 Fri to avoid colliding with the Mid-Sprint Surprise.
- **AWS Bedrock posture:** InvokeModel + token/cost accounting authorized per D-060 Bedrock-W2-onward exception to D-050. Bedrock Guardrails (managed service) is NOT in scope this week — flagged as W5+ in W04-SA-2.
