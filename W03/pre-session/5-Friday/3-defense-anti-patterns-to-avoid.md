---
week: W03
day: 5-Friday
topic_slug: defense-anti-patterns-to-avoid
topic_title: Defense Anti-Patterns to Avoid
parent_overview: W03/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://martinfowler.com/articles/exploring-gen-ai/i-still-care-about-the-code.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://martinfowler.com/articles/exploring-gen-ai/humans-and-agents.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://github.com/joelparkerhenderson/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://staffeng.com/guides/staff-archetypes/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://lethain.com/getting-in-the-room/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-06-06
---

# Defense Anti-Patterns to Avoid

> [!NOTE]
> **From earlier:** Mon HITL #3: every ADR cites a source. Thu HITL #5: show the `interrupt()` call and the audit row, not "we wired a hard gate." Those standards apply to every panel answer today.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the five defense anti-patterns and recognize each in their own draft answers.
- Distinguish owning a decision from explaining one someone else made.
- Lead every answer with the load-bearing fact before any framing.
- Use the Phase-1 Defense Rubric (Sec 3.2) to self-assess readiness before the panel opens.

## 2. Introduction

The defense is where someone asks "why did you do it this way?" — and your answer is the artifact being evaluated. Reviewers are not testing whether the system works in the demo. They are testing whether *you* understand the trade-offs you made and would catch the failure mode they are about to ask about.

AI-assisted work is hard to defend because the artifact wasn't produced line-by-line by the person defending it. Böckeler (Thoughtworks) frames every AI-accepted change as a bet — impact × probability × detectability. The defense makes that bet legible. The same five mistakes recur; internalize them now.

## 3. Core Concepts

### 3.1 The five defense anti-patterns

| # | Anti-pattern | Tell | Counter-move |
|---|-------------|------|-------------|
| 1 | Defending decisions you don't own | "We decided" — but the decision was your partner's or a model suggestion you accepted | "My partner owned the framework choice; I owned the retrieval layer" — calibrated ownership beats performed omniscience |
| 2 | Citing sources you haven't read | You drop FAR 15.308 or OWASP LLM06 and can't answer the follow-up | Paraphrase the principle and name the gap: "I haven't read the clause, but the discipline we applied was X" |
| 3 | Buried-the-lede answers | Asked "biggest production risk?" — two minutes of framing before naming the risk | First sentence carries the lede: "The biggest risk is stale documents ranking above fresh in 3% of queries" |
| 4 | Pretending the work was clean | Sanitized story — the project went roughly to plan | "The filter took 3 days not 1.5 — RLS edge case. Here's what we learned." Discovery named = competency shown |
| 5 | Skipping the evidence | Traces, eval reports, ADR docs, audit rows — all in the folder, none on screen | Lead with the LangSmith screenshot; verbal description supports the evidence, doesn't replace it |

> [!TIP]
> Before the panel: tag each planned answer with its anti-pattern risk and write the counter-move.

### 3.2 Phase-1 Defense Rubric

**Scoring instrument — walk in with this.**

| Dimension | 5/5 — Strong pass | 2/5 — Weak / finding |
|-----------|-------------------|----------------------|
| **HITL touchpoint discipline** — #3 ADR (W3 Mon), #4 multi-agent handoff (W3 Wed), #5 formal `interrupt()` (W3 Thu) | Shows code + audit row for each; explains what each gate protects and which FAR clause anchors it | Names touchpoints but can't show implementation; claims "HITL is wired" without evidence; conflates soft and hard interrupts |
| **ADR citation quality** | Every ADR cites a `/web-research` URL or named clause; can answer follow-up on cited source content | ADRs cite only internal discussion; cites a regulation but can't state what it says; no ADR log present |
| **Framework primitive correctness** — LangGraph v1.0, LangChain v1.0, Bedrock InvokeModel | `Command(resume=...)` via `graph.invoke()` for HITL resume; no `Chain` class; no LCEL `\|` pipes as fundamental; correct Bedrock model IDs | LangGraph v0.x `.run()` resume; `Chain` class or LCEL pipe advocated as core (D-033 violation); stale Bedrock model IDs |
| **Evaluator → consensus → SSA handoff** | Shows topology end-to-end: evaluator nodes score, consensus aggregates, SSA gate fires at handoff boundary; LangSmith trace screenshot present | Describes topology verbally only; no trace; handoff boundary implicit; no audit row for the handoff event |
| **Audit-row schema with FAR citations** | `hitl_events` rows include `event_type`, `agent_id`, `tenant_id`, `far_clause` (e.g., `FAR 15.308`), `decision`, `timestamp`; can explain what each field defends in an OIG review | Rows exist but lack FAR clause field; can't explain field purpose; no audit rows for HITL events |

> [!IMPORTANT]
> **Tier calibration.** Senior FDEs: production-shape bar (cost, on-call, FedRAMP, scale). Entry FDEs: applied-recognition bar (name the pattern, recognize the failure mode, explain the trade-off, escalate when out of depth). Different bars; same rubric.

## 4. Generic Implementation

Same Q&A, two ways. Setting: fintech engineering review of an AI-assisted fraud-detection feature.

