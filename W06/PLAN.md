---
week: W06
title: "Client Deliverability + Final Defense + Client Showcase + Cohort Retro"
phase: "Phase 2 (Modernization) → Deployment Gate"
gate: "**Deployment Gate / Final Defense + Client Showcase + Cohort Retro** (Thu 2 Jul 2026 for Cohort #1; Fri canonical)"
showcase_audience: "Karsun managers (3-5 internal, federal-acq-literate) per D-060"
density_target: "5–8 topics/day (Production/Capstone band)"
last_verified: 2026-05-23
cohort1_compression: "Fri 3 Jul = Independence Day observed (Jul 4 falls on Sat). W6 runs Mon–Thu (4 days) for Cohort #1. Original Fri Final Defense moves to Thu. Original Wed documentation block compresses to absorb Thu dry-run."
---

# W06 PLAN — Mon–Thu at a glance

> Master plan for Week 6 per `RevaturePro_FDE-Karsun_v2.pdf` §W6 + `deliverables/phase-2-deliverable-spec.md` + **D-060 (Karsun-managers showcase audience)**, **compressed for Cohort #1's loss of Fri 3 Jul (Independence Day observed)**. Mon = Checkpoint 4 + W6 Plan Day (§0 retro spans Phase 1+2 arc end-to-end). Tue = runbook authoring + ADR catalog curation. Wed = eval report + security attestation + HITL authority-boundaries synthesis + dry-run prep (compressed from original Wed + Thu). Thu = Final Defense + Client Showcase to **Karsun managers** + Cohort Retro (moved from Fri). Future cohorts without the holiday clash revert to the canonical 5-day W6 shape with gate on Fri.

## Mon — W6 Plan Day + **Checkpoint 4 Audit/Exam** *(7 topics, at Production band cap)*

*Carries the 4th and final checkpoint event (2-hour block). Plan Day for the client-deliverability arc runs morning; Checkpoint 4 audit + exam runs early afternoon; deliverable-taxonomy planning runs late afternoon.*

**Morning (Plan Day — `war-room/D1.md`)** — Gate-Boundary 1:1s with each pair (20 min each, 60 min total — the **final** gate-boundary cadence per `pipeline/PIPELINE.md` §4). **§0 retrospective spans Phase 1+2 arc end-to-end** (per D-036, extended scope for W6) — cohort reflects on W1-W5: what we adopted (Phase 1: LLM essentials → RAG → agentic), what we discovered (Phase 1 close-out at W3 Fri), what we modernised (Phase 2: AI-Native SDLC → AIOps → final adversarial review), and finalises the **deliverability package shape**. Trainer 1:1 + AI-Assist Project Audit. Pair commits W6 plan-spec (this is the **last** weekly plan-spec; W6 carries gate prep + handoff package authorship, not new tech adoption).

**Early afternoon (Checkpoint 4 audit + exam, 2-hour block — `assessments/checkpoint-4/`)** — Both tiers' exam runs simultaneously (0:00–1:30, 90 min). Audit walkthrough + debrief 1:30–2:00. Covers W5 content: AIOps, governance, compliance, deployment readiness. Calibration: **full** (gate-prep posture). Senior exam includes the deployment-readiness write-up format (FedRAMP-aligned checklist + 1 honest known-weakness declaration); Entry exam includes a runbook-completion task.

**Late afternoon (practical)** — Each pair drafts the deliverable taxonomy for their Phase 2 Project: runbook outline, ADR catalog INDEX scaffolding, eval report shape, handoff README skeleton, security-attestation framing, **HITL authority-boundaries doc skeleton (Wed-anchor capstone — synthesises all 7 programme touchpoints per D-043/D-044)**. Modeled as a stack of `Finding` entities opened against the pair-project repo + acquire-gov — mirrors the OIG Findings Tracker (`/admin/findings`) workflow the cohort built in W4/W5 (meta-runbook framing per `feature-inventory-target.md` line 391). Time-boxed: ship taxonomy skeletons by EOD; content lands Tue–Wed.

**Conceptual (`pre-session/1-Monday/1-DailyTopicOverview.md`)** — Pre-reading: stakeholder communication, three-levels rule (90-second / 5-min / 20-min versions), trade-off-naming honesty.

## Tue — Runbook Authoring + ADR Catalog Curation + Stakeholder Calibration *(7 topics)*

