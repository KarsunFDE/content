---
template: pre-session-reading
week: W04
day: Wed
phase: SDLC
topic: "AI Security Engineering — OWASP LLM Top 10:2025 + HITL #6 + Adversarial review consolidation"
estimated_total_minutes: 60
last_verified: 2026-05-26
fde_situations: [4, 5, 7, 8, 9, 11]
tech: [OWASP-LLM-Top-10-2025, JWT-validation, Pydantic-v2, AWS-Bedrock-Guardrails]
sources_research_briefs: [research/owasp-llm-top-10-20260522.md, research/pydantic-v2-20260522.md]
author: instructor
---

# W4 Wed Pre-Session — AI Security Engineering Day

> Read Tue night, before Wed AI Security Day. ~60 min — at the upper cap by design. Wed is the OWASP-depth day, the HITL #6 day, and the dedicated `/web-research` day all at once. Tomorrow you exercise OWASP LLM Top 10:2025 against `acquire-gov`'s actual surfaces in three exercises: (1) build a working JWT-skip exploit, (2) ship Pydantic v2 input validators on the 4 prompt-injection-via-stored-content surfaces, (3) produce the HITL #6 authority-boundary table for the 8 ai-orchestrator endpoints. This brief gives you the vocabulary, the threat surfaces, and the discipline frame for each.

## 1. Prompt-Injection Testing — the 4 stored-content surfaces of acquire-gov (10 min)

OWASP LLM01:2025 covers *both* direct and indirect prompt injection. The federal-acquisitions surface in `acquire-gov` is dominantly **indirect**: a vendor submits content that an agency user later asks the LLM to process. The vendor is the attacker; the agency drafter is the victim.

`acquire-gov`'s four stored-content surfaces (per `training-project/feature-inventory-target.md` line 352):

1. `POST /api/solicitations` — solicitation description field
2. `POST /api/solicitations/{id}/qa` — vendor question text
3. `POST /api/awards/{id}/debrief-request` — debrief narrative
4. `POST /api/contracts/{id}/cpars/{cparId}/rebuttal` — CPARS rebuttal

Each feeds either `POST /draft-amendment` or `POST /answer-qa` downstream. Tomorrow's AM war-room walks the indirect-injection path on each surface; the AM Beat 1 deliverable is a 4-row map (surface × LLM01 vector × downstream artifact × mitigation candidate) committed to `planning/W04/security/01-stored-content-injection-map.md`.

**Why this is a federal-acquisitions threat, not just a generic LLM threat:** a vendor whose injected Q&A text causes the agency to publish a fabricated FAR clause has just induced a published-solicitation defect. That triggers re-opening Q&A, OIG observation, possibly a bid-protest record. The blast radius is the agency's procurement integrity, not just a malformed model response.

