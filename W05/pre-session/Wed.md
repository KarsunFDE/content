---
week: W05
day: Wed
title: "Pre-session — AIOps governance, HITL #7 auto-remediation authority, OWASP LLM continuation, /web-research day mechanics"
audience: All cohort members
time_on_task_minutes: 55
last_verified: 2026-05-23
research_recency_windows: [hot-tech-3mo, federal-regulatory-6mo]
hitl_touchpoint: 7
---

# W5 Wed Pre-Session — AIOps Governance + HITL #7 + Research Day

> Read **before** W5 Wed morning. ~55 min total. Wed is **HITL touchpoint #7 of 7** — the closing of the HITL programme thread. Wed afternoon is the **dedicated research day per D-040** — pairs work all 3 W05 scenario-alternatives prompts.

## 1. HITL #7 — the closing touchpoint (15 min)

The HITL theme has run through the programme as 7 touchpoints (per D-043 + D-044 extension):

| # | Week / Day | Touchpoint | Status |
|---|-----------|-----------|--------|
| 1 | W1 Fri | LLM Engineering Essentials — flagged-output review | Closed W1 |
| 2 | W2 Thu | RAG fallback when retrieval confidence low | Closed W2 |
| 3 | W3 Mon | Plan Day ADR — HITL gates per endpoint | Closed W3 |
| 4 | W3 Wed | Multi-agent handoffs — interrupt nodes | Closed W3 |
| 5 | W3 Thu | LangGraph deep-dive — `interrupt_before` / `interrupt_after` | Closed W3 |
| 6 | W4 Wed | AI Security / OWASP LLM06 framing — excessive agency | Closed W4 |
| **7** | **W5 Wed** | **AIOps auto-remediation authority** | **Today** |

W5 Wed closes the thread. The canonical case (per `training-project/week-dependency-map.md` §W5):

> **"Auto-fix-the-audit-gap" case.** Datadog Watchdog AI detects that an Item 2 race condition just produced an audit gap — the write to `Solicitation` succeeded, the write to `AuditEvent` didn't. The AIOps platform has access to the original request payload (via the OTel trace from Tue's work) and *could* replay the missing AuditEvent write atomically. Three candidate platform behaviours:
>
> - **(A) Full auto** — auto-replay the missing AuditEvent without human gate.
> - **(B) Propose-and-await-approval** — generate a fix PR + page sys_admin; merge after human review.
> - **(C) Escalate-only** — page on-call human; the platform does not touch data.

Wed morning's war-room produces an **ADR per pair** committing one of A/B/C with rationale. The decision affects: FedRAMP audit posture (who has authority to write the audit table?), reversibility (is the replay reversible if wrong?), trust model (does the AI ever write the audit table without a human?). **This decision lands in the pair's Fri PR as a load-bearing ADR.**

[`skills/aiops-curriculum/references/ai-sre-patterns.md` §6 Auto-remediation, in-repo]
[`training-project/feature-inventory-target.md` Item 2 (audit-log race), in-repo]

## 2. AI-SRE Pattern Walkthrough — what you'll see Wed morning (10 min)

Wed war-room walks all six AI-SRE patterns against `acquire-gov`. Read `skills/aiops-curriculum/references/ai-sre-patterns.md` end-to-end before Wed. Map each pattern to an `acquire-gov` example:

| Pattern | `acquire-gov` example you'll see Wed |
|---------|--------------------------------------|
| Investigation | Datadog timeline reconstruction of yesterday's `/draft-solicitation` request that confabulated a CFR clause |
| Correlation | Latency spike correlated with Atlas Vector Search index rebuild + FAR-clause batch update |
| Root-cause | Embedding drift after agency-supplement clauses were ingested without re-tokenization |
| Fix-PR generation | AI-SRE proposes a re-tokenization PR + regression test (NOT auto-merged) |
| Runbook augmentation | Runbook entry for "retrieval quality dropping over days" gains a new section |
| **Auto-remediation** | **HITL #7 case (above)** |

Pattern 6 is where Wed's war-room ADR work happens.

## 3. OWASP LLM Top 10 continuation — AIOps-discoverable risks (10 min)

