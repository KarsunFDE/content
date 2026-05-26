---
week: W06
title: "Client Deliverability + Final Defense + Client Showcase + Cohort Retro"
phase: "Phase 2 (Modernization) → Deployment Gate"
gate: "Deployment Gate / Final Defense + Client Showcase + Cohort Retro — Thu 2 Jul 2026 for Cohort #1; Fri canonical"
audience_for_showcase: "Karsun managers (3-5 internal, federal-acquisitions-literate) — per D-060"
density_target: "5–8 topics/day (Production/Capstone band)"
last_verified: 2026-05-23
---

# W06 · Client Deliverability + Pair-Project Defence (Deployment Gate)

*Phase:* Capstone / Phase 2 close-out (per D-044 — v1 W7 renumbered to v2 W6)
*Gate:* **Deployment Gate / Final Defense + Client Showcase + Cohort Retro** — **Thu 2 Jul 2026 for Cohort #1** (moved from Fri 3 Jul due to Independence Day observed; canonical Fri restored for future cohorts)
*Showcase audience:* **Karsun managers** (3-5 internal, federal-acq-literate) per D-060. Cohort #2 graduates to mock-client panel.
*Project phase:* Phase 2 — Modernization → Gate. The full arc from W3 Fri's "what we adopted + what we discovered" through W6 Thu's "what we delivered to the client" is the cohort's lasting artifact.
*Weekly assessments:* Deployment Gate Defense (rubric — pair + per-individual) + Client Showcase (rubric — Karsun-manager-calibrated). No MCQ + no Live Defense in W6 (gate replaces them).

## v2 day-by-day (per `RevaturePro_FDE-Karsun_v2.pdf` + D-060 calibration)

| Day | Headline | Anchor topics |
|-----|----------|---------------|
| **Mon** | W6 Plan Day + **Checkpoint 4 Audit/Exam** + Deliverable Taxonomy | §0 retro on W1-W5 arc end-to-end; Gate-Boundary 1:1s; Checkpoint 4 (90-min exam + 30-min audit walkthrough); deliverable-taxonomy scoping (runbook, ADR catalog, eval report, handoff README, security attestation) |
| Tue | Runbook Authoring + ADR Catalog Curation | OIG-Findings-Tracker-as-meta-runbook framing; runbook structure (incident response + AIOps signals from W5 + escalation paths); ADR catalog INDEX from W2-W5 ADRs with "what we'd revisit now" annotation |
| Wed | Eval Report + Security Attestation + HITL Authority Boundaries + Dry-Run | Quantitative LLM eval report (RAGAS + faithfulness trend from W2 RAG + W3 agentic); security attestation (W4 OWASP LLM03/06/07/08 + W5 governance synthesis); **HITL Authority Boundaries doc** (7-touchpoint synthesis — the W6 capstone deliverable); compressed dry-run of Thu gate |
| **Thu Gate Day** | **Final Defense + Client Showcase to Karsun Managers + Cohort Retro** | AM = Phase 2 Final Defense per pair (15-min demo + 10-min Agency-CIO Q&A + 10-min OIG-Auditor Q&A + 20-min per-individual architecture defence). PM block 1 = Client Showcase to Karsun managers (25 min/pair). PM block 2 = Gate decisions delivered. EOD = Cohort Retro + Phase 3 framing. |

## acquire-gov substrate this week touches

Per `training-project/week-dependency-map.md` §W6 + `feature-inventory-target.md`:

- **OIG Findings Tracker (`/admin/findings`)** is the **meta-runbook surface** — the cohort's own runbook + ADR catalog + eval report are modeled as `Finding` entities (`POST /api/findings` in evaluation-service) closed against `acquire-gov`. Each `Finding` has `opened_by`, `evidence_requests[]`, `remediation_status`, `due_at` — mirrors how GSA OIG audit findings actually close. [GSA OIG Audit A210064 Contract Administration of Federal Acquisition Service Information Technology Contracts, retrieved 2026-05-23, https://www.gsaig.gov/sites/default/files/audit-reports/A210064_3.pdf]
- **CPAR + Vendor Rebuttal (Workflow 6)** is the **meta-mirror** of W6's defense + retro shape — the cohort produces a "rating + rebuttal" of their own programme using the same data model they built (per inventory line 396). PM (cohort) rates; cohort gets 60-min rebuttal window in the cohort retro.
- **5 Reports list** (from inventory lines 300-307): Acquisition Pipeline, Vendor Past Performance, Contract Spend by Agency, **OIG Findings Status**, **Audit-log Activity**. The latter two are this week's runbook + attestation evidence base.
- **8 AI endpoints in ai-orchestrator** (`/draft-solicitation`, `/draft-amendment`, `/answer-qa`, `/eval/ssdd-draft`, `/rag/clause-search`, `/agent/intake-triage`, plus the two LangGraph evaluator→consensus→SSA handoff endpoints from W3) all need an entry in the HITL authority-boundaries table.
- **Workflow 4 Mid-Sprint Surprise** (from W4 Fri — Item 3 load incident) closure status is part of the runbook + security attestation evidence.

