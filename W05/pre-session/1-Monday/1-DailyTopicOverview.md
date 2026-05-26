---
week: W05
day: Mon
title: "Pre-session — AIOps Plan Day: spec-driven dev, platform walkthrough, AIOps decisions, Ops Governance Spec, Production AI scope, D-050/D-060 carve-out"
audience: All cohort members
time_on_task_minutes: 60
last_verified: 2026-05-26
research_recency_windows: [hot-tech-3mo, federal-regulatory-6mo, foundation-stable-12mo]
---

# W5 Mon Pre-Session — AIOps Plan Day prep

> Read **before** W5 Mon afternoon (morning is Checkpoint 3 + audit walkthroughs). ~60 min total. This week is the AIOps anchor; Mon is Plan Day. **Six topics promote the load-bearing PDF-canonical Plan-Day artifacts** — iterative spec-driven dev (the §0 retro discipline), AIOps platform walkthrough, the three Mon ADRs, the Ops Governance Spec, the Production AI Scope Survey, and the cohort-specific D-050/D-060 carve-out that makes this whole week possible. Tue–Fri builds on what you commit today.

## 1. Iterative Spec-Driven Dev — §0 retro as W5's load-bearing discipline (10 min)

W5 Mon is the **fifth Plan Day in a row** the cohort opens with `§0 retrospective on last week's plan-spec` (per D-029 + D-036 in `pipeline/DECISIONS.md`). That pattern *is* iterative spec-driven dev — you write a spec, you ship against it, you retro the spec before writing the next one. W4's spec doesn't get a passing grade just because the week ended; it gets a passing grade only if it survives contact with the Mid-Sprint Surprise.

Bring to Mon afternoon:

- Your pair's W4 plan-spec link.
- A one-paragraph answer to: *"What did the Mid-Sprint Surprise (per D-049 — the `acquire-gov` prod incident in Workflow 4) reveal that the W4 plan-spec didn't anticipate?"* Hint: Item 3 (absent circuit breaker on the evaluation-service → solicitation-service hop) caused a retry storm. Item 2 (AuditEvent race) left the SSDD-sign un-logged.
- A one-paragraph answer to: *"What's the W5 fix work that **must** land in the Fri Final Adversarial Review PR?"* Item 3 + Item 2 are required. Anything else you choose to harden is bonus.

This is the §0 retro shape. Spec-driven dev is not a W4 topic you read about; it's the *living discipline* you've been practising since W2 Mon and that ramps to its sharpest form this week — every Mon ADR you author this afternoon traces to a Fri PR commit, or it doesn't earn ADR status.

If your pair didn't run the W4 Surprise (calendar slip, exam disruption, etc.), surface this at Mon morning's 1:1 — the instructor will calibrate.

Source: `pipeline/DECISIONS.md` D-029, D-036, D-049 (in-repo, retrieved 2026-05-26 via `/web-research`).

## 2. AIOps Platform Walkthrough — Datadog hands-on + Dynatrace/New Relic/Coralogix compare-set (12 min)

Per D-060, **Datadog AI** is the hands-on pick this week. Cohort onboards the Datadog Agent into `acquire-gov` Tue afternoon and runs the AIOps surface against the running stack.

**Why Datadog AI for cohort #1 hands-on** (per D-060):

- Cohort-familiar UI; Datadog dashboards are common in federal-modernization engagements.
- Strongest AI-on-AI features for the cohort's 2026 timeframe — Watchdog AI (anomaly detection) + LLM Observability (per-request trace, hallucination signals, cost-per-tenant).
- Karsun-likely-licensed (engagement assumption).
- FedRAMP High via GovCloud authorization — load-bearing for federal-acquisitions work.

Read once, lightly:

