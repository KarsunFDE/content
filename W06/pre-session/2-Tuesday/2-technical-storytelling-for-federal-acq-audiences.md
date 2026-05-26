---
week: W06
day: Tue
topic_slug: technical-storytelling-for-federal-acq-audiences
topic_title: "Technical storytelling for federal-acq audiences"
parent_overview: W06/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://www.acquisition.gov/far/15.308
    retrieved_on: 2026-05-26
    recency_category: federal-regulatory
  - url: https://www.codewithsense.com/blog/technical-storytelling-for-engineering-leaders-aligning-vision-with-business-strategy
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://sloanreview.mit.edu/article/become-a-better-problem-solver-by-telling-better-stories/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://oig.federalreserve.gov/audits-what-we-do.htm
    retrieved_on: 2026-05-26
    recency_category: federal-regulatory
  - url: https://www.harness.io/blog/the-executive-playbook-communicating-engineering-metrics-for-maximum-business-impact
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Technical storytelling for federal-acq audiences

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish a *controls-language* technical story from a *capability-language* one, and explain why the former lands better with audit-oriented audiences.
- Identify the three structural moves that make a technical story defensible (clause-anchor → control-frame → declared-weakness).
- Pick the right audience register (executive / manager / auditor) for a given storytelling moment and adjust depth + emphasis accordingly.
- Apply a "broken-but-named-broken" disclosure pattern to known limitations rather than burying them.
- Critique a peer's draft technical story for the three most common failure modes (cleverness-over-defensibility, hidden weakness, register mismatch).

## 2. Introduction

