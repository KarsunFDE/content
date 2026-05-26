---
week: W06
day: Wed
topic_slug: why-d-060-picked-karsun-managers
topic_title: "Why D-060 picked Karsun managers"
parent_overview: W06/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://demosecret.com/articles/how-to-demo-to-technical-audience
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.linkedin.com/advice/3/how-do-you-adjust-technical-demos-different-audiences
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://kindatechnical.com/effective-communication-engineers/client-presentations-and-demos.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://ohiostate.pressbooks.pub/feptechcomm/chapter/2-audience/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Why D-060 picked Karsun managers

## 1. Learning Objectives

By the end of this reading, the learner can:

- Explain why audience calibration is a first-order decision when designing a technical defense or showcase, not a stylistic afterthought.
- Distinguish at least three orthogonal audience dimensions (technical proficiency, decision-making role, domain literacy) that shape what an engineer should foreground vs. background.
- Articulate the "translation" framing of technical communication and why senior engineers consistently under-calibrate to non-engineer-but-domain-literate stakeholders.
- Name the failure modes of mis-calibrated technical defenses (jargon flooding, marketing-speak, hidden limitations) and how each erodes credibility with a specific audience type.
- Apply an audience-calibration checklist to a hypothetical defense scenario, deciding what to lead with and what to hold for follow-up.

## 2. Introduction

Every technical demonstration is, at its core, an act of translation. The engineer carries detailed knowledge of a system; the audience needs enough of that knowledge to make a decision. Effective communicators treat audience as a first-order input to *what* they show and *how* they show it — not just a delivery-style adjustment they make at the end [4]. Ohio State's engineering-communication curriculum frames this directly: *how* you choose to transmit information must depend to a large extent on who your audience is and what their goals are [4].

The problem is that engineers under pressure tend to default to a single audience model: their own. They show what they would want to see if they were watching, calibrating depth to their own comfort with the material. This works exactly once — when the audience happens to share their background — and fails predictably whenever the audience differs by even one dimension (less technical depth, different domain, different decision-making role).

Programmes designed to train engineers in production-deployment skills therefore make audience calibration an explicit object of practice rather than a soft skill picked up by osmosis. The decision to put an engineer in front of a *specific tier* of audience for a defense is itself part of the curriculum — different audiences exercise different muscles. A panel of fellow engineers exercises technical depth; a panel of executives exercises summarization and business framing; a panel of domain-literate non-engineer managers exercises the hardest combination of both. This reading covers the generic principles of audience calibration; the daily overview connects them back to the specific Wednesday-evening preparation work the cohort does.

## 3. Core Concepts

### 3.1 Audiences are not a one-dimensional spectrum

A common mental shortcut is to imagine audience as a single spectrum from "non-technical" to "technical." It isn't. Audience composition has at least three orthogonal dimensions [4]:

- **Technical proficiency** — how much of the implementation vocabulary the audience can absorb without translation. A senior engineer is high on this axis; a CFO is low; a sales engineer and a CTO sit somewhere in the middle.
- **Decision-making role** — what action the audience will take after the communication. A staffing manager decides whether to put you on a project. An architect decides whether to integrate your system. A compliance officer decides whether to sign off on a control. The action shapes what content matters.
- **Domain literacy** — how fluent the audience is in the *problem space* even if not the *implementation*. A federal-acquisitions manager knows the regulations, contract structures, and procurement vocabulary cold without knowing which database your service uses.

A high-domain-literacy / mid-technical-proficiency / staffing-decision-role audience is a very specific combination. It is not the same as an "executive audience" (typically low technical proficiency) and not the same as a "technical audience" (typically high technical proficiency, low decision-making role for staffing). Defending to this audience requires its own playbook.

### 3.2 The translation principle

Ohio State's framing is worth quoting directly: think of communication "as an act of translation — you possess information and knowledge, and you need to deliver that information to your audience in a way they will understand" [4]. The translator's job is not to dilute the content but to re-encode it in symbols the audience can decode without losing the meaning.

What this implies for an engineer defending a system:

- The *facts* you have do not change with the audience. The *encoding* does.
- Translation is **lossy in both directions**: omit too much technical depth and a senior engineer in the audience loses trust; preserve too much and a manager-tier audience cognitively checks out.
- Good translators front-load the highest-signal-per-symbol content for the *specific* audience and keep the rest available on demand.

### 3.3 The audience-fit failure modes

When a technical defense is mis-calibrated to its audience, the failures fall into a small number of named patterns [1][2][3]:

