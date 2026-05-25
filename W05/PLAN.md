---
week: W05
title: "AIOps Anchor Week — OpenTelemetry on AI + AI-SRE + Governance + Final Adversarial Review PR"
phase: "Phase 2 — Modernization (operationalisation week)"
gate: "Final Adversarial Review PR (Fri 26 Jun) — v2 replacement for v1 Exit Technical Exam"
density_target: "5–8 topics/day (Production density)"
last_verified: 2026-05-23
hitl_touchpoint: "7th of 7 — auto-remediation authority boundaries (Wed)"
hands_on_aiops_platform: "Datadog AI (Watchdog AI + LLM Observability) per D-060"
aiops_compare_set: ["Dynatrace Davis AI", "New Relic AI Monitoring", "Coralogix AI Observability"]
aws_managed_services_first_lit: "Bedrock Knowledge Bases, Agents-for-Bedrock, OpenSearch Managed (per D-050 + D-060)"
decisions_load_bearing: ["D-038 single pair-project arc", "D-043 7 HITL touchpoints", "D-044 Phase 2 = W4–W6", "D-049 W4 surprise = acquire-gov prod incident (sets up W5 fix work)", "D-050 AWS managed services deferred to W5", "D-060 Datadog AI hands-on + Bedrock W2-onward exception"]
---

# W5 PLAN — AIOps Anchor Week at a glance

> Phase 2 operationalisation week. Pair Project now has real federal-modernization shape from W4. W5 lights up **observability + AIOps + governance + AWS managed services + final adversarial review** as the operational maturity layer. Every theme answers a question Phase 1 (or W4) surfaced — *"what would the AI-SRE have caught?"*, *"what does the OIG need in the binder?"*, *"what's the on-call surface?"*. **Final Adversarial Review PR Fri 26 Jun = v2's replacement for v1's Exit Technical Exam** (same security + failure-sim rigor; framed as a culminating PR review rather than a separate exam).

## Mon 22 Jun — Plan Day: AIOps integration + AWS managed-service onboarding plan *(7 topics)*

*§0 retro on W4 plan-spec lands the Mid-Sprint Surprise findings as the W5 fix backlog. Item 3 circuit breaker work is the canonical W5 fix.*

**Morning (war-room — `war-room/Mon.md`)** — Trainer 1:1 (gate-boundary cadence per `PIPELINE.md` §10). **§0 Plan retrospective on W4** — pair names what the Mid-Sprint Surprise revealed (per D-049: Workflow 4 evaluator-fan-out + Item 3 absent circuit-breaker + Item 2 audit-log race left SSDD-sign un-logged). Those discoveries drive W5's fix targets.