## How this folder gets filled

Daily pre-session reading lives at `pre-session/<N-DayName>/1-DailyTopicOverview.md` — one per teaching day. W06 (Cohort #1) only has D1–D3 (Mon–Wed): **no D4** — Thu = gate, no advance reading needed; **no D5** — Fri = holiday. Authored by `pre-session-author`. See `weeks-overview.md` for which days each week has.

| Subfolder | Artifact type | Skill / source |
|-----------|---------------|----------------|
| `war-room/D1.md..D3.md` + `war-room/D4-defense-showcase-retro.md` | Morning war-room (D1-D3) + gate-day full agenda (D4) | `war-room-scenario` (D1-D3); instructor (D4 — gate is reproducible from this file) |
| `scenarios/` | 3 W6-specific deliverability prompts (handoff depth, HITL boundary table shape, eval report shape) — NOT live-defense-eligible (no Fri Live Defense W6); used as **deliverable-shaping exercises** Tue afternoon | `scenario-alternatives` (adapted) |
| `retros/` | **Cohort Retro template** (programme-wide retrospective format; captures lessons for Cohort #2) | `weekly-retro` (adapted to cohort scope) |
| `assessments/` | **Deployment Gate Defense rubric** + **Client Showcase rubric** (Karsun-manager-tier calibration per D-060); Checkpoint 4 working copy mirrors `KarsunFDE/assessment-ec` private repo | `live-defense-rubric` (adapted to gate format); rubric calibrated to D-060 |
| `codex-reviews/` | Handoff-package adversarial review template (Codex reviews `docs/` PRs against handoff completeness checklist) | `codex-adversarial-review` |
| `gate-day/` | Gate-decision logs, per-individual gate verdicts, Phase 3 commitment cards | Instructor (Thu PM block 2-3) |
| `hitl-authority-boundaries.md` | **Standalone capstone artifact** synthesizing all 7 HITL programme touchpoints (D-043/D-044) into one authority-boundary table per acquire-gov AI endpoint + per pair-project AI surface | Authored Wed afternoon by each pair against the canonical template in this folder |

## Special notes for W6 (Cohort #1)

- **Fri 3 Jul = Independence Day observed, no class.** W6 runs Mon–Thu (4 days). Original Fri Final Defense moves to Thu 2 Jul; original Thu dry-run absorbed into Wed afternoon. Future cohorts without the holiday clash revert to 5-day W6 with gate on Fri.
- **W6 Mon carries Checkpoint 4** (2-hour block — 90-min exam + 30-min audit walkthrough). Plan Day morning + Checkpoint afternoon + deliverable-taxonomy planning late afternoon.
- **Wed is dense (8 topics — at Production band cap).** Documentation push + HITL synthesis doc + dry-run all on one day. Risk: dry-run reveals a fix needed and pairs have only Wed evening to address it. Pairs should expect a Wed-night push.
- **Client Showcase audience = Karsun managers (3-5 internal) per D-060.** Defense rubric calibrated to federal-acq-literate manager tier — NOT pair-specific-stack-deep. Rubric asks "did the cohort produce a defensible handoff a Karsun manager could pick up and staff?" NOT "did they explain Resilience4j circuit-breaker config." Cohort #2 graduates to mock-client panel.
- **Karsun-manager panel staffing fallback:** if internal Karsun-manager panel can't be staffed Thu 2 Jul, degrade to instructor + 2 senior FDEs from another Karsun project as substitute panel. Same rubric applies; substitute panel briefed on D-060 calibration before showcase. See `assessments/client-showcase-rubric.md` §Fallback.
- **No Mid-Sprint Surprise W6** (only W4 carries that pattern per D-037).
- **No Live Defense Friday W6** (gate replaces the war-room block; Thu is the gate day, not Fri).
- **No new scenario-alternatives prompts W6 in the W3-W5 sense** (per D-040 W6 override). The 3 scenarios authored here are **deliverable-shaping exercises**, not technology-comparison ADRs.
- **Final Adversarial Review PR closed W5 Fri** (the W5 culminating review feeds W6 Mon's runbook + eval report content). One additional codex review runs this week — on the **handoff package PR** Wed afternoon (per `codex-reviews/handoff-package-adversarial-review.md`).
- **Phase 3 handoff** is the Thu gate-day's closing arc (not a separate W7).
- Topic density: 5–8/day (Production/Capstone band per `pipeline/PIPELINE.md` §4).

## What W6 produces that lasts beyond the programme

1. **Handoff package per pair-project** — runbook + ADR catalog + eval report + security attestation + threat model + cost analysis + deployment plan + HITL authority-boundaries doc. Lives in each pair-project repo.
2. **HITL authority-boundaries synthesis doc** — single reference table covering all 7 programme touchpoints across all 8 ai-orchestrator endpoints + the pair-project AI surfaces. Cohort #2 inherits this as a baseline; refreshed each cohort.
3. **Cohort retro** — captures lessons for Cohort #2 + Phase 3 commitments per learner.
4. **Phase 3 commitment cards** — one improvement each pair would ship in their first Phase 3 week.
