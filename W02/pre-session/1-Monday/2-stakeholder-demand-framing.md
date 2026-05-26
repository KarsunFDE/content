---
week: W02
day: Mon
topic_slug: stakeholder-demand-framing
topic_title: "Stakeholder demand framing — translating ambiguous asks into operational scope"
parent_overview: W02/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://en.wikipedia.org/wiki/Requirements_elicitation
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.atlassian.com/work-management/project-management/acceptance-criteria
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.altexsoft.com/blog/acceptance-criteria-purposes-formats-and-best-practices/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://blog.planview.com/expectation-management-the-art-of-under-promise-and-over-deliver/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.pmi.org/learning/library/stakeholder-management-pain-points-perils-prosperity-5955
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Stakeholder demand framing — translating ambiguous asks into operational scope

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish a **business question** from a **technical question** in a stakeholder utterance, and explain why most stakeholder asks are the former.
- Restate an ambiguous, prose-form stakeholder ask as a set of **operational scope statements** (what the system WILL do) plus **explicit out-of-scope statements** (what it definitively WILL NOT do this iteration).
- Name three forces — accountability, audit, and budget — that push stakeholders to ask "is this safe to commit to?" rather than "is this technically possible?"
- Apply the **MoSCoW** prioritisation lens (Must / Should / Could / Won't) and the **negative-scenario** lens (what input/state must produce a defined failure response) to a single ambiguous ask and produce a one-page scope envelope.

## 2. Introduction

Stakeholders almost never ask the question they need answered. They ask a question shaped by **what they're going to be measured on next** — the budget meeting, the audit cycle, the executive review, the customer escalation. The job of the engineer receiving that ask is to translate the surface-form question into operational scope: what the system will do, what it will not do, what the failure mode looks like, and what evidence will exist when it ships.

This is the central skill of **requirements elicitation**, and the published literature on it is mature. Wikipedia frames elicitation as "the practice of researching and discovering the requirements of a system from users, customers, and other stakeholders" — emphasising that the stakeholder's own statement of the requirement is rarely complete or unambiguous on first contact. The IIBA, Atlassian, and PMI literature all converge on the same point: stakeholders speak in terms of **outcomes they care about**; engineers must translate those into **specifications that can be built and verified**.

Two anti-patterns dominate the failure mode. The first is **over-promising** — the engineer hears "AI-assisted contracting tool" and nods to everything, then discovers at sprint end that the stakeholder's mental model included three features nobody scoped. The second is **under-promising and over-delivering** — once romanticised by Tom Peters in 1987, now widely critiqued (Planview, PMI, ProjectTimes all carry the rebuttal) because it conditions stakeholders to discount your commitments and amplifies anger when a real problem hits. The middle path the literature recommends is **honest scoping with explicit boundaries**: promise X, deliver X, and write down what X is not.

This reading focuses on the translation step: how you take an emotionally-loaded, accountability-driven stakeholder utterance and produce a one-page scope envelope your team can stand behind for the rest of the week.

## 3. Core Concepts

### 3.1 The stakeholder utterance is a signal, not a spec

Stakeholders compress a lot of information into a few sentences. A useful decomposition is to read every stakeholder utterance for three layers:

- **The surface ask** — the literal sentence ("walk me through what you're building").
- **The accountability anchor** — what the stakeholder is going to be held accountable for ("I have to walk into the CIO's office Wednesday").
- **The disqualifier** — the condition under which the stakeholder's signature is impossible ("OIG told her any tool that can't produce a citation trail won't pass audit").

The disqualifier is the most important of the three, and the most often missed. It's the unstated "if X is true, none of this ships." Surfacing the disqualifier early is what separates a scoping conversation from a sales pitch.

### 3.2 Translate to a six-question scope envelope

Once you have surface / anchor / disqualifier, restate the ask as answers to six standard questions. This is sometimes called the **W6H pattern** in the requirements literature (arXiv 1508.01954), and it's a reliable way to flush out missing scope:

1. **Who** is the primary user, and who is the accountable signer?
2. **What** functional behaviour ships this iteration (specific verbs)?
3. **When** — what is the time-box, and what's the demo moment?
4. **Where** does it run — what environments, what data sources?
5. **Why** — what changes for the stakeholder when this ships (the outcome metric)?
6. **How will we know it worked** — what's the acceptance criterion + the negative scenario?

The sixth question is the load-bearing one. If you cannot name a negative scenario — a specific input or state that must produce a defined failure response — the requirement is not yet scoped.

### 3.3 MoSCoW and the explicit "Won't"

MoSCoW prioritisation (Must / Should / Could / Won't have this iteration) is widely used in business analysis and consistently underused for one reason: teams forget to fill in the **Won't**. The Atlassian, Altexsoft, and IIBA guides on acceptance criteria all emphasise that **negative scope** (the explicit Won't list) is what protects a team from scope drift mid-iteration.

A useful test: read your scope back to the stakeholder and ask "what's missing from this list that you'd be unhappy not to see Friday?" The items they name go into either the Must list (you accept the work) or the Won't list (you write down explicitly that it's deferred and to which iteration). The worst answer is "we'll see how it goes."

### 3.4 Acceptance criteria + Definition of Done

Acceptance criteria (per-story) and Definition of Done (per-team-standing-policy) are not synonymous, and stakeholders routinely conflate them. Atlassian's working definition: acceptance criteria are the **boundaries of a single user story** — the specific conditions that must be true for THIS story to be considered shipped. Definition of Done is the **standing checklist** that every story must pass regardless of its specific content (e.g., "tests pass, code reviewed, deployed to staging, security scan green").

When a stakeholder says "I need X by Friday," the engineer's translation reflex should be: what are the per-story acceptance criteria for X, and does the team's Definition of Done already cover the cross-cutting concerns (auditability, observability, accessibility, regulatory citation), or does THIS story need to extend DoD this iteration?

### 3.5 The accountability inversion

The deepest stakeholder framing trick is the **accountability inversion**: who carries the risk if the system is wrong? In most enterprise contexts, the answer is *not* the engineering team. The signer of a regulated artefact, the doctor signing a prescription, the loan officer signing a credit decision, the contracting officer signing a solicitation — these are the accountable parties, and the platform exists to **earn their signature**, not to substitute for it.

Reading a stakeholder ask through the accountability inversion lens often reveals that the real requirement is not the AI capability but the **evidence trail** — what the platform produces so the human signer has the artifact they need to defend their signature in a downstream review (audit, legal discovery, malpractice case, regulator inquiry).

## 4. Generic Implementation

A worked example outside the federal-acquisitions domain. Imagine a fintech team building a **fraud-screening assistant** for a credit-card issuer's call centre. The stakeholder ask:

> "Build me something that flags suspicious transactions so the agents don't have to read 200 lines of history per call. Compliance is on my back about Reg E disputes — we're missing the 10-day window."

A first-pass scope envelope might be:

| Question | Answer |
|---|---|
| **Who** | Call-centre agents (primary user); fraud-ops director (accountable signer); compliance officer (disqualifier holder). |
| **What** | A side-panel widget that highlights the three most-anomalous transactions in the last 90 days with one-sentence rationales. |
| **When** | Pilot to one team of 12 agents in 6 weeks; one demo to compliance at week 3, one to fraud-ops director at week 5. |
| **Where** | Embedded in the existing call-centre desktop; reads transactions from the existing ledger DB; no new data flow out of the secure perimeter. |
| **Why** | Reg E dispute resolution must complete within 10 business days; agents currently miss the window ~8% of the time because they cannot triage history fast enough. |
| **How will we know it worked** | Acceptance criterion: agents resolve dispute calls in median <7 minutes (currently 11). **Negative scenario:** when the model has < 80% confidence on a transaction, it MUST NOT highlight it; the widget shows "no flag" instead of a guess. |
| **Must** | Three highlighted transactions, one-sentence rationale, links to source transaction records. |
| **Should** | Audit log of every highlight rendered, replayable for compliance. |
| **Could** | Adaptive learning from agent thumbs-up/thumbs-down (deferred to iteration 2). |
| **Won't** | Auto-resolve disputes. Auto-issue refunds. Auto-close cases. The agent ALWAYS makes the call. (This is the accountability-inversion clause.) |

The **Won't** row is what makes this scope envelope defensible. The compliance officer's disqualifier ("you can't have AI making Reg E decisions") is addressed head-on by the negative-scope statement. The accountable signer (fraud-ops director) keeps signing authority. The platform's job is to compress the agent's reading time, not to substitute for the agent's judgment.

## 5. Real-world Patterns

**Healthcare — clinical decision support.** A 2023 multi-stakeholder study of an AI-decision-support tool at a German university hospital (NCBI PMC10155093, retrieved 2026-05-26) surfaced conflicting requirements: patients prioritised data protection and transparency, AI developers needed clinical collaboration and good UI surfaces, clinicians needed the system to **stop short of recommending action** because malpractice liability sits with the physician. The team's scope envelope explicitly listed "auto-prescription" and "auto-referral" as Won't items — the system surfaces evidence but never takes the action. The platform's job is to earn the physician's signature, not to substitute for it.

**Fintech — Reg E and the accountable signer.** A 2024 industrial study of FinTech requirements engineering under regulatory pressure (arXiv 2405.02867) reported that 73% of surveyed teams used a **regulation-traceability matrix** as a scoping artifact — every functional requirement mapped to the specific CFR clause it was implementing, plus the negative-scope statement of what the system would NOT decide. Teams without this matrix consistently mis-scoped: they built more automation than the regulator allowed, then had to roll back.

**E-commerce — returns automation.** A large online retailer (described in Atlassian's acceptance-criteria guide) ran into trouble when the executive sponsor asked for "an AI returns assistant." The engineering team's first scope envelope included "auto-approve refunds under $50." Compliance pushed back: state-by-state consumer-protection laws made auto-approve a legal exposure. The revised scope envelope moved auto-approve into the Won't list and replaced it with "pre-fill the refund form so the human CSR clicks Approve in one step." The accountability stayed with the human; the platform compressed the work.

**Logistics — last-mile dispatch.** A delivery-routing company (PMI library, "Stakeholder management pain points") learned the hard way that the executive who funded the project (CFO, cost-focused) and the executive who would adopt the project (VP of Operations, reliability-focused) had non-overlapping success criteria. The team's eventual scope envelope had two acceptance-criteria columns — one per executive — and explicitly named the trade-offs. The negative-scope clause: the platform will NOT optimise pure cost without an operational-reliability floor below which a route cannot be assigned.

## 6. Best Practices

- **Read every stakeholder utterance for three layers** — surface, accountability anchor, disqualifier. The disqualifier is the most often missed and the most load-bearing.
- **Surface the negative scope before the positive scope.** Ask "what would make this not worth shipping?" before "what should it do?" The answer to the first question constrains the answer to the second.
- **Use MoSCoW with all four buckets populated.** A scope envelope with an empty Won't column is not a scope envelope — it's a wishlist.
- **Treat acceptance criteria as boundary objects.** Each criterion must be testable by the stakeholder, not just by the engineer. If the stakeholder cannot read the criterion and say "yes that's what I meant," it isn't done.
- **Write the negative scenario into every acceptance criterion.** What input, state, or condition must produce a defined failure response? If you cannot name one, the criterion is not yet scoped.
- **Never under-promise to over-deliver.** Honest scope, delivered as promised, builds far more trust than dramatic over-delivery. The latter conditions stakeholders to discount your future commitments.
- **Keep the accountable signer's name in the scope envelope.** Who signs off when this ships? Whose career is on the line? That person's standards are the real acceptance criteria; everyone else is advisory.

## 7. Hands-on Exercise

**Time: 10–15 minutes.** Take this stakeholder utterance:

> "We need a tool that helps engineers write better incident post-mortems. The CTO is tired of reading the same root-cause analysis over and over. Get it done this quarter — make it work with our Jira and our Slack."

Produce a one-page scope envelope with:

- The three layers (surface ask / accountability anchor / disqualifier).
- The six W6H answers.
- A MoSCoW table with at least two entries in each of the four buckets, including the **Won't** column.
- One acceptance criterion that includes a named negative scenario.

**What good looks like:** Your disqualifier should name something the CTO would refuse to ship over (e.g., "if the tool drafts a post-mortem that misattributes blame to a specific engineer, ship is blocked"). Your Won't list should include auto-publishing post-mortems and auto-assigning action items — both are accountability-inversion footguns. Your negative scenario should be testable in 10 lines of code (e.g., "when the source incident ticket has fewer than three comments, the tool MUST refuse to draft and surface a 'need more input' message").

## 8. Key Takeaways

- Can you distinguish a stakeholder's **surface ask** from their **accountability anchor** and their **disqualifier**, and explain why the disqualifier is the most load-bearing?
- Can you restate an ambiguous prose-form ask as a six-question (W6H) scope envelope with a populated Won't column?
- Can you name a negative scenario for every acceptance criterion in your scope envelope?
- Can you explain why under-promising to over-deliver is the failure mode the literature warns against, and what honest scoping looks like instead?
- Can you read a stakeholder ask through the **accountability-inversion** lens and identify the evidence trail the platform must produce so the accountable human can defend their signature?

## Sources

1. [Requirements elicitation — Wikipedia](https://en.wikipedia.org/wiki/Requirements_elicitation) — retrieved 2026-05-26
2. [Acceptance Criteria — Atlassian Work Management](https://www.atlassian.com/work-management/project-management/acceptance-criteria) — retrieved 2026-05-26
3. [Acceptance Criteria for User Stories in Agile: Purposes, Formats, Best Practices — AltexSoft](https://www.altexsoft.com/blog/acceptance-criteria-purposes-formats-and-best-practices/) — retrieved 2026-05-26
4. [Expectation Management: The Art of Under-Promise and Over-Deliver — Planview](https://blog.planview.com/expectation-management-the-art-of-under-promise-and-over-deliver/) — retrieved 2026-05-26
5. [Stakeholder management: pain points, perils and prosperity — PMI](https://www.pmi.org/learning/library/stakeholder-management-pain-points-perils-prosperity-5955) — retrieved 2026-05-26

Last verified: 2026-05-26
