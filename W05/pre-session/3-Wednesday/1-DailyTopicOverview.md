---
week: W05
day: Wed
title: "Pre-session — AIOps governance, HITL #7 auto-remediation authority, AIOps constraints, OWASP LLM continuation, /web-research day mechanics"
audience: All cohort members
time_on_task_minutes: 60
last_verified: 2026-05-26
research_recency_windows: [hot-tech-3mo, federal-regulatory-6mo]
hitl_touchpoint: 7
---

# W5 Wed Pre-Session — AIOps Governance + HITL #7 + Research Day

> Read **before** W5 Wed morning. ~60 min total. Wed is **HITL touchpoint #7 of 7** — the closing of the HITL programme thread. Wed afternoon is the **dedicated `/web-research` day per D-040** — pairs work all 3 W05 scenario-alternatives prompts. Every topic below is Karsun-applied to `acquire-gov` so your morning ADR + Fri PR are grounded in the brownfield you already know.

## 1. HITL #7 — Auto-remediation authority, the closing touchpoint (15 min)

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

W5 Wed closes the thread. The canonical case (per `training-project/week-dependency-map.md` §W5 and Wed war-room Act 2):

> **"Auto-fix-the-audit-gap" case.** At 03:14 last night Datadog Watchdog AI detected an Item 2 race condition on `acquire-gov`. The CO published solicitation SOL-2026-0421 — the `Solicitation` table write committed; the `AuditEvent` insert required by FedRAMP AU-2 did not (SIGKILL between commits; OOM on ai-orchestrator mid-large-prompt for an unrelated request). The platform has the original request payload via Tue's OTel trace and *could* replay the missing `AuditEvent` atomically with a `replayed_by_ai_sre: true` flag. The on-call sys_admin is asleep. The CO is offline. The OIG audit window opens in 6 weeks. Three candidate platform behaviours:
>
> - **(A) Full Auto** — auto-replay the missing AuditEvent without human gate.
> - **(B) Propose-and-Await-Approval** — generate a fix PR + page sys_admin; merge after human review.
> - **(C) Escalate-only** — page on-call human; the platform does not touch data.

Wed morning's war-room produces an **ADR per pair** committing one of A/B/C with rationale, using `weeks/W05/hitl-7-authority-adr-template.md`. The decision affects: FedRAMP audit posture (who has authority to write the audit table?), reversibility (is the replay reversible if wrong?), trust model (does the AI ever write the audit table without a human?). **This decision lands in the pair's Fri PR as a load-bearing ADR.**

No option is objectively right. The ADR's quality lives in the reasoning + the boundary-naming. *"It depends"* is not an ADR — pick a position, defend it.

