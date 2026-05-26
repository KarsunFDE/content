---
week: W06
day: Mon
topic_slug: stakeholder-vocabulary-calibration
topic_title: "Stakeholder vocabulary calibration — 4-audience framework"
parent_overview: W06/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://writingcommons.org/article/audience-analysis-primary-secondary-and-hidden-audiences/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.archbee.com/blog/audience-in-technical-writing
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://medium.com/@toddlarsen/how-can-you-communicate-technical-concepts-to-non-technical-stakeholders-74bee2549fd6
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.technical-leaders.com/post/technical-vs-non-technical-audiences-communication-strategies
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://reintech.io/blog/bridging-communication-gap-technical-non-technical-stakeholders
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Stakeholder vocabulary calibration — 4-audience framework

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish primary, secondary, and tertiary audiences for a technical deliverable and explain what each reads first.
- Adapt vocabulary, level of abstraction, and emphasis to each audience type without losing technical accuracy.
- Recognise the most common audience-mismatch failure modes (over-detailed for executives, under-detailed for engineers, ambiguous about decision rights).
- Use a four-audience framework — operational, executive, audit, and customer — to plan the artefacts of a multi-stakeholder handoff.

## 2. Introduction

Defending a piece of engineering work in front of a panel of mixed audiences is a different task from defending it in front of any single audience. The mixed-audience defence is where most engineering presenters fail, and the failure mode is consistent across industries: the engineer optimises the talk for the audience they understand best (other engineers) and loses the audience they understand least (executives, auditors, customers). The result is a presentation that engineers enjoyed and decision-makers tuned out — and decision-makers are usually the ones whose approval matters.

The technical-writing tradition has had a working model for this for decades: the **primary / secondary / tertiary audience** decomposition, where each role has different questions, different vocabulary, and different attention budgets. Writing Commons' canonical treatment names primary readers as the "action takers" (the decision-makers), secondary as the advisors who help the primary, and tertiary as everyone else who might read the document for ancillary reasons. The same decomposition works at a defence panel: who is in the room, what do they need, what is their attention budget?

