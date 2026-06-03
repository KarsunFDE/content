---
checkpoint: 4
title: "QC Audit Expectations — Checkpoint 4: AIOps, Governance, Deployment Readiness"
covers: ["W05 AIOps anchor week", "W05 OpenTelemetry on AI", "W05 AI-SRE + governance + OWASP LLM Top 10", "W05 AWS managed services"]
delivery_date: 2026-06-26
last_verified: 2026-06-03
read_time_min: 12
audience: learner
---

# Checkpoint 4 — What to Expect (Fri 26 Jun 2026)

> Learner-facing prep brief. Covers **W5 AIOps anchor week** — observability on AI workloads, AI-SRE patterns, governance + OWASP LLM Top 10 final pass, AWS managed-service onboarding, and the Final Adversarial Review PR. **Lands the Friday of W5 — the same day as the Final Adversarial Review PR, closing out the week.** Exam runs early PM; audit interviews async across W6.

This is the **last formal checkpoint** before the Deployment Gate / Final Defense (Thu 2 Jul). The auditor knows this. Expect the audit to feel more like a *senior-engineer technical conversation* than a quiz — you're being calibrated for client deliverability.

---

## What's different about C4

C1 tested AI fundamentals. C2 tested agentic topology. C3 tested SDLC discipline + AI security. **C4 tests whether you can operate the AI system in production.** That means observability, incident response, governance binders for the OIG/OMB, AWS managed-service trade-offs, and what auto-remediation authority looks like when an AI-SRE has tool-use power.

**HITL #7 (Wed)** landed in W5 — the *last* HITL touchpoint of the programme. It frames the **auto-remediation authority boundary**: when AI-SRE can autonomously restart, rollback, or scale, vs. when a human MUST sign off. This is the audit's spine for C4.

**Final Adversarial Review PR** runs the morning of the same day (Fri 26 Jun), closing out W5. If your pair shipped or failed it, that's fair game — the auditor will press on the security findings + load test results in the afternoon audit.

---

## Topics in scope

- [W5 PLAN — week shape](../W05/PLAN.md)

**Mon — AIOps integration + AWS managed-service plan**
- [Iterative spec-driven dev](../W05/pre-session/1-Monday/2-iterative-spec-driven-dev.md)
- [AIOps platform walkthrough](../W05/pre-session/1-Monday/3-aiops-platform-walkthrough.md)
- [AIOps decisions — three ADRs](../W05/pre-session/1-Monday/4-aiops-decisions-three-adrs.md)
- [Ops governance spec](../W05/pre-session/1-Monday/5-ops-governance-spec.md)
- [Production-AI scope survey](../W05/pre-session/1-Monday/6-production-ai-scope-survey.md)
- [AWS managed-services carve-out](../W05/pre-session/1-Monday/7-aws-managed-services-carve-out.md)

**Tue — OpenTelemetry on AI workloads**
- [Why OTel on AI matters](../W05/pre-session/2-Tuesday/2-why-otel-on-ai-matters.md)
- [Trace propagation — W3C traceparent](../W05/pre-session/2-Tuesday/3-trace-propagation-w3c-traceparent.md)
- [Span attributes — GenAI semantic conventions](../W05/pre-session/2-Tuesday/4-span-attributes-gen-ai-semconv.md)
- [OTel instrumentation — Python AI service](../W05/pre-session/2-Tuesday/5-otel-instrumentation-python-ai-service.md)
- [Cost as an AIOps signal](../W05/pre-session/2-Tuesday/6-cost-as-aiops-signal.md)
- [Drift detection — agentic systems](../W05/pre-session/2-Tuesday/7-drift-detection-agentic-systems.md)

