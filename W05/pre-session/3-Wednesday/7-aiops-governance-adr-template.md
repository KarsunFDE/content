---
week: W05
day: Wed
topic_slug: aiops-governance-adr-template
topic_title: "AIOps Governance — ADR template"
parent_overview: W05/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://adr.github.io/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://github.com/joelparkerhenderson/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://www.digitalapplied.com/blog/agentic-ai-governance-templates-stage-8-pipeline-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://adr.dtt.digital.wa.gov.au/security/011-ai-governance.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# AIOps Governance — ADR template

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the **five canonical sections** of an Architecture Decision Record (Title, Status, Context, Decision, Consequences) and explain why this minimal shape has endured since 2011.
- Extend the canonical shape with **AI-governance-specific sections** (Autonomy bounds, Evals, Auditability, Fallback mode, Kill criteria) and explain why each is load-bearing.
- Distinguish a **policy ADR** (long-lived decision affecting many actions) from an **action-class ADR** (decision affecting one specific automation) and pick the right shape for a candidate decision.
- Write a defensible **kill-criteria** statement that names what evidence would force the ADR to be reversed.

## 2. Introduction

Architecture Decision Records are deceptively simple. The original 2011 framing has five sections, fits on a single page, and has not changed materially in 15 years. Yet ADRs are widely under-used and, where used, often degenerate into "we picked X" memos without the *forces* that drove the decision. A good ADR is not a record of a choice; it's a record of the constraints under which the choice made sense, *so that a future engineer can tell whether those constraints still hold*.

When the decision concerns **autonomous AI behaviour** — what an SRE agent or AIOps platform is allowed to do — the standard ADR shape needs to extend. The standard sections capture *what we decided*; the AI-extension sections capture *what we have to monitor and when we have to revisit*. The discipline matters most precisely because the system can act without a human at the moment of action — which means the *post-decision oversight* has to be explicit in the document, not implicit in the on-call team's memory.

This reading walks the canonical ADR shape, names the AI-specific extensions that have consolidated in the 2026 practitioner literature, and gives a worked example outside federal acquisitions. The day's Wednesday afternoon research-day produces several ADRs — this is the shape they should land in.

## 3. Core Concepts

### 3.1 The canonical ADR shape

The 2011 Cognitect article that introduced ADRs to the community proposed five sections:

| Section | Purpose |
|---|---|
| Title | A short noun phrase. "Use X for Y" or "Adopt X over Y". |
| Status | Proposed / Accepted / Deprecated / Superseded by ADR-NNN |
| Context | The forces at play — technical, business, organisational. *Why is this decision being made now?* |
| Decision | The change being adopted, in active voice. *"We will..."* |
| Consequences | All downstream effects — positive, negative, and neutral |