W4 Wed lit OWASP LLM Top 10 (continuation of LLM06 Excessive Agency for HITL #6). W5 Wed extends with the categories AIOps observability specifically helps with:

**LLM05 Improper Output Handling.** OTel spans on the 4 AI endpoints capture *what the model returned*. AIOps alerting fires when output type drift exceeds threshold (e.g., `/eval/ssdd-draft` returning prose instead of structured JSON). Per `skills/codex-adversarial-review/references/owasp-llm-top-10.md`:

> *"Flag example. `subprocess.run(model_output)` or `cursor.execute(model_output)` anywhere in a diff."*

`acquire-gov`'s **Item 4** (no structured output validation across 4 distinct AI endpoints) is the LLM05 surface. AIOps detects the drift; the fix is the Pydantic schema gate that lands in Fri's PR.

**LLM09 Misinformation.** AIOps surfaces hallucination signals — citation pattern mismatch, RAG retrieval k=0 results that still got a confident response. The Datadog LLM Observability product treats this as a first-class signal. `acquire-gov`'s `/answer-qa` (no citation grounding) is the LLM09 surface.

**LLM10 Unbounded Consumption.** Tue's `gen_ai.usage.*` instrumentation surfaces cost amplification — agent loops, retrieval fan-out, prompt-stuffing. Item 3's retry-storm scenario (evaluator-agent → solicitation-service without circuit breaker) is the canonical LLM10 reproducer. The Fri PR's Resilience4j circuit breaker is the mitigation.

[OWASP LLM Top 10 for 2025, retrieved 2026-05-23, https://owasp.org/www-project-top-10-for-large-language-model-applications/]
[`skills/codex-adversarial-review/references/owasp-llm-top-10.md` (in-repo curated reference)]

## 4. Research day mechanics per D-040 (8 min)

D-040 locked **Wednesday as the dedicated scenario-alternatives research day** — 2–3 scenarios × 2–3 techs each, pair-collaborative, dedicated practical block. W5 has 3 scenario prompts:

- **W05-SA-1** — AIOps platform compare-set (Datadog vs Dynatrace vs New Relic vs Coralogix). 4 techs.
- **W05-SA-2** — Auto-remediation authority (Full Auto vs Propose-and-Approve vs Escalate-only). 3 techs.
- **W05-SA-3** — Bedrock managed-services migration cost-benefit (direct InvokeModel vs Knowledge Bases vs Agents-for-Bedrock). 3 techs.

Pair splits the research work; comes back together with one consolidated comparative ADR per scenario by EOD Wed. **All three feed Thu's compare-matrix workshop + Fri's PR.**

All research routes through `/web-research` per `pipeline/RESEARCH-PROTOCOL.md` D-046. **Recency windows applied:**
- AIOps platforms (W05-SA-1) — hot-tech 3-month.
- Auto-remediation authority (W05-SA-2) — hot-tech 3-month + federal-regulatory 6-month (FedRAMP AU authority controls).
- Bedrock managed services (W05-SA-3) — hot-tech 3-month (AWS ships Bedrock features monthly).

## 5. AIOps governance ADR template (5 min)

Wed afternoon's research also produces governance ADRs your Fri PR will reference. Per `templates/scenario-alternatives.md` shape, each ADR commits:

- **Decision** — chosen platform / authority level / migration target.
- **Context** — what `acquire-gov` need this serves.
- **Alternatives** — at least 2 considered, not dismissed.
- **Consequences** — what could go wrong.
- **Rollback** — migration cost if the decision is wrong.

Specifically for HITL #7's authority ADR, add:

- **FedRAMP boundary** — does the chosen authority cross AC-2 (account management) or AU-9 (audit log integrity) controls? [FedRAMP Rev 5 controls, retrieved 2026-05-23, https://www.fedramp.gov/rev5/]
- **Reversibility analysis** — what if the auto-remediation fires incorrectly?
- **Audit-of-the-audit** — every auto-remediation action is itself an AuditEvent. The platform doesn't get a silent action.

## 6. Wed afternoon target (2 min)

EOD Wed: 3 consolidated comparative ADRs per pair (W05-SA-1, W05-SA-2, W05-SA-3) + the HITL #7 authority ADR (more rigorous than the scenario-alternatives; this one is load-bearing for Fri's PR). Thu morning's compare-matrix workshop assumes all four exist.

---

## Sources (all retrieved 2026-05-23 via `/web-research`)

- `pipeline/DECISIONS.md` D-040 + D-043 + D-046 (in-repo)
- `skills/aiops-curriculum/references/ai-sre-patterns.md` (in-repo)
- `skills/codex-adversarial-review/references/owasp-llm-top-10.md` (in-repo curated)
- `training-project/feature-inventory-target.md` (in-repo, for Item 2 + Item 3 + Item 4 surfaces)
- OWASP LLM Top 10 for 2025 — https://owasp.org/www-project-top-10-for-large-language-model-applications/ (hot-tech 3-month — 2025 list current as of 2026-05-23; OWASP refreshes every 1–2 years)
- FedRAMP Rev 5 controls — https://www.fedramp.gov/rev5/ (federal-regulatory 6-month window)
- W3C Trace Context — https://www.w3.org/TR/trace-context/ (foundation-stable, referenced for HITL #7 context)