A useful working model for engineering-handoff presentations is **four audiences**, each anchored to a role rather than a person: an *operational* audience (will inherit operations of the system), an *executive* audience (will fund or staff the next phase), an *audit* audience (will assess risk and compliance), and a *customer/end-user* audience (will use the system or interact with the team's work product). Each audience asks different questions; each audience uses different vocabulary; each audience needs the team's presentation tuned to their attention budget.

This reading covers the four-audience model, what each audience reads first, and the common vocabulary-mismatch failure modes.

## 3. Core Concepts

### 3.1 Primary, secondary, tertiary

The classical audience-analysis decomposition has three roles:

- **Primary audience.** The people who will take action based on the work — the decision-makers. They read first; their question is *should we approve / fund / accept / staff this?*
- **Secondary audience.** The advisors who brief the primary audience — experts in the field, technical reviewers, advisors. They read in support of the primary's decision; their question is *is the work credible enough that I will stake my reputation on advising the primary to approve?*
- **Tertiary audience.** Anyone else who might read the document — community, evaluators, future archive readers. Their question varies widely; the team usually does not optimise for them but should not actively confuse them.

Audience type matters because the *order in which the audience reads* dictates the structure of the deliverable. The primary reads the front page; the secondary reads the appendix; the tertiary reads what is published. A document structured front-loaded with executive summary, with technical depth pushed to appendices, serves the structure of the read.

### 3.2 The four-audience working model for handoff

For a handoff defence, the four-audience model maps roles to questions:

| Audience | Identity | Primary question | What they read first | Attention budget |
|----------|----------|------------------|----------------------|------------------|
| Operational | The team that will run the system | "Can we operate this in production without you?" | The runbook + the known-weaknesses inventory | High for procedural detail; low for strategy |
| Executive | The decision-maker who funds / staffs the next phase | "Is this fit-for-purpose for the next engagement / customer?" | The handoff README + the cost analysis | Low for procedural detail; high for ROI and risk |
| Audit | The role that assesses risk, compliance, and reproducibility | "Is the audit trail complete and the decisions defensible?" | The ADR catalog + the security attestation + the AI-assist log | Medium for everything; very high for evidence |
| Customer / end-user | The role that will use the system or interact with the work product | "Does this solve my problem, and can I trust it?" | The eval report + a demo + the authority-boundary doc | Variable; usually time-boxed; story-driven |

The four are not always all present at any given gate — some gates collapse executive and customer into one audience, some collapse audit and operational. But planning for all four ensures that when one shows up unexpectedly, the team has the artefact and the vocabulary.

### 3.3 Vocabulary calibration per audience

The Technical Leaders blog characterises the skill as **contextual bilingualism** — the ability to switch between technical depth and strategic overview based on audience. The Reintech treatment puts it more bluntly: each technical term risks disconnecting your non-technical audience, so word choice is a deliberate act, not a default.

Concrete calibration moves:

- **For the executive audience.** Lead with the business outcome (cost, risk, scale, time-to-market). Defer architecture diagrams to backup slides. Use analogies anchored in their domain (logistics, financial, market). Avoid stack-deep terminology — say "the system tolerates two-thirds of nodes failing" not "the Raft quorum is N/2 + 1." The decision-makers reading and listening at this level want to know: *what did we get for the money, and what does the next quarter cost?*
- **For the audit audience.** Lead with reproducibility and evidence. Every claim ties to an artefact link. Vocabulary borrows from the audit tradition (finding, criterion, condition, remediation, accepted-risk) more than from the engineering tradition (PR, MR, deploy, rollback). Auditors prize *honest naming of what didn't work* more than polished claims of success — a known-weakness inventory carries more credibility than a complete-success narrative.
- **For the operational audience.** Lead with procedures and on-call expectations. Vocabulary mirrors the operational reality: alert names, escalation paths, dashboard URLs, SLO definitions. The operational reader's first question is "what fires my pager and what do I do about it?" — answer that before anything else.
- **For the customer / end-user audience.** Lead with the user's task and how the system supports it. Vocabulary borrows from the customer's domain, not the engineering team's. A demo or screenshot is usually worth more than a description.

### 3.4 The common audience-mismatch failure modes

Three failure modes recur:

#### Over-detailed for executives
The engineer presents the architecture diagram first because that is the artefact they are proudest of. The executive disengages within minutes because they came for a decision, not an education. Detection: time-the-disengagement — if the executive picks up their phone before slide 5, you are over-detailed.

#### Under-detailed for engineers
The presenter pitches the executive-summary version to a panel that includes the engineering reviewer who will inherit the code. The reviewer cannot evaluate the work because the depth they need is missing. Detection: the technical Q&A is shallow because the audience cannot find a foothold to ask substantive questions.

#### Ambiguous about decision rights
The presenter does not state who is being asked to approve what. The executive thinks the auditor is the decision-maker; the auditor thinks the customer is; the presenter walks out without a decision. Detection: the meeting ends without a named action, owner, and date.

### 3.5 The two-pass speech: outcome first, evidence second

A working pattern for mixed-audience presentations is the **two-pass speech**: pass 1 covers outcomes (what was achieved, what it cost, what the risks are) at executive depth; pass 2 covers evidence (architecture, tests, design decisions) at technical depth. The structure lets the executive audience disengage cleanly after pass 1, lets the audit audience cross-check pass 1 against pass 2, and lets the engineering audience get the depth they need in pass 2.

The discipline of the structure is in the *cleanness of the boundary*. If pass 1 leaks into architectural detail or pass 2 leaks into business outcomes, the structure collapses and both audiences lose the thread.

## 4. Generic Implementation

Below is a four-audience handoff-presentation outline for a **regional-airline operations-platform handoff** — the team is handing the platform to the airline's internal operations group after a 14-week engagement. Entirely outside federal acquisitions.

```
HANDOFF PRESENTATION OUTLINE — Regional Airline Ops Platform v2
================================================================

Length: 60 minutes (45 talk + 15 Q&A)
Audiences in the room:
  - Operational: Airline Ops Center supervisor + on-call rotation lead
  - Executive: VP Operations (decision-maker)
  - Audit: IT Risk Officer (compliance)
  - Customer/end-user: Crew-scheduling representative (downstream user)

Pass 1 — Outcome (15 min)
  - 1 slide: business outcome (on-time-performance delta; cost-per-flight delta)
  - 1 slide: scope shipped vs scope deferred (named deferrals as findings)
  - 1 slide: cost shape under three load scenarios
  - 1 slide: top three risks named (with named remediation owners)

Pass 2 — Evidence (25 min, layered)
  - Operational layer (8 min): runbook walkthrough + alert coverage
  - Audit layer (8 min): ADR catalog + security attestation + AI-assist log
  - Engineering layer (9 min): architecture diagram + integration boundaries

Pass 3 — Pointer (5 min)
  - Where each artefact lives
  - Who owns what post-handoff
  - First three weeks of operations cadence

Q&A (15 min, ordered)
  - Executive questions first (decision)
  - Audit questions second (compliance)
  - Operational questions third (procedural)
  - End-user questions last (interaction)
```

Annotations:

- The **pass-1 / pass-2 structure** lets the VP Operations leave at the 15-minute mark if needed; the rest of the audiences get pass 2.
- The **layered pass 2** is the secret to keeping multiple audiences engaged: each layer has its primary audience and each lasts long enough to be substantive without exhausting the others.
- The **Q&A ordering** matches decision-rights: the executive's question is answered first because their decision is the gate; the operational question last because procedural detail can be followed up offline.

## 5. Real-world Patterns

**Investor pitches vs technical deep-dives at a healthcare-startup ramp (healthcare).** Healthcare startups frequently present the same product to two distinct audiences: investors (who care about market and ROI) and FDA reviewers (who care about safety and evidence). The lead-with-outcome / drill-into-evidence pattern works for both — but the *outcome* differs (investor: market size; FDA: clinical evidence quality). The discipline is keeping the structure constant while swapping the outcome anchor.

**Game-studio publisher reviews (entertainment).** Game studios pitching a milestone build to a publisher run a multi-audience presentation: business team (sales projections), design leads (creative direction), QA leads (regression status), and external press (preview content). The pattern is the same two-pass: outcome (business + creative direction) first, evidence (build quality, content readiness) second. Studios that do not segregate their pass-1 from pass-2 lose the publisher's business audience in the first 10 minutes and miss the funding decision.

**Cloud-platform engineering reviews at a hyperscaler (developer tools).** Hyperscaler cloud-platform engineering reviews (AWS, Azure, GCP) routinely pull together SRE, product, and customer success in the same review. The internal convention is "exec summary slide, then drill" — the first slide is for the platform GM, the next 20 slides are for the platform engineers. SREs in particular are notorious for over-detailing the exec slide; the convention exists because the failure mode recurs.

**Public-utility regulatory hearings (logistics / utilities).** Public-utility presentations to regulators (rate cases, infrastructure proposals) present to a panel of commissioners (decision-makers), expert staff (advisors), and public commenters (tertiary). The presentation routinely runs a two-pass structure — outcome and rate impact first, technical justification second — and the structure has been stable for decades because the audience mix is stable. Engineering handoffs benefit from copying the format.

## 6. Best Practices

- **Name your audiences before you write a slide.** A presentation drafted without explicit audience identification defaults to the author's preferred audience.
- **Order content by reader role.** Primary first, secondary second, tertiary last. The structure mirrors the read.
- **Calibrate vocabulary per audience.** Each technical term risks disconnecting a non-technical audience; word choice is deliberate.
- **Use the two-pass structure for mixed-audience defences.** Outcome first, evidence second; clean boundary.
- **Lead with business outcome for executives, with evidence for auditors, with procedure for operators, with the user-task for end-users.** The first sentence sets the audience-frame.
- **Name decision rights explicitly.** "We are asking the VP to approve X by Date Y." Without this, mixed-audience defences end ambiguously.
- **Time-the-disengagement during dry-runs.** If the executive audience peels off before pass 1 ends, the pass-1 needs tightening.

## 7. Hands-on Exercise

**Whiteboarding prompt (10–15 min).** Pick a recent project deliverable you presented (or could imagine presenting) to a mixed audience. Draft the *first 90 seconds* of the talk for each of the four audiences — operational, executive, audit, customer — separately.

Expected components:

- Four short scripts (3–5 sentences each), one per audience.
- The opening sentence of each script should anchor on that audience's primary question (operate, fund, audit, use).
- Vocabulary should be visibly different — executive script uses business vocabulary; audit script uses compliance vocabulary; operational script uses runbook vocabulary; customer script uses the customer's domain vocabulary.
- One sentence at the end of the exercise naming the SAME outcome restated four ways — proving that the substance is constant while the framing varies.

**What good looks like.** A reader skimming the four scripts should know which audience each is for from the first sentence. The vocabulary should not overlap heavily between any two scripts (more than 20% overlap suggests under-calibration). The final outcome restatement should be substantively the same fact, with four different verbs and four different objects-of-care.

## 8. Key Takeaways

After this reading the learner should be able to answer:

- What are the primary, secondary, and tertiary audience roles, and how does the audience hierarchy dictate the structure of a technical deliverable? *(maps to LO 1)*
- What are the four audience-archetypes for a handoff defence (operational / executive / audit / customer), and what is each audience's primary question? *(maps to LO 4)*
- How do you calibrate vocabulary and emphasis per audience without losing technical accuracy, and what is the contextual-bilingualism skill? *(maps to LO 2)*
- What are the three common audience-mismatch failure modes (over-detailed for executives / under-detailed for engineers / ambiguous decision rights), and how do you detect each in a dry-run? *(maps to LO 3)*
- What is the two-pass structure for mixed-audience presentations, and why does the clean boundary between outcome and evidence matter? *(maps to LO 2, 4)*

## Sources

1. [Audience Analysis for Technical Documents — Writing Commons](https://writingcommons.org/article/audience-analysis-primary-secondary-and-hidden-audiences/) — retrieved 2026-05-26
2. [Audience Analysis in Technical Writing — Archbee](https://www.archbee.com/blog/audience-in-technical-writing) — retrieved 2026-05-26
3. [How Can You Communicate Technical Concepts to Non-Technical Stakeholders? — Todd Larsen on Medium](https://medium.com/@toddlarsen/how-can-you-communicate-technical-concepts-to-non-technical-stakeholders-74bee2549fd6) — retrieved 2026-05-26
4. [Technical vs Non-Technical Audiences: Communication Strategies — Technical Leaders](https://www.technical-leaders.com/post/technical-vs-non-technical-audiences-communication-strategies) — retrieved 2026-05-26
5. [Bridging the communication gap between technical and non-technical stakeholders — Reintech](https://reintech.io/blog/bridging-communication-gap-technical-non-technical-stakeholders) — retrieved 2026-05-26

Last verified: 2026-05-26
