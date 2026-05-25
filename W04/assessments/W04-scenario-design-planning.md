---
template: scenario-design-planning
week: W04
pair: <pair-1 | pair-2 | pair-3>
brief_released_at: 2026-06-15T09:00 (Mon W4 09:00)
artifacts_due_by: 2026-06-15T17:30 (Mon W4 17:30)
client_persona: Karsun delivery lead (with agency CIO + Karsun Security Engineering Lead context)
last_verified: 2026-05-23
fde_situations: [3, 4, 5, 6, 10, 11]
graded: true
graded_artifact_w4_per_claude_md: true
---

# Scenario Design Planning — Week 4 (Phase 2 begins)

Released: Mon W4 09:00. Artifacts due: same-day 17:30. Pair: <pair-1 | pair-2 | pair-3>.

> **W4 is one of three graded Scenario Design Planning weeks per v2 cadence** (W2, W4, W6 per `templates/scenario-design-planning.md`). This is the first front-of-Phase-2 design assessment. Brief is deliberately under-specified — pair surfaces gaps.

## 1. The brief (deliberately under-specified)

> *(written as Karsun delivery lead's Monday-morning email, with attached transcript from Friday's Phase 1 Defense)*
>
> **From:** Karsun delivery lead
> **To:** Pair-<N>
> **Date:** Mon W4 09:00
> **Subject:** Phase 2 modernization — scope for the week, ADRs by 17:30
>
> *"Friday's Phase 1 Defense was good — agency liked the AI adoption story. Now they want what's underneath. The agency CIO has signed off on the Bedrock budget through end of fiscal; what they want this week is a Phase 2 modernization scope they can defend at Friday's exec brief. The OpenRewrite hop on Java + Spring is in scope (Thursday). The 4 stored-content surfaces that came up in Wednesday's AI Security drop need a defense (Wednesday). The HITL authority-boundary table for the 8 ai-orchestrator endpoints is something Security has been asking for since W2 (Wednesday PM). I want ADRs on my desk by 17:30 today. Pick the 2–3 modernization items + the OpenRewrite hop that Phase 1 actually surfaced — don't try to do everything. The Risk Register needs to address what we deferred and why. And — heads up — Friday is going to be a long day. I won't say more than that. Plan accordingly."*
>
> **Attached:** transcript excerpts from Pair-<N>'s Friday Phase 1 Defense (pair gets their own pair-specific transcript).
>
> **NOT in the brief but the pair should surface:** budget for managed services (Bedrock Guardrails — D-050 says W5+); how to handle the Friday surprise the delivery lead alluded to; whether codex Full strictness blocks the Thursday hop PRs.

## 2. What the pair must produce (no code yet)

Six artifacts in `planning/W04/`, in this order:

1. **§0 Plan retrospective on W3 Phase 1 Defense** (1 page) — `planning/W04/00-retro.md`. **W4-specific addition to the standard six**: name verbatim which Phase 1 findings drive W4 modernization scope. This is the moment Phase 1 discoveries become a Phase 2 plan.
2. **Requirements synthesis** (1–2 pages) — `planning/W04/01-requirements.md`. Distil the brief: functional, non-functional, explicit, implied, contradictory. Call out missing info (managed-service budget, Friday surprise scope).
3. **6R / system map + diagram** (1 page + diagram) — `planning/W04/02-6r-map.md`. Apply the 6R framework to the legacy stack. Diagram acquire-gov's current state + the Thursday hop target state.
4. **Three to four ADRs** (1 page each) — `planning/W04/adrs/`:
   - `01-w4-modernization-scope.md` — 2–3 debt items + OpenRewrite hop, with Phase 1 rationale.
   - `02-ai-security-threat-model.md` — 3–4 OWASP LLM:2025 entries the pair exercises Wed.
   - `03-sb-3.5-future-hop.md` — drafted Mon, finalized Thu with Pass 3 evidence floor.
   - `04-sb-4.0-future-hop.md` — drafted Mon, finalized Thu with Pass 3 evidence floor.
5. **Estimate** (1 page) — `planning/W04/05-estimate.md`. Size each ADR's execution in hours. Risk-adjust. Name dependencies (OpenRewrite hop depends on Tue pre-flight).
6. **Risk register** (1 page) — `planning/W04/06-risks.md`. Top 8–10 risks across: OpenRewrite hop blowup, AI Security false-positives blocking publish, Mid-Sprint Surprise (unknown shape), codex Full strictness blocking modernization PRs, pair-project per-pair-unique debt collisions, audit-log race (debt item 2) surfacing during hop, Bedrock cost runaway, sys_admin role gaps.
7. **Open questions for the client** (½ page) — `planning/W04/07-open-questions.md`. Managed-service budget? Friday surprise scope/shape? Codex Full strictness blocking PRs?

