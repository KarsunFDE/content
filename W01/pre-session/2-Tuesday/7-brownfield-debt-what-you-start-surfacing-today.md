---
week: W01
day: Tue
topic_slug: brownfield-debt-what-you-start-surfacing-today
topic_title: "Brownfield debt — what an inventory is and how to do one"
parent_overview: W01/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://aws.amazon.com/blogs/migration-and-modernization/a-framework-for-accelerated-modernization-and-technical-debt-reduction/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.ibm.com/think/insights/reimagining-brownfield-application-modernization
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.mckinsey.com/capabilities/mckinsey-digital/our-insights/breaking-technical-debts-vicious-cycle-to-modernize-your-business
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://arxiv.org/html/2403.06484v1
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://agileengine.com/technical-debt-and-modernization-three-lessons-learned/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Brownfield debt — what an inventory is and how to do one

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define "brownfield" in contrast to "greenfield" and explain why brownfield modernisation is the **default** at federal-systems integrators (and most established enterprises).
- Name at least four distinct **categories** of technical debt (code, architecture, infrastructure, security, dependency / supply-chain) and give an example of each.
- Describe the **inventory-first principle**: why no modernisation conversation produces credible estimates without an inventory, and why "we'll figure it out as we go" is the dominant failure mode.
- Apply the **six standard inventory dimensions** (artefact + location + severity + risk + effort + suggested-R) to a real debt item.

## 2. Introduction

A *brownfield* project builds on top of existing systems, code, or infrastructure — as opposed to *greenfield*, which starts from a blank slate. The IBM Think piece captures the asymmetry: "brownfield projects involve building on top of existing systems, codebases, or infrastructure, with development teams working within constraints while integrating new features, migrating components, or modernising platforms without starting from zero." That "without starting from zero" clause is doing all the work. At a federal-systems integrator, **brownfield is the default**. Greenfield work — pure new-build with no prior code, no existing data — is exceptional. The interesting question is therefore "how do we modernise brownfield well?"

McKinsey Digital 2023 names the dominant antipattern: organisations modernise reactively, in response to incidents or audits, without ever building a complete picture of what they own. The result is a "vicious cycle" — every fix produces two new urgencies because the team never holds the landscape in view. The way out is unsexy: **an explicit inventory**. The AWS 2023 framework, the IBM Think piece, and the 2024 arXiv survey all converge: the inventory **is** the first deliverable, and it's a *living* document.

This reading walks the inventory discipline generically. The 12 deliberate debt items in `acquire-gov` are the *applied* version; the inventory you start Tuesday afternoon is the muscle. At a client engagement, you'll start an inventory in Week 1 of every brownfield project.

## 3. Core Concepts

### 3.1 Greenfield vs brownfield in one diagram

```
GREENFIELD                              BROWNFIELD
─────────                              ─────────
blank slate                            existing system (5–20 yrs of code)
no users yet                           users in prod right now
no SLAs                                SLAs you can't violate
no legacy data                         data with history & quirks
team picks the stack                   stack inherited & constrained
"what should we build?"                "what do we already have, and what
                                        do we change without breaking it?"
```

The AgileEngine 2024 article on technical-debt-and-modernization observes that the *engineering muscles* are completely different. Greenfield engineering rewards rapid prototyping, frequent rewrites, optimisation later. Brownfield engineering rewards **comprehension**, **risk modelling**, **incremental change with safety nets**. The Karsun-FDE programme spends most of its time training the brownfield muscle because that's what the post-W6 engagements actually need.

### 3.2 What "technical debt" actually means

The 2024 arXiv survey "Technical Debt Management: The Road Ahead for Successful Software Delivery" frames technical debt as **the gap between the codebase's current state and the state it would be in if every shortcut, deferred upgrade, and accumulated workaround had been addressed at the time it was incurred**. The metaphor is borrowed from finance: shortcuts are loans that accrue interest (in the form of slower feature work, increased bug rates, more incidents) until paid down.

