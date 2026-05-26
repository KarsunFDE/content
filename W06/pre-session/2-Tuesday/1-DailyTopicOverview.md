---
week: W06
day: Tue
title: "Pre-session — Stakeholder storytelling for federal-acq audiences (Tue activity) + Wed-prep eval/attestation/HITL concept load"
audience: All cohort members
time_on_task_minutes: 50
last_verified: 2026-05-26
---

# W6 Tue Pre-Session — Stakeholder Storytelling (Tue-targeting) + Wed-Prep Concept Load

> Read before Tue morning. ~50 min. **Two halves.**
>
> **Front half (Tue-targeting, ~20 min):** the storytelling discipline you'll rehearse Tue PM in the stakeholder rotation — same artifact, three audiences (Agency CIO / OIG Auditor / Contracting Officer).
>
> **Back half (Wed-prep, ~30 min):** the three Wed-targeting concepts (eval report metrics and reader structure, FedRAMP attestation language, HITL authority-boundaries capstone framing). Tue authors the runbook + ADR catalog; Wed authors the eval report + security attestation + HITL synthesis. We load the Wed concepts tonight so Tue's authoring stays focused.

---

## Front half — Tue activity briefing (stakeholder storytelling)

## 1. Technical storytelling for federal-acq audiences (8 min)

Senior-engineer technical storytelling fails federal audiences for a specific reason: it over-indexes on cleverness ("we picked a fancy retrieval ranker") and under-indexes on **auditability** ("we picked this ranker because the eval set was constructed this way and we can defend the choice in 5 years"). The federal-acquisitions register is different.

What a federal-acq technical story actually looks like:

- **Clause-anchored.** Mention the FAR / DFARS / FedRAMP clause that constrains the decision before you mention the technical choice. "FAR 15.308 says the Source Selection Authority decision cannot be delegated — so our LangGraph evaluator→consensus→SSA flow puts a hard human interrupt at the SSDD-draft step." That ordering is the register.
- **Controls-language framing.** Frame the AI behaviour as a *control*, not a *capability*. "The RAG fallback path is a `propose-and-await-approval` control on retrieval-confidence-below-threshold" reads better to a Karsun-manager / OIG-Auditor audience than "we have a confidence threshold." See §4 below for the four authority levels (`full-auto` / `propose-and-await-approval` / `escalate-only` / `never-AI`).
- **Honest known-weakness declared up front.** The "broken-but-named-broken" rule — Tue's war-room (`war-room/D2.md` §C Known-issue inventory) hardens this. A story that opens with "here are the 2 things we'd revisit with another week" beats a story where the auditor *discovers* the same 2 things 8 minutes in. The Karsun-manager-fluent storytelling register names tradeoffs before the auditor finds them.

**Karsun-manager-fluent register vs senior-engineer register.** A Karsun manager (Thu PM showcase per D-060) is federal-acq-fluent + staffing-decision-oriented, NOT pair-stack-deep. The story you tell a manager is *"could a new FDE pick this up in a week?"* — not *"here's our Resilience4j circuit-breaker config."* If the storyteller can't switch registers between audiences, the artifact reads as if every audience is the same — and three out of four audiences feel patronised or under-served. The stakeholder rotation Tue PM is the rehearsal.

## 2. Three-levels rule (5 min)

Same story, three lengths:

- **90-second version** — elevator pitch to an Agency CIO walking by. Just the headline + the cost shape + one known-weakness. *"acquire-gov + grants-portal-modern: RAG-grounded solicitation drafting with HITL at every irreversible step. Token cost ~$0.04/draft at projected volumes. Open weakness: Item 12 GHA lint deferred to Phase 3."*
- **5-minute version** — Karsun-manager showcase Q&A. 90-second version + 3-4 details that matter to staffing + cost defensibility (eval-set provenance summary, per-tenant cost line, the HITL touchpoint count, Phase 3 commitment).
- **20-minute version** — OIG-Auditor deep-dive. 5-minute version + methodology + per-tenant breakdowns + reproducer steps for each open `Finding`.

The three levels **nest** — the 5-min IS the 90-second + 3-4 details; the 20-min IS the 5-min + methodology + breakdowns. Don't author three independent stories. Author the 20-min, then **subtract** to 5 and 90s. If your 5-min doesn't fit inside your 20-min, you have two stories and one of them isn't true.

Which version for which audience:

| Version | Audience | When |
|---------|----------|------|
| 90s | Agency CIO at the door | Hallway / elevator / arrival |
| 5min | Karsun manager | Thu PM showcase Q&A |
| 20min | OIG Auditor | Thu AM gate Q&A |