**Morning (war-room — `war-room/D2.md`)** — **Runbook authoring workshop** modelled on the OIG Findings Tracker (`/admin/findings`) the cohort built into acquire-gov in W4/W5. Each pair drafts their runbook **as a stack of `Finding` entities** — `opened_by`, `evidence_requests[]`, `remediation_status`, `due_at` — mirroring how GSA OIG audit findings actually close. The runbook covers: all 12 acquire-gov baseline debt items closure status (some still open is fine — name them); per-pair-unique debt items (per D-059); W4 modernization hop status (SB 2.7→3.0 + J11→17 per D-054); W5 AIOps integration (Datadog AI per D-060 #4); the W4 Fri Workflow 4 + Item 3 load incident closure status; pair-project incident-response runbook. **Stakeholder calibration mini-block (45 min):** instructor walks cohort through how the **Karsun manager** audience differs from CIO/OIG/Contracting Officer (the Tue afternoon roles W6 rehearses) — Karsun managers are federal-acq-fluent + staffing-decision-oriented, NOT pair-stack-deep. The same artifact gets framed differently for each.

**Afternoon (practical)** — **ADR catalog curation.** Each pair walks their W2 ADRs (RAG choices), W3 ADRs (multi-agent + LangGraph HITL), W4 ADRs (modernization hops + AI security), W5 ADRs (AIOps governance) into one consolidated `docs/adrs/INDEX.md` — every programme ADR with a "what we'd revisit now" annotation. Stakeholder interview simulation rotates: instructor plays Agency CIO (10 min/pair), OIG Auditor (10 min/pair), Contracting Officer (10 min/pair) — same artifact, three audiences. Live-demo discipline + "broken-but-named-broken" framing rehearsed.

**Conceptual (`pre-session/2-Tuesday/1-DailyTopicOverview.md`)** — Pre-reading: eval report shape (RAGAS quantitative + faithfulness trend), FedRAMP Moderate attestation language, OWASP LLM Top 10 (2025 v2.0) attestation patterns, HITL authority-boundaries framing for Wed's capstone synthesis doc.

## Wed — Eval Report + Security Attestation + HITL Authority Boundaries Synthesis + Dry-Run *(8 topics, at Production band cap)*

*Compressed: absorbs original Wed (documentation) + original Thu (dry-run prep). Long, dense day. Pre-event Tue night the pair has Tue's runbook + ADR catalog drafted; Wed = eval report + attestation + HITL synthesis capstone + dry-run.*

**Morning (war-room — `war-room/D3.md`)** — **FDE SME Q&A** session — instructor plays a senior FDE who has done 4 federal client deliverables; cohort asks "what do they look for at handoff?" Open-ended Q&A — pairs use this to stress-test their `known-weaknesses.md` framing + their handoff README clarity. **Codex handoff-package adversarial review fires** mid-morning against each pair's `docs/` PR (per `codex-reviews/handoff-package-adversarial-review.md`).

**Afternoon (practical, dense)** — **Documentation finalization push** (Tue artifacts close; Wed artifacts ship):
- Eval report (`eval/REPORT.md`) — **quantitative LLM eval results** from W2 RAG (RAGAS faithfulness + answer-relevancy + context-precision over time) + W3 agentic flow eval (intake-triage routing accuracy, evaluator→consensus agreement rate, SSDD-draft HITL approval rate). Karsun-manager-readable in 5 min, OIG-readable in 30 min.
- Known-weaknesses inventory (`docs/known-weaknesses.md`) — final pass; explicitly names which of the 12 acquire-gov baseline debt items remain open + why (e.g., "Item 12 GHA lint disabled remains open — opened as `Finding/F-2026-W6-001` in OIG Findings Tracker; closure deferred to Phase 3").
- Handoff README (`docs/handoff.md`) — written for the *next FDE* who picks up this Project.
- Security attestation (`docs/security-attestation.md`) — controls-met vs gaps-known vs gaps-blockers, mapped against **FedRAMP Moderate Rev 5** + **OWASP LLM Top 10 (2025 v2.0)**. Synthesises W4 OWASP work (LLM01 prompt-injection-via-stored-content on the 4 user-input surfaces; LLM03 supply chain `:latest`; LLM06 Excessive Agency on JWT-skip Item 1; LLM07 multi-tenant Item 10) + W5 governance (audit-log race Item 2 detector + auto-remediation authority).
- **HITL Authority Boundaries doc (`docs/hitl-authority-boundaries.md`)** — **THE W6 capstone deliverable**. Synthesises all 7 programme HITL touchpoints (D-043/D-044: #1 W1 Fri LLM Essentials, #2 W2 Thu RAG fallback, #3 W3 Mon Plan-Day ADR, #4 W3 Wed multi-agent handoffs, #5 W3 Thu LangGraph deep-dive, #6 W4 Wed OWASP LLM06, #7 W5 Wed AIOps auto-remediation) into one authority-boundary table per acquire-gov AI endpoint (all 8 in ai-orchestrator) + per pair-project AI surface. Each endpoint × touchpoint cell answers: full-auto / propose-and-await-approval / escalate-only / never-AI. Template lives at `weeks/W06/hitl-authority-boundaries.md`.
- Operational runbook (`docs/runbook.md`) — Tue's draft finalised: incident response + on-call surface + escalation paths + AIOps signals to watch (Datadog AI dashboards from W5). HITL authority section cross-references the synthesis doc.
- ADR catalog (`docs/adrs/INDEX.md`) — Tue's curation finalised.

**Late afternoon (dry-run, was original Thu)** — Each pair runs a 25-min dry-run of their Final Defense (15 demo + 10 Q&A). Instructor plays the Karsun manager + Agency CIO + OIG Auditor + Contracting Officer in rotation per dry-run pass. Other 2 pairs observe; provide 1 critical + 1 supportive note. Pairs revise overnight. **Compressed dry-run** — only one full pass, no time for second iteration. Risk acknowledged in §Density notes below.

**Conceptual (`pre-session/3-Wednesday/1-DailyTopicOverview.md`)** — Pre-reading: final defense format details, Karsun-manager-tier showcase calibration (D-060), cohort retro framing, Phase 3 commitment framing.

## Thu — **Deployment Gate / Final Defense + Client Showcase + Cohort Retro** *(GATE DAY)*

*Was originally Fri 3 Jul; moved to Thu 2 Jul due to Independence Day observed. All-day gate event.*

**Morning (gate block 1, 09:00–12:00)** — **Phase 2 Final Defense per pair** (~75 min per pair × 3 pairs ≈ 225 min):

Per pair format (per `deliverables/phase-2-deliverable-spec.md`):
- **15-min paired demo** — pair walks through the *modernised* system: what Phase 1 discovered, what Phase 2 changed, what production-ready looks like for the pair's federal-acquisitions aspect.
- **10-min Agency CIO Q&A** — instructor plays Agency CIO; probes FedRAMP posture, deployment readiness, cost shape, security + failure-handling.
- **10-min OIG Auditor Q&A** — instructor plays OIG Auditor; probes reproducibility, audit trail completeness, who-decided-what.
- **20-min per-individual architecture defence** — each pair member defends separately for gate rigour; questions span Phase 1 + Phase 2 architecture choices. **Per-individual format = gate decisions are per-learner, not per-pair.**

**Lunch (12:00–13:00)** — Cohort + instructor + any visiting stakeholders.

**Early afternoon (gate block 2, 13:00–14:30)** — **Client Showcase to Karsun managers** (per D-060). Each pair runs a 25-min showcase (15 demo + 10 Q&A) for invited **Karsun managers** (3-5 internal, federal-acq-literate). Polished demo + take questions from the Karsun-manager panel. Rubric calibrated to manager-tier: *"defensible handoff a Karsun manager could pick up and staff?"* NOT *"explain Resilience4j circuit-breaker config"* — see `assessments/client-showcase-rubric.md`. **Fallback:** if Karsun-manager panel can't be staffed Thu 2 Jul, degrade to instructor + 2 senior FDEs from another Karsun project; same rubric applies, substitute panel briefed on D-060 calibration before showcase. Cohort #2 graduates to mock-client panel.

**Mid-afternoon (gate block 3, 14:30–15:30)** — **Gate decisions delivered**:
- Per-pair pass / pass-with-notes / not-yet decisions
- Per-individual pass / pass-with-notes / not-yet decisions (the per-individual gate from D-?? )
- Live cohort discussion (~30 min) — what each pair did well, trade-off framing that landed, what would be a different choice with more time
- Pass-with-notes pairs/individuals get a named Phase 3 development target

**Late afternoon (15:30–17:00)** — **Cohort Retro** (~60 min) + Phase 3 introduction + cohort celebration (~30 min):
- Programme-wide retro: what worked, what didn't, what we'd change for Cohort #2
- Each cohort member names: (a) one thing they learned that surprised them, (b) one thing they'd tell themselves at W1 Tue if they could time-travel
- Phase 3 framing: 6-month on-the-job development continues; each pair commits to ONE improvement they'd ship in their first Phase 3 week
- Pair Project repos archived as cohort artifacts (revisited across Phase 3)

## Special notes for W6 (Cohort #1)

- **Fri 3 Jul = Independence Day observed, no class.** W6 runs Mon–Thu (4 days). Original Fri Final Defense moves to Thu 2 Jul; original Thu dry-run absorbed into Wed afternoon.
- **W6 Mon carries Checkpoint 4** (2-hour block early afternoon). Plan Day morning + Checkpoint afternoon + deliverable-taxonomy planning late afternoon. Dense Mon but topics stay within Production band (5–8).
- **Wed is dense** — documentation push + dry-run all on one day. Risk: dry-run reveals a fix needed and pairs have only Wed evening to address it (no Thu morning buffer). Pairs should expect a Wed-night push.
- **No Mid-Sprint Surprise W6** (only W4 carries that pattern per D-037).
- **No Live Defense Friday W6** (gate replaces the war-room block, but Thu is the gate day, not Fri).
- **No new scenario-alternatives prompts W6** (per D-040 `count: 0` override for W6 — deliverability week).
- **Final Adversarial Review PR closed W5 Fri** (the W5 culminating review feeds W6 Mon's runbook + eval report content). No further codex passes scheduled in W6 itself.
- **Phase 3 handoff** is the gate-day's closing arc (not a separate W7).

## Assets in this folder

- `PLAN.md` — this file.
- `README.md` — week-at-a-glance + D-060 Karsun-manager calibration + acquire-gov substrate.
- `pre-session/<N-DayName>/1-DailyTopicOverview.md` — pre-session reading material (Cohort #1 Mon–Wed only; **no D4** — Thu = Final Defense gate, no advance reading needed; **no D5** because no Fri / Independence Day observed).
- `war-room/D1.md` through `D3.md` — morning war-room scenarios. `war-room/D4-defense-showcase-retro.md` = Thu full-day gate agenda (instructor-facing, reproducible from this file alone if Charles is unavailable).
- `assessments/checkpoint-4/` — Checkpoint 4 audit + Senior exam + Entry exam (canonical: `KarsunFDE/assessment-ec` private repo; working copy here).
- `assessments/deployment-gate-defense-rubric.md` — pair + per-individual gate rubric, Karsun-manager-tier calibration per D-060.
- `assessments/client-showcase-rubric.md` — 25-min showcase rubric, Karsun-manager-panel scoring + fallback panel framing.
- `scenarios/` — 3 deliverable-shaping prompts (NOT live-defense-eligible — used Tue afternoon as deliverable-shape exercises): handoff package depth, HITL authority boundary matrix shape, eval report shape.
- `retros/cohort-retro-template.md` — Cohort Retro template (programme-wide, captures Cohort #2 lessons + Phase 3 commitment cards). Per-pair retros deferred to Phase 3 follow-up.
- `codex-reviews/handoff-package-adversarial-review.md` — codex review template for the Wed handoff-package PR (completeness checklist + handoff-grade rigor).
- `gate-day/` — populated Thu — gate-decision logs, per-individual gate verdicts, Phase 3 commitment cards.
- `hitl-authority-boundaries.md` — **W6 capstone synthesis doc template** — pair instantiates one per pair-project; canonical template lives at the week root for cross-cohort reference.

## Density check

| Day | Topic count | Production band |
|-----|-------------|-----------------|
| Mon | 7 | ≤8 ✓ |
| Tue | 6 | ≤8 ✓ |
| Wed | 8 | ≤8 ✓ (at cap) |
| Thu | gate day | n/a |

Wed at cap — Documentation-finalization push has 7 named deliverable types + dry-run. If Wed afternoon runs over, push runbook polish to Wed-evening homework + run dry-run earlier (1500 not 1600). Don't drop deliverables.