Technical storytelling, in industry generally, is the practice of wrapping a technical decision in narrative scaffolding so non-technical decision-makers can act on it. Engineering-leadership writers describe it as "aligning vision with business strategy" — the engineer presents *not* the cleverness of an implementation, but the path from a business problem through a technical choice to a measurable outcome ([CodeWithSense, retrieved 2026-05-26](https://www.codewithsense.com/blog/technical-storytelling-for-engineering-leaders-aligning-vision-with-business-strategy)). MIT Sloan frames it more bluntly: "numbers inform; stories transform" — the same data point lands very differently inside a narrative arc than as a slide bullet ([MIT Sloan Review, retrieved 2026-05-26](https://sloanreview.mit.edu/article/become-a-better-problem-solver-by-telling-better-stories/)).

The audit-oriented variant of this discipline is the one this reading is about. When the audience is responsible for *signing off* on a system — a regulator, an internal auditor, a board reviewing risk — the story they need is not the same story you tell a product VP. The product VP wants to know the system *enables* something. The auditor wants to know what the system *constrains* and how that constraint is *verifiable*. Same facts, different shape.

The bridge concept is **controls language vs capability language**. Capability language describes what the system *can do* ("the model summarises the customer note"). Controls language describes what the system *will not do without human approval*, what evidence trail exists when it does act, and which clause or policy authorises the boundary ("the summariser writes to a draft area only; publication requires an authorised reviewer; every publication leaves an audit row with the reviewer's identity"). The capability framing is true; the controls framing is *defensible*. Audit-oriented audiences hear both, but they make decisions on the second.

This reading covers the discipline generically. The day's overview applies it to the W6 Tue stakeholder-rotation exercise; treat this reading as the underlying technique.

## 3. Core Concepts

### 3.1 The three structural moves

A defensible technical story makes three moves in this order:

1. **Clause-anchor.** Open by naming the rule, contract, policy, or regulatory clause that constrains the decision. The auditor's first question is always "what is the basis?" — answering it before they ask shifts the conversation from interrogation to confirmation.
2. **Control frame.** Describe the system's behaviour as a *control* (a verifiable constraint) rather than a *capability* (a feature). "The system requires reviewer approval before publishing" is a control; "the system lets reviewers approve before publishing" is a capability. The first asserts a boundary; the second describes a workflow option.
3. **Declared weakness.** End by naming what the control does *not* cover, where coverage is shallow, and what the remediation plan is. Auditors find weaknesses; the only question is whether *you* found them first.

The federal-acquisitions example case is FAR 15.308, which requires that a Source Selection Decision "represent the SSA's independent judgment" and that the rationale be documented in an SSDD ([Acquisition.gov FAR 15.308, retrieved 2026-05-26](https://www.acquisition.gov/far/15.308)). A technical story about an AI-assisted source-selection tool that opens with "we built an AI evaluator" fails. A technical story that opens with "FAR 15.308 requires the SSA's independent judgment — so our evaluator produces *drafts* the SSA reviews, never decisions; the SSDD always carries the SSA's signature and a record of the AI's contribution" passes. Same system, different story shape.

### 3.2 Why capability framing fails audit audiences

Capability framing creates an unbounded surface area. If the story says "the model can summarise notes," the auditor's next question is "can it also write notes? approve notes? delete notes?" Each answer opens new questions. Controls framing inverts this: by naming what the system *will not do without approval*, you constrain the surface area to what is enumerable and testable ([Federal Reserve OIG, retrieved 2026-05-26](https://oig.federalreserve.gov/audits-what-we-do.htm) — OIGs assess "the effectiveness of internal controls" as a primary lens, so framing your story in those terms aligns with how the audience already reads).

The Harness engineering-metrics playbook makes a related point about executive communication: lead with the *boundary* and the *evidence*, not the *mechanism* ([Harness, retrieved 2026-05-26](https://www.harness.io/blog/the-executive-playbook-communicating-engineering-metrics-for-maximum-business-impact)). The mechanism details belong in an appendix the auditor can drill into; the headline must be the boundary.

### 3.3 The four authority levels

When framing AI-system behaviour as a control, name the authority level explicitly. Four levels cover the practical range:

- **`full-auto`** — the system acts without human review. Reserved for reversible, low-stakes, high-volume actions.
- **`propose-and-await-approval`** — the system drafts; a human reviews and approves before action takes effect. The default for any action with downstream consequence.
- **`escalate-only`** — the system does not act; it identifies a condition and surfaces it to a human owner.
- **`never-AI`** — a class of actions the system is structurally prohibited from taking, typically because a regulation or policy assigns the decision to a named role.

Audit-oriented audiences understand this taxonomy quickly because it maps to the controls vocabulary they already use (preventive / detective / corrective controls). Capability framing has no equivalent vocabulary.

### 3.4 Declared-weakness discipline

A *named* gap is half-closed; a *hidden* gap is whole-open and growing. Auditors approach every system on the assumption that gaps exist — they're going to find some. If you find them first and name them, the conversation becomes "do we agree on the remediation plan?" If they find them first, the conversation becomes "what else have you not told us?" Those are not symmetric.

The Federal Reserve OIG's published audit-process documentation describes the typical pattern: preliminary findings are discussed during fieldwork, an exit conference allows management to comment, and a formal draft report follows ([Federal Reserve OIG, retrieved 2026-05-26](https://oig.federalreserve.gov/audits-what-we-do.htm)). The exit conference is where declared-weakness discipline pays off — if your story has already named the same weaknesses the auditor is about to surface, there is no exit-conference surprise.

The pattern for declaring a weakness:

> Named tradeoff (two values in tension) → adjudicating policy (the clause or rule that makes the tradeoff acceptable) → remediation plan (what closes the gap and when).

## 4. Generic Implementation

A short worked example outside federal acquisitions: a fintech compliance team is presenting a new fraud-scoring model to the bank's internal audit function before launch.

**Capability-framed version (fails):**

> "The model scores each transaction in real time and blocks the ones above a fraud threshold. Accuracy is 94% on our holdout set."

The auditor's next questions: How is the threshold set? Who can change it? What happens to blocked legitimate transactions? Is the 94% on the holdout set representative of production traffic? What is the model's behaviour on novel attack patterns? You're answering questions for the next forty minutes.

**Controls-framed version (passes):**

> "Bank Secrecy Act §1020.320 requires SAR filing on suspicious activity within 30 days of detection. The fraud-scoring model is a *detection control*: it flags transactions above a threshold for a human reviewer; it does not file SARs and it does not block transactions autonomously. The threshold is owned by the Compliance Director, change-tracked in our governance log; reviewers act under our existing four-eyes process. Holdout accuracy is 94% across the four customer segments in §3.2 of the eval report. Two known weaknesses: (a) the holdout under-represents commercial real-estate flows — closing in Q3; (b) the model has not been re-tuned for the 2026 ACH same-day window change — re-tune scheduled for the next governance cycle."

The story now opens with the clause (BSA §1020.320), frames the model as a detection control (not "fraud detection capability"), names the authority boundaries (threshold ownership, four-eyes review, no autonomous SAR or block), and declares two weaknesses with remediation owners and dates. The auditor's next question is probably "show me the governance log for the threshold" — a question with a known, prepared answer. The conversation has moved from interrogation to confirmation.

Notice what is *not* in the second version: anything about the model's architecture, training data size, gradient-boosting vs neural-net choice, or any other engineering detail. Those belong in an appendix the auditor can request. The headline is the control, the boundary, and the named gap.

## 5. Real-world Patterns

**E-commerce — Black Friday post-mortem storytelling.** When a major e-commerce platform loses checkout availability during peak retail hours, the engineering team narrates the incident to the executive leadership review the following week. The disciplined format opens with the **SLO contract** the platform commits to publicly (e.g., 99.95% checkout availability against a documented quarterly target); frames each decision point as a *control* the on-call team applied or could have applied ("the auto-scaling policy was configured to fire at 75% CPU; we observed 81% sustained for 90s before traffic-shed triggered"); and ends with a **declared learning** — the named runbook gap and the named code-owner accountable for closing it before the next peak event, surfaced before the executive audience extracts it. The structural rule is the same three-move pattern: SLO-anchor → control-frame → declared-weakness ([CodeWithSense on aligning technical narrative with regulator audience, retrieved 2026-05-26](https://www.codewithsense.com/blog/technical-storytelling-for-engineering-leaders-aligning-vision-with-business-strategy)).

**Healthcare — FDA 510(k) clearance submissions.** A medical-device firm seeking 510(k) clearance for an AI-assisted ECG interpretation tool opens its technical story with the predicate device and the relevant FDA guidance document, frames every model capability as a *clinical decision support* control (the AI never diagnoses; it presents candidate interpretations to a clinician), and pre-discloses the populations under-represented in training data. This is the same three-move structure: clause-anchor → control-frame → declared-weakness ([MIT Sloan on stories that earn audience trust, retrieved 2026-05-26](https://sloanreview.mit.edu/article/become-a-better-problem-solver-by-telling-better-stories/)).

**Finance — model-risk management (SR 11-7).** US bank model-risk frameworks under the Federal Reserve's SR 11-7 supervisory letter require model documentation to include intended use, model limitations, and ongoing performance monitoring. Banks that present new models to internal model-risk teams in this exact order — intended use as a *bounded* statement, limitations as a named list, monitoring as a verifiable evidence stream — pass review faster than banks that lead with model accuracy numbers ([Federal Reserve OIG audit framing, retrieved 2026-05-26](https://oig.federalreserve.gov/audits-what-we-do.htm)).

**Industrial controls — IEC 61508 functional safety.** Safety-instrumented systems in chemical plants are presented to certifying bodies via Safety Integrity Level (SIL) claims — controls language is the *only* language. Capability claims ("the burner-management system shuts the valve") are insufficient; the story must say what the system *will do under named failure modes*, what evidence supports the SIL rating, and which failures the analysis did not cover. The discipline transfers directly to AI-system audit storytelling.

## 6. Best Practices

- **Lead with the clause, not the cleverness.** The first sentence of any audit-facing technical story names the rule that constrains the decision; the implementation comes second.
- **Frame behaviour as bounded controls, not unbounded capabilities.** Replace "the model can X" with "the model is permitted to X under condition Y and only with evidence Z."
- **Pre-declare known weaknesses with a remediation owner and date.** A weakness without an owner is an unsolved problem; a weakness with an owner is a managed risk.
- **Match register to audience.** Executives get the 90-second version; managers get the 5-minute version; auditors get the 20-minute version. Same facts, different surface area — see the three-levels-rule reading.
- **Keep mechanism out of the headline.** Architecture choices, model size, vendor selection, framework versions — all of these belong in an appendix the audience can drill into. The headline is the boundary.
- **Use the four authority levels by name.** `full-auto` / `propose-and-await-approval` / `escalate-only` / `never-AI` is a vocabulary the audience can act on. "It's mostly automated but with a human in the loop sometimes" is not.
- **Rehearse the exit-conference moment.** Before any audit-facing presentation, imagine the auditor saying "what didn't you tell me?" If the answer is "nothing — it's all in §5 known weaknesses," you're ready.

## 7. Hands-on Exercise

**Time:** 15 minutes. **Format:** rewriting.

You have a candidate technical-story opening for a fictional system. Rewrite it from capability framing to controls framing using the three-move structure (clause-anchor → control-frame → declared-weakness). The system is generic on purpose.

**Capability-framed opening (rewrite this):**

> "We built an AI-assisted code-review tool. It reads pull requests, summarises the changes, and flags potential security issues. Accuracy is 87% on a benchmark we built internally. Developers love it because it speeds up review by 40%."

**Constraints for your rewrite:**

- Open with a rule, policy, or standard the tool's behaviour respects (you may invent a plausible internal policy or cite a real standard — e.g., NIST SSDF, SOC 2, internal change-management policy).
- Frame the tool as a control, not a capability — name the authority level explicitly.
- Declare at least one named weakness with an owner and date.
- Keep the rewrite to 150 words or fewer.

**What good looks like:** the rewrite opens with the policy clause; the next sentence describes the tool as `propose-and-await-approval` (it drafts review comments; humans approve before they land); the 87% accuracy claim is reframed as *evidence supporting the control* and bounded to the benchmark population; the developer-velocity claim is moved out of the headline (it's a *side effect*, not the audit-relevant fact); and one weakness is declared (e.g., "benchmark under-represents Rust changes — closing Q3 — owner: tooling team lead"). If your rewrite still leads with the model accuracy or the developer velocity, you're still in capability framing.

## 8. Key Takeaways

- Can I name the rule, contract, or policy that constrains the AI behaviour I'm describing, *before* I describe the behaviour?
- Can I state what the system will not do without human approval, in one sentence, using one of the four authority levels?
- Can I name at least one weakness in my own system, with an owner and a remediation date, before the audience asks?
- Can I tell the same story at 90-second, 5-minute, and 20-minute length without contradicting myself across versions?
- If I imagine the auditor at exit conference asking "what didn't you tell me?" — can I answer "nothing"?

## Sources

1. [FAR 15.308 — Source selection decision (Acquisition.gov)](https://www.acquisition.gov/far/15.308) — retrieved 2026-05-26
2. [Technical Storytelling for Engineering Leaders (CodeWithSense)](https://www.codewithsense.com/blog/technical-storytelling-for-engineering-leaders-aligning-vision-with-business-strategy) — retrieved 2026-05-26
3. [Become a Better Problem Solver by Telling Better Stories (MIT Sloan Review)](https://sloanreview.mit.edu/article/become-a-better-problem-solver-by-telling-better-stories/) — retrieved 2026-05-26
4. [Audits — What We Do (Federal Reserve OIG)](https://oig.federalreserve.gov/audits-what-we-do.htm) — retrieved 2026-05-26
5. [The Executive Playbook: Communicating Engineering Metrics (Harness)](https://www.harness.io/blog/the-executive-playbook-communicating-engineering-metrics-for-maximum-business-impact) — retrieved 2026-05-26

Last verified: 2026-05-26
