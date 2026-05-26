---
week: W06
day: Mon
title: "Pre-session — Phase 1+2 arc retrospective frame + deliverable taxonomy"
audience: All cohort members
time_on_task_minutes: 50
last_verified: 2026-05-26
---

# W6 Mon Pre-Session — Phase 1+2 Arc Retrospective Frame + Deliverable Taxonomy

> Read before Mon morning. ~50 min. Mon is Plan Day **and** Checkpoint 4 — dense day. The §0 retro this Mon is the **biggest** of the programme: it spans Phase 1 + Phase 2 end-to-end (W1-W5), not just last week. Today's pre-session covers eight topics — the densest reading of the programme, and the last pre-session reading you'll do as a cohort (Tue–Wed have pre-sessions; Thu is the gate, no pre-session).

## 1. Why this §0 retro is different (5 min)

Every Plan Day from W2 Mon onward opened with **§0 Plan retrospective on last week's spec** (per D-036). W6 Mon's §0 retro is the **only one** that retros the full programme arc — Phase 1 (W1 Thu–W3 Fri: AI Adoption into brownfield) plus Phase 2 (W4–W5: Modernization driven by Phase-1 discoveries) per D-038 + D-039.

Five things the retro names (the pair brings rough notes; instructor structures the conversation):

1. **What we adopted in Phase 1** — Bedrock + LLM Essentials (W1 Thu-Fri), RAG over FAR/DFARS via Atlas Vector Search (W2 — corpus scope locked at FAR+DFARS ~3,500 pages per D-060 #1), agentic workflows + LangGraph HITL (W3 — both intake-triage AND evaluator→consensus→SSA per D-060 #2).
2. **What we discovered at W3 Fri Phase 1 gate** — the modernization targets that Phase 2 inherited. Most pairs surfaced acquire-gov debt items 2 (audit-log race), 5 (legacy LangChain), 10 (multi-tenant boundary) as gating Phase 2.
3. **What we modernised in Phase 2** — AI Security + OpenRewrite SB 2.7→3.0 + J11→17 (W4 Thu hop per D-054), AIOps with Datadog AI hands-on (W5 per D-060 #4), final adversarial PR review (W5 Fri).
4. **What didn't survive contact with reality** — the moments the plan-spec was wrong. Most pairs hit at least one: a W2 ADR (e.g., chunk size, embedding model) revised after W2 Thu RAG eval results; a W3 LangGraph interrupt-node placement changed after multi-agent debugging; a W4 modernization stage that took 2 days instead of 1.
5. **What we'd tell ourselves at W1 Wed when we picked the pair-project aspect** — the Phase 3 commitment seed.

## 2. The deliverability package shape — the deliverable taxonomy (10 min)

By Thu 2 Jul afternoon's Client Showcase, each pair owes a **handoff package** in their pair-project repo. **This eight-artifact list IS the deliverable taxonomy** — the named, bounded set of things the package contains. Mon afternoon's deliverable-taxonomy block scopes them:

| Artifact | Path | Audience | Drafts Tue | Ships Wed |
|----------|------|----------|------------|-----------|
| Operational runbook | `docs/runbook.md` | next on-call FDE | Tue AM | Wed PM |
| ADR catalog | `docs/adrs/INDEX.md` | future FDE inheriting the codebase | Tue PM | Wed AM |
| Eval report | `eval/REPORT.md` | Karsun manager (5 min skim) + OIG-style auditor (30 min depth) | — | Wed AM |
| Known-weaknesses inventory | `docs/known-weaknesses.md` | next FDE + Karsun manager | Tue PM (sketch) | Wed PM (final) |
| Handoff README | `docs/handoff.md` | the *next FDE* picking up the repo | — | Wed PM |
| Security attestation | `docs/security-attestation.md` | Karsun manager + (eventual) sponsor 3PAO | — | Wed PM |
| HITL authority boundaries | `docs/hitl-authority-boundaries.md` | next FDE + future cohort | — | **Wed (capstone)** |
| Threat model + cost analysis | `docs/threat-model.md` + `docs/cost-analysis.md` | Karsun manager | Tue PM | Wed AM |

Mon afternoon you ship **skeletons + outlines** of each — content lands Tue-Wed. The taxonomy is closed: there is no ninth artifact. If you find yourself authoring a "documentation site" or a "demo video" or a "deployment automation script," you are scope-creeping (see §3).

## 3. Scope Discipline — the skeletons-only rule (5 min)

Mon afternoon is the *single biggest* trap of the W6 arc. The deliverable taxonomy is eight artifacts; the temptation is to draft real prose on Mon, because the gate is Thu and Mon feels like the moment to "get ahead." **Resist.** Mon afternoon is skeletons only.

Three rules that operationalise scope discipline this week:

1. **Skeletons-only on Mon.** Table of contents + section headers + 1-line stubs. NOT first draft. NOT prose. If your `docs/runbook.md` is 4 pages by Mon EOD, you over-invested and your Tue is now over-committed — Tue morning's runbook authoring workshop assumes you're starting from a skeleton, not refactoring a draft. The instructor walkthrough explicitly hard-redirects pairs that bring prose to Mon.
2. **The eight-artifact taxonomy is a scope bound.** No ninth artifact. No "let me build a docs-system from scratch" detour. No demo-video production. The bounded set is what the Karsun-manager panel will read Thu; the bounded set is what Phase 3 inherits. Anything else is gold-plating that costs you Wed dry-run prep time.
3. **Trade-off-naming honesty IS scope discipline.** When you find a known weakness in your pair-project that you cannot fix this week, the disciplined move is to *name it* in `docs/known-weaknesses.md` as a `Finding` entity (see §4) — not to extend scope by fixing it. The W3 Fri Phase-1 retro taught this distinction; W6 is where it pays off. "We picked latency over completeness on the RAG fallback path because the FAR 15.206 acknowledgement window is 24h, not real-time" is a declared trade-off. Silently fixing the RAG fallback in W6 — when it should be a known-weakness — is scope creep dressed as engineering.

The cohort-facing war-room/D1.md preamble enforces this same rule. The reason it appears in both pre-session and war-room: this is the failure mode that historically eats W6 across cohorts. Mon afternoon discipline buys Tue–Wed sanity.

## 4. OIG-Findings-Tracker-as-meta-runbook framing (10 min)

The acquire-gov surface you built in W4 Wed AI Security + W5 Wed AIOps governance — **`/admin/findings`** (the OIG Findings Tracker, evaluation-service) — is the **meta-runbook** for W6.

Each finding entity has:
- `opened_by` (role + name)
- `evidence_requests[]` (linked artifact paths)
- `remediation_status` (`open` / `evidence-pending` / `remediation-in-progress` / `closed-remediated` / `closed-accepted-risk`)
- `due_at`

Your W6 runbook + ADR catalog + eval report + known-weaknesses inventory should be **modeled as `Finding` entities** opened against your pair-project repo. This is not metaphor — it's the same data model. The pattern mirrors how **GSA OIG audit findings actually close**: open finding → request evidence → remediate or accept risk → close with rationale + artifact link.

Reference for OIG audit-closure patterns (read for the *form*, not the substance): [GSA OIG Audit A210064 Contract Administration of Federal Acquisition Service Information Technology Contracts, retrieved 2026-05-23 via /web-research, https://www.gsaig.gov/sites/default/files/audit-reports/A210064_3.pdf] — note how each finding is paragraphed with `Condition` / `Cause` / `Effect` / `Recommendation` / `Management Response` / `Auditor's Reply`. Your runbook entries follow the same shape.

**Why this matters for Thu:** the Karsun manager + OIG-Auditor probe at Thu's Final Defense will ask "how do you track open known-weaknesses past handoff?" The honest answer is: as `Finding` entities in the same tracker the codebase exposes. You built the tracker; use it for your own work.

## 5. AI-Assist Project Audit — the per-PR honest record (8 min)

The Checkpoint 4 audit walkthrough probes your **AI-Assist Project Audit** — the per-PR record of *how AI was used to author the work*, kept honestly enough that you can defend it five years from now to a federal auditor (per `pipeline/DECISIONS.md` D-035). This topic gets its own section because the audit-trail discipline is half of what Thu's OIG-Auditor role probes, and it's the single most undertaught surface of the programme.

**What the per-PR annotation captures.** Each PR description carries an `assistance_mode` annotation. The three canonical values:

| Annotation | Meaning | When to use |
|------------|---------|-------------|
| `pair-led` | Pair authored; AI used as accelerant (autocomplete, refactor suggestions, doc lookup) | Default for most PRs. The pair owns the design, AI shaved minutes off the typing |
| `ai-led` | AI authored the first draft; pair reviewed + verified + revised | Use sparingly + honestly. If the prompt was "write me a Spring Boot Resilience4j config for the Bedrock call surface" and you accepted >50% of the output, it's `ai-led`. Defend it |
| `hand-authored` | No AI involvement | Use when you turned AI off deliberately (e.g., the W4 codex-adversarial-review response, where you wanted to think through the critique without AI mediation) |

**Why this matters for federal-acq audit-trail expectations.** Federal-acq engagements increasingly carry FAR/DFARS-clause-anchored audit-trail requirements; the OIG-Auditor lens reads your AI-Assist log as a "could you defend this work product in five years?" signal. Three failure modes the Mon audit walkthrough surfaces:

1. **Performative auditing.** PR carries `assistance_mode: pair-led` because that's the default, but the actual authoring was ai-led. The auditor catches this by asking "Pick one PR where you marked `pair-led` and walk me through the design decisions." If you cannot, the annotation was performative — flag.
2. **Annotation drift.** Early PRs (W1–W2) carry honest annotations; by W4–W5 the annotations are stale because nobody enforced. The auditor probes this with "Show me your last five PRs and the trajectory."
3. **Cross-link gaps.** Your AI-Assist log references W4 OWASP findings? Your W5 AIOps governance decisions? If not, the audit-trail is missing the chain that ties prompt → output → adversarial review → revision.

**The Karsun-manager lens (the Thu PM showcase audience).** Karsun managers read the AI-Assist audit-trail as a staffing signal: *can I hand this codebase to a new FDE next month and have them inherit not just the code, but the honest history of how it was built?* A clean audit-trail is a staffing asset. A performative one is a staffing liability — the new FDE inherits invisible debt.

**What to bring to the audit walkthrough (Mon 10:30):**
- Your last five pair-project PRs open in browser tabs (with `assistance_mode` annotations visible).
- One PR you'd flag as `ai-led` and would defend confidently.
- One PR where the annotation is wrong (be honest — surfacing this proactively is a growth signal, not a flag).

## 6. Checkpoint 4 prep — what's on the exam (10 min)

Checkpoint 4 fires Mon 09:00–10:30 (90-min exam, in-person AM per D-061; was originally early afternoon, moved to AM in the Cohort #1 walkthrough per memory `checkpoint-time-budget-2hr.md`). Audit walkthrough runs 10:30–12:00 (30 min per learner, 2 auditors × 3 trainees each per memory `programme-calendar-cohort1.md`). **Calibration: full (gate-prep posture).**

| Tier | Exam shape | Anchored on |
|------|-----------|-------------|
| Senior FDE | Deployment-readiness write-up (FedRAMP-aligned checklist) + 1 honest known-weakness declaration | W5 AIOps + governance content; AWS-Bedrock token/cost accounting; Datadog AI dashboards |
| Entry FDE (applied recognition per memory `entry-tier-applied-recognition.md`) | Runbook-completion task on the Item 2 audit-log race scenario (given a skeleton + scenario, fill it correctly) | W5 AIOps signals + W4 OWASP LLM Top 10 + the OIG Findings Tracker data model |

The audit walkthrough surface is the AI-Assist Project Audit (see §5). The exam itself is technical-recognition; the audit walkthrough is honest-record defense. Both feed Thu's per-individual gate calibration.

Bring to the exam:
- Your W5 plan-spec.
- Your pair-project repo `docs/known-weaknesses.md` current state.
- Your AI-Assist audit log (the per-PR record of `assistance_mode`).

## 7. Stakeholder vocabulary calibration (5 min)

Tue morning's war-room walks the four Karsun audiences you'll defend to over W6:

| Audience | What they care about | What they DON'T |
|----------|---------------------|------------------|
| **Karsun manager** (Thu PM showcase audience per D-060) | Can they staff this onto a real federal engagement next quarter? Does the handoff package let a new FDE come up to speed in a week? Is the cost shape defensible? | Pair-stack-deep config (e.g., Resilience4j circuit-breaker tuning); LangGraph node-ID semantics |
| **Agency CIO** (Thu AM gate panel role) | FedRAMP posture, ATO timeline, vendor lock-in, total cost of ownership | Code-level tradeoffs |
| **OIG Auditor** (Thu AM gate panel role) | Reproducibility, audit trail completeness, who-decided-what-and-when, evidence-of-decision | The aesthetic of the demo |
| **Contracting Officer** (Tue rehearsal role) | Does the AI output respect FAR/DFARS clause boundaries? Is the HITL gate honest? Will I get audited on this? | The infra layer |

**D-060 calibration:** The W6 Thu **showcase** audience is **Karsun managers** specifically. The gate Q&A (Agency CIO + OIG Auditor roles) is still played by instructor as a defense probe, but the showcase rubric (`assessments/client-showcase-rubric.md`) calibrates to manager-tier knowledge. Cohort #2 graduates to mock-client panel.

## 8. The retro framing for Thu's Cohort Retro (5 min)

Start thinking now about three questions for Thu EOD's Cohort Retro (the programme's last 60 minutes):

1. *"What's one thing I learned that surprised me?"*
2. *"What would I tell myself at W1 Tue if I could time-travel?"*
3. *"What's the ONE improvement I'd ship in my first Phase 3 week?"*

Phase 3 framing: 6-month on-the-job development continues after the programme. Your Phase 3 commitment card (Thu EOD artifact) names the one improvement you'd ship in your first Phase 3 week — concrete, ticketed, deliverable.

## What you'll do W6 Mon

- 09:00–10:30 — **Checkpoint 4 exam** (90 min, in-person; both tiers simultaneous; per D-061 Cohort #1 schedule).
- 10:30–12:00 — **Audit walkthrough** (30 min per learner, 2 auditors × 3 trainees in rotation).
- 12:00–13:00 — Lunch.
- 13:00–14:00 — **Final gate-boundary 1:1s** (20 min × 3 pairs; while you wait, work skeleton outlines).
- 14:00–15:00 — **§0 Phase 1+2 arc retrospective** (all-cohort, instructor authors retro notes live).
- 15:00–16:45 — **Deliverable-taxonomy skeleton push** — ship 8 artifact skeletons in `docs/handoff-package-skeleton` PR.
- 16:45–17:00 — EOD wrap + Tue handoff brief.
- 17:00 — Read Tue pre-session.

## Sources

- GSA OIG Audit A210064 Contract Administration of Federal Acquisition Service Information Technology Contracts. GSA Office of Inspector General. Retrieved 2026-05-23 via /web-research. https://www.gsaig.gov/sites/default/files/audit-reports/A210064_3.pdf
- D-035 (AI-Assist Project Audit + per-individual Deployment Gate), D-036 (§0 Plan retrospective), D-038/D-039 (Phase 1 / Phase 2 arc), D-046 (research protocol), D-054 (W4 Thu modernization hop), D-060 (Karsun-manager showcase audience + corpus scope), D-061 (Checkpoint 4 time budget). Pipeline-internal decisions: `pipeline/DECISIONS.md`.
- Memory references: `checkpoint-time-budget-2hr.md`, `entry-tier-applied-recognition.md`, `programme-calendar-cohort1.md`. Pipeline-internal.

Last verified: 2026-05-26