- Primary reading: [OWASP GenAI Security Project — LLM01:2025 Prompt Injection](https://genai.owasp.org/llmrisk/llm012025-prompt-injection/) (~15 min), retrieved 2026-05-22 via `/web-research`. Focus on the **indirect injection** subsection.

**Question to bring to AM war-room:** for each of the 4 surfaces, what's the smallest realistic injected payload that would cause `/draft-amendment` to emit fabricated content into a published artifact?

## 2. PII Protection + Multi-tenant Boundary Validation (8 min)

LLM02:2025 (Sensitive Information Disclosure) and LLM08:2025 (Vector and Embedding Weaknesses) overlap in `acquire-gov`'s data model. The canonical cross-tenant leakage surface is **debt item 10**: the Atlas Vector Search index over FAR/DFARS clauses uses `findAll` without an `agency_id` filter. A query crafted on behalf of Agency A can return embeddings the system was meant to isolate to Agency B.

PII in `acquire-gov` is federal-employee/contractor data: CO names, evaluator identities in SSDD narratives, debrief recipients. Federal context: under FOIA Exemption 6 and FAR 24.202, this data has specific disclosure constraints. LLM02 is not abstract — it's "did the model just summarise a debrief in a way that names the evaluator panel?"

**Multi-tenant boundary** in this codebase = inter-agency boundary. Same code base, multiple agency tenants, one Atlas index. The boundary is `agency_id` discipline at every retrieval seam.

- Primary reading: [OWASP GenAI Security Project — LLM08:2025 Vector and Embedding Weaknesses](https://genai.owasp.org/llmrisk/llm082025-vector-and-embedding-weaknesses/) (~10 min), retrieved 2026-05-22 via `/web-research`. Cross-tenant embedding leakage is the canonical pattern.

## 3. Spring Security Controls — the JWT-skip exploit surface (10 min)

`api-gateway`'s `JwtSignatureFilter` has `validateSignature = false` on the `/api/public/**` route (debt item 1). Downstream services trust the gateway's auth filter to have validated the JWT — they read the `sub` and `role` claims directly. That trust chain is the exploit surface.

Tomorrow PM Exercise 1: build a working exploit that (a) mints an unsigned JWT (the `{alg: "none"}` shape), (b) hits `GET /api/public/opportunities`, (c) escalates to a downstream route that assumes authentication, (d) demonstrates the privilege escalation end-to-end. **The exploit document lives in `weeks/W04/wed-jwt-skip-exploit.md` — instructor-owned location, NOT committed to `acquire-gov` main. Do not push it.**

OWASP framing: this is **LLM07:2025 System Prompt Leakage** + **LLM08:2025 Vector/Embedding Weaknesses** path. Once past auth, the attacker can probe ai-orchestrator system prompts (LLM07) and query Atlas Vector Search across tenant boundaries (LLM08, via debt item 10). The JWT-skip is the foothold; LLM07/LLM08 are the lateral moves.

> **2025 vs 2023 trap:** LLM07:2025 is **System Prompt Leakage**. The 2023 LLM07 was "Insecure Plugin Design" and is stale. If a blog post or AI-generated note cites the 2023 ID set, treat it as out-of-date and re-anchor on `research/owasp-llm-top-10-20260522.md`.

- Primary reading: [OWASP GenAI Security Project — LLM07:2025 System Prompt Leakage](https://genai.owasp.org/llmrisk/llm072025-system-prompt-leakage/) (~10 min), retrieved 2026-05-22 via `/web-research`.

**Question to bring to AM war-room:** once you have the unsigned JWT and the public-route response, what's the second-hop downstream call you'd try, and which service-trust assumption does it exploit?

## 4. HITL #6 — Excessive Agency (LLM06) as authority-boundary design (10 min)

This is the **6th of 7 programme HITL touchpoints** (W1 Fri / W2 Thu / W3 Mon / W3 Wed / W3 Thu / **W4 Wed** / W5 Wed). It is also a numbered topic on its own — not a subsection of anything else.

OWASP LLM06:2025 Excessive Agency: *"an LLM-based system is granted authority to interface with other systems, take actions, or make decisions that exceed what is required for its purpose."* The mitigation is **not** "make the LLM less smart." It is **authority-boundary design** — for each tool the LLM can invoke, name: (a) who can call the LLM endpoint, (b) what authority the LLM exercises through it, (c) what gate requires a human approval, (d) what the rollback path looks like if the LLM was wrong.

This is HITL framed as a **security control**, not a UX pattern. Earlier touchpoints framed HITL as workflow (W1 Fri Essentials) or as agent orchestration (W3 LangGraph). Wed's HITL #6 frame is **OWASP-compliant authority gating in FedRAMP language**. Same discipline; different vocabulary; same artifact.

The artifact: **8-endpoint × 4-column authority-boundary table** for the ai-orchestrator (`/draft-amendment`, `/answer-qa`, `/rag/clause-search`, `/summarize-proposal`, `/consensus-draft`, `/debrief-draft`, `/cpars-rebuttal-draft`, `/health`). Each row names caller, AI authority, human gate, rollback. PM Exercise 3 produces this; it lands as `planning/W04/security/03-hitl-authority-boundary.md`.

> **Over-gating is itself an LLM06 finding.** Don't write "human approves before publish" on every row. `/health` exercises no AI authority. `/rag/clause-search` is read-only retrieval — the gate is downstream where the retrieved content gets *used*, not at retrieval. Naming what does NOT need a gate is the artifact discipline. A FedRAMP package cluttered with hollow controls fails its own audit.

> [!instructor-review] **Density carry-over guidance:** if PM Exercise 1 (JWT-skip exploit demo) runs over, defer Exercise 1 wrap to Thu AM's first 30 min — **never** defer the HITL #6 authority-boundary table. The table IS the touchpoint; the touchpoint IS the table.

- Primary reading: [OWASP GenAI Security Project — LLM06:2025 Excessive Agency](https://genai.owasp.org/llmrisk/llm062025-excessive-agency/) (~15 min), retrieved 2026-05-22 via `/web-research`. The five mitigation strategies (minimize extensions, minimize functionality, avoid open-ended functions, minimize permissions, execute in user's context) map 1:1 to the 8 ai-orchestrator endpoints.

**Question to bring to AM war-room:** for `/draft-amendment` specifically, what's the rollback once a draft is published (vs pre-publish), and how does the human-gate column differ between those two states?

## 5. Security Validation as Automated Tests + Audit-Trail Completeness Check (8 min)

Two PDF topics share a frame: a security control that isn't *tested* is decorative. PM Exercise 2 (debt item 9 prompt-injection fix) lands a PR with **Pydantic v2 input validators** + **output sanitisation** on the 4 stored-content surfaces. The discipline is: the PR contains the tests that prove the validator fires on adversarial input *and* lets legitimate input through.

Pydantic v2 specifics for this fix:
- `field_validator` for input shape + content sanitisation — strip dangerous patterns, length limits, character whitelisting where appropriate.
- Output sanitisation — remove any echoed user-input segments from the LLM response before returning to the caller (a partial defense against echo-back exfil).
- Test the *negative* cases: adversarial input must raise `ValidationError`, not silently pass. Test the *positive* cases: a normal vendor Q&A must round-trip cleanly. Both shapes in the PR.

**Audit-trail completeness check:** review the `audit_logs` write paths on every ai-orchestrator endpoint. Does each call (a) emit an event, (b) bind tenant + caller identity, (c) preserve the prompt/response shape an investigator would need? Gaps in the audit trail are themselves an OWASP LLM02 + LLM06 finding — you can't gate authority you can't observe.

- Pydantic v2 details: `research/pydantic-v2-20260522.md` (in-repo research brief).

## 6. Adversarial Review Consolidation from Yesterday's PRs + Spec-Driven Discipline Checkin (6 min)

Wed AM second-half consolidation: pair walks Tue's PRs through their codex Full output. **Codex strictness is at Full per D-034 starting W4** — P0 and P1 findings block merge. The discipline:

1. Open each Tue PR. Read the codex Full output line-by-line.
2. Re-check each P0/P1 finding against Mon's plan-spec (`planning/W04/`). Did the spec anticipate this? Did Tue's code drift from the spec?
3. For each finding: fix now, stage for Thu modernization hop with an ADR amendment, or defer to W5 with a documented gap. Name the call.
4. Surface to the AM war-room the findings that gate Thu's hop (e.g., a Spring Security regression that would compound the OpenRewrite work).

**Spec-Driven Discipline Checkin** is the meta-frame: the §0 retro discipline you've practised 4× by now (W2/W3/W4 Mon + Tue workshop) gets exercised again on Tue's deliverables. The spec is not a Mon-only artifact — it's the lens you re-check every day's output against.

## 7. Research: Alternative Scenario Tech + Bedrock Guardrails preview (8 min)

Wed afternoon is the **dedicated `/web-research` slot per D-040** — research W04-SA-1 (circuit-breaker), W04-SA-2 (prompt-injection defenses), W04-SA-3 (modernization sequencing) in the same block as PM Exercises 2 and 3. Output: `planning/W04/research/W04-SA-{1,2,3}-research.md`.

**AWS Bedrock Guardrails is the elephant in the W04-SA-2 room.** Bedrock Guardrails is AWS's managed prompt-injection / content-filtering layer — it would solve a meaningful chunk of LLM01/LLM02 surface in `acquire-gov`. **It is deferred to W5 per D-050** (Karsun-paid AWS account doesn't come online until W5; managed services are out-of-scope this week). W04-SA-2 explicitly asks you to defend an *unmanaged* prompt-injection defense this week — Pydantic-layer validators + output sanitisation, not Bedrock Guardrails. Recognition-tier awareness of Guardrails tonight; depth in W5.

- Bedrock Guardrails recognition reading: [AWS Bedrock — Guardrails overview](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html) (~10 min), retrieved 2026-05-23 via `/web-research`. **Recognition-tier only this week.**
- Bedrock catalog context: `research/bedrock-claude-catalog-20260522.md` (in-repo research brief).

## 8. Further reading (optional)

- [MITRE ATLAS — Adversarial ML Threat Matrix](https://atlas.mitre.org/) (~20 min), retrieved 2026-05-23 via `/web-research`. Companion framework to OWASP — STRIDE-style threat matrix. Useful prep for the AM war-room threat-modeling exercise if you want a second lens.
- [LangChain Security Best Practices](https://python.langchain.com/docs/security) (~15 min), retrieved 2026-05-23 via `/web-research`. `acquire-gov`'s ai-orchestrator uses LangChain v1.0; this page maps OWASP IDs to LangChain primitives (tool definitions, retriever wrappers, prompt templates).

Last verified: 2026-05-26