## 3. Required submission format

- Markdown files in `planning/W04/` per the structure above.
- A `planning/W04/README.md` linking each artifact + naming lead author per artifact (pair must split authorship — both names appear; each artifact has a lead).

## 4. Time-boxing

- 09:00–10:30 — read brief + W3 retro transcript, draft §0 retro + open questions (artifacts 0, 7).
- 10:30–12:30 — requirements synthesis + 6R map (artifacts 1, 2).
- 13:30–15:00 — ADRs 01 + 02 (the load-bearing two — Modernization Scope + AI Security Threat Model).
- 15:00–16:00 — ADRs 03 + 04 (future-hop drafts; finalized Thursday with Pass 3 evidence).
- 16:00–17:00 — estimate + risk register (artifacts 5, 6).
- 17:00–17:30 — pair review, lead-author commit, submission.

## 5. Scoring rubric (instructor only)

Score each artifact 1–5. *Pass = ≥3 on every artifact; ≥4 average overall. Failure on any single artifact triggers a re-do during Tue practical block.*

| Artifact | 1 (poor) | 3 (acceptable) | 5 (strong) | Score |
|----------|----------|----------------|------------|-------|
| §0 retro on W3 Phase 1 Defense | Vague "what we learned" framing | Names 2–3 Phase 1 discoveries that drive W4 scope | Names discoveries verbatim from W3 transcript with quoted text + maps each to a W4 ADR | <> |
| Requirements synthesis | Misses managed-service constraint | Captures most requirements; flags some gaps | Captures everything; explicitly flags the Fri surprise + managed-service deferral as open | <> |
| 6R map | No 6R logic | Applies 6R credibly to the legacy stack | 6R justifies each choice; diagram makes the Thursday hop target obvious | <> |
| ADR 01 Modernization Scope | Pure list of debt items; no Phase 1 rationale | 2–3 items + hop, rationale per item | Items named with Phase 1 quote citations + rollback story + codex Full strictness mitigation | <> |
| ADR 02 AI Security Threat Model | Generic OWASP citation | 3–4 entries mapped to specific endpoints | Mapped to specific endpoints; HITL #6 authority-boundary outline included; 2023 vs 2025 ID delta named | <> |
| ADR 03/04 Future-hop drafts | Pure paper / blog-post citations | Acknowledges Pass 3 floor; placeholder for Thu evidence | Pass 3 floor planned: which module compiles; `dryRun` command identified | <> |
| Estimate | Single-point hours | Ranges with assumptions | Ranges + dependencies + Fri-surprise reserve time | <> |
| Risk register | Generic | 8+ risks with mitigations | Risks domain-specific; Mid-Sprint Surprise reserved budget; codex Full + audit-log race named | <> |
| Open questions | Trivial | Useful + prioritised | Prioritised + assume-if-no-answer fallback for each | <> |

## 6. Calibration note (consistency across cohorts)

Reference `skills/scenario-design-planning/references/calibration-examples/` for prior cohort submissions scored 1, 3, and 5 on each artifact (Cohort #1 = first delivery; calibration set seeded from instructor-authored examples).

## 7. W4-specific calibration warnings (NEW)

- **Pairs that omit §0 retro** are graded ≤2 on that artifact regardless of other quality — Plan Day discipline is the whole point of the assessment, not optional.
- **Pairs whose Modernization Scope ADR doesn't cite the W3 Phase 1 Defense transcript** are graded ≤3 on Modernization Scope — "Phase 1 surfaced X" must be quoted, not paraphrased.
- **Pairs whose AI Security Threat Model uses 2023 OWASP IDs** (e.g., "LLM07 Insecure Plugin Design") are graded ≤2 on that artifact — current canonical revision is 2025 (v2.0); using 2023 IDs is a known-bad-pattern per `research/owasp-llm-top-10-20260522.md`.
- **Pairs whose Risk Register doesn't allocate reserve time for the Mid-Sprint Surprise** are graded ≤3 on Risk Register — the surprise is on the schedule; not budgeting for it is a planning gap.