**Wed — AIOps governance + HITL #7 + research slot**
- [HITL #7 — auto-remediation authority](../W05/pre-session/3-Wednesday/2-hitl-7-auto-remediation-authority.md)
- [AI-SRE pattern walkthrough](../W05/pre-session/3-Wednesday/3-ai-sre-pattern-walkthrough.md)
- [AIOps constraints — reversibility & blast radius](../W05/pre-session/3-Wednesday/4-aiops-constraints-reversibility-blast-radius.md)
- [OWASP LLM Top 10 — AIOps-discoverable](../W05/pre-session/3-Wednesday/5-owasp-llm-top-10-aiops-discoverable.md)
- [Research-day mechanics](../W05/pre-session/3-Wednesday/6-research-day-mechanics.md)
- [AIOps governance ADR template](../W05/pre-session/3-Wednesday/7-aiops-governance-adr-template.md)
- [Load-bearing Fri PR — Wed commits](../W05/pre-session/3-Wednesday/8-load-bearing-fri-pr-wed-commits.md)

**Thu — AIOps platform comparison + AWS managed-service onboarding**
- [AIOps platform compare-matrix workshop](../W05/pre-session/4-Thursday/2-aiops-platform-compare-matrix-workshop.md)
- [Bedrock Knowledge Bases migration](../W05/pre-session/4-Thursday/3-bedrock-knowledge-bases-migration.md)
- [Agents-for-Bedrock migration](../W05/pre-session/4-Thursday/4-agents-for-bedrock-migration.md)
- [Audit logging & responsible AI](../W05/pre-session/4-Thursday/5-audit-logging-and-responsible-ai.md)
- [Drift-monitoring policy](../W05/pre-session/4-Thursday/6-drift-monitoring-policy.md)
- [Human-oversight escalation workflows](../W05/pre-session/4-Thursday/7-human-oversight-escalation-workflows.md)
- [Deployment planning — incident-response runbook](../W05/pre-session/4-Thursday/8-deployment-planning-incident-response-runbook.md)

**Fri — Final Adversarial Review PR** (replaces standard war-room; same day as this checkpoint)

- W5 war-room incidents — `W05/war-room/`.

### Cross-cutting

- **OpenTelemetry on AI workloads** — what to span (LLM call, retrieval, rerank, generation, validation), what attributes to attach (model ID, token counts, latency-per-stage, faithfulness scores, retrieval scores), how spans nest across the agent graph
- **AI-SRE patterns** — anomaly detection on AI signals (faithfulness drift, retrieval-score drift, latency p99 drift, cost-per-request drift), auto-remediation playbooks + their authority boundaries
- **AIOps platform comparison** — Datadog AI (Watchdog AI + LLM Observability — what your cohort got hands-on with), Dynatrace Davis AI, New Relic AI Monitoring, Coralogix AI Observability. Know one deeply; know the trade-offs across
- **AWS managed-service onboarding** — Bedrock Knowledge Bases, Agents-for-Bedrock, OpenSearch Managed. What you trade when you adopt them; what stays in your hands
- **Governance + OWASP LLM Top 10** — final pass, now in a deployment-readiness frame. What does the OIG binder need? What's the FedRAMP-relevant evidence?
- **OWASP LLM Top 10:2025** — same categories as C3, now exercised through an *operations* lens (LLM10 unbounded consumption, LLM08 vector/embedding weaknesses, LLM03 supply chain in particular)
- **Final Adversarial Review PR** — Fri's security-and-failure-sim review; what the cohort found, what blocked, what shipped

---

## Example scenarios (illustrative — distinct from your W5 incidents)

### Example A — OpenTelemetry span design for a multi-agent flow

*Supporting reading:* [Span attributes — GenAI semantic conventions](../W05/pre-session/2-Tuesday/4-span-attributes-gen-ai-semconv.md) · [OTel instrumentation — Python AI service](../W05/pre-session/2-Tuesday/5-otel-instrumentation-python-ai-service.md) · [Trace propagation — W3C traceparent](../W05/pre-session/2-Tuesday/3-trace-propagation-w3c-traceparent.md)

> *"Your evaluator-consensus-SSA flow takes a proposal and emits an audit-log row 4–8 minutes later. The CO is asking 'where did the time go?' for one proposal that took 22 minutes. Your current OpenTelemetry instrumentation has a single top-level span for the whole flow. Walk me through the span tree you'd refactor to."*

**What an opening answer should touch:**

- The right unit of instrumentation is **per agent boundary + per LLM call + per retrieval**. A single span tells you total latency and nothing else
- Sketch a span tree: top span = `evaluator-consensus-ssa-flow`; children = `evaluator-agent` (with grandchildren `retrieve-cpar`, `llm-call-claude-sonnet`, `validate-output`); siblings = `consensus-agent` (similar shape); `await-ssa-review` (the interrupt — durations here include human wait time, name it accordingly); `ssa-review-commit`
- Attributes that matter: model ID + version, input token count, output token count, retrieval doc count, retrieval scores, faithfulness score, correlation ID, tenant ID
- The 22-min outlier is most likely *await-ssa-review* — name that distinction; human-wait spans aren't latency bugs, they're business-process spans. The CO needs to see that clearly to ask the right next question

**Likely follow-up presses:**
- *Does this instrumentation work if you swap LangGraph out for a hand-rolled state machine?* (Decouple span attributes from framework; OTel semantic conventions where possible.)
- *What's the cardinality risk on tags?* (Tenant ID + correlation ID are high-card; cost-shape on the observability backend matters.)
- *How do you correlate a faithfulness drop in eval with the OTel traces of the same period?* (Common trace-ID surface; faithfulness as a span attribute, not a separate metric.)

### Example B — Auto-remediation authority

*Supporting reading:* [HITL #7 — auto-remediation authority](../W05/pre-session/3-Wednesday/2-hitl-7-auto-remediation-authority.md) · [AIOps constraints — reversibility & blast radius](../W05/pre-session/3-Wednesday/4-aiops-constraints-reversibility-blast-radius.md) · [AI-SRE pattern walkthrough](../W05/pre-session/3-Wednesday/3-ai-sre-pattern-walkthrough.md)

> *"Your AI-SRE detects that the clause-drafter's faithfulness score has dropped from 92% to 81% over the last 4 hours. The remediation playbook says: route traffic to the v1 of the prompt template. Your AI-SRE has the authority to execute this. Last week, a junior auditor caught the same playbook flipping a prompt rollback that masked a real regression for 3 days. Defend the authority boundary you'd draw."*

**What an opening answer should touch:**

- The HITL #7 framing — auto-remediation authority is **bounded by reversibility + blast radius + observability of side-effects**
- Things AI-SRE can do unsupervised: anything that's reversible inside one decision cycle + within one tenant + that surfaces a clear signal to humans afterward (auto-scale, restart a stuck worker, route around a degraded backend)
- Things AI-SRE should NOT do unsupervised: anything that *changes the model surface to users* (prompt rollback, model swap, retrieval-source change) — those need a human signoff because they mask exactly the signal you'd use to diagnose the underlying regression
- The right boundary: AI-SRE *proposes the remediation*, opens the ticket, gates traffic to a canary, but the *cutover-to-v1-prompt-for-all-users* needs the SRE-on-call to sign
- This isn't bureaucracy — it's the difference between *fixing* a regression and *hiding* it

**Likely follow-up presses:**
- *Who needs to know within the first hour if the AI-SRE has executed an auto-remediation?* (On-call SRE, instructor, eventually the OIG binder if it touches a contracting flow.)
- *What's the audit-log shape for an auto-remediation? You need to be able to reconstruct what the AI-SRE saw and decided.* (Captured signal + threshold + remediation chosen + outcome + human-review timestamp.)
- *What's the OWASP LLM10 (unbounded consumption) angle here?* (Auto-remediation that loops or amplifies the problem.)

### Example C — AWS managed-service trade

*Supporting reading:* [Bedrock Knowledge Bases migration](../W05/pre-session/4-Thursday/3-bedrock-knowledge-bases-migration.md) · [AWS managed-services carve-out](../W05/pre-session/1-Monday/7-aws-managed-services-carve-out.md) · [AIOps platform compare-matrix workshop](../W05/pre-session/4-Thursday/2-aiops-platform-compare-matrix-workshop.md)

> *"Your tech lead is proposing you move from your self-hosted hybrid RAG pipeline (Atlas Vector Search + your own retrieval + your own reranker) to Bedrock Knowledge Bases. The pitch is: less infrastructure, faster onboarding for new corpora, native Bedrock integration. Defend or push back."*

**What an opening answer should touch:**

- This is a **build-vs-buy at the retrieval layer** — same shape as the W2 vector-store ADR, now at a higher abstraction
- What you give up: control over chunking strategy, control over reranker choice, control over embedding model (Bedrock KBs gate the choices), control over the audit-log fields on retrieval, control over the multi-tenant filter contract
- What you gain: less ops surface, AWS-native IAM (one fewer credential to manage), tighter Bedrock integration (potentially lower latency), managed retrieval index lifecycle
- Press the *non-obvious risks*: lock-in to AWS's chunking + retrieval defaults; harder to swap providers later; debugging is shallower (the auditor wants you to recognise this)
- The right answer is rarely "always managed" or "always self-hosted" — it's "for these workloads, managed; for these, self-hosted." Map it to your pair's actual surface

**Likely follow-up presses:**
- *How do you migrate without downtime? Dual-write? Shadow read?*
- *What's the FedRAMP-relevant trade?* (Managed services have a different boundary-of-responsibility footprint; the OIG binder shifts shape.)
- *What does this lock you out of if the cohort's roadmap includes Graph RAG in 6 months?* (Bedrock KBs aren't strong on KG/CG patterns yet; you'd be migrating back out.)

---

## What "meets bar" looks like

For C4 specifically:

1. **Span design instinct** — when the auditor describes a latency problem, you should be able to sketch the span tree before they finish describing it. OpenTelemetry isn't optional context — it IS the conversation
2. **HITL #7 authority defence** — you can draw the line between *what AI-SRE can do unsupervised* and *what needs human signoff*, and defend that line against pressure from both sides ("too much human in the loop" + "too much agent autonomy")
3. **Managed-service trade-off articulacy** — when the auditor proposes a managed-service migration, you can name what you give up, what you gain, and where the lock-in lives. *"Easier to manage"* is not an answer; map it to a specific operational concern
4. **OIG / governance framing** — at least once in the audit, you should be able to anchor an answer in *what the OIG audit binder needs* or *what the FedRAMP boundary looks like*. This is the deployability lens
5. **Final Adversarial Review reference** — if your pair shipped the Fri PR, you should be able to name what blocked, what shipped, and what the load test surfaced

Common stuck points:

- **Treating observability as a feature to add later** — for the audit, observability IS the deployability story. *"We'd add logging"* is below bar
- **Over-promising auto-remediation** — every auto-remediation has a failure mode. If you can't name the failure mode, you're proposing a problem disguised as a fix
- **Forgetting cost** — AIOps platforms, AWS managed services, observability backends all have cost shapes. *"This is more reliable"* is not a complete answer without *"and here's what it costs and why we'd pay it."*
- **Vague governance answers** — the OIG binder is real. The auditor will press on *what exact field, in what exact log, satisfies what exact regulatory ask.* Don't gesture; name

---

## How to prep

1. **Re-walk your W5 ADRs** — Datadog vs alternatives, AWS managed-service migration boundary, HITL #7 authority table. The third one is the load-bearing one for the audit
2. **Sketch your service's full OpenTelemetry span tree from memory** — top-level span down through agent boundaries, LLM calls, retrievals. If you can't draw it without looking, fix that today
3. **Read the Final Adversarial Review PR after-action notes** — what blocked, what shipped, what the load test found. If your pair didn't run the PR, read what the cohort did
4. **Re-read the HITL #7 framing material** — auto-remediation authority is the single most-pressed concept in C4 audits
5. **Cheat-sheet the AWS managed-service trade-offs** — Bedrock KBs, Agents-for-Bedrock, OpenSearch Managed. One paragraph each on what you give up and what you gain
6. **Know one OWASP LLM:2025 deeply through an operations lens** — LLM10 (unbounded consumption) is the most operational; LLM08 (vector/embedding weaknesses) is the most subtle. Pick one, internalise the production-detection story

---

## Logistics

- **Fri 26 Jun is a heavy day** — Final Adversarial Review PR review in the morning + exam early PM, closing out W5. Sleep Thursday matters
- **Audit interview** — async across W6. **W6 runs Mon–Thu only** (Fri 3 Jul is Independence Day observed; lost). Final Defense is **Thu 2 Jul**. Book your audit slot for Tue or Wed; avoid Thu because Final Defense is the same day
- **No retake** — there's no checkpoint after this one. The Deployment Gate / Final Defense is the next assessment surface, and it's a different beast (live presentation + client showcase). If C4 surfaces a gap, the Mon–Wed of W6 is your last window to close it

---

*Brief last verified 2026-06-03. AIOps platform feature parity moves fast — re-verify hot-tech references (Datadog AI Watchdog, Dynatrace Davis AI, New Relic AI Monitoring) against your `research/` briefs if prepping >30 days post-write date.*