The Tue PM stakeholder rotation (instructor plays CIO / OIG / Contracting Officer per `war-room/D2.md` §Act 5) rehearses transitions across these three lengths.

## 3. Tradeoff honesty (5 min)

How to name a tradeoff in federal-acq register:

> *"We picked latency over completeness on the RAG fallback path because the FAR 15.206 amendment acknowledgement window is 24 hours, not real-time — so missing the absolute last paragraph of a corrigendum is acceptable; missing the 24-hour SLA is not."*

What that sentence has:

1. **Two named values in tension** (latency, completeness) — not one disguised value.
2. **The clause that adjudicates the tension** (FAR 15.206 24-hour window) — so the auditor can verify the adjudication is correct, not just stated.
3. **The acceptance** ("acceptable") — explicit, not implied.

The "broken-but-named-broken" rule (from `war-room/D3.md` preview): a *named* gap is half-closed; a *hidden* gap is whole-open and growing. Karsun-manager trust falls off a cliff when the manager discovers a gap the pair didn't name. Conversely, naming gaps **before** they're discovered builds trust — the manager learns the pair has internal honesty and doesn't have to do the auditing themselves.

Common patterns to avoid:

- "We didn't have time" — non-falsifiable, reads as excuse. Replace with "we deferred X to Phase 3 because Y had higher coupling to the W4 modernization hop."
- "It works fine in practice" — Karsun-managers + OIG-Auditors both hear "we don't have evidence."
- "Edge case we'll handle later" — name the case + the Phase 3 `Finding` ID + the ETA.

## 4. Live-demo discipline (5 min)

Tue is the **briefing**; Wed late afternoon is the **practice** (the dry-run per `war-room/D3.md` §Dry-run discipline). Pre-Thu rehearsal checklist:

- **Time the demo.** 15-min target. If it runs 22 minutes Wed dry-run, it'll run 25 Thu — Thu is not the rehearsal, Thu is the gate.
- **Graceful-degradation plan.** Bedrock rate-limit → pre-recorded fallback video staged on local laptop. LangGraph interrupt-node timeout >4hr → script for explaining the HITL escalation path verbally. Datadog AI dashboard down → screenshot from yesterday-evening saved in `docs/demo-fallback/`.
- **No live `grep`s, no live searches.** Have the laptop tab pre-loaded. The terminal pre-pasted with the command. The dashboard URL bookmarked. Live typing on a 15-min demo loses 90 seconds you don't have.
- **Per-individual gate awareness.** The Thu defence is per-individual (gate decisions are per-learner, not per-pair, per `weeks/W06/PLAN.md` §Thu). The demo is one shared piece; the architecture defence is two. Both pair members should be able to drive any segment of the demo if Bedrock fails over to fallback at minute 8.

Cross-link: this is the **briefing**; Wed §3 Dry-run discipline is the **practice**. By Thu morning the muscle memory is set.

## 5. Audit-trail walkthrough (5 min)

How to walk an OIG-Auditor through your audit-log live:

1. **Pick the most material incident** — for Cohort #1 that's the W4 Fri Workflow 4 + Item 3 load-incident-during-modernization (per D-060 #3). Real, recent, and the cohort closed it.
2. **Pull up `/admin/findings`** — the OIG Findings Tracker UI you built into acquire-gov in W4 + W5. The W4 Fri incident has a `Finding` entity with `opened_by`, `evidence_requests[]`, `remediation_status`, `due_at`, and a state-transition log.
3. **Walk the state transitions in order** — `open` → `evidence-pending` → `remediation-in-progress` → `closed-remediated`. Show every state change has a timestamp + actor + linked artifact (PR / runbook section / Datadog screenshot).
4. **Show the audit row in the database** — the `audit_logs` collection entry that anchors the state transition. Karsun-manager + OIG-Auditor both want to see this — *"is there a row, or is there just a UI?"*
5. **Show the AI-assist annotation** — per-PR `assistance_mode` (Claude-Code-assisted / hand-authored / codex-suggested per D-035). For an AI-assisted system, audit-trail completeness includes *which work was AI-touched*. The Mon pre-session §3.5 (AI-Assist Project Audit) framed this — Tue rehearses it live.

The `/admin/findings` UI is **itself the audit-trail walkthrough surface**. You don't have to invent one for the dry-run. Use what you built.

