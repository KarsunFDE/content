---
week: W06
day: Mon
topic_slug: deliverable-taxonomy
topic_title: "The deliverability package shape — the deliverable taxonomy"
parent_overview: W06/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 13
sources:
  - url: https://medium.com/@sparknp1/10-docs-that-compound-adrs-runbooks-why-files-f9fb1c38b582
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://rootly.com/incident-response/runbooks
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://adr.github.io/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://multishoring.com/blog/it-project-handover-checklist-steps-for-a-seamless-transition/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://medium.com/@leoyeh.me/deliverables-artifacts-and-building-blocks-decoding-the-architecture-content-framework-in-togaf-5c308ff9ef35
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# The deliverability package shape — the deliverable taxonomy

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define what a deliverable taxonomy is and why a closed (bounded) list is more useful than an open one.
- Name the standard handoff artefacts (runbook, ADR catalog, known-weaknesses inventory, handoff README, etc.) and the audience each is written for.
- Distinguish artefacts that target an *operator* from artefacts that target a *successor engineer* from artefacts that target a *manager or auditor*.
- Choose which artefact a given piece of project knowledge belongs in — and recognise when knowledge does NOT fit in any artefact (signal of scope creep).

## 2. Introduction

When a delivery team hands a system to its next owner — a successor team, an operations group, or a client — the team produces some collection of documents that travel with the code. In the worst case the collection is whatever happened to get written during the engagement, sorted by accident. In the best case the collection is a small, named, bounded set of artefacts where each has a stated audience, a stated acceptance criterion, and a stated lifecycle past the handoff.

The bounded set is what professional-services and consulting practice generally call the **deliverable taxonomy** or **handoff package**. The word "taxonomy" matters: it implies a closed classification, where every artefact has a slot, and where the absence of a slot for a piece of work tells you the work probably is not a deliverable. Open documentation produces sprawl; a taxonomy produces scope discipline.

This reading walks through the canonical artefacts that show up in modern engineering handoff packages, the audiences each serves, and the trade-offs of expanding vs. contracting the taxonomy. The reading is deliberately generic — your programme's specific taxonomy is named in the overview; here we cover the engineering practice that surrounds *any* such taxonomy.

## 3. Core Concepts

### 3.1 Three audiences, not one

The single most useful mental move when assembling a handoff package is to recognise that the artefacts do **not** share an audience. They split roughly into three reader groups:

- **Operators.** People who keep the system running. They need step-by-step procedures, alert thresholds, escalation paths.
- **Successor engineers.** People who will change the code next. They need design rationale, known weaknesses, architectural context, and getting-started instructions.
- **Managers and auditors.** People who need to assess the system without reading code. They need cost shape, risk posture, decision history, and security attestations.