**Afternoon (practical)** — Plan-spec author session (Scenario Design Planning artifact graded EOD). Six artifacts due: requirements (AIOps integration), 6R map, ADRs (Datadog vs alternatives, AWS managed-service migration boundary, HITL #7 authority table), estimate, risk register, open questions. **AWS managed-service onboarding gate opens** — D-050 carve-out lifts, cohort's single Karsun-paid AWS account gets Bedrock Knowledge Bases, Agents-for-Bedrock, OpenSearch Managed turned on.

**Conceptual (`pre-session/Mon.md`)** — D-050 vs D-060 framing recap, AIOps platform-walkthrough overview, Datadog hands-on intent.

## Tue 23 Jun — OpenTelemetry on AI Workloads *(8 topics)*

**Morning (war-room — `war-room/Tue.md`)** — Audit Log Search drill: cohort tails a real `POST /draft-solicitation` request and tries to reconstruct its end-to-end path across Angular RUM → api-gateway → solicitation-service → ai-orchestrator → Bedrock → back. They fail because **Item 6** (inconsistent correlation IDs) breaks the join — exactly the W3C `traceparent` problem OTel solves.

**Afternoon (practical)** — Cohort installs the Datadog Agent into `acquire-gov` docker-compose, instruments the Python ai-orchestrator with `opentelemetry-instrumentation-fastapi`, the 3 Java services with the OTel Java auto-agent, and the Angular SPA with Datadog RUM. **Bedrock token + cost metrics fan-in** — every InvokeModel emits input/output token counts as OTel span attributes; Datadog dashboard rolls cost-per-endpoint × tenant. **Item 6 (correlation IDs) modernised end-to-end this afternoon.**

**Conceptual (`pre-session/Tue.md`)** — OTel for AI primer; W3C `traceparent`; span-attribute conventions for LLM (`gen_ai.*` semconv); cost-as-a-signal framing.

## Wed 24 Jun — AIOps Governance + HITL #7 + Dedicated `/web-research` Slot *(7 topics)*

> **HITL touchpoint #7 (last of 7 per D-043 + D-044).** Closes the HITL programme thread before W6 re-asserts it as a deliverable.

**Morning (war-room — `war-room/Wed.md`)** — **AI-SRE Pattern Walkthrough** (per `skills/aiops-curriculum/references/ai-sre-patterns.md`). Canonical case: **"auto-fix-the-audit-gap"** — Datadog Watchdog AI detects an Item 2 race-condition gap (write succeeded, audit row missing). Three candidate platform behaviours: (A) auto-replay the missing AuditEvent, (B) escalate to sys_admin with a structured incident, (C) page on-call human. **The cohort decides which authority the AI-SRE has** — and writes the policy ADR. This decision *is* HITL #7.

**Afternoon (practical, dedicated research slot per D-040)** — Scenario research day. Pairs work scenario W05-SA-1 (AIOps platform compare-set) + W05-SA-2 (auto-remediation authority) + W05-SA-3 (Bedrock managed-services migration cost-benefit). All three feed Thu's compare-matrix + Fri's adversarial-review PR.

**Conceptual (`pre-session/Wed.md`)** — OWASP LLM Top 10 continuation (LLM05, LLM09, LLM10 — emphasis on AIOps-discoverable risks); HITL #7 framing; D-040 research-day mechanics.

## Thu 25 Jun — AIOps Platform Comparison + AWS Managed-Service Onboarding *(8 topics)*

**Morning (war-room — `war-room/Thu.md`)** — Compare-matrix workshop. Datadog AI is hands-on (cohort has been running it since Tue afternoon). Cohort compares against **Dynatrace Davis AI** (causation-graph approach), **New Relic AI Monitoring** (apdex-driven), **Coralogix AI Observability** (streaming-pipeline-native). Each pair owns one platform's research output from Wed; Thu morning is structured argument. **No vendor wins by default** — the matrix names the dimensions where each wins.

**Afternoon (practical)** — **AWS managed-service migration of `acquire-gov`.** Cohort migrates `/rag/clause-search` from direct Atlas Vector Search calls toward a **Bedrock Knowledge Base** wrapper (managed retrieval + citation surfacing). Migrates `POST /agent/intake-triage` from custom LangGraph implementation toward **Agents-for-Bedrock** (managed agent orchestration). Each migration produces an ADR that names *whether the managed service is worth it* (latency, vendor lock-in, cost, FedRAMP boundary, debuggability). Direct InvokeModel stays for endpoints where the managed service adds friction without payoff.

**Conceptual (`pre-session/Thu.md`)** — AWS managed services in federal context; vendor lock-in vs FedRAMP boundary; Bedrock Knowledge Bases vs custom RAG decision dimensions.

## Fri 26 Jun — **Final Adversarial Review PR** *(replaces standard war-room)*

> **v2 replacement for v1 Exit Technical Exam.** Same substance (security + failure-sim maturity); culminating PR review instead of separate exam.

**Morning (Final Adversarial PR session — `war-room/Fri-final-adversarial-pr.md`)** — Each pair submits a **hardening PR** against their pair-project + a parallel PR against `acquire-gov`. The PR must contain: (1) **Resilience4j circuit breaker on Item 3** (evaluation-service → solicitation-service hop); (2) **AuditEvent race fix on Item 2** (atomic transaction or outbox pattern); (3) **OTel instrumentation across all 4 services** with consistent `traceparent`; (4) **OWASP LLM Top 10 mitigations** for at least 3 categories surfaced by codex; (5) **Auto-remediation authority ADR** capturing the Wed HITL #7 decision. Instructor opens the PR review live + **codex-adversarial-review runs at Full strictness** (per D-034 ramp). Findings discussed in real time; pair defends or accepts each.

**Afternoon** — **Final Adversarial PR rubric scoring** (`assessments/W05-final-adversarial-pr-rubric.md`). Instructor scores per dimension: Correctness, Code Quality, Architectural Fit, Test Coverage, Brownfield Awareness, PR Clarity, plus a Security Pass score (OWASP LLM Top 10 coverage). **Pair retro + cohort retro.** Pair commits remediation tickets for findings deferred to W6.

**Conceptual (no pre-session — Fri is the assessment)** — N/A. Pre-session for W6 Mon (`weeks/W06/pre-session/Mon.md`) ships EOD Fri.

## Special notes for W5 (Cohort #1)

- **Datadog AI is the hands-on platform per D-060.** Not Dynatrace, not New Relic. Compare-set authoring (W05-SA-1) does the rigorous compare across all four. Karsun-licensed Datadog is the cohort-acquisition assumption.
- **D-050 carve-out vs D-060 exception** — cohort understands by EOD Mon: D-060 let Bedrock InvokeModel land W2 onward; D-050 kept Bedrock *managed services* (Knowledge Bases, Agents-for-Bedrock, OpenSearch Managed) deferred until **this week**. W5 Mon is the gate-open moment; W5 Thu is the migration work.
- **HITL #7 closes the 7-touchpoint thread.** W6 Wed re-asserts the full table as a deliverable. Wed war-room must produce a real ADR; don't let cohort hand-wave it.
- **Final Adversarial Review PR is the W5 assessment + the cohort gate.** Replaces v1's Exit Technical Exam. Rubric is in `assessments/W05-final-adversarial-pr-rubric.md`; codex output template is in `codex-reviews/`.
- **§0 retro Mon names W4 Mid-Sprint Surprise findings as W5 fix backlog.** Item 3 circuit breaker + Item 2 audit-race fix are *required* fixes for the Fri PR. Pair doesn't get to defer them.
- **Density: 5–8 topics/day (Production density).** Extra time for AIOps platform onboarding (Tue afternoon = Datadog install + instrument all 5 components) + Thu afternoon (managed-service migration). Don't push to 12.
- **`/web-research` recency:** AIOps platforms = hot-tech 3-month window (Datadog AI features ship monthly; re-verify pre-cohort). AWS managed services = 3-month window. OWASP LLM = 3-month (2025 list still current). OTel = 12-month stable.

## Assets in this folder

- `PLAN.md` — this file.
- `pre-session/Mon.md..Thu.md` — pre-session reading (one per day; **no Fri pre-session** — Fri is the Final Adversarial PR assessment).
- `war-room/Mon.md..Thu.md` — morning war-room scenario.
- `war-room/Fri-final-adversarial-pr.md` — Final Adversarial PR session shape (replaces standard war-room).
- `scenarios/` — 3 scenario-alternatives prompts (AIOps platform compare; auto-remediation authority; Bedrock managed-services migration).
- `assessments/W05-final-adversarial-pr-rubric.md` — the W5 assessment + cohort gate.
- `assessments/W05-MCQ-senior.md` + `W05-MCQ-entry.md` — supplementary MCQ (lighter than the PR rubric; covers AIOps + OTel + governance + HITL #7 vocabulary).
- `codex-reviews/W05-codex-template.md` — adversarial-review output format.
- `codex-reviews/W05-codex-rubric.md` — scoring rubric for Fri's codex pass.
- `retros/W05-pair-retro.md` + `retros/W05-cohort-retro.md` — Fri retros.

## What the cohort takes into W6 Mon Plan Day

- Datadog AI dashboard live on `acquire-gov`; AuditEvent Activity report wired; Bedrock token/cost rolling.
- Item 2, Item 3, Item 6 closed or remediation-ticketed.
- Final Adversarial PR review findings sorted into "fixed this week" / "deferred to W6" / "documented as accepted risk".
- HITL #7 ADR committed — auto-remediation authority boundary table seeded for W6 Wed's full deliverable.
- AWS managed-service migration ADRs committed — cohort can defend *which* migration was worth it and *which* wasn't.
