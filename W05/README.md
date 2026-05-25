# W05 · AIOps Anchor Week: OpenTelemetry on AI + AI-SRE + Governance + Final Adversarial Review PR

*Phase:* Phase 2 — Modernization (operationalisation week). AIOps + governance + compliance + deployment readiness all surface as *operational maturity* the modernised system needs before it can be handed to a federal client. Each is an answer to a question Phase 1 (or W4 Mid-Sprint Surprise) surfaced — "what would the AI-SRE have caught?", "what does the OIG need in the binder?", "what's the on-call surface?". **v2 renumbered from v1 W6 → v2 W5 per D-044.**
*Gate:* **Final Adversarial Review PR** (W5 Fri 26 Jun) — v2 framing; replaces v1's Exit Technical Exam (same substance, less exam-style framing).
*Project phase:* Pair Project Phase 2 — operationalisation.
*Weekly assessments:* MCQ (supplementary, senior + entry) + **Final Adversarial Review PR** (Fri = the assessment + the cohort gate) + Codex on PRs (Full strictness per D-034)
*HITL:* **Touchpoint #7 of 7** lands Wed — auto-remediation authority boundaries (per D-043 + D-044 extension). Closes the HITL programme thread; re-asserted as a deliverable W6 Wed.
*AIOps hands-on platform:* **Datadog AI** (Watchdog AI + LLM Observability) per D-060. Dynatrace / New Relic / Coralogix are compare-set, NOT hands-on.
*AWS managed services:* **First lit this week** per D-050 + D-060. Bedrock Knowledge Bases, Agents-for-Bedrock, OpenSearch Managed onboard Mon afternoon; cohort migrates `acquire-gov` Thu afternoon and *defends whether each migration is worth it*.

## v2 day-by-day

| Day | Headline | Anchor topics |
|-----|----------|---------------|
| **Mon Plan Day** | AIOps integration + AWS managed-service onboarding plan | **Trainer 1:1** (gate-boundary cadence); **§0 retro on W4 plan** — names Mid-Sprint Surprise findings as W5 fix backlog (Item 3 circuit breaker, Item 2 audit-race); AIOps Platform Walkthrough (Datadog AI features); Scenario Design Planning artifact (graded brief due EOD); AWS managed-service onboarding gate opens (D-050 carve-out lifts) |
| Tue | OpenTelemetry on AI Workloads | W3C `traceparent` across LLM + non-LLM hops; OTel `gen_ai.*` semconv; Bedrock token/cost as OTel span attrs; **Item 6 modernised end-to-end** (Angular RUM + 3 Java auto-agents + Python instrumentation); Datadog Agent install into docker-compose |
| Wed | AIOps Governance + HITL #7 + Dedicated `/web-research` Slot | **AI-SRE Pattern Walkthrough**; **HITL #7** (auto-remediation authority — canonical case: auto-fix-the-audit-gap); AIOps governance ADRs; **OWASP LLM Top 10** continuation (LLM05, LLM09, LLM10 AIOps-discoverable risks); **Research day per D-040** — pairs work all 3 W05 scenario-alternatives prompts |
| Thu | AIOps Platform Comparison + AWS Managed-Service Migration | Datadog vs Dynatrace vs New Relic vs Coralogix compare-matrix; **Bedrock Knowledge Base migration** of `/rag/clause-search`; **Agents-for-Bedrock migration** of `POST /agent/intake-triage`; managed-service worth-it ADRs |
| **Fri** | **Final Adversarial Review PR** | Hardening PR with (1) Resilience4j on Item 3, (2) AuditEvent race fix on Item 2, (3) OTel across all 4 services, (4) OWASP LLM Top 10 mitigations, (5) Auto-remediation authority ADR; **codex-adversarial-review Full strictness**; instructor scoring per rubric; pair + cohort retro |

## How this folder is filled (cohort #1 — authored)

