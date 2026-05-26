---
week: W04
title: "AI-Native SDLC + AI Security + Brownfield Modernization (Phase 2 begins)"
phase: "Phase 2 — Modernization driven by Phase-1 discoveries"
gate: "Mid-Sprint Surprise Fri (per D-049/D-060) — not a programme gate, but a high-stakes live exercise"
density_target: "5–8 topics/day (per CLAUDE.md AIOps/Capstone budget — W4 sits at the upper end of the foundation-vs-specialisation boundary; we hold 5–8 to leave room for the Fri incident)"
last_verified: 2026-05-23
hitl_touchpoint: "#6 of 7 — Wed (AI Security / OWASP LLM06 Excessive Agency framing of HITL authority boundaries)"
mid_sprint_surprise: "Fri = acquire-gov prod incident, Workflow 4 + Item 3 load incident (per D-060). NOT Item 1 JWT-skip (that's the W4 Wed OWASP LLM07/LLM08 demo)."
modernization_anchor: "OpenRewrite hop SB 2.7.18 + Java 11 → SB 3.0 + Java 17, javax → jakarta (per D-054 + D-056 single-branch design)"
phase_2_begins: true
---

# W04 PLAN — Mon–Fri at a glance

> Master plan for Week 4. **Phase 2 (Modernization) begins.** §0 plan retrospective on the W3 Phase 1 Defense findings drives W4 modernization scope. Three themes interlock this week: **spec-driven dev as living discipline** (cohort has now practised the §0 retro 3× — W2/W3/W4 Mon — and this week's Tue workshop deepens it), **AI Security** (OWASP LLM Top 10:2025), and **modernization execution** (the OpenRewrite hop per D-054 lands Thursday on the cohort's own `acquire-gov` legacy stack per D-056). Friday's Mid-Sprint Surprise tests all three under load.

## Mon Plan Day — Phase 2 framing + W4 modernization ADR + AI Security threat model *(7 topics)*

*First Plan Day of Phase 2. §0 retro covers W3 Phase 1 Defense findings — Phase 1 discoveries drive W4's modernization scope. Plan Day is the moment "what did we discover?" turns into "what are we modernizing?"*

**Morning (war-room, Plan-Day mode — `war-room/D1.md`)** — §0 plan retrospective on W3 plan-spec: which Phase 1 modernization needs were named, what the pair's defense surfaced, what the cohort owes the legacy stack now. Scenario Design Planning brief released (W4 cadence per D-029 + CLAUDE.md — graded artifact, EOD due).

**Afternoon (practical)** — Pair authors `planning/W04/` six-artifact set. Two ADRs land today as draft commits: (a) **W4 Modernization Scope ADR** (which 2–3 of the 12 debt items + per-pair-unique items the pair will execute this week — must include the OpenRewrite hop or explain why deferred); (b) **AI Security Threat Model ADR** for Wed — pair walks the 8 ai-orchestrator endpoints (per `feature-inventory-target.md`) against OWASP LLM01–LLM10:2025 and names which apply where. No code lands Monday.

**Conceptual (`pre-session/1-Monday/1-DailyTopicOverview.md`)** — Phase 2 framing + spec-driven dev as living discipline + AI Security pre-read.

**Diagnostic recast (EOD):** *"name the single Phase 1 finding that most narrows the W4 modernization scope — and the single discovery your pair did NOT yet make."* Feeds Tue spec-driven workshop seeding.

## Tue — Spec-driven Dev Workshop + Brownfield Modernization Planning *(6 topics)*

*Cohort has practised the §0 retro 3× by now (W2/W3/W4 Mon). Today's workshop turns spec-driven dev from a Plan-Day-only artifact into a daily discipline. Cohort discovers their own PRs aren't actually linted (debt item 12) — the meta-joke that drives the whole day.*

**Morning (war-room — `war-room/D2.md`)** — Spec-driven dev workshop: instructor frames the gap between "we wrote a spec on Monday" and "every PR this week ships against that spec." Cohort opens their own merged W2–W3 PRs in the training-project + pair-project repos, then opens the GitHub Actions runs, and discovers the lint job is `if: false`-disabled (debt item 12). Adversarial fact: their PRs were never actually linted. The OIG-Findings-Tracker meta-joke (per inventory line 387) becomes real — first PR of the week opens a finding against the repo's own CI.

**Afternoon (practical)** — API modernization pattern work: fix debt item 8 (frontend hardcodes service URL bypassing the gateway) as the warm-up. Then pair plans the Thu OpenRewrite hop — pre-flight checks (Java 17 toolchain installed? `dependency-check` baseline? rescue branch named?). Pair-project parallel work: each pair plans the same hop against their pair-project repo's slice.