What audit-trail completeness looks like for an AI-assisted system: every irreversible action has (a) an audit row, (b) the prompt + model + temperature, (c) the human-confirmation event if HITL gated it, (d) the AI-assist annotation if a human added downstream code. Anything less is a gap — name it as such.

---

## Back half — Wed-prep concept load

## 6. Wed-prep — Eval report metrics and reader structure (20 min)

W6 Wed's eval report (`eval/REPORT.md`) is the **quantitative half** of your handoff package — the runbook + ADR catalog are qualitative; the eval report is numbers. This combined briefing covers **what to measure** (the three layers of metrics) and **how to lay it out** (the 5/30 reader structure).

### Source signals — what to measure

Three signals you've been collecting since W2:

**W2 RAG eval signals** — RAGAS metrics over your FAR + DFARS corpus (the ~3,500-page scope locked at D-060 #1):
- **Faithfulness** — does the answer stay grounded in retrieved chunks? (target ≥ 0.85)
- **Answer relevancy** — does the answer address the question? (target ≥ 0.80)
- **Context precision** — are the retrieved chunks the *right* ones? (target ≥ 0.75)
- **Context recall** — did we retrieve everything we needed? (target ≥ 0.70)

[RAGAS metrics reference, Ragas docs retrieved 2026-05-23 via /web-research, https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/overview/]

**W3 agentic flow eval signals** — for both the intake-triage flow AND the evaluator→consensus→SSA handoff (per D-060 #2):
- Intake-triage routing accuracy (did the proposal land at the right TEP member? — count vs ground-truth assignments).
- Evaluator→consensus agreement rate (after HITL #5 LangGraph interrupt at "consensus complete").
- SSDD-draft HITL approval rate (how often does the SSA actually approve the AI draft vs revise vs reject?).

**W5 AIOps signals** — Datadog AI dashboards (per D-060 #4):
- p95 + p99 latency per AI endpoint (`/draft-solicitation`, `/draft-amendment`, `/answer-qa`, `/eval/ssdd-draft`, `/rag/clause-search`, `/agent/intake-triage`).
- Bedrock token + cost per request, per tenant (`agency_id`).
- Item 2 audit-log race detection rate (W5 Wed's AIOps signal — how often does the detector catch a missing audit row?).
- Item 3 circuit-breaker activations (post-W4-Fri-Workflow-4-incident).

### Reader structure — how to lay it out

The eval report has two readers per D-060:
- **Karsun manager** (5-min skim) — wants the headline numbers + the trend direction + 1-2 known failure modes.
- **OIG-style auditor** (30-min depth) — wants the methodology, the ground-truth dataset construction, the per-tenant breakdowns, the unresolved questions.

Structure that works for both:

```
eval/REPORT.md
├─ §1 Executive summary (300 words, Karsun-manager facing — headline numbers + trend + 2 known failure modes)
├─ §2 Methodology (RAGAS construction; ground-truth eval set provenance; eval cadence; CI integration)
├─ §3 Quantitative results (faithfulness + answer-relevancy + context-precision over time; per-tenant breakdown)
├─ §4 Agentic flow results (intake-triage routing accuracy; evaluator→consensus agreement; SSDD-draft HITL stats)
├─ §5 Known failure modes (named, with reproducer + remediation status — `Finding` IDs from /admin/findings)
├─ §6 Unresolved questions for Phase 3 (the "what we'd test next if we had time" list)
└─ §7 Appendix (raw RAGAS output, per-PR eval diff, Datadog AI dashboard links)
```

§1 is what gets read at the Showcase. §2-§7 is what gets read during the OIG-Auditor Q&A. This is the **same 5-min/20-min nesting rule** as the three-levels rule (§2 above) — §1 is the 5-minute version; §2-§7 is the 20-minute version. They should not contradict.

## 7. Wed-prep — FedRAMP attestation language (10 min)

Your security attestation (`docs/security-attestation.md`) maps your controls to **FedRAMP Moderate Rev 5** + **OWASP LLM Top 10 (2025 v2.0)**.

[FedRAMP Rev 5 Authorization Baselines, retrieved 2026-05-23 via /web-research, https://www.fedramp.gov/rev5/baselines/] — Moderate is the right tier for the kind of work acquire-gov represents (federal data, not classified). Rev 5 is the current revision as of 2026-05-23 (federal-regulatory 6mo window holds at 2026-05-26).

[OWASP LLM Top 10 2025 v2.0, retrieved 2026-05-23 via /web-research, https://genai.owasp.org/llm-top-10/] — the 2025 list reordered + reframed some categories from the 2023 list; use the 2025 v2.0 list, not earlier (3mo hot-tech window holds at 2026-05-26; `research/owasp-llm-top-10-20260522.md` confirms no canonical 2026 revision).

Three buckets for attestation language:

- **Controls met** — "Implemented and verified. See `<artifact-path>` for evidence."
- **Controls met with gaps** — "Implemented with named gap. Gap: `<one-sentence>`. Tracked as `Finding/F-2026-W6-NNN`."
- **Controls not met (blocker for next FAR-clause-eligible deployment)** — "Not implemented. Blocker for FedRAMP authorization at Moderate. Remediation owner: `<role>`. ETA: `<date>`."

**Honesty rule:** never claim "controls met" without a linked artifact. Karsun-manager and OIG-Auditor both probe this — they'd rather see 5 honest "controls met with gap" entries than 50 "controls met" claims with no evidence. The W6 Wed dry-run will surface this if you got it wrong. This is the §3 tradeoff-honesty rule applied to security controls.

For each acquire-gov debt item (1-12) you actually closed in W4-W5: include it as "controls met" with the PR link as evidence. For items still open (Item 12 GHA lint is the canonical example — it's intentionally left as Phase 3 work): include it as "controls met with gap" with the Phase 3 `Finding` ID.

## 8. Wed-prep — HITL authority boundaries (Wed's capstone framing) (10 min)

Wednesday afternoon you author **`docs/hitl-authority-boundaries.md`** — the W6 capstone deliverable.

It synthesises **all 7 HITL programme touchpoints** (D-043 + D-044):

| # | Touchpoint | Week | Shape |
|---|-----------|------|-------|
| 1 | LLM Essentials | W1 Fri | Hallucination-flagging on `/draft-solicitation` — CO confirms before publish |
| 2 | RAG fallback | W2 Thu | When retrieval below threshold → escalate to CO, don't ship a guess |
| 3 | Plan-Day ADR | W3 Mon | Pair commits interrupt-node ADR at week start |
| 4 | Multi-agent handoffs | W3 Wed | Supervisor agent requires human approval before delegating irreversible task |
| 5 | LangGraph deep-dive | W3 Thu | Soft vs hard interrupts; audit trail; resuming graphs after approval (the technical anchor) |
| 6 | OWASP LLM06 | W4 Wed | Excessive Agency defence — narrow tool scopes, HITL on irreversible actions |
| 7 | AIOps auto-remediation | W5 Wed | What can AI-SRE do alone vs what must escalate (the auto-fix-the-audit-gap decision) |

The synthesis doc is a **table per AI endpoint × authority level**. For each of the 8 ai-orchestrator endpoints (the 6 from inventory + the 2 LangGraph evaluator→consensus→SSA handoff endpoints from W3) + your pair-project AI endpoints, name the authority level: `full-auto` / `propose-and-await-approval` / `escalate-only` / `never-AI`.

A FedRAMP-aligned attestation reader (auditor + Karsun manager) reads this doc and gets "what does this system let AI do without a human, and what doesn't it?" in 5 minutes. That clarity is the entire point — and it's the controls-language framing from §1 applied across the whole AI surface.

Template: `weeks/W06/hitl-authority-boundaries.md` (canonical) — you instantiate one per pair-project Wed afternoon.

---

## What you'll do W6 Tue

- 09:00–12:00 — Runbook authoring workshop (war-room) + stakeholder calibration mini-block. (§§1-5 inform this directly.)
- 13:00–17:00 — ADR catalog curation + **stakeholder rotation** (CIO / OIG / CO — same artifact, three audiences, three lengths). (§§1-5 are the rehearsal frame.)
- 17:00 — Read Wed pre-session.

## Sources

- RAGAS metrics, Ragas docs, retrieved 2026-05-23 via /web-research, <https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/overview/>
- FedRAMP Rev 5 Authorization Baselines, retrieved 2026-05-23 via /web-research, <https://www.fedramp.gov/rev5/baselines/>
- OWASP LLM Top 10 2025 v2.0, retrieved 2026-05-23 via /web-research, <https://genai.owasp.org/llm-top-10/>
- Internal: `research/owasp-llm-top-10-20260522.md` — confirms 2025 v2.0 still canonical as of 2026-05-26 (no OWASP-published 2026 revision)
- Internal: `research/federal-oig-audit-response-20260525.md` — OMB Circular A-50 audit-followup framework cited in §5 audit-trail walkthrough framing

Last verified: 2026-05-26