| Subfolder | Artifact type | Status |
|-----------|---------------|--------|
| `pre-session/Mon..Thu.md` | Pre-session reading (no Fri — Fri is the assessment) | authored |
| `war-room/Mon..Thu.md` | Morning war-room scenario | authored |
| `war-room/Fri-final-adversarial-pr.md` | Final Adversarial PR session shape | authored |
| `scenarios/W05-SA-{1,2,3}.md` + `_index.md` | 3 scenario-alternatives prompts | authored |
| `assessments/W05-final-adversarial-pr-rubric.md` | Final Adversarial PR rubric (the W5 gate) | authored |
| `assessments/W05-MCQ-{senior,entry}.md` | Supplementary MCQ (study aid; NOT the gate) | authored |
| `codex-reviews/W05-codex-{template,rubric}.md` | Codex output template + scoring rubric | authored |
| `retros/W05-{pair,cohort,eod-probes}-retro.md` | Pair retro template + cohort retro + EOD probes | authored |
| `PLAN.md` | One-page week plan (Mon–Fri grid) | authored |

## Special notes for W5 (Cohort #1)

- **W5 is the heaviest Production-AI week.** AIOps + Governance + AWS Managed Services + Final Adversarial Review compressed into 5 days. **Density cap: 5–8 topics/day** (Production density — extra time for Datadog onboarding Tue afternoon + managed-service migration Thu afternoon). Don't push to 12.
- **Datadog AI is the hands-on pick per D-060.** Dynatrace / New Relic / Coralogix appear in W05-SA-1 as compare-set, NOT hands-on. The cohort defends the Datadog choice rigorously — they don't just accept it.
- **D-050 + D-060 framing must land EOD Mon.** D-060 carved out Bedrock InvokeModel for W2-onward; D-050 kept managed services deferred *to this week specifically*. Without this framing, the cohort wonders why managed services suddenly land Thu.
- **HITL #7 is the closing touchpoint of 7.** Wed war-room produces a real ADR — auto-fix-the-audit-gap. W6 Wed re-asserts the full 7-touchpoint authority-boundary table as a deliverable; W5 Wed seeds the last row.
- **Final Adversarial Review PR Fri 26 Jun is the W5 assessment + the cohort gate.** Replaces v1's Exit Technical Exam. Same security + failure-sim rigor, framed as a culminating PR review. Rubric is in `assessments/W05-final-adversarial-pr-rubric.md`.
- **Karsun-specific request:** AIOps platform shootout (W05-SA-1) lands Wed — D-040's dedicated research-day slot is the natural fit.
- **Wed = research day per D-040.** All 3 scenario-alternatives prompts get pair-collaborative research time that afternoon. The compare-matrix workshop Thu morning depends on this work.

## Map to acquire-gov features (per `training-project/week-dependency-map.md` §W5)

- **Audit-log Activity report** (from inventory's 5-report list) = canonical AIOps observability surface. Cohort sees AuditEvents grouped by actor + action (FedRAMP AU-2/AU-6 evidence) — gaps from **Item 2** race become detectable here.
- **LLM response-quality dashboard** instrumented via OTel spans on the 8 ai-orchestrator endpoints; cost-vs-quality dial visible per endpoint. Bedrock token/cost metrics fan in.
- **Agent retry-storm scenario** — evaluator-agent calling `solicitation-service` for proposal text without circuit breaker (**Item 3**) under TEP-week concurrent load is the canonical reproducer; cohort builds Resilience4j circuit + idempotency keys in Fri's PR.
- **Auto-remediation authority boundary (HITL #7)** — the canonical AIOps-governance case is "auto-fix-the-audit-gap" decision against **Item 2** race.
- **OIG Findings Tracker (`/admin/findings`)** — **Item 12** (GHA lint debt) could be opened as a finding here during AIOps-governance work; remediation tracked via the same surface the cohort built.

Reports surface clarifications:
- W5 instruments the underlying AuditEvent table.
- W6 documents the runbook + alert-routing + on-call rotation that *uses* this instrumentation. W5 is the instrumentation layer; W6 is the human-process layer.