**Conceptual (`pre-session/2-Tuesday/1-DailyTopicOverview.md`)** — Spec-driven dev deepening + OpenRewrite primer.

## Wed AI Security Day — OWASP LLM Top 10 + HITL #6 + dedicated research slot *(8 topics, at upper cap)*

*Dedicated `/web-research` slot per D-040 + AI Security depth day. Density at upper cap because the OWASP coverage is load-bearing for the Fri surprise + W5 AIOps governance. **HITL #6 lands here** — framed as OWASP LLM06 Excessive Agency = authority-boundary design.*

**Morning (war-room — `war-room/D3.md`)** — AI Security framing. Instructor speaks as Karsun Security Engineering Lead: a recent OIG audit at a sibling agency found that an AI drafter accepted a prompt-injected vendor question and emitted FAR clauses that didn't exist into a published Q&A response. Pair's task: walk the 4 stored-content surfaces (`POST /api/solicitations` description, `POST /api/solicitations/{id}/qa`, `POST /api/awards/{id}/debrief-request`, `POST /api/contracts/{id}/cpars/{cparId}/rebuttal`) and demonstrate how each feeds `POST /draft-amendment` or `POST /answer-qa` as an LLM01 indirect prompt-injection vector.

**Afternoon (practical, AI Security + dedicated /web-research)** — Three exercises in sequence: (1) **Item 1 JWT-skip exploit demo** (OWASP LLM07 + LLM08 — pair builds a working exploit that mints an unsigned JWT, hits `GET /api/public/opportunities`, escalates via downstream-trust convention; the exploit lives in `weeks/W04/wed-jwt-skip-exploit.md` as instructor-owned cohort takeaway, NOT committed to acquire-gov main); (2) **Item 9 prompt-injection-via-stored-content fix** — Pydantic input validators + output sanitisation on the 4 surfaces; (3) **HITL #6 authority-boundary table** — for each of the 8 ai-orchestrator endpoints, pair names: who can call it, what authority the AI exercises, what gate requires a human, what the rollback looks like. This is the **OWASP LLM06 Excessive Agency** translation into FedRAMP-language. **Dedicated research slot** doubles up — pairs research the Wed scenario-alternatives prompts (W04-SA-1/2/3) in the same block.

**Conceptual (`pre-session/3-Wednesday/1-DailyTopicOverview.md`)** — OWASP LLM Top 10:2025 deep-dive + HITL authority-boundary framing.

## Thu — Modernization Execution Day — OpenRewrite SB 2.7→3.0 + J11→17 *(7 topics)*

*The cohort's biggest execution day of W4. Per D-054 + D-056, `acquire-gov` main IS the legacy stack — cohort modernizes it forward. The 5-checkpoint shape (`legacy-baseline → stage-J17 → stage-3.0-rewrite → stage-3.0 → final-expected-PR`) is the instructor's safety net; cohort works the J17 + 3.0-rewrite + 3.0 stages. Two future-hop ADRs land too (SB 3.5 + SB 4.0) per D-054.*