- **Jargon flooding.** Defaulting to the implementation vocabulary the engineer is comfortable in. To a domain-literate manager, this reads as "I cannot explain my work to people outside the implementation." To a fellow engineer, it can pass; to anyone else, it actively erodes credibility.
- **Marketing-speak.** The opposite failure: replacing the technical content with hand-waving abstractions. To a manager-tier audience, this can briefly pass, but only briefly — any probing question reveals the absence of substance underneath. The DemoSecret guidance is direct: "Slides feel like a pitch. A live product feels like proof. Use the real product, with realistic data" [1].
- **Hidden limitations.** Avoiding or deflecting questions about what doesn't work. Both engineer audiences and manager audiences are sensitive to this — the engineer because they smell technical dishonesty, the manager because they smell project risk that hasn't been named. "Deflection erodes trust. Honesty builds it" [1].
- **Over-rehearsed delivery.** Practicing past the point of authenticity. The kindatechnical guide notes that "over-rehearsing can make delivery sound scripted, reducing authenticity" [3] — rehearsal helps timing and transitions but should stop short of memorization.
- **Wrong-tier decision content.** Loading the defense with content the audience cannot or will not act on. A manager who decides staffing does not need the LangGraph node-ID semantics; they need cost shape, risk shape, and time-to-productivity for the next FDE.

### 3.4 The "decision-making power" lens

Per kindatechnical's audience framework, "prioritize content that addresses the needs of those who will approve or fund the project" [3]. This is the single most actionable lens for audience calibration: ask *what decision is this audience going to make in the next 24 hours*, and front-load the content that informs that decision.

Different audiences make different decisions:

- A peer engineer is deciding whether your code review feedback should be accepted.
- An architect is deciding whether your component can be integrated into a larger system.
- A staffing manager is deciding whether you can be put on a new engagement next quarter, and at what level.
- A compliance officer is deciding whether your system can pass an audit.
- A CFO is deciding whether to fund the next phase.

The same system can be defended five different ways for these five audiences, and *should be*. The engineer who can flex between framings is the engineer who gets promoted; the one who can only produce the engineer-to-engineer framing is constrained to one slot of the org chart.

## 4. Generic Implementation

For a topic about audience calibration, the implementation is a worked example rather than a code snippet. Imagine a software engineer at a fintech company who has just finished a six-month project to migrate a payment-reconciliation service from a monolith to microservices. They will defend the work three times in three different rooms next week. Same system, three audiences, three calibrations:

**Audience A — Engineering staff meeting (peer engineers).** Lead with the architecture diagram showing the new service boundaries. Walk the bounded-context decisions and the database-per-service tradeoff. Show the integration tests covering the cross-service flows. Open the trace viewer for a real reconciliation. Spend most of the time on the two design decisions that almost went the other way. Q&A will probe the schema-evolution strategy and the rollback plan.

**Audience B — Quarterly business review (VP-tier, low technical proficiency, high decision-making power).** Lead with the business outcome: reconciliation latency dropped from 6 hours to 12 minutes; daily-error backlog dropped from ~400 to <20. Show the cost shape: infrastructure spend up 12%, engineer-on-call hours down 60%, net annualized savings $X. Mention the architecture only as "we split one service into three that can scale independently." Q&A will probe forecasted savings, headcount implications, and the next quarter's roadmap.

**Audience C — Compliance review (domain-literate, mid-technical, audit-decision role).** Lead with how the audit-trail requirements (SOX, PCI-DSS) are preserved across the split — same events, same retention, new ID-stitching at the service boundaries. Show the access-control matrix per service. Walk one full audited reconciliation end-to-end with timestamps and actor IDs. Q&A will probe the failure-mode coverage and the disaster-recovery story.

The *system* is identical in all three rooms. The *defense* is three different defenses. The engineer who walks into all three with the engineering-staff version of the deck loses two of three rooms.

## 5. Real-world Patterns

**Healthcare — clinical engineering review boards.** Hospital systems doing major IT changes often defend the change to a mixed board: clinicians (high domain literacy, low implementation literacy), IT directors (high implementation literacy, decision authority), and patient-safety officers (high domain literacy, regulatory decision authority). Published case-studies on clinical-systems implementation note that successful defenses lead with the patient-safety story for the safety officers, the integration story for IT, and the workflow-fit story for clinicians — three threads woven, not one thread told once [3].

**Fintech — board-level demos.** Sales-engineering literature consistently distinguishes between "champion demos" (deep technical sessions with the engineering champion who will advocate internally) and "executive demos" (short, business-outcome-led sessions with the budget owner). The same product, demoed identically to both, loses one or the other reliably [1][2]. The pattern that works in production sales: the champion demo is exploratory and technical; the executive demo is structured around the three business outcomes the champion has confirmed matter, in priority order.

**E-commerce — vendor selection committees.** Mid-2020s e-commerce platform evaluations are typically run by a committee with a procurement lead, a technical lead, and a business-unit lead. Vendor demo guides advise running three sessions or one carefully sequenced session that surfaces the right content at the right time for each role — opening with the business outcome (for the BU lead), pivoting to architecture (for the technical lead), then closing with deployment timeline and commercials (for procurement) [2]. Trying to compress all three into a single undifferentiated pitch is the modal failure.

