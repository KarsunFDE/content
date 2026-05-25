---
week: W05
day: Mon
title: "Pre-session — AIOps Plan Day: D-050 vs D-060 framing + Datadog walkthrough + W4 retro"
audience: All cohort members
time_on_task_minutes: 55
last_verified: 2026-05-23
research_recency_windows: [hot-tech-3mo, foundation-stable-12mo]
---

# W5 Mon Pre-Session — AIOps Plan Day prep

> Read **before** W5 Mon morning. ~55 min total. This week is the AIOps anchor; Mon is Plan Day. **Heavy framing read** — D-050 vs D-060, Datadog AI walkthrough, AWS managed-service onboarding mechanics. Tue–Fri builds on this framing.

## 1. The D-050 / D-060 carve-out — why AWS managed services light up *now* (10 min)

Two decisions have been governing AWS access since W1:

- **D-050** locked "AWS deferred to W5 single Karsun-paid account" as a cost-blast-radius decision. Cohort #1 has one Karsun-paid AWS account; managed services that meter usage (Bedrock Knowledge Bases, Agents-for-Bedrock, OpenSearch Managed) are blast-radius-shaped enough that they wait until the AIOps week where governance lands.
- **D-060** (2026-05-23) carved out **Bedrock InvokeModel specifically** as a W2-onward exception. The cohort needed Bedrock as the *model layer* from W2 to do real RAG + agentic work. Direct InvokeModel + token/cost accounting was authorized then; AWS *managed services* still deferred.

W5 Mon is the gate-open moment for the deferred services:

| Service | Status W2–W4 | Status W5 onward |
|---------|--------------|------------------|
| Bedrock `InvokeModel` (Claude) | Live (D-060 exception) | Live (continues) |
| Bedrock token/cost metering | Live (D-060 exception) | Live (now fans into Datadog) |
| **Bedrock Knowledge Bases** | Deferred (D-050) | **Lights up Mon afternoon** |
| **Agents-for-Bedrock** | Deferred (D-050) | **Lights up Mon afternoon** |
| **OpenSearch Managed** (vector + log) | Deferred (D-050) | **Lights up Mon afternoon** |
| MongoDB Atlas Vector Search | Live (not AWS-managed; D-050 carve-out N/A) | Live (continues; **migration question** Thu) |

Atlas Vector Search has carried the cohort's RAG since W2. Thu afternoon's practical asks: *now that Bedrock Knowledge Bases is available, is migrating `/rag/clause-search` worth it?* That's the canonical W5 managed-service question — not a foregone conclusion. The cohort must defend the answer.

[`pipeline/DECISIONS.md` D-050 + D-060, retrieved 2026-05-23]

## 2. AIOps platform landscape — Datadog AI hands-on, others compare-set (15 min)

Per D-060, **Datadog AI** is the hands-on pick this week. Cohort onboards the Datadog Agent into `acquire-gov` Tue afternoon and runs the AIOps surface against the running stack.

**Why Datadog AI for cohort #1 hands-on (per D-060):**

- Cohort-familiar UI; Datadog dashboards are common in federal-modernization engagements.
- Strongest AI-on-AI features in 2026 — Watchdog AI (anomaly detection) + LLM Observability (per-request trace, hallucination signals, cost-per-tenant).
- Karsun-likely-licensed (engagement assumption).
- Best `/web-research` signal mass for the hot-tech 3-month window (Datadog ships AI features monthly — re-verified pre-cohort).

Read once, lightly:

