---
week: W01
day: Tue
topic_slug: what-this-intensive-is-programme-arc-plus-the-12-fde-situations
topic_title: "What this intensive is — programme arc + the 12 FDE situations"
parent_overview: W01/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://newsletter.pragmaticengineer.com/p/forward-deployed-engineers
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://jobs.lever.co/palantir/dab396d4-2f14-4796-aac0-0d82883dccf0
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.everestgrp.com/palantir-inside-the-category-of-one-forward-deployed-software-engineers-blog/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://interview.norahq.com/interview-guides/palantir-technologies-forward-deployed-engineer-associate-interview-guide-2025
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# What this intensive is — programme arc + the 12 FDE situations

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define the Forward Deployed Engineer (FDE) role in their own words and explain how it differs from a traditional software engineer, a sales engineer, and a management consultant.
- Name at least eight of the twelve "FDE situations" that vocabulary the rest of the programme uses and identify which two they are least comfortable with from prior work.
- Articulate the three-phase arc — pre-cohort foundation, intensive, on-the-job — that this six-week programme sits inside.
- Explain why the role has exploded in demand since the 2024 wave of LLM-based enterprise products.

## 2. Introduction

The Forward Deployed Engineer is one of the fastest-growing engineering job categories of the mid-2020s. Pragmatic Engineer's 2024 deep-dive on the role notes that a16z called it "the hottest job in tech," and the same piece tracks adoption from its Palantir origins in the early 2010s into OpenAI, Anthropic, Ramp, Salesforce, and a long tail of AI startups. The proximate cause is the same one every cohort here will live with: enterprises buying LLM-based products cannot integrate them on their own, so the vendor sends engineers in.