**Gaming — internal pitches to studio heads.** Indie-game studios pitching new projects to studio leadership commonly mis-calibrate by leading with game-design depth (the team's natural language) when the studio head needs cost-and-schedule first. Post-mortem write-ups of greenlit vs. shelved pitches consistently identify audience mis-calibration as a top-three failure cause — not because the game wasn't good, but because the pitch answered questions the audience wasn't asking yet [4].

## 6. Best Practices

- **Identify the audience's *decision* before you write a single slide.** Every piece of content earns its place by serving that decision, or it gets cut.
- **Map the audience on all three dimensions** (technical proficiency, decision-making role, domain literacy) — not just the technical axis. Mid-tech-proficiency / high-domain-literacy is the most commonly mis-calibrated combination.
- **Lead with the audience's natural vocabulary.** Mirror their terms; introduce new terms only after grounding them in something the audience already knows [1].
- **Prepare a technical appendix or follow-up doc** so you can say "more in the appendix" rather than carrying that depth in the main flow [1]. This is how you serve mixed-audience rooms without picking one to alienate.
- **Acknowledge limitations explicitly and route them to a tracked artifact** (a finding, a known-weakness entry, a backlog ticket). Both engineer and manager audiences respect named, owned, dated gaps; both penalize hand-waving [1].
- **Rehearse to the *audience*, not to your own mirror.** Find one person who resembles the actual audience and rehearse with them, accepting the feedback that emerges from their frame of reference [3].
- **Time-box the demo to the audience's attention shape.** Executive audiences cap around 15 minutes; technical audiences can sustain 30–45 minutes if probing is welcome; manager-tier audiences typically sit at 20–25 minutes before they want the conversation to shift to Q&A.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** You have built an internal recommendation engine for an e-commerce platform. You must defend it next week in two rooms, back-to-back: (a) the platform's senior staff engineers, who will review the architecture; and (b) the head of category management, who is a domain expert (knows merchandising, vendor relationships, inventory) but is not an engineer and does not know your stack.

On a piece of paper or in a markdown file, write **two 5-bullet opening agendas** — one for each room. For each bullet, name (a) the content, (b) the vocabulary you will use (sample phrasing), and (c) the question the audience is likely to ask after that bullet.

**What good looks like:** The two agendas share ~20% of their content (the core business outcome — what the engine does and how well it works) and ~80% diverges (the staff-engineer agenda goes deep on retrieval architecture, ranking model, evaluation pipeline; the category-manager agenda goes deep on which categories see the biggest lift, how merchandising rules interact with the engine, what the override-and-pin workflow looks like). The vocabulary in the engineer agenda uses terms like "feature store," "two-tower model," "offline eval"; the vocabulary in the category-manager agenda uses terms like "category lift," "merchandising override," "seasonal pin." The questions you anticipate are different in kind, not just in depth.

## 8. Key Takeaways

- *Why is audience a first-order input to technical communication, not a styling choice applied at the end?* (Because the encoding — which content earns space, which vocabulary is used, which depth is reached — flows from the audience's decision and translation needs.)
- *What are the three orthogonal audience dimensions, and why does collapsing them to one axis mis-calibrate?* (Technical proficiency, decision-making role, domain literacy; collapsing them produces a single "technical/non-technical" mental model that misses the high-domain-literacy / mid-technical-proficiency combination entirely.)
- *What is the "translation" framing of communication and how does it bound the engineer's job?* (You possess the information; the audience needs it in a form they can decode; your job is the re-encoding, not the dilution.)
- *What are the named failure modes of mis-calibrated technical defenses, and which audience type triggers each?* (Jargon flooding for non-engineers; marketing-speak for engineers; hidden limitations for everyone; over-rehearsal for any audience; wrong-tier decision content for the audience whose decision you missed.)
- *What single lens collapses most of the audience-calibration question into one actionable question?* ("What decision is this audience going to make in the next 24 hours" — content earns space by serving that decision.)

## Sources

1. [How to Demo to Technical Audience (DemoSecret)](https://demosecret.com/articles/how-to-demo-to-technical-audience) — retrieved 2026-05-26
2. [How to Tailor Technical Demos to Different Audiences (LinkedIn)](https://www.linkedin.com/advice/3/how-do-you-adjust-technical-demos-different-audiences) — retrieved 2026-05-26
3. [Client Presentations and Demos (kindatechnical)](https://kindatechnical.com/effective-communication-engineers/client-presentations-and-demos.html) — retrieved 2026-05-26
4. [Understanding Your Audience — Fundamentals of Engineering Professional Communication (Ohio State)](https://ohiostate.pressbooks.pub/feptechcomm/chapter/2-audience/) — retrieved 2026-05-26

Last verified: 2026-05-26