[`skills/aiops-curriculum/references/ai-sre-patterns.md` §6 Auto-remediation, in-repo, retrieved 2026-05-26]
[`training-project/feature-inventory-target.md` Item 2 (audit-log race), in-repo, retrieved 2026-05-26]
[NIST 800-53 AU-9 (Protection of Audit Information), retrieved 2026-05-26 via `/web-research`, https://csrc.nist.gov/projects/risk-management/sp800-53-controls/release-search#!/control?version=5.1&number=AU-9]

## 2. AI-SRE Pattern Walkthrough — 6 patterns mapped to `acquire-gov` (10 min)

Wed war-room walks all six AI-SRE patterns against `acquire-gov`. Read `skills/aiops-curriculum/references/ai-sre-patterns.md` end-to-end before Wed. Map each pattern to a Karsun-applied example you'll see live on the projector:

| Pattern | `acquire-gov` example you'll see Wed |
|---------|--------------------------------------|
| Investigation | Datadog Bits AI timeline reconstruction of yesterday's `/draft-solicitation` request that confabulated a CFR clause (W1 hallucination scenario revisited with full Tue observability) |
| Correlation | Watchdog AI auto-correlates evaluation-service latency spike Tuesday 14:30 with a solicitation-service config push at 12:30 (connection-pool sizing change) |
| Root-cause | Embedding-drift on the FAR/DFARS Atlas Vector Search index after agency-supplement clauses were ingested without re-tokenization — Bits AI calls out the *process gap*, not a model regression |
| Fix-PR generation | Bits AI proposes a re-tokenization PR + regression test for the ingestion pipeline; PR is NOT auto-merged — human reviews, edits, merges. This is HITL #7 Option B in code. |
| Runbook augmentation | After the embedding-drift incident, Bits AI proposes a new section in `acquire-gov/runbook.md` ("retrieval quality dropping over days → check tokenizer per batch") — the muscle that fills W6 Mon's deliverable runbook |
| **Auto-remediation** | **HITL #7 case (above) — Pattern 6 is where federal-context governance gets serious** |

Pattern 4 is what distinguishes AI-SRE from traditional AIOps — the platform drafts a fix; the human merges. Pattern 6 is where the cohort's W5 ADR work happens.

[Bits AI in live incidents, Datadog blog, retrieved 2026-05-26 via `/web-research`, https://www.datadoghq.com/blog/bits-ai-incident-management/]
[`skills/aiops-curriculum/references/ai-sre-patterns.md` (in-repo curated taxonomy)]

## 3. AIOps Constraints — FedRAMP AU-9 + reversibility + blast-radius (8 min)

Before you commit a HITL #7 position, name the constraints that bound *any* AIOps decision on a federal-acquisitions surface. These are the dimensions Wed's ADR must reason against — not after-the-fact lenses.

**FedRAMP AU-9 (Protection of Audit Information).** The audit table is a controlled artifact. AU-9 requires audit information be protected from unauthorized modification — including *writes* by parties without authority. Option A (Full Auto) asks: does the AI-SRE have AU-9-aligned authority? Option B asks: does proposed-with-human-merge satisfy AU-9? Option C asks: is the only AU-9-defensible write the human's?

**Reversibility.** Every auto-remediation must answer *"what if this fires wrongly?"* An auto-replayed AuditEvent with `replayed_by_ai_sre: true` is reversible (delete the row, page human, audit the action). An auto-applied schema migration on `Solicitation` is not. The Wed ADR names which side of the reversibility line your chosen option lives on.

**Blast-radius on the cohort-paid AWS account.** Per D-050, the cohort runs on a single Karsun-paid AWS account this week — no per-tenant boundary in infra. An auto-remediation that mis-fires affects every tenant served by the account. Wed's ADR must call this out: production-grade options pre-suppose tenant-bounded blast-radius your demo environment doesn't have.

**Vendor lock-in.** Datadog Bits AI is the hands-on platform per D-060. Whatever authority you grant the AI-SRE *here* is a contract with *this vendor*. Thu's compare-matrix forces the question: does your chosen authority level transfer to Dynatrace, New Relic, Coralogix? If not, you've over-coupled.

**Audit-of-the-audit.** Every auto-remediation is itself an `AuditEvent`. The AI-SRE doesn't get a silent action — the platform's *own* writes are first-class audit rows the OIG sees in the binder. Your ADR commits the schema for that row (who acted, what acted, what the trace showed, what the human-readable summary is).

[FedRAMP Rev 5 baseline (AU-9, AC-2, AC-3), retrieved 2026-05-26 via `/web-research`, https://www.fedramp.gov/rev5/]
[NIST 800-53 Rev 5.1 (AU-9 Protection of Audit Information), retrieved 2026-05-26 via `/web-research`, https://csrc.nist.gov/projects/risk-management/sp800-53-controls/release-search]

## 4. OWASP LLM Top 10 continuation — LLM05 / LLM09 / LLM10 as AIOps-discoverable risks (10 min)

W4 Wed lit OWASP LLM Top 10 (continuation of LLM06 Excessive Agency for HITL #6). W5 Wed extends with the categories AIOps observability specifically helps with. The 2025 list is current; OWASP refreshes every 1–2 years.

**LLM05 Improper Output Handling.** OTel spans on the 4 AI endpoints capture *what the model returned*. AIOps alerting fires when output type drift exceeds threshold (e.g., `/eval/ssdd-draft` returning prose instead of structured JSON). Per `skills/codex-adversarial-review/references/owasp-llm-top-10.md`:

> *"Flag example. `subprocess.run(model_output)` or `cursor.execute(model_output)` anywhere in a diff."*

`acquire-gov`'s **Item 4** (no structured output validation across 4 distinct AI endpoints) is the LLM05 surface. AIOps detects the drift; the Pydantic schema gate lands in Fri's PR as the mitigation.

**LLM09 Misinformation.** AIOps surfaces hallucination signals — citation pattern mismatch, RAG retrieval `k=0` results that still got a confident response. Datadog LLM Observability treats this as a first-class signal. `acquire-gov`'s `/answer-qa` (no citation grounding — Item 9 in the W4-surfaced inventory) is the LLM09 surface. Fri PR re-asserts the W2 citation-token requirement.

**LLM10 Unbounded Consumption.** Tue's `gen_ai.usage.*` instrumentation surfaces cost amplification — agent loops, retrieval fan-out, prompt-stuffing. Item 3's retry-storm scenario (evaluator-agent → solicitation-service without circuit breaker) is the canonical LLM10 reproducer. The Fri PR's Resilience4j circuit breaker + per-tenant token budget mitigates LLM10. Wed afternoon you wire the Watchdog AI watch on the Tue cost dashboard so the alert fires before the next spike.

Each pair commits which 3+ OWASP LLM categories the Fri PR will mitigate. Capture in your pair's pre-research scratch doc Wed AM.

[OWASP Top 10 for LLM Applications 2025, retrieved 2026-05-26 via `/web-research`, https://genai.owasp.org/llm-top-10/]
[`skills/codex-adversarial-review/references/owasp-llm-top-10.md` (in-repo curated reference)]

## 5. Research Day mechanics per D-040 — 3 scenario prompts (8 min)

D-040 locked **Wednesday as the dedicated scenario-alternatives research day** — 2–3 scenarios × 2–3 techs each, pair-collaborative, dedicated practical block. W5 has 3 scenario prompts (all under `weeks/W05/scenarios/`):

- **W05-SA-1** — AIOps platform compare-set (Datadog vs Dynatrace Davis AI vs New Relic AI Monitoring vs Coralogix AI Observability). 4 techs.
- **W05-SA-2** — Auto-remediation authority (Full Auto vs Propose-and-Approve vs Escalate-only). 3 techs.
- **W05-SA-3** — Bedrock managed-services migration cost-benefit (direct `InvokeModel` vs Bedrock Knowledge Bases vs Agents-for-Bedrock). 3 techs.

Pair splits the research work; comes back together with one consolidated comparative ADR per scenario by EOD Wed. **All three feed Thu's compare-matrix workshop + Fri's PR.**

All research routes through `/web-research` per `pipeline/RESEARCH-PROTOCOL.md` D-046. **If `/web-research` is unavailable, surface to instructor — do NOT fall back to model-internal knowledge.** Model knowledge fails the rubric on Fri.

**Recency windows applied:**
- AIOps platforms (W05-SA-1) — hot-tech 3-month (AIOps vendors ship monthly).
- Auto-remediation authority (W05-SA-2) — hot-tech 3-month + federal-regulatory 6-month (FedRAMP AU/AC controls).
- Bedrock managed services (W05-SA-3) — hot-tech 3-month (AWS ships Bedrock features monthly).

**Sub-30-day source HITL gate.** If your `/web-research` surfaces a source dated within the last 30 days, that's a HITL gate per `cohort1-locked-decisions.md` — surface to instructor for adopt/defer/omit decision *before* including. Do not auto-magic recent sources into your ADRs.

[`pipeline/DECISIONS.md` D-040 + D-046 (in-repo)]
[`pipeline/RESEARCH-PROTOCOL.md` (in-repo, recency windows + citation discipline)]

## 6. AIOps Governance — ADR template + FedRAMP overlay (5 min)

Wed afternoon's research produces governance ADRs your Fri PR will reference. Per `templates/scenario-alternatives.md` shape, each ADR commits:

- **Decision** — chosen platform / authority level / migration target.
- **Context** — what `acquire-gov` need this serves (anchor to W4 surprise + brownfield item number).
- **Alternatives** — at least 2 considered, not dismissed by hand-wave.
- **Consequences** — what could go wrong; what's the failure mode.
- **Rollback** — migration cost if the decision is wrong.

Specifically for HITL #7's authority ADR (use `hitl-7-authority-adr-template.md`, not the generic scenario template — it's more rigorous), add:

- **FedRAMP boundary** — does the chosen authority cross AC-2 (account management), AC-3 (access enforcement), or AU-9 (audit log integrity) controls?
- **Reversibility analysis** — what if the auto-remediation fires incorrectly? What's the un-do path?
- **Audit-of-the-audit** — schema for the `AuditEvent` row the auto-remediation itself writes (per §3 constraint).
- **Authority delegation** — who in the org grants this authority? CO? sys_admin? FedRAMP AO? Name them.

[FedRAMP Rev 5 (AC-2, AC-3, AU-9), retrieved 2026-05-26 via `/web-research`, https://www.fedramp.gov/rev5/]
[`weeks/W05/hitl-7-authority-adr-template.md` (in-repo)]
[`templates/scenario-alternatives.md` (in-repo)]

## 7. What's load-bearing for Fri's PR — the Wed ADR commit (4 min)

Wed produces four deliverables Thu and Fri reference. **None are optional; all land in the Fri Final Adversarial Review PR (v2's replacement for v1's Exit Technical Exam).**

1. **HITL #7 authority ADR** (per pair) — A/B/C committed with rationale. The Fri PR cites this ADR as the policy basis for *whatever auto-remediation behavior* your circuit-breaker / outbox / OTel wiring implements.
2. **3 consolidated comparative ADRs** (W05-SA-1 / SA-2 / SA-3) — feed Thu's compare-matrix workshop; the worth-it ADRs Thu authors are downstream of these.
3. **OWASP LLM Top 10 mitigation plan** — 3+ categories your Fri PR addresses, with the specific `acquire-gov` surface each maps to (Item 3 → LLM10; Item 4 → LLM05; `/answer-qa` → LLM09; etc.).
4. **Watchdog AI watches configured** on the Tue cost dashboard — at least one LLM10 detection rule live; one drift detection rule live (Tue's `gen_ai.*` semconv enables both).

If you're behind by EOD Wed: the HITL #7 ADR is the *one* thing you cannot defer. Thu morning's compare-matrix can absorb a half-done W05-SA-{1,2,3}; Fri's PR cannot absorb a missing HITL #7 ADR.

---

## Sources (all retrieved 2026-05-26 via `/web-research`)

- `pipeline/DECISIONS.md` D-040 + D-043 + D-044 + D-046 + D-050 + D-060 (in-repo)
- `pipeline/RESEARCH-PROTOCOL.md` (in-repo, recency windows + citation discipline)
- `skills/aiops-curriculum/references/ai-sre-patterns.md` (in-repo curated taxonomy)
- `skills/codex-adversarial-review/references/owasp-llm-top-10.md` (in-repo curated reference)
- `training-project/feature-inventory-target.md` (in-repo, for Item 2 + Item 3 + Item 4 + Item 6 surfaces)
- `weeks/W05/hitl-7-authority-adr-template.md` (in-repo)
- `weeks/W05/scenarios/W05-SA-{1,2,3}.md` (in-repo)
- OWASP Top 10 for LLM Applications 2025 — https://genai.owasp.org/llm-top-10/ (hot-tech 3-month — 2025 list current; OWASP refreshes every 1–2 years)
- FedRAMP Rev 5 baseline (AC-2, AC-3, AU-9) — https://www.fedramp.gov/rev5/ (federal-regulatory 6-month window)
- NIST 800-53 Rev 5.1 controls (AU-9 Protection of Audit Information) — https://csrc.nist.gov/projects/risk-management/sp800-53-controls/release-search (federal-regulatory 6-month window)
- Bits AI in live incidents — https://www.datadoghq.com/blog/bits-ai-incident-management/ (hot-tech 3-month — Datadog ships Bits AI features monthly; re-verify pre-cohort)
- W3C Trace Context recommendation — https://www.w3.org/TR/trace-context/ (foundation-stable, referenced for HITL #7 context)

Last verified: 2026-05-26