The arXiv survey emphasises a point easy to miss: "A large portion of technical debt was introduced many years prior; it remains hidden and undocumented, and involves design or architecture issues that cannot be mined by analysing source code." Static analysis tells you about code-style debt and dependency-version debt; it does not tell you about architectural debt (the monolith that should have been microservices) or about *missing* artefacts (the documentation that should exist and doesn't). Inventory is how those debts surface.

### 3.3 The five debt categories every brownfield system has

Synthesising AWS, IBM, McKinsey, and arXiv-survey taxonomies, technical debt clusters into five canonical categories. Every brownfield inventory should have items in each:

| Category | What it covers | Example debt item |
|----------|----------------|--------------------|
| **Code-level** | Code smells, anti-patterns, dead code, duplicated logic, untested paths | "Form validation missing on user-registration endpoint" |
| **Architecture-level** | Wrong service boundaries, gateway-bypass paths, missing layers, tight coupling | "Frontend bypasses API gateway and calls service directly" |
| **Infrastructure / deployment** | Outdated container base images, missing healthchecks, `:latest` tags, no IaC | "Postgres volume not persisted; data resets on container restart" |
| **Dependency / supply-chain** | Outdated library versions, EOL frameworks, deprecated SDKs, unpinned versions | "Spring Boot 2.7.18 (community-EOL Aug 2025); AWS SDK v1 (general support ended Dec 2023)" |
| **Security / compliance** | Skipped validation, hard-coded secrets, missing audit trails, outdated TLS, OWASP issues | "JWT signature validation skipped if header malformed" |

The McKinsey 2023 piece adds a sixth, often-omitted category — **knowledge debt**: the cohort of senior engineers who carry the system in their heads. When they leave, the system becomes harder to modernise because the implicit knowledge evaporates. Knowledge debt is hardest to inventory because it isn't in the repo — but the *consequences* (undocumented business rules, unclear ownership) show up in code and infra.

### 3.4 The inventory-first principle

The AWS 2023 framework paper makes the inventory-first argument explicit: "Any serious tech modernisation effort starts with an accurate and detailed accounting of current technical debt, documenting assets, data, and their links to business value, which enables building meaningful support in the business, setting realistic budgets, making accurate allocations, and prioritising initiatives." The McKinsey piece reinforces it: without inventory, every modernisation conversation devolves into "I think we should rewrite X" — *I think* is the problem; the inventory removes the *think*.

The 2024 arXiv survey identifies the antipattern: teams jump to fixing the *visible* debt (the recent incident's proximate cause) without holding the rest of the landscape in view. The result is whack-a-mole — every fix produces two new urgencies because changes ripple through unmapped dependencies. The inventory **precedes** prioritisation, which **precedes** any fix.

### 3.5 The six standard inventory dimensions

An inventory item should record at minimum:

| Dimension | Question it answers | Example |
|-----------|---------------------|---------|
| **Artefact** | What is the debt item? One specific named thing. | "Long-lived AWS access keys in `.github/workflows/ci.yml`" |
| **Location** | Where exactly? File + line, service + endpoint. | `.github/workflows/ci.yml:23-27` |
| **Category** | Which of the 5 (or 6) canonical categories? | Security / Compliance |
| **Severity** | Worst-case impact if unaddressed? | High — credential exfiltration → AWS account compromise |
| **Effort** | T-shirt size. S (≤1d) / M (≤1w) / L (≤1 sprint) / XL (multi-sprint). | M — OIDC + IAM role + workflow change |
| **Suggested R** | Mapping to 6R/7R: Retain / Rehost / Replatform / Refactor / Repurchase / Retire. | Refactor |

A worthwhile extra for federal contexts: **Compliance touch-point** — does this item trigger an OIG / FedRAMP / FISMA control? Often the load-bearing argument for prioritising.

### 3.6 The 6R / 7R framing

The AWS 2023 framework references the canonical six Rs of modernisation:

| R | What it means |
|---|---------------|
| **Retain** | Keep as-is. Right for systems near EOL or with no modernisation business case. |
| **Rehost** | "Lift and shift" — move to new infra (on-prem → cloud) without changing the app. |
| **Replatform** | Change the underlying platform without changing the app (self-managed Postgres → RDS). |
| **Refactor** | Change the application itself — break the monolith, modernise the framework. |
| **Repurchase** | Drop the in-house system; buy SaaS that does the same job. |
| **Retire** | Turn it off. No replacement; the business no longer needs the capability. |

Some frameworks add **Relocate** (cross-region / cross-provider) for 7R; 6R is sufficient for inventory purposes.

The crucial point: 6R is **per-service**, not per-application. The same brownfield system can have a Retain service, a Refactor service, and a Retire service in the same modernisation. Every debt item's suggested-R may differ.

## 4. Generic Implementation

A generic five-row inventory for a hypothetical mid-size **logistics platform** (a regional carrier-management SaaS, ~8 years old, recently AWS-migrated but not modernised). Every entry uses the six-dimension schema:

```markdown
| # | Artefact | Location | Category | Severity | Effort | Suggested R |
|---|----------|----------|----------|----------|--------|-------------|
| 1 | Long-lived AWS keys in CI | .github/workflows/deploy.yml:12-16 | Security | High | M | Refactor (→ OIDC) |
| 2 | Node 14 runtime (EOL Apr 2023) | services/route-planner/Dockerfile:1 | Dependency | High | M | Replatform (→ Node 22 LTS) |
| 3 | No correlation IDs across services | every service's log pipeline | Architecture | Medium | L | Refactor (add OpenTelemetry) |
| 4 | Postgres `:latest` tag | docker-compose.yml:34 | Infrastructure | Medium | S | Replatform (pin to 16.x) |
| 5 | Carrier-portal SPA bypasses gateway | frontend/src/app/services/carrier.ts:8 | Architecture | High | M | Refactor (route via gateway) |
```

Reading the table: row 1 + row 5 are both high-severity but address different categories; both need to be fixed but the order matters (gateway-bypass is the prereq for centralising auth at the gateway, which is the prereq for retiring the long-lived AWS keys). The inventory **does not by itself prioritise** — it surfaces the artefacts and lets prioritisation be a downstream conversation with stakeholders.

For each row, the FDE adds prose underneath when "what's the worst case?" deserves more than a single severity word:

> **#1 — Long-lived AWS keys in CI.** Severity high because credential exfiltration (CI logs, fork-PR exposure, departed-employee retention) escalates to full AWS account compromise. Effort medium: one-time IAM-role + trust-policy setup, workflow change, secret deletion. Suggested R: Refactor the auth pattern itself — not Replatform (no infra move) and not Repurchase (no SaaS to swap to).

That density — six dimensions plus a sentence of context per item — is the inventory artefact federal CIOs read in a budget meeting.

## 5. Real-world Patterns

**Fintech (mid-stage SaaS post-acquisition).** A venture-funded fintech gets acquired by a larger bank; the bank's compliance team demands a full debt inventory before integration. The acquired team spends 4–6 weeks producing it and discovers half their critical paths run on EOL Node, hardcoded keys, and a single-AZ Postgres. The pattern: **M&A diligence as forcing function for the inventory** — and the team is usually grateful afterwards because they finally see what they have.

**Healthcare IT (EHR-vendor implementation).** Epic and Oracle Health (Cerner) implementations start with a multi-month inventory of the hospital's existing systems — every interface, every data feed, every shadow-IT spreadsheet. The inventory feeds the 6R conversation explicitly: which feeds Retain, which Replatform onto the new EHR, which Retire. Pattern: **inventory as contractual scope-of-work artefact**.

**E-commerce (Shopify's "Built for Shopify" review).** Apps applying for the "Built for Shopify" badge undergo a Shopify-led technical audit that is functionally a third-party inventory: location, category, severity, effort. Pattern: **third-party audit-as-inventory** — sometimes you don't produce the inventory; you receive one.

**Gaming (live-ops at long-lived MMOs).** Studios like CCP Games (EVE Online) publish quarterly debt updates in dev-blogs — engineers name specific items, their suggested R, and the quarter they intend to address. Pattern: **inventory as community-facing transparency**, where players become advocates for the debt that affects their gameplay most.

## 6. Best Practices

- **Always inventory before you fix.** Even when an incident is in flight, write the item down before patching — otherwise it disappears from the queue.
- **Use the six-dimension schema consistently.** Artefact, location, category, severity, effort, suggested-R. Drop dimensions only when they genuinely do not apply.
- **Inventory items are short and specific.** "Improve security" is not an inventory item; "JWT signature validation skipped if header malformed in `api-gateway/SecurityConfig.java:41`" is.
- **Categorise even when uncertain.** Force-pick a category from the canonical list; the act of categorising is itself diagnostic.
- **Hold severity and effort separate.** A high-severity / low-effort item should pop to the top; a high-severity / high-effort item should drive sprint planning; a low-severity / low-effort item should fill bus-factor recovery slots.
- **Treat the inventory as living.** Re-review monthly; promote stale items, demote resolved ones, add newly-discovered ones.
- **Make the inventory the artefact you point CIOs at.** "What are we doing about technical debt?" deserves a document answer, not a slide.

## 7. Hands-on Exercise

**Whiteboarding inventory drill (12-15 min, no code):**

You are dropped into a hypothetical **fintech-startup** monorepo on Day 1 of an engagement. You have 30 minutes to skim the repo before the kickoff call. Produce a five-row preliminary inventory based on the (made-up) observations below. Use the six-dimension schema.

Observations from your 30-minute skim:

- `package.json` shows Node 16 (EOL Sep 2023) and Express 4.17 (current EOL roadmap published).
- The frontend repo calls `https://api.fintech.internal/v2/...` directly bypassing the public gateway in 3 places.
- `.github/workflows/deploy.yml` has `aws-access-key-id: ${{ secrets.AWS_KEY }}`.
- No structured logging anywhere — all `console.log` strings.
- The Postgres production instance is `db.t3.medium` Single-AZ (no Multi-AZ replica).

**What good looks like:** five rows, each with all six dimensions populated, severities span at least three levels (high / medium / low), at least three categories are represented, the suggested-R column shows at least two different Rs (probably Refactor and Replatform). Each row should also have a one-sentence "worst case if unaddressed" note. The artefact is the *kind* of document you'd send to the client engagement lead at end of Day 1 — concise, specific, prioritisable.

## 8. Key Takeaways

- *Can I define brownfield vs greenfield and explain why brownfield is the default at federal-systems integrators?* (Maps to LO 1.)
- *Can I name four+ categories of technical debt and give an example of each?* (Maps to LO 2.)
- *Can I explain why inventory-before-fix is the discipline and what the failure mode of skipping it is?* (Maps to LO 3.)
- *Can I apply the six standard inventory dimensions to a real debt item?* (Maps to LO 4.)

## Sources

1. [AWS — A Framework for Accelerated Modernization and Technical Debt Reduction](https://aws.amazon.com/blogs/migration-and-modernization/a-framework-for-accelerated-modernization-and-technical-debt-reduction/) — retrieved 2026-05-26
2. [IBM Think — Reimagining brownfield application modernization](https://www.ibm.com/think/insights/reimagining-brownfield-application-modernization) — retrieved 2026-05-26
3. [McKinsey Digital — Breaking technical debt's vicious cycle to modernize your business](https://www.mckinsey.com/capabilities/mckinsey-digital/our-insights/breaking-technical-debts-vicious-cycle-to-modernize-your-business) — retrieved 2026-05-26
4. [arXiv 2024 — Technical Debt Management: The Road Ahead for Successful Software Delivery](https://arxiv.org/html/2403.06484v1) — retrieved 2026-05-26
5. [AgileEngine — Technical debt and modernization: three lessons learned](https://agileengine.com/technical-debt-and-modernization-three-lessons-learned/) — retrieved 2026-05-26

Last verified: 2026-05-26