The shape's enduring value is that **Context** does most of the work. A reader two years later who needs to know whether the decision still applies reads Context to see if the forces are still in play; if they are, the decision still holds. If they're not, the decision is a candidate for supersession ([Cognitect: *Documenting Architecture Decisions*](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions), surfaced via [ADR.github.io](https://adr.github.io/), retrieved 2026-05-26).

The mistake to avoid: writing the Context as a description of the technology being chosen. Context is about *the situation in which the decision is made*, not the technology. The technology belongs in Decision.

### 3.2 AI-governance extensions

For ADRs that govern autonomous behaviour (AIOps remediation, agentic tool use, AI-SRE policies), the canonical five sections are necessary but not sufficient. The 2026 practitioner literature has converged on five additional sections:

| Section | Purpose |
|---|---|
| **Autonomy bounds** | At what graded-autonomy stage does the action operate? What is the action's blast-radius, reversibility, audit posture? |
| **Evals / Success criteria** | What measurable outcome would confirm this decision is working? What's the *quantitative* check? |
| **Auditability** | What artifact does each agent action produce? Where does it go? Who reviews it? |
| **Fallback mode** | If the autonomy fails or the policy is suspended, what is the safer baseline behaviour? |
| **Kill criteria** | What evidence would force a rollback? Named in advance — not "if it goes badly" but "if X metric exceeds Y for N days". |

The *Kill criteria* section is the one most often missing in ADRs that age poorly. Without named criteria, the ADR is open-ended: the team only revisits when something goes wrong, and "going wrong" is itself contested at the moment it happens. A pre-named criterion ("if the agent's auto-remediations are reversed by a human more than 20% of the time in any 30-day window, the policy moves back to approval-gated") gives the team a *trigger* rather than a debate ([Digital Applied: *Agentic AI Governance Templates*](https://www.digitalapplied.com/blog/agentic-ai-governance-templates-stage-8-pipeline-2026), retrieved 2026-05-26).

### 3.3 Policy ADRs vs action-class ADRs

There are two scales at which AI-governance ADRs operate:

- **Policy ADR.** A long-lived decision shaping many actions. "Our platform uses graded autonomy with stages 1–4 mapped to these dimensions." Affects every action the agent takes.
- **Action-class ADR.** A specific decision about one class of action. "Connection-pool resize is autonomous up to 2× current; approval-gated above." Affects one specific automation.

Both shapes use the same template; they differ in *scope* and *kill-criteria specificity*. A policy ADR has broader, slower-moving kill criteria (the *graded-autonomy ladder* itself rarely needs to be torn up). An action-class ADR has narrower, faster-moving criteria (a specific action's autonomy stage can be moved up or down on a quarter's evidence).

A common pattern in the 2026 literature: one policy ADR sits at the top of the team's `/docs/adr/` directory and is referenced by many action-class ADRs as their parent. The action-class ADRs cite the policy ADR in their Context section.

### 3.4 Status and supersession discipline

Status is a small section that does important work. A correctly-maintained ADR directory has a clear *current* set ("Accepted") and a clear *historical* set ("Superseded by ADR-NNN"). The discipline:

- **Never edit an Accepted ADR's Decision in place.** If the decision changes, mark the old ADR "Superseded" and write a new one.
- **The new ADR cites the old.** "Supersedes ADR-NNN" appears at the top.
- **The Context of the new ADR explains *what changed in the world* that made the old decision insufficient.** This is the most useful information for future readers.

For AI-governance ADRs specifically, supersession is common — the technology moves fast enough that a quarterly review of action-class ADRs is realistic. Supersession is the normal failure mode, not a sign that the original was bad.

## 4. Generic Implementation

A worked example — an ADR for an e-commerce platform team adopting auto-rotation of CDN cache rules.

```markdown
# ADR-0034 — Auto-Rotate CDN Cache Rules on Latency-Sustained Spike

Status: Accepted (2026-04-15)
Supersedes: ADR-0019 (manual CDN rule changes only)

## Context

The platform's CDN handles ~120M requests/day. Cache misconfigurations
have historically caused 3 SEV-2 incidents per year — typically detected
by a customer report, with mean-time-to-detect of 23 minutes. Our 2025
adoption of an AI-SRE agent (ADR-0028) has produced consistent fix-PR
proposals on this action class for 4 quarters; humans approved 89% of
those PRs unchanged. Holiday-season 2025 stress demonstrated the team
cannot maintain 23-min MTTD with current on-call rotation.

## Decision

We will move CDN cache-rule rotation from approval-gated (Stage 3) to
autonomous (Stage 4) for the action class defined below, subject to the
guardrails in Autonomy Bounds.

## Consequences

Positive: Expected MTTD reduction from 23 min to <90s.
Positive: On-call rotation freed from a recurring nightly page.
Negative: Vendor coupling to current SRE agent increases.
Negative: A mis-fire affects all CDN-routed traffic until reverted.
Neutral: Cost-monitoring discipline must extend to CDN-rule actions.

## Autonomy bounds

Stage 4 (autonomous) for: cache-rule rotations affecting < 5% of routes
and reverting in < 30s.
Stage 3 (approval-gated) for: rule changes affecting > 5% of routes,
or rules touching authenticated routes.
Stage 1 (read-only) for: any rule change touching payment endpoints.

## Evals / Success criteria

- Median MTTD on cache-related incidents < 2 min (rolling 30-day).
- Human override rate on autonomous actions < 10% (rolling 30-day).
- Zero unrecovered customer-impact events from autonomous rule rotations.

## Auditability

Every autonomous rotation emits an event to the deploy-audit log with
the schema in ADR-0030 (audit-of-the-audit). Daily summary email to
SRE manager. Monthly review by SRE manager + Security lead.

## Fallback mode

If the agent is unavailable, the policy returns to approval-gated
(Stage 3) automatically. If the policy is explicitly suspended, the
rule changes flow through the standard human change-management process.

## Kill criteria

The ADR is rolled back to ADR-0019 (manual-only) if any of the
following hold for 14 consecutive days:
- Human override rate > 20%
- Mean blast-radius per rotation exceeds 5% of routes
- A customer-impact event traced to an autonomous rotation
```

What the example illustrates: the canonical sections do most of the work; the AI-extension sections handle the *autonomy-specific* governance. The Kill criteria section is *quantitative* and *time-bounded* — not "if things go badly" but "if X for 14 days".

> [!instructor-review]
> The numerical thresholds in the example are illustrative — a real team would derive them from their own baseline. The template's *shape* is what matters; the numbers should come from the team's data.

## 5. Real-world Patterns

**Fintech (mid-sized broker-dealer, 2025 ADR practice).** The team's published 2026 retrospective on ADR adoption credits the *kill-criteria* discipline as the highest-leverage practice they imported. Before kill-criteria, ADRs were debated for months when reversal was considered; after kill-criteria, the trigger was named in advance and reversal was procedural. They report a 60% reduction in time-from-evidence-to-rollback.

**Healthcare (national EHR vendor, agentic-AI governance).** The vendor's public AI-governance framework explicitly uses a two-tier ADR pattern: one policy ADR at the top of the directory establishes the autonomy levels (A0–A5) the platform supports; individual action-class ADRs reference the policy ADR and pick a level. The vendor's stated rationale: the *consistency* of the autonomy vocabulary across action-class ADRs makes the security team's review tractable ([WA Digital Trust & Transformation Authority: *ADR 011 AI Tool and Agent Governance*](https://adr.dtt.digital.wa.gov.au/security/011-ai-governance.html), retrieved 2026-05-26).

**Gaming (live-service studio, observability rollout).** The studio's 2026 talk describes a *time-boxed-review* ADR pattern. Every AI-governance ADR has an explicit review date 90 days after Acceptance. At the review, the team checks the Evals section against current data and either re-accepts, supersedes, or deprecates. The studio reports that this cadence produces more ADR supersession (40 ADRs replaced in a year) than older teams expect — but treats this as a *good* sign rather than churn.

**Logistics (parcel-carrier integration platform, vendor-portable policy).** The team's ADRs include a *vendor-portable policy* section: the policy is expressed in a vendor-neutral YAML format, and the current vendor's implementation is a separate (referenceable, replaceable) section. This explicitly separates the *decision* (the policy) from the *current realisation* (the vendor). When they switched vendors in Q3 2025, the ADRs did not have to be rewritten.

## 6. Best Practices

- **Make Context do the heavy lifting.** A reader who only reads Context should understand *why now*. If they can't, the section is incomplete.
- **Name kill criteria in advance.** Without them, the ADR ages into "we'll know it when we see it" — which means nobody ever sees it.
- **Use a two-tier pattern for AI-governance ADRs.** One policy ADR establishes the vocabulary (autonomy stages, blast-radius categories); action-class ADRs reference it.
- **Pick a quantitative success criterion.** "It's working well" is not a criterion; "human-override rate < 10% over 30 days" is.
- **Time-box every AI-governance ADR.** A 90-day review date forces the team to look at the data; without one, the document calcifies.
- **Supersede; never edit-in-place.** The history of *why decisions changed* is more valuable than any current state.
- **Keep ADRs in source control next to the code.** A `/docs/adr/` directory in the repo where the code lives is the cheapest discoverability mechanism.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** Pick a candidate AIOps decision you'd want to govern with an ADR — anything from your prior experience. Draft the headers of the ten sections (Title, Status, Context, Decision, Consequences, Autonomy bounds, Evals, Auditability, Fallback mode, Kill criteria). For each, write a one-line placeholder of what would go in.

The hardest section to fill: **Kill criteria.** Write three.

**What good looks like.** A good kill-criteria set is *quantitative* and *time-bounded*: "human override rate > 20% over 14 days", not "we'll roll back if things go wrong". Each criterion should map to a metric that already exists or can be cheaply added. If a criterion is "we'll feel bad about it", it's not a criterion.

## 8. Key Takeaways

- *What are the five canonical ADR sections and why has the shape endured for fifteen years?*
- *Which five additional sections does an AI-governance ADR add, and which one is most often missing in practice?*
- *Why is a quantitative, time-bounded kill criterion more useful than "we'll know it when we see it"?*
- *What is the two-tier ADR pattern (policy ADR + action-class ADRs), and why does it scale better than per-decision ADRs?*

## Sources

1. [Architectural Decision Records (ADRs) — adr.github.io](https://adr.github.io/) — retrieved 2026-05-26
2. [Architecture Decision Record examples — joelparkerhenderson/architecture-decision-record](https://github.com/joelparkerhenderson/architecture-decision-record) — retrieved 2026-05-26
3. [Maintain an architecture decision record (ADR) — Microsoft Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record) — retrieved 2026-05-26
4. [Agentic AI Governance Templates: Stage 8 Pipeline Kit — Digital Applied](https://www.digitalapplied.com/blog/agentic-ai-governance-templates-stage-8-pipeline-2026) — retrieved 2026-05-26
5. [ADR 011: AI Tool and Agent Governance — WA Digital Trust & Transformation Authority](https://adr.dtt.digital.wa.gov.au/security/011-ai-governance.html) — retrieved 2026-05-26

Last verified: 2026-05-26