A single document trying to serve all three audiences is almost always worse than three short documents each tuned to one. This is the structural reason most engineering blogs that publish handoff-package designs (Syntal's "10 docs that compound" is a good example) keep the runbook separate from the architecture-decision record separate from the known-issues file: each has a different reader.

### 3.2 The core artefact set

Across modern engineering handoff packages, a recurring set of named artefacts shows up. Each is described below with its audience and its acceptance shape.

#### Runbook
A tactical, step-by-step checklist for recurring operational tasks and incident response. The runbook is the document an on-call engineer opens at 3 AM when an alert fires; it should answer "what do I do right now?" with concrete, named commands. Rootly's runbook guidance lists role assignments, communication templates, validation checks, and runbook-version metadata as standard components. Audience: operators. Acceptance: every named alert in the monitoring system has a runbook entry; every entry has been dry-run within the last quarter.

#### ADR catalog
An Architectural Decision Record (ADR) captures a single architectural decision, the context that produced it, the alternatives considered, and the consequences accepted. The ADR.GitHub.io community page treats the collection as a **decision log** of the system. Audience: successor engineers. Acceptance: every load-bearing architectural choice in the system has an ADR; every ADR has the same template (typically Context / Decision / Status / Consequences); an INDEX links them.

> **ADR-amendment convention.** Multi-amendment ADRs follow the convention established in W04 Fri (`5-Friday/5-amended-plan-spec.md`). Each amendment appends underneath the previous with its own date + status header; if a third amendment refines the first amendment, slot 1 references the first amendment explicitly.

#### Known-weaknesses inventory
A list of the failure modes, gaps, and technical debt the team is *aware of* and *deliberately leaving*. The point is honesty: each entry names the weakness, the reason it was not addressed (cost, scope, timing), and the conditions under which it would warrant addressing later. This artefact is often missing from open-source projects but is a hallmark of mature engagements — it is the artefact that lets a successor team distinguish "this is a bug we'll find" from "this is a known compromise the original team named." Audience: successor engineers + managers.

#### Handoff README
The single entry-point document a new engineer reads first. It says where the code lives, how to run it locally, where the runbook is, where the ADRs are, who to ask for what. Multishoring's IT project handover checklist treats this as the "knowledge transfer" artefact and lists it as the failure-mode-prevention step that distinguishes a controlled handoff from a chaotic one. Audience: successor engineers. Acceptance: a new engineer can be productive within their first day using only the README's pointers.

#### Eval report
A structured account of how the system was evaluated — what was measured, what the results were, what the test coverage was, and what the known evaluation gaps are. For systems with AI components this includes model-evaluation metrics; for any system it includes performance and correctness metrics. Audience: managers + auditors.

#### Security attestation
A statement of the security posture: threat model, control implementations, known unaddressed risks, dependency inventory and known-vulnerability status. Audience: managers + auditors + (for regulated systems) third-party assessors.

#### Threat model + cost analysis
Often paired. The threat model is a structured account of what could go wrong from an adversarial perspective (STRIDE or similar). The cost analysis is a forward-looking statement of operational cost shape — what scales with usage, what is fixed, where the cost cliffs are. Audience: managers.

#### Authority-boundary or HITL document
For systems with autonomous or AI-driven behaviour, a statement of *what the system is allowed to do without a human in the loop* and *what requires human approval*. This artefact is increasingly load-bearing for AI-integrated systems and is often the artefact that audit conversations focus on. Audience: managers + auditors + operators.

### 3.3 What "closed" means in a taxonomy

A taxonomy is **closed** if the team has agreed in advance that the artefact list is fixed and any candidate document either fits an existing slot or gets rejected. Closing the taxonomy has two effects:

1. **It bounds scope.** When an engineer thinks "I should write a deployment-automation guide," the closed taxonomy forces the question "which artefact does this belong in?" The answer is usually "the runbook" — and the work becomes a runbook section, not a new document.
2. **It surfaces gaps.** When the team finds something that *doesn't* fit any slot, that is information. Either the taxonomy is missing an artefact (a real gap), or the work is out of scope (a scope-discipline signal).

Open documentation produces an ever-growing wiki. Closed taxonomies produce a small, navigable handoff package. The TOGAF Architecture Content Framework formalises this distinction between "deliverables" (the named contractual outputs) and "artefacts" (the building blocks inside them); the same principle applies at the engagement scale.

### 3.4 Lifecycle beyond handoff

Each artefact also has a stated lifecycle past the handoff: does it freeze at handoff, or does the successor team continue to update it? Runbooks must continue to evolve as the system evolves. ADRs typically freeze (decisions are made once; new decisions get new ADRs). Known-weaknesses inventories get items closed-out as they are addressed. Security attestations get refreshed on a cadence. Specifying the lifecycle at handoff prevents the next team from inheriting documentation they don't know whether they own.

## 4. Generic Implementation

Below is a concrete example of a deliverable-taxonomy declaration for a **regional-bank online-banking platform handoff** — a fintech system, deliberately outside the federal-acquisitions domain.

```yaml
# handoff-taxonomy.yml — the closed artefact list for the OnlineBanking v3 handoff
# Generated 2026-05-26. Reviewed by delivery lead + ops lead + product manager.

artefacts:

  - id: runbook
    path: docs/runbook.md
    audience: ops on-call (Tier 1 + Tier 2)
    acceptance:
      - every alert in PagerDuty has a runbook section
      - every section has been dry-run by an ops engineer in the last 60 days
      - rollback procedure tested in staging within the last quarter
    lifecycle: continues to evolve post-handoff (ops owns)

  - id: adr-catalog
    path: docs/adrs/INDEX.md
    audience: successor engineering team
    acceptance:
      - one ADR per load-bearing architectural choice
      - uniform template (Context / Decision / Status / Consequences / Date)
      - INDEX.md links every ADR with one-line summary
    lifecycle: append-only post-handoff (new decisions = new ADRs)

  - id: known-weaknesses
    path: docs/known-weaknesses.md
    audience: successor engineering + product
    acceptance:
      - each entry: weakness / why-deferred / re-evaluation-trigger
      - reviewed line-by-line in handoff meeting
    lifecycle: items close-out as addressed; new items added as discovered

  - id: handoff-readme
    path: docs/handoff.md
    audience: incoming engineer (day 1)
    acceptance:
      - links to every other artefact
      - local-setup instructions verified on a clean laptop
      - 'first day' checklist for incoming engineer
    lifecycle: frozen at handoff, archived after 90 days

  - id: eval-report
    path: eval/REPORT.md
    audience: client product owner + risk officer
    acceptance:
      - test-coverage metrics for backend + frontend
      - load-test results (p50/p95/p99)
      - fraud-detection model evaluation if AI components present
    lifecycle: refresh quarterly

  - id: security-attestation
    path: docs/security-attestation.md
    audience: bank CISO + auditor
    acceptance:
      - control list (PCI-DSS scoped components)
      - SBOM (software bill of materials)
      - known unaddressed risks named
    lifecycle: refresh on regulatory cadence (annual minimum)

  - id: threat-model-cost
    path: docs/threat-model.md, docs/cost-analysis.md
    audience: client architecture review board
    acceptance:
      - STRIDE-style threat enumeration
      - per-month cost shape under three load scenarios
    lifecycle: refresh on architectural change

# CLOSED LIST. New candidate artefacts go through architecture review.
```

Annotations:

- The list is **declared closed**. New candidates are not added casually.
- Each artefact has a **named audience** so a writer can ask "would *this reader* find *this content* useful?" before adding it.
- Each artefact has an **acceptance criterion** so "done" is testable.
- Each artefact has a **lifecycle**, which prevents the successor team from inheriting maintenance ambiguity.

## 5. Real-world Patterns

**SRE runbook libraries at large search/ads companies.** Google's SRE practice (documented widely in the *Site Reliability Engineering* book series) treats the runbook as a first-class artefact and explicitly separates it from the architecture-decision history. Google SRE teams maintain runbook directories per service with a stable structure (alert → symptom → diagnosis → mitigation → contact); the architectural reasoning lives elsewhere. This separation enforces audience-discipline: an on-call engineer never has to read an ADR to answer "is this alert real?"

**ADR adoption at a healthcare-EHR vendor.** Healthcare EHR vendors (Epic, Cerner) and the broader medical-imaging vendor space have adopted ADRs because the regulatory requirement for design-history traceability (FDA design controls for medical devices, ISO 13485) maps almost directly onto the ADR pattern. The same template that satisfies an engineering audience satisfies a regulatory auditor — because both want "decision + alternatives + rationale + consequences." This is a case where one artefact, *because of audience-discipline*, manages to serve two distant audiences.

**Game-studio "post-mortem packages" as taxonomy.** Game studios (CD Projekt, Naughty Dog, Larian) publish post-mortem retrospectives months after a major title ships. The post-mortem package is a small, closed set of artefacts: technical post-mortem, design post-mortem, production post-mortem. Each has a distinct audience (engineers, designers, producers). The closed-set discipline is what makes the post-mortems publishable — the studio knows exactly what it's saying and to whom.

**E-commerce platform-migration handoffs.** Large e-commerce migrations (Shopify Plus rollouts, BigCommerce enterprise moves) routinely use a handoff README + runbook + ADR triad as the contractual deliverable. The handoff README is the document the client's incoming team opens first; the runbook is the document the merchant's ops team opens at 3 AM; the ADRs are the documents the client's CTO opens when deciding next year's roadmap. Three audiences, three artefacts, one closed list.

## 6. Best Practices

- **Declare the taxonomy closed up front.** Decide the artefact list at the start of the handoff phase, not as you go. A closed list is a scope-discipline tool.
- **Name the audience for every artefact.** If you cannot name one specific role that will read it, the artefact does not belong in the taxonomy.
- **Write to one audience per artefact.** Multi-audience documents are read by no one — the operator skips the strategy, the strategist skips the procedures.
- **State an acceptance criterion that is testable.** "Runbook is done" should mean something checkable, not a feeling.
- **State a lifecycle past handoff.** Will the artefact freeze, refresh on a cadence, or continue to evolve? Without this, the next team inherits ambiguity.
- **Reject candidates that do not fit a slot.** The work either belongs in an existing artefact or is out of scope. Resist creating a ninth artefact at the last minute.
- **Index the artefacts from one front-door document.** The handoff README is the entry point; everything else is reachable from it within one click.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** You are the delivery lead for a 12-week engagement that ships a recommendation engine for a mid-sized e-commerce site. The client's incoming team is one platform engineer (who will own day-to-day operations) plus one principal engineer (who will own future changes). Draft a **closed deliverable taxonomy** for the handoff.

Expected components:

- An artefact list (likely 5–8 items) with one-line descriptions.
- For each artefact: which of the two incoming engineers is the primary reader, plus any third audience (e.g., the merchant's CTO who reviews quarterly).
- For each artefact: one testable acceptance criterion.
- For each artefact: lifecycle past handoff (freeze / refresh-cadence / continues-to-evolve).
- One sentence declaring the list closed and naming the review path for adding a new artefact.

**What good looks like.** A reader of your taxonomy should be able to predict which artefact a given piece of project knowledge belongs in — and should be able to spot, for any candidate document, whether it fits a slot or is out of scope. The acceptance criteria should be checkable by someone who was not on the project. The lifecycle declarations should remove all ambiguity about post-handoff ownership.

## 8. Key Takeaways

After this reading the learner should be able to answer:

- Why is a *closed* (bounded) deliverable taxonomy more useful than an open one for handoff scope discipline? *(maps to LO 1)*
- What is the canonical set of handoff artefacts (runbook, ADRs, known-weaknesses, README, eval report, security attestation, threat model + cost, authority-boundary doc) and which audience does each serve? *(maps to LO 2, 3)*
- How do you decide which artefact a given piece of project knowledge belongs in, and how do you recognise when it does not fit any slot? *(maps to LO 4)*
- What changes about an artefact's specification when its lifecycle past handoff is "freeze" vs. "refresh-cadence" vs. "continues-to-evolve"? *(maps to LO 2, 4)*

## Sources

1. [10 Docs That Compound: ADRs, Runbooks & "Why" Files — Syntal on Medium](https://medium.com/@sparknp1/10-docs-that-compound-adrs-runbooks-why-files-f9fb1c38b582) — retrieved 2026-05-26
2. [Incident Response Runbooks: Templates, Examples & Guide — Rootly](https://rootly.com/incident-response/runbooks) — retrieved 2026-05-26
3. [Architectural Decision Records (ADRs) — adr.github.io](https://adr.github.io/) — retrieved 2026-05-26
4. [IT Project Handover Checklist – 8 Steps — Multishoring](https://multishoring.com/blog/it-project-handover-checklist-steps-for-a-seamless-transition/) — retrieved 2026-05-26
5. [Deliverables, Artifacts, and Building Blocks — Decoding the TOGAF Architecture Content Framework](https://medium.com/@leoyeh.me/deliverables-artifacts-and-building-blocks-decoding-the-architecture-content-framework-in-togaf-5c308ff9ef35) — retrieved 2026-05-26

Last verified: 2026-05-26