**Morning (war-room — `war-room/D4.md`)** — Instructor speaks as Karsun delivery lead: the agency CIO has asked for a Spring Boot 3.x posture statement by Friday's exec brief. Pair's task: execute the OpenRewrite hop on `solicitation-service` + `evaluation-service`, document expected-failures (from instructor's pre-staged `known-failures.md` per stage), and draft the SB 3.5 + SB 4.0 future-hop ADRs as evidence-backed (≥1 OpenRewrite `dryRun` per future hop, ≥1 representative-module compile attempt, per D-054 Pass 3 floor).

**Afternoon (practical)** — Execute the hop. Each pair owns one service (third pair owns the cross-cutting `javax.*` → `jakarta.*` work in the shared modules). Rescue branch named per stage. `known-failures.md` updated as failures land. EOD: each pair opens a PR **from their per-pair feature branch into `main`** (per D-056 — `acquire-gov` main IS the legacy stack being modernized forward; the `v0.1-legacy-baseline` git tag preserves pre-modernization state for rollback) and tags it for codex adversarial review (Full strictness per D-034). Future-hop ADRs (SB 3.5, SB 4.0) committed to `planning/W04/adrs/`.

**Conceptual (`pre-session/4-Thursday/1-DailyTopicOverview.md`)** — OpenRewrite recipe authoring + Spring Boot 3.x migration breaking-change catalog + ADR evidence floor.

## Fri MID-SPRINT SURPRISE — Workflow 4 + Item 3 load incident *(6 topics, incident-driven)*

*Per D-049 + D-060: Mid-Sprint Surprise replaces the standard war-room. **Workflow 4 (evaluation → consensus → SSDD-sign → award) under TEP-week load** drops at 09:00. No prior warning. No code-fix afternoon — the afternoon is **detect → respond → RCA → fix-or-stage-for-W5**. Live Defense is on the amended plan-spec, NOT scenario ADRs.*

**Morning (incident — `war-room/Fri-mid-sprint-surprise.md`)** — At 09:00 the instructor turns to the cohort, opens `acquire-gov`'s Grafana dashboard (instructor-staged + pre-seeded with the incident signal), and reads the Karsun delivery-lead's escalation: *"GSA's source-selection authority for the cloud-modernization solicitation is signing the SSDD right now. The platform just stopped responding. The audit log has gaps. The OIG observer in the next office is watching. What do we do?"* Cohort works the incident. Full detail in `war-room/Fri-mid-sprint-surprise.md`.

**Afternoon (practical — amended plan-spec + Live Defense)** — Pair amends Mon's `planning/W04/` ADRs in-place: which decisions held, which broke, what gets fixed today vs staged for W5 AIOps work (Resilience4j circuit + audit-log race detector are the canonical hand-off). Live Defense at 15:30 — instructor probes the amended plan; rubric in `assessments/W04-live-defense.md`. EOD: post-incident retro (`retros/Fri-post-incident-retro.md`) — what changed in the pair's mental model of "production"?

**Conceptual (`pre-session/5-Friday/1-DailyTopicOverview.md`)** — Pre-reading drop for W5 Mon Plan Day (AIOps anchor week framing — Resilience4j + OpenTelemetry + W3C traceparent).

## Special notes for W4 (Cohort #1)

- **Phase 2 begins Monday.** W3 Fri's Phase 1 Defense + Mid-Program Retro happened last Friday — Mon §0 retro names which Phase 1 discoveries drive W4 modernization.
- **Codex Adversarial Review at Full strictness per D-034.** W1 was Light, W2 Ramping, W3 Near-full, W4 Full. P0 + P1 findings block merge.
- **Modernization branch protection.** Pair PRs go **from per-pair feature branches into `main`** (per D-056 — main IS the legacy stack; no sibling `legacy-baseline` branch exists; the `v0.1-legacy-baseline` git tag preserves pre-modernization state for rollback). The `legacy-baseline → stage-J17 → stage-3.0-rewrite → stage-3.0 → final-expected-PR` 5-checkpoint shape in §Thu is the instructor's *safety-net checkpoint naming convention* (private to instructor), NOT a set of cohort-target branches. Instructor's `final-expected-PR` branch stays private until W4 Fri retro to avoid solution leakage (per D-054).
- **The Mid-Sprint Surprise is not scheduled.** The cohort knows W4 Fri is "the surprise day" but does NOT know the shape. Do not pre-leak Workflow 4 / Item 3. Instructor pre-stages the incident signal in `acquire-gov`'s observability surface Thu evening (feature-flag-gated chaos injection per D-049).
- **HITL #6 is NOT the Fri surprise's HITL.** HITL #6 is Wed's authority-boundary table (LLM06 Excessive Agency framing). The Fri surprise exercises the W4 HITL muscle implicitly but does not introduce a new HITL touchpoint.
- **AWS posture:** Bedrock InvokeModel authorized per D-060 Bedrock-W2-onward exception. AWS managed services (Knowledge Bases, Agents-for-Bedrock, OpenSearch Managed) STILL deferred to W5 per D-050. No Bedrock Guardrails this week — flagged as managed-service in W04-SA-2.
- **Density risk Wed:** at the 8-topic upper cap. If the JWT-skip exploit demo runs over, defer the HITL authority-boundary-table to Thu morning's first 30 minutes (acceptable carry-over).
- **Cohort #1 calendar:** No holidays in W4 (Mon 15 Jun – Fri 19 Jun 2026). Full 5-day week.

## Assets in this folder

- `PLAN.md` — this file.
- `README.md` — week framing + HITL #6 + Mid-Sprint Surprise + D-049 + D-054 + D-060 context.
- `pre-session/<N-DayName>/1-DailyTopicOverview.md` — pre-session reading (5 days, Mon–Fri; **D1 = Mon since no holiday compression in W4**).
- `war-room/D1.md..D4.md` — Mon–Thu morning war-rooms.
- `war-room/Fri-mid-sprint-surprise.md` — instructor-facing Mid-Sprint Surprise script + incident shape + scoring rubric.
- `scenarios/` — 3 scenario-alternatives prompts (W04-SA-1 circuit-breaker, W04-SA-2 prompt-injection defenses, W04-SA-3 modernization sequencing).
- `assessments/` — Scenario Design Planning (Mon) + MCQ (Fri) + Live Defense (Fri PM, scoped to amended plan-spec) + Adversarial PR Rubric.
- `codex-reviews/` — modernization PR adversarial review template.
- `retros/` — Fri post-incident retro + EOD prompts.