The role is not new in shape — embedded consulting engineers existed before — but the *naming* matters because it draws a line between three job categories that get conflated. A traditional software engineer (a "Dev" in Palantir's vocabulary) builds **one capability for many customers**: a feature that ships in the product. A management consultant builds **a recommendation document**: a slide deck the client implements themselves. An FDE sits between them — building **many capabilities for one customer**, in their environment, on production-ready software the client will run after the FDE leaves. The Pragmatic Engineer piece quotes Palantir's framing directly: "you'll work in small teams and own end-to-end execution of high-stakes projects."

The Karsun-FDE intensive teaches that mode of work specifically for federal-modernization engagements. Federal clients carry constraints — FedRAMP authorisation, FAR/DFARS contracting rules, multi-tenant systems serving multiple agencies, OIG audit trails — that commercial FDEs rarely face. The 12-situation vocabulary you learn this week is the operational shorthand the programme uses to teach those constraints alongside the AI-engineering content.

## 3. Core Concepts

### 3.1 The Forward Deployed Engineer role — three definitions worth holding together

The Palantir definition (from the company's own job postings, captured at jobs.lever.co): FDEs **embed directly with customers** to configure existing software platforms to solve the customer's toughest problems. The Everest Group analysis frames Palantir's FDE practice as "the category of one" because, until 2016, Palantir had more FDEs than ordinary software engineers — the company's commercial growth was bottlenecked by deployment capacity, not feature development.

The Pragmatic Engineer 2024 framing widens the lens: "FDEs alternate between embedding with customer teams and working with core product engineering teams … combining software development, sales support, and platform engineering." This dual identity is what differentiates the FDE from a sales engineer (whose deliverable is the sale) or a solutions architect (whose deliverable is a reference architecture). The FDE's deliverable is **working code in the client's environment**.

The 2025 Palantir interview-guide framing emphasises what *day-to-day* feels like: the canonical FDE week mixes "deploying and customizing" platforms, "monitoring, debugging, deploying, or configuring" software, and "reviewing pull requests" — alongside customer-facing workshops, requirements synthesis sessions, and stakeholder demos. The work is engineering work; the *context* is the customer's mission.

### 3.2 The three-phase programme arc

| Phase | Duration | What happens |
|-------|----------|--------------|
| **Phase 1 — pre-cohort foundation** | Variable | Async self-paced foundation: language fundamentals (Java/Python), basic web, Git, an LLM intro. Already complete before W1 Mon. |
| **Phase 2 — this 6-week intensive** | 6 weeks | What you are in. Cohort-based, instructor-led, 8 hr/day Mon–Fri, paired work, real codebase, two gate defences and a final showcase. |
| **Phase 3 — on-the-job development** | 6 months | Post-W6. Coach check-ins monthly. You're on real client engagements but with a named coach you can escalate to. |

Phase 2 is where the *role-specific* muscles get built. Pre-cohort foundation gives you "engineer" — Phase 2 gives you "forward-deployed." Phase 3 gives you "experienced."

### 3.3 The 12 FDE situations — the vocabulary the programme operates in

These 12 names recur every day for six weeks. They are not a curriculum sequence — they are a **diagnostic vocabulary** for the situations FDEs actually face at clients. Read them as questions you'll ask of every system you walk into.

1. **Data comprehension** — what is this data, who owns it, what's stale, what's missing?
2. **Tech-stack comprehension** — what is this codebase, why is it shaped this way, where are the seams?
3. **Requirements synthesis** — what does the client actually need (vs what they asked for)?
4. **Estimation** — what is this going to cost in time, money, complexity?
5. **6R and system mapping** — Retain / Rehost / Replatform / Refactor / Repurchase / Retire. Knowledge-graph + Context-graph thinking.
6. **Non-functional requirements** — performance, security, observability, compliance, accessibility.
7. **Cloud thinking** — what does production look like on AWS?
8. **DevOps thinking** — how does code get from a PR into production safely?
9. **Database thinking** — relational, document, vector — which when?
10. **Security thinking** — OWASP, OWASP LLM Top 10, multi-tenant boundaries, secret management.
11. **Architecture thinking** — ADRs, trade-off framing, defending decisions under probing.
12. **Design thinking** — user outcomes, not feature lists; what does the actual end-user need?

The list overlaps deliberately — situations 5 (6R) and 7 (cloud) both touch migration; situations 6 (NFRs) and 10 (security) both touch compliance. The overlap is the point. Real client problems do not respect category boundaries.

### 3.4 Why demand exploded in 2024-2025

The Pragmatic Engineer piece is unambiguous about the cause: enterprises bought LLM-based products faster than their internal teams could integrate them. The same pattern that drove Palantir's FDE practice in the 2010s (complex data-integration software needs embedded engineers to land) now drives OpenAI's, Anthropic's, Salesforce's, and Ramp's. The Everest Group analysis frames this as a structural shift: vendors of "platform" software whose value is realised through customer-specific configuration *must* have an FDE practice or they will not scale their bookings.

For Karsun and other federal-systems integrators, the same dynamic applies with a federal twist. AI capability is rapidly available; the integration into FedRAMP-bounded, FAR-contracted, multi-tenant federal systems is not. The FDE role exists to close that integration gap.

## 4. Generic Implementation

There is no "implementation" of a role in the code-snippet sense. The closest analogue is a **role rubric** — the explicit framing an FDE uses to triage what kind of work they're doing in any given hour. Below is a generic FDE-week rubric, drawn from the Palantir interview-guide and Pragmatic Engineer descriptions, that you can apply across industries:

```
WEEK RUBRIC — Forward Deployed Engineer (generic)

Each work block, name the situation:
  [ ] Comprehension       — I am reading code/data/docs to understand the system
  [ ] Synthesis           — I am reframing what the client said into what they need
  [ ] Build               — I am writing/modifying production code in the client env
  [ ] Deploy/operate      — I am putting code into prod or keeping it running
  [ ] Communicate         — I am telling the client (or internal team) what I found
  [ ] Influence-product   — I am feeding back what's missing in our platform

If a week has zero [Build] blocks: you may be drifting into consulting.
If a week has zero [Communicate] blocks: you may be drifting into pure engineering.
If a week has zero [Influence-product] blocks: the platform team loses a feedback channel.
```

The rubric is deliberately generic — it works for a fintech FDE landing a payments-platform deploy, for a healthcare FDE configuring a clinical-decision-support system, or for a logistics FDE wiring a fulfilment-orchestration tool into a 3PL's stack. The categories don't change; the systems do.

## 5. Real-world Patterns

**Fintech (Ramp).** Ramp's engineering blog and the Pragmatic Engineer piece both highlight Ramp's FDE practice as a sales-multiplier: an FDE lands with a new enterprise customer in week 1, builds the custom-rules integration into the customer's general-ledger workflow over weeks 2–4, then transitions to support while the customer's own engineers take over. The pattern is **land-build-transition**, not land-and-leave.

**AI platforms (OpenAI, Anthropic).** Both companies hired aggressively into FDE roles in 2024-2025 to address LLM-integration complexity at enterprise customers. OpenAI's job postings (visible on the company's careers page during 2024-2025) describe the FDE as the engineer who actually wires Assistants API or fine-tuned-model deployments into the customer's product. The pattern is **vendor-engineer-as-integration-specialist**, replacing the older "professional services" model that pure SaaS vendors used to outsource.

**Healthcare IT (Epic, Cerner / Oracle Health).** The traditional electronic-health-record vendors have long had on-site implementation engineers who functionally serve the FDE role — embedding with a hospital for the 12–18-month rollout, configuring schemas to the hospital's workflows, and training the hospital's IT team on the handoff. The Everest Group piece notes this is the original "FDE-shaped" job, predating Palantir by decades; Palantir's contribution was the *name* and the explicit framing.

**E-commerce (Shopify Plus).** Shopify Plus's "Launch Engineer" role for large-merchant migrations is functionally an FDE — embedded with the merchant for the 60–90-day migration window, building custom apps and Liquid templates in the merchant's environment. Worth knowing if you ever interview at e-commerce platforms post-Karsun.

## 6. Best Practices

- **Always name the FDE situation you're in at the top of every work block.** "I am doing tech-stack comprehension on the api-gateway right now" beats "I'm just looking around."
- **Treat consultant-style deliverables (slide decks, recommendation docs) as supporting artefacts, not primary deliverables.** Your primary deliverable is code in the client environment.
- **Feed product feedback to the platform team weekly, in writing.** If your platform's gaps are only visible to FDEs in the field, the platform won't improve.
- **Estimate in ranges, not points.** "2–5 days, depending on whether the auth integration goes through their SSO" beats "3 days."
- **Verify the client's stated requirement against the client's actual data before you build.** Requirements-synthesis (situation 3) precedes build, always.
- **Default to reading code over reading docs.** Federal codebases especially: the README is older than the code in most cases.
- **Keep a personal "12-situations" diary for the first six months on engagement.** When you can name your weak situations, your coach can target them.

## 7. Hands-on Exercise

**Self-diagnostic (10 minutes, no code, whiteboard or notebook):**

For each of the 12 FDE situations, rate yourself on a 1–5 scale based on prior work experience (1 = "I've never done this," 5 = "I could lead a client conversation on this tomorrow"). Then identify:

- The **two situations** you rated lowest. These are your W1 Tue diagnostic answers.
- The **one situation** you rated highest. This is the situation you'll be expected to mentor your pair on during W2–W3.
- The **one situation** you'd most like to grow in by W6. This goes in your private 1:1 doc; the W3 Mon 1:1 will reference it.

**What good looks like:** an honest, specific diagnostic. "Database thinking, 2 — I've used Postgres on small apps but never designed for multi-tenant audit trails" is more useful than "Database thinking, 3 — average." The diagnostic informs pair assignment and coach attention; specific weaknesses get specific attention.

## 8. Key Takeaways

- *Can I define an FDE in one sentence and contrast them with a Dev, a sales engineer, and a consultant?* (Maps to LO 1.)
- *Can I name at least 8 of the 12 FDE situations without looking at the list?* (Maps to LO 2.)
- *Can I explain the three-phase programme arc and place Phase 2 (this intensive) inside it?* (Maps to LO 3.)
- *Can I explain why the FDE role exploded in demand around the 2024 enterprise-LLM wave?* (Maps to LO 4.)

## Sources

1. [Forward Deployed Engineers: why they're in demand](https://newsletter.pragmaticengineer.com/p/forward-deployed-engineers) — retrieved 2026-05-26
2. [Palantir — Forward Deployed Software Engineer (job posting)](https://jobs.lever.co/palantir/dab396d4-2f14-4796-aac0-0d82883dccf0) — retrieved 2026-05-26
3. [Everest Group — Palantir: Inside the category of one — forward deployed software engineers](https://www.everestgrp.com/palantir-inside-the-category-of-one-forward-deployed-software-engineers-blog/) — retrieved 2026-05-26
4. [Palantir FDE Associate Interview Guide 2025](https://interview.norahq.com/interview-guides/palantir-technologies-forward-deployed-engineer-associate-interview-guide-2025) — retrieved 2026-05-26

Last verified: 2026-05-26