- [Datadog Bits AI in live incidents](https://www.datadoghq.com/blog/bits-ai-incident-management/) — conversational troubleshooting on the Datadog stack. ~5 min.
- [Datadog LLM Observability docs landing](https://docs.datadoghq.com/llm_observability/) — per-request LLM trace, evaluation metrics, cost-per-tenant. ~5 min skim (don't deep-read; Tue afternoon is hands-on).

**The compare-set** — Dynatrace Davis AI (causation-graph approach), New Relic AI Monitoring (apdex-driven + assistant framing), Coralogix AI Observability (streaming-pipeline-native, per-GB ingest pricing) — **lands in depth Wed afternoon during the dedicated research slot** (per D-040). For Mon: know the four names + know that this week's compare-matrix work (`aiops-compare-matrix.md` in this folder) *is the rigorous defense of Datadog as the hands-on choice* — not loyalty to the default.

Sources (all retrieved 2026-05-26 via `/web-research`, hot-tech 3-month window):
- Datadog Bits AI — https://www.datadoghq.com/blog/bits-ai-incident-management/
- Datadog LLM Observability — https://docs.datadoghq.com/llm_observability/
- Datadog Watchdog AI — https://docs.datadoghq.com/watchdog/
- `pipeline/DECISIONS.md` D-060 (in-repo).

## 3. AIOps Decisions — the 3 ADRs you draft Mon afternoon (8 min)

Your pair commits **three named ADRs by EOD Mon 17:00** as part of the W5 Scenario Design Planning artifact (per `templates/scenario-design-planning.md`). These are the load-bearing Mon decisions — every one of them feeds something downstream.

| ADR | What it commits | Downstream consumer |
|-----|------------------|---------------------|
| **ADR-1: Datadog vs alternatives (platform choice)** | Pair commits the hands-on choice (Datadog per D-060) **with** named pivot conditions (where you'd choose Dynatrace / New Relic / Coralogix for a different engagement shape). Don't restate D-060; defend it. | Thu compare-matrix workshop + Fri PR's platform-attestation paragraph. |
| **ADR-2: Bedrock Knowledge Base migration boundary** | Pair commits *whether* `/rag/clause-search` migrates from Atlas Vector Search to Bedrock Knowledge Bases Thu afternoon — and, if yes, what boundary (latency budget, FedRAMP boundary, multi-tenant filter on `agency_id`, vendor lock-in cost). Migrate / hybrid / no-migrate are all defensible; *"we'll decide Thu"* is not. | Thu migration practical (`aws-managed-service-onboarding.md` Migration 1). |
| **ADR-3: HITL #7 authority seed** | Pair commits a **starting position** (Full Auto / Propose-and-Approve / Escalate-only) for the audit-gap auto-remediation question — knowing the *final* ADR lands Wed. Mon's seed is the gut call; Wed's ADR is the defended position. Capture this in your private 1:1 doc as well. | Wed war-room (HITL #7 walkthrough) + the formal `hitl-7-authority-adr-template.md` Wed EOD. |

These three ADRs are not interchangeable with the open-questions list. Open questions are what Wed research answers; ADRs are what your pair commits Mon and defends Fri.

Source: `weeks/W05/PLAN.md` Mon row + `weeks/W05/scenarios/W05-SA-1..SA-3.md` (in-repo, retrieved 2026-05-26 via `/web-research`).

## 4. Ops Governance Spec — what it commits (8 min)

The **Ops Governance Spec** is the artifact your pair commits Mon afternoon that names *how the AIOps stack is governed once it's live* — separate from the platform-choice ADR (that's ADR-1). The Governance Spec is the closest analogue to what a Karsun engagement would hand a federal client's CISO before turning AIOps loose on their environment.

What the spec names:

- **AIOps integration scope** — which `acquire-gov` services + endpoints the AIOps stack monitors this week (Angular SPA via RUM, 3 Java services via OTel Java auto-agent, Python ai-orchestrator via FastAPI auto-instrumentation, Bedrock via `gen_ai.*` semconv span attributes).
- **Alerting policy** — what fires a Datadog Watchdog AI anomaly alert (token-cost spike, latency p95 breach, audit-gap pattern, drift signal on agentic flow).
- **On-call surface** — who gets paged, on what severity, with what runbook entry. For cohort #1 this is the instructor + the pair (no real on-call); the spec still names the shape so Fri's PR runbook section is grounded.
- **FedRAMP boundary call-outs** — which managed services cross the FedRAMP boundary (Bedrock Knowledge Bases on GovCloud = inside; Datadog SaaS US1 = outside unless GovCloud tenant; OpenSearch Managed in cohort region = inside if commercial, inside-GovCloud if relocated). Reference `aws-managed-service-onboarding.md` for the per-service status.
- **Audit-of-the-audit** — what audit row the platform itself writes when it fires an automated action. Seeds the HITL #7 conversation.

The Governance Spec is **not** the FedRAMP authorization document — that lives downstream. It's the *operating contract* the pair commits to before turning the AIOps stack on. Format: 1–2 pages markdown, committed to your pair-project repo at `docs/ops-governance-spec.md`.

Source: `weeks/W05/aiops-compare-matrix.md` + `weeks/W05/aws-managed-service-onboarding.md` (in-repo, retrieved 2026-05-26 via `/web-research`).

## 5. Production AI Scope Survey — `acquire-gov` surfaces lighting up this week (8 min)

The **Production AI Scope Survey** is the Mon artifact that names *which surfaces of `acquire-gov` get instrumented + monitored + governed this week*. It's the bridge between the W4 Mid-Sprint Surprise discoveries and the Tue/Thu/Fri implementation work.

What the survey lists, per surface:

- **Item 6 (inconsistent correlation IDs)** — modernised end-to-end Tue afternoon. Lights up under: Angular RUM `traceparent` propagation, OTel Java auto-agent on the 3 Java services, `opentelemetry-instrumentation-fastapi` on the Python ai-orchestrator, Bedrock `gen_ai.*` span attributes on every InvokeModel call. **Required for Fri PR.**
- **Item 3 (absent circuit breaker, evaluation-service → solicitation-service hop)** — fixed Fri PR via Resilience4j. Lights up under: Datadog Watchdog AI alerting on retry-storm signature; OTel error-span attributes on circuit-open transitions. **Required for Fri PR.**
- **Item 2 (AuditEvent race)** — fixed Fri PR via atomic transaction or outbox pattern. Lights up under: Datadog audit-gap detection + the HITL #7 ADR's enforcement code. **Required for Fri PR.**
- **`/rag/clause-search`** — candidate for Thu migration to Bedrock Knowledge Bases (per ADR-2). If migrated, the multi-tenant filter on `agency_id` metadata is non-negotiable.
- **`POST /agent/intake-triage`** — candidate for Thu migration to Agents-for-Bedrock. HITL fidelity check: the pair's W3 multi-agent HITL #5 ADR must align with the Agents user-confirmation surface, or the pair commits to no-migrate.
- **Bedrock token + cost metering** — fans into Datadog Tue per the D-060 carve-out; per-tenant cost rolls into the dashboard.

What the survey excludes: anything that isn't surfaced by `acquire-gov`'s 5-component topology Mon afternoon (e.g., AWS Lambda is technically available but isn't a primary W5 target per `aws-managed-service-onboarding.md`). Scope discipline is the point — you commit to what gets instrumented this week so Fri's PR has a defensible boundary.

Format: 1-page table committed to your pair-project repo at `docs/production-ai-scope.md`.

Source: `training-project/feature-inventory-target.md` (in-repo, retrieved 2026-05-26 via `/web-research`) — Items 2, 3, 6, 10 are the load-bearing references.

## 6. D-050 + D-060 carve-out — why AWS managed services light up *now* (8 min)

Two decisions have been governing AWS access since W1:

- **D-050** locked "AWS deferred to W5 single Karsun-paid account" as a cost-blast-radius decision. Cohort #1 has one Karsun-paid AWS account; managed services that meter usage (Bedrock Knowledge Bases, Agents-for-Bedrock, OpenSearch Managed) are blast-radius-shaped enough that they wait until the AIOps week where governance lands.
- **D-060** (2026-05-23) carved out **Bedrock InvokeModel specifically** as a W2-onward exception. The cohort needed Bedrock as the *model layer* from W2 to do real RAG + agentic work. Direct InvokeModel + token/cost accounting was authorized then; AWS *managed services* still deferred.

W5 Mon afternoon is the gate-open moment for the deferred services:

| Service | Status W2–W4 | Status W5 onward |
|---------|--------------|------------------|
| Bedrock `InvokeModel` (Claude) | Live (D-060 exception) | Live (continues) |
| Bedrock token/cost metering | Live (D-060 exception) | Live (now fans into Datadog) |
| **Bedrock Knowledge Bases** | Deferred (D-050) | **Lights up Mon afternoon** |
| **Agents-for-Bedrock** | Deferred (D-050) | **Lights up Mon afternoon** |
| **OpenSearch Managed** (vector + log) | Deferred (D-050) | **Lights up Mon afternoon** |
| MongoDB Atlas Vector Search | Live (not AWS-managed; D-050 carve-out N/A) | Live (continues; **migration question** Thu) |

Atlas Vector Search has carried the cohort's RAG since W2. Thu afternoon's practical asks: *now that Bedrock Knowledge Bases is available, is migrating `/rag/clause-search` worth it?* That's the canonical W5 managed-service question — not a foregone conclusion. The cohort must defend the answer (this is ADR-2 from topic 3).

Verification you can run before Mon afternoon (after AWS SSO login):

```
aws sso login --profile cohort1
aws bedrock-agent list-knowledge-bases --region us-east-1
aws bedrock-agent list-agents --region us-east-1
aws opensearch list-domains --region us-east-1
```

Each should return cleanly (empty lists for Bedrock; cohort domain in Active or Stopped state for OpenSearch). If anything fails, surface in Mon afternoon's first 15 min — instructor walks IAM trust policy live.

Sources (all retrieved 2026-05-26 via `/web-research`):
- `pipeline/DECISIONS.md` D-050 + D-060 (in-repo, current).
- `weeks/W05/aws-managed-service-onboarding.md` (in-repo).

---

## Closing pointers (not numbered topics)

**AI-SRE preview.** The six AI-SRE patterns (Investigation, Correlation, Root-cause, Fix-PR generation, Runbook augmentation, Auto-remediation-with-authority) are covered end-to-end Wed pre-session + Wed morning war-room. Mon-evening glance at `skills/aiops-curriculum/references/ai-sre-patterns.md` if you want to walk in Wed already fluent in the vocabulary. Pattern 6 is where HITL #7 lands. ~2 min.

**Fri PR preview.** Fri 26 Jun is the **Final Adversarial Review PR**, the v2 replacement for v1's Exit Technical Exam. By Fri morning your pair submits a hardening PR with at minimum: (1) Resilience4j circuit breaker on Item 3, (2) AuditEvent race fix on Item 2, (3) OTel instrumentation across all 4 services with consistent `traceparent`, (4) OWASP LLM Top 10 mitigations for at least 3 codex-surfaced categories, (5) Auto-remediation authority ADR from Wed's HITL #7 decision. Instructor opens the PR review live; codex-adversarial-review runs at Full strictness per D-034. ~2 min.

**Mon Scenario Design Planning artifact checklist (EOD 17:00).** Per `templates/scenario-design-planning.md`: (a) Requirements synthesis — AIOps integration scope + managed-service migration scope. (b) 6R / system map — `acquire-gov` services + Datadog Agent placement + Bedrock managed-service touchpoints. (c) Three named ADRs (per topic 3). (d) Estimate — Tue OTel install + Thu managed-service migration + Fri PR work. (e) Risk register — Datadog data egress, cost-per-day cap, vendor lock-in, FedRAMP boundary breach. (f) Open questions for Wed's research slot. Plan-spec integrity check by instructor 17:00 Mon. ~2 min.

---

## Sources (all retrieved 2026-05-26 via `/web-research` per `pipeline/RESEARCH-PROTOCOL.md` D-046)

**Hot-tech 3-month window:**
- Datadog Bits AI in live incidents — https://www.datadoghq.com/blog/bits-ai-incident-management/
- Datadog LLM Observability docs — https://docs.datadoghq.com/llm_observability/
- Datadog Watchdog AI — https://docs.datadoghq.com/watchdog/

**Foundation-stable 12-month + in-repo references:**
- `pipeline/DECISIONS.md` D-029 (Plan-Day pattern), D-036 (§0 retro discipline), D-040 (Wed research slot), D-046 (research protocol), D-049 (W4 Mid-Sprint Surprise = acquire-gov prod incident), D-050 (AWS deferred to W5), D-060 (Datadog AI hands-on + Bedrock W2-onward exception).
- `weeks/W05/aiops-compare-matrix.md` (in-repo).
- `weeks/W05/aws-managed-service-onboarding.md` (in-repo).
- `weeks/W05/hitl-7-authority-adr-template.md` (in-repo).
- `skills/aiops-curriculum/references/ai-sre-patterns.md` (in-repo).
- `training-project/feature-inventory-target.md` (in-repo) — Items 2, 3, 6, 10.

Hot-tech 3-month window applied to all Datadog content; re-verify within 30 days of cohort start per cohort-instantiation checklist (`pipeline/PIPELINE.md` §17). Federal-regulatory 6-month window applies to FedRAMP boundary references in topic 4 — current FedRAMP High GovCloud authorization for Datadog confirmed via vendor security page within window.

Last verified: 2026-05-26