- [Datadog Bits AI in live incidents](https://www.datadoghq.com/blog/bits-ai-incident-management/) — conversational troubleshooting on the Datadog stack. ~5 min.
- [Datadog LLM Observability docs landing](https://docs.datadoghq.com/llm_observability/) — per-request LLM trace, evaluation metrics, cost-per-tenant. ~5 min skim (don't deep-read; Tue afternoon is hands-on).

**The compare-set (Dynatrace / New Relic / Coralogix) appears Wed afternoon during the dedicated research slot.** You'll read in depth then. For Mon: know the four names + know that this week's compare-matrix work *is the rigorous defense of Datadog as the hands-on choice* — not loyalty to the default.

## 3. The W4 §0 retrospective — what to bring to Mon morning (10 min)

Mon morning opens with **§0 Plan retrospective on the W4 plan-spec** (per D-036 — every Plan Day starts with last week's spec retro). W4's plan-spec landed before the Mid-Sprint Surprise. The Surprise (per D-049) was an `acquire-gov` prod incident: **Workflow 4 (evaluator → consensus → SSA → award) under load**, where Item 3 (no circuit breaker on evaluation-service → solicitation-service hop) caused a retry storm + Item 2 (audit-log race) left the SSDD-sign action un-logged.

Bring to Mon:

- Your pair's W4 plan-spec link.
- A one-paragraph answer to: *"What did the Mid-Sprint Surprise reveal that the W4 plan-spec didn't anticipate?"*
- A one-paragraph answer to: *"What's the W5 fix work that *must* land in the Fri Final Adversarial PR?"* (Hint: Item 3 + Item 2 are the required fixes. Anything else you choose to harden is bonus.)

If your pair didn't run the Surprise (calendar slip, etc.), surface this at Mon morning's 1:1. Instructor will calibrate.

## 4. AI-SRE as a discipline — what the cohort signs up for this week (10 min)

The cohort hears the term "AI-SRE teammate" a lot in federal-modernization marketing. This week defines what that actually *is*. Read `skills/aiops-curriculum/references/ai-sre-patterns.md` once — the six patterns:

1. **Investigation** — pull telemetry from logs/metrics/traces/LangSmith, correlate, produce timeline.
2. **Correlation** — *"what changed in the last N hours that could explain what we're seeing now?"*
3. **Root-cause analysis** — reason forward from correlated change to symptom.
4. **Fix-PR generation** — propose a PR, not a chat recommendation. Human reviews + merges.
5. **Runbook augmentation** — write back after every incident.
6. **Auto-remediation (with explicit authority)** — *can* close the loop, but only with policy-granted authority.

Pattern 6 is where **HITL #7** lands Wed. Read the AI-SRE patterns doc once before Mon to have vocabulary.

## 5. The Final Adversarial PR — what it asks Fri (8 min)

Fri 26 Jun is the **Final Adversarial Review PR**, the v2 replacement for v1's Exit Technical Exam. By Fri morning, your pair submits a hardening PR with at minimum:

1. **Resilience4j circuit breaker on Item 3** (evaluation-service → solicitation-service hop)
2. **AuditEvent race fix on Item 2** (atomic transaction or outbox pattern)
3. **OTel instrumentation across all 4 services** (Angular RUM + 3 Java + Python) with consistent `traceparent`
4. **OWASP LLM Top 10 mitigations** for at least 3 categories surfaced by codex
5. **Auto-remediation authority ADR** (from Wed's HITL #7 decision)

Instructor opens the PR review live; **codex-adversarial-review runs at Full strictness per D-034**. Findings are surfaced in real time; your pair defends or accepts each. Rubric scoring follows per the 7-dimension rubric in `assessments/W05-final-adversarial-pr-rubric.md`.

**This is the cohort gate.** Plan Mon's Scenario Design Planning artifact with Friday's PR in mind — every ADR should trace to a Fri PR commit.

## 6. Mon Scenario Design Planning artifact — six artifacts due EOD (2 min)

Per `templates/scenario-design-planning.md`, due EOD Mon:

1. Requirements synthesis — AIOps integration scope + AWS managed-service migration scope.
2. 6R / system map — `acquire-gov` services + Datadog Agent placement + Bedrock managed-service touchpoints.
3. ADRs — (a) Datadog as hands-on platform vs alternatives, (b) Bedrock Knowledge Base migration boundary, (c) HITL #7 authority table seed.
4. Estimate — Tue OTel install + Thu managed-service migration + Fri PR work.
5. Risk register — Datadog data egress / cost / cohort-licensed-account-blast-radius / managed-service vendor lock-in.
6. Open questions — write them down; Wed's dedicated research slot is where they get answered.

**Plan-spec integrity check by instructor 17:00 Mon.**

---

## Sources (all retrieved 2026-05-23 via `/web-research`)

- `pipeline/DECISIONS.md` D-050 + D-060 (in-repo, current).
- `pipeline/DECISIONS.md` D-040 (Wed = scenario-research day).
- `pipeline/DECISIONS.md` D-049 (W4 Mid-Sprint Surprise = acquire-gov prod incident).
- `skills/aiops-curriculum/references/ai-sre-patterns.md` (in-repo).
- Datadog Bits AI in live incidents — https://www.datadoghq.com/blog/bits-ai-incident-management/
- Datadog LLM Observability docs — https://docs.datadoghq.com/llm_observability/

Hot-tech 3-month window applied to Datadog content; re-verify the LLM Observability docs page within 30 days of cohort start.