```
ANTI-PATTERN (buries lede, unread citation, skips evidence):
"We follow industry standard practices around evaluation. There's
some research we used as guidance — I believe it was a Stripe paper
— and we ran a lot of tests. Overall the false-negative rate is
acceptable."

COUNTER-MOVE (leads with case, reads what it cites, shows evidence):
"Worst case: card-not-present under $35 from a new merchant — the
rules engine fires first, the LLM doesn't weigh in. Two false
negatives last quarter, both chargebacks. Eval report on screen:
4.1% false-negative rate vs. 1.2% target. Defense-in-depth — I
haven't read the specific OWASP page, that was our security lead's
call — but the trade-off was bounding LLM latency vs. catching
marginal cases."
```

The counter-move's first sentence carries the lede; everything else supports it. The anti-pattern's lede never lands.

## 5. Real-world Patterns

**Healthcare: FDA pre-submission defense.** An AI-assisted radiology triage team required every 510(k) citation to be read — not summarized — before the pre-submission consultation. They built a cite-and-read tracker. The reviewer's first probe was a follow-up on a cited precedent. They answered cleanly.

**Fintech: model-risk-management.** A US bank's LLM credit-decisioning submission was rejected because validators found production cases quietly fixed without naming the failure class. Discovery named = competence. Discovery hidden = finding.

> [!NOTE]
> Both patterns map to anti-pattern 4. The team knew the discovery and chose not to surface it.

## 6. Best Practices

- Build an "owned vs. explained" list; rehearse the owned column aloud.
- Read every source you intend to cite — the actual section, not a summary.
- Rehearse leading with the failure-shaped sentence; have a partner cut you off when you bury the lede.
- Assemble the evidence pack (traces, eval reports, ADR list, audit-row snapshots) as named files you can open in five seconds.

> [!WARNING]
> **Anti-patterns: `defense-no-citation` + `hand-wave-hitl`.** (1) "Standard practice" without a `/web-research` citation — OIG needs a URL or clause number. (2) "We implemented HITL #5" without showing `interrupt_before=['ssa_approval']` and the `hitl_events` row. Both root cause: asserting without evidence.

## 7. Hands-on Exercise

**Mock-defense — 10 minutes, pair format.** Partner A presents (3 min). Partner B asks in order and notes which anti-pattern each answer commits:

1. "Worst failure mode in production today?"
2. "Cite one source that informed a key decision — what does it say?"
3. "Walk me through a trade-off and tell me who owned it."

Under 90 seconds each. Debrief, swap.

> [!NOTE]
> **Self-check** (30s — answer mentally before expanding)
>
> 1. Which of the five anti-patterns maps directly to the HITL #3 ADR discipline, and what is the tell?
> 2. In the buried-lede anti-pattern, what is the structural fix the counter-move applies?

<details>
<summary>Show answers</summary>

1. Anti-pattern 2 (Citing sources you haven't read). The tell: you cite a standard in an ADR but cannot answer "what does that source actually say?" — exactly what the OIG Q&A probes.

2. Make the first sentence carry the lede: state the risk or trade-off before any context. Everything that follows supports the opening sentence rather than building toward it.

</details>

## 8. Key Takeaways

- Five anti-patterns: defending decisions you don't own, citing unread sources, burying the lede, pretending the work was clean, skipping the evidence.
- Each collapses *detectability* — actual risk never reaches the reviewers who need it.
- Own vs. explain: mis-claiming ownership is the most common individual failure mode.
- Evidence leads: traces, eval reports, ADRs, and audit rows beat verbal recall every time.
- Walk in with the Sec 3.2 rubric — it is the scoring instrument; find gaps now, not during the panel.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://martinfowler.com/articles/exploring-gen-ai/i-still-care-about-the-code.html — retrieved 2026-05-26 — hot-tech
- https://martinfowler.com/articles/exploring-gen-ai/humans-and-agents.html — retrieved 2026-05-26 — hot-tech
- https://github.com/joelparkerhenderson/architecture-decision-record — retrieved 2026-05-26 — foundation-stable
- https://staffeng.com/guides/staff-archetypes/ — retrieved 2026-05-26 — foundation-stable
- https://lethain.com/getting-in-the-room/ — retrieved 2026-05-26 — foundation-stable

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

**The OIG audit model.** OIG reviewers apply a reconstruction test drawn from OMB Circular A-123: can the system's behavior be reconstructed from the audit record alone, without relying on the engineer's testimony? That test is why the audit-row schema matters more than accuracy metrics in an OIG Q&A.

**Kief Morris's "on-the-loop" framing.** The defender's job is to demonstrate they supervised the AI's loop — not that they reviewed every token. The evidence: eval reports that ran on schedule, HITL interrupts that fired with documented decisions, ADRs that record rationale for configuration choices. If none of those exist, the panel finds the gap.

**Staff archetypes and ownership calibration.** Will Larson's taxonomy (Tech Lead, Architect, Solver, Right Hand) maps cleanly: own the architectural decision you were Tech Lead or Architect on; explain implementation decisions you delegated; explicitly name what you don't know and how you'd discover it. Interviewers who know the taxonomy reward that calibration.

</details>

Last verified: 2026-06-06
