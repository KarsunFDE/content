---
week: W01
day: Wed
topic_slug: brownfield-debt-inventory
topic_title: "Brownfield-debt inventory — Wed PM continuation"
parent_overview: W01/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://arxiv.org/html/2403.06484v1
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://utkrusht.ai/blog/challenges-with-brownfield-codebases
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://onlinelibrary.wiley.com/doi/10.1155/2015/898514
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.itconvergence.com/blog/strategies-for-managing-technical-debt-in-legacy-software-systems/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.monterail.com/blog/legacy-systems-and-technical-debt-guide-to-modernization
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Brownfield-debt inventory — Wed PM continuation

> Tuesday's PM block started a brownfield-debt inventory; Wednesday continues it as the Microservices Foundation walkthrough surfaces new items. The goal by end-of-Wednesday is **7+ items logged in a structured catalogue**. This generic deep-dive lays out the **discipline of technical-debt cataloguing** — what the field knows about how to find debt, how to record it, and how to prioritise it without falling into the "fix it now" reflex. The overview (`1-DailyTopicOverview.md` §5) gives you the entry format `acquire-gov` uses and the four anchor debt items the instructor will name during the walkthrough.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define **technical debt** in the four-quadrant Fowler framing (deliberate vs inadvertent, reckless vs prudent) and place a given debt item in the correct quadrant.
- Author a **structured debt entry** with the fields a debt catalogue needs: ID, location, symptom, impact dimension, suspected fix vector, target horizon.
- Apply an **impact × effort prioritisation matrix** to a debt catalogue and distinguish "quick wins" from "strategic investments" from "tech-debt that is not actually debt".
- Resist the **fix-it-now reflex** during inventory phases and articulate why naming-without-fixing is its own discipline.
- Recognise **deliberate / pedagogical debt** in training scaffolds and distinguish it from the organic accidental debt found in production systems.

## 2. Introduction

Every long-lived software system carries debt. The metaphor — borrowed by Ward Cunningham in the 1990s, formalised by Martin Fowler and others over the subsequent two decades — frames the gap between the system's current state and the state it would be in if every decision had been made with full information [3]. The metaphor is durable because the financial analogy is accurate: debt accumulates interest (every feature ships slower than it would in a clean codebase), debt can be deliberate (a pragmatic shortcut taken with eyes open) or inadvertent (accumulated without anyone noticing), and debt eventually has to be paid down or it consumes the organisation's capacity to ship new value.

Recent industry data puts the scale in perspective: technical debt may represent up to 40% of the technology estate in large enterprises, and CIOs estimate that 10–20% of the budget intended for new products is consumed by debt-servicing work [5]. The federal context is no exception — agencies running 30-year-old systems alongside cloud-native modern ones have some of the most acute debt loads in the industry, which is precisely why programmes like Karsun ReDuX exist [6].

What the field has learned, painfully, is that you cannot fix what you have not catalogued. Engineering teams that walk into a brownfield system with the impulse "let me just clean this up" produce two outcomes: they leave the codebase no cleaner because they did not have a plan, and they introduce new bugs because they did not understand the existing constraints. The pre-modernisation discipline is the *inventory* — name every debt item, locate it, characterise its impact, and *defer the fix*. Inventory is a meta-skill; fixing is a specialist skill. The two should not happen simultaneously.

This reading walks the inventory discipline in the generic — across industries, outside any federal-acquisitions context. The afternoon block applies it to `acquire-gov`'s deliberate debt items.

## 3. Core Concepts

### 3.1 Fowler's four-quadrant technical-debt framing

Martin Fowler's widely cited four-quadrant model crosses two axes [3]:

|              | **Deliberate**                 | **Inadvertent**                |
|--------------|--------------------------------|--------------------------------|
| **Reckless** | "We don't have time for design" | "What's layering?"             |
| **Prudent**  | "We must ship now and deal with consequences" | "Now we know how we should have done it" |

The four quadrants demand different responses:

- **Deliberate-reckless** debt is the most dangerous: someone chose to skip discipline they knew was needed. The fix is cultural, not technical.
- **Deliberate-prudent** debt is the most defensible: the team took a calculated shortcut to hit a deadline and recorded the consequences. The fix is scheduled.
- **Inadvertent-reckless** debt is the largest category in most legacy systems: the team did not know what they did not know. The fix is education-plus-refactor.
- **Inadvertent-prudent** debt is the kindest category: the team did the best they could with the information available, and now better patterns exist. The fix is incremental modernisation.

A debt catalogue gains explanatory power when each item is placed in its quadrant — the same code-level symptom can sit in two different quadrants depending on the history, and the right response differs.

### 3.2 The catalogue entry — minimum viable fields

The TD-Tracker tool research [3] and subsequent industry tooling converge on a minimum viable entry shape:

```
ID:                  unique identifier (numeric or scheme-prefixed; never reused)
Title:               short noun phrase
Location:            file path + line range, or service + module
Symptom:             one sentence — what the reader observes
Impact dimension:    production / security / observability / cost / dev-experience / compliance
Severity:            high / medium / low
Effort estimate:     S / M / L (or hours/days/weeks if precision available)
Suspected fix vector: one line — what kind of change addresses it
Target horizon:      next sprint / this quarter / next major modernisation / never
Quadrant:            deliberate-reckless / deliberate-prudent / inadvertent-reckless / inadvertent-prudent
Discovered:          date + context (who found it, in what activity)
```

Three fields are load-bearing: **Impact dimension** (security debt prioritises differently than dev-experience debt), **Target horizon** (forces commitment to *when* the team will pay this down, not just *that* they will), and **Quadrant** (forces honesty about how the debt got there).

The temptation to skip the "Quadrant" field is strong because it requires admitting the team's own contribution to the debt; that is exactly why it is worth keeping.

### 3.3 Impact × effort prioritisation matrix

Once a catalogue has 10+ items, prioritisation becomes a question of allocating scarce engineering time. The standard frame is a 2×2 [4]:

|                | **Low effort**     | **High effort**         |
|----------------|--------------------|-------------------------|
| **High impact** | Quick wins         | Strategic investments   |
| **Low impact**  | Background hygiene | Tech-debt that is not actually debt |

The fourth quadrant — *low impact, high effort* — is the most important one to recognise. Items in that quadrant are theoretically debt but practically a poor use of engineering time; documenting them is enough. Treating every catalogued item as something that must be fixed is how cataloguing exercises destroy themselves.

The other three quadrants drive scheduling: quick wins go into the next sprint's slack capacity, strategic investments become modernisation projects with their own ADRs and timelines, background hygiene happens during downstream feature work as opportunistic improvements.

### 3.4 The "do not fix during inventory" discipline

The hardest discipline during a debt-inventory phase is *not fixing*. The reflex to fix-as-you-find is strong: an engineer notices a bug, has the diff in their head, and "it would take five minutes" to land the fix. The problem is twofold:

1. **The five-minute fix usually isn't.** A fix without context has a non-trivial chance of introducing a regression — especially in brownfield code where the "broken" behaviour is sometimes load-bearing for a downstream consumer no one has read recently.
2. **Inventory loses honesty when items get fixed mid-pass.** A catalogue that says "12 items found, 4 fixed, 8 remaining" loses the ability to characterise *the state of the system before any team intervened*. That state is the input to modernisation planning; corrupting it makes planning harder.

The discipline is: log the item, link the file location, write a one-line suspected fix vector, *move on*. Resist the diff. Fixing happens later, deliberately, with a plan.

### 3.5 Deliberate / pedagogical debt vs accidental debt

Training scaffolds — like `acquire-gov` — contain a third category that production code does not: **deliberately seeded debt**, planted by the curriculum author to be discovered by the cohort. Pedagogical debt:

- Is *named* in a public document (the scaffold's README or design spec) so instructors can verify the cohort found it.
- Is *bounded* — it does not cascade into systemic dysfunction the way production debt does.
- Is *not* meant to be fixed during the inventory phase; fixing it happens in a later, deliberately scheduled modernisation phase as a teaching activity.

When working in a brownfield codebase that has both kinds, the inventory discipline differs slightly: deliberate debt is logged with a note ("pedagogical — scheduled for week-N modernisation block"), accidental debt is logged with normal prioritisation. Mixing the two confuses planning later.

### 3.6 The cognitive trap: bug-hunting vs debt-cataloguing

Bug-hunting and debt-cataloguing feel similar — both involve reading code looking for problems — but they reward different mental modes. Bug-hunting rewards depth: trace one issue end-to-end, reproduce it, write the test. Debt-cataloguing rewards breadth: skim widely, note many problems, capture each in one line, do not stop to reproduce.

A team that confuses the two ends the day with one bug fully understood and twenty unwritten catalogue entries. The exercise was meant to produce twenty entries.

## 4. Generic Implementation

A generic example: a fintech team has acquired a 6-year-old loan-origination microservice from a defunct subsidiary and has two weeks to assess what they bought. The catalogue entry format the team adopts (using the minimum viable fields from §3.2) looks like this in practice:

```markdown
## DEBT-014 — JWT signature validated only at gateway

Location: services/loan-eligibility/src/main/java/.../AuthFilter.java:42-67
Symptom: AuthFilter reads X-User-Id from headers without re-validating the JWT
         signature; trusts the gateway to have done it upstream.
Impact dimension: security
Severity: high
Effort estimate: M (3-5 days incl. tests + integration verification)
Suspected fix vector: re-validate JWT signature at filter entry; introduce JWKS
                      cache with TTL; document mTLS posture in ADR.
Target horizon: next quarter (after current PCI re-cert)
Quadrant: deliberate-prudent (team chose this when first split out of the monolith
          with the gateway-trust posture; documented in original ADR-0007, but
          posture review now overdue)
Discovered: 2026-05-22 by Sam during AuthN walkthrough
```

The entry takes a single experienced engineer ~5 minutes to write once they have the information. Multiply by 20-40 items in a typical brownfield assessment and the cataloguing phase consumes 2-4 person-hours per service. That is the right scale — much less and the entries are too sparse to be useful; much more and the team has slipped into bug-hunting.

After collection, the team produces a single summary view — typically a markdown table or a spreadsheet — sorted by severity, with the impact-effort quadrant called out:

```markdown
| ID       | Title                              | Impact         | Severity | Effort | Quadrant            |
|----------|------------------------------------|----------------|----------|--------|---------------------|
| DEBT-001 | Logging format inconsistent        | observability  | medium   | M      | inadvertent-prudent |
| DEBT-014 | JWT not re-validated downstream    | security       | high     | M      | deliberate-prudent  |
| DEBT-022 | Postgres docker volume not mounted | production     | high     | S      | inadvertent-reckless|
| ...      | ...                                | ...            | ...      | ...    | ...                 |
```

The summary view is the artifact that goes to the engineering manager and the modernisation-planning meeting. It is short enough to read in one sitting. The full entries are the substrate.

## 5. Real-world Patterns

**Healthcare — EHR migration debt audits.** When U.S. health systems transitioned from on-premise EHR systems to cloud-hosted ones in 2020-2024, the consulting work that preceded each transition included a 4-6 week debt-inventory phase. Published case studies report typical findings: 200-400 catalogued items per major system, with roughly 60% in the inadvertent-prudent quadrant (accumulated organically over a decade), 25% inadvertent-reckless, 10% deliberate-prudent (documented shortcuts during prior upgrades), and 5% deliberate-reckless [4][5].

**Fintech — Stripe's "Code Yellow" patterns.** Stripe has published in engineering posts about its approach to high-impact code-quality audits: time-boxed cataloguing sprints where senior engineers walk a subsystem for 1-2 weeks producing a debt catalogue, with an explicit rule that no code changes ship during the walk. The catalogue then feeds a prioritised remediation backlog. The discipline is intentionally separated from feature work [1].

**Logistics — DHL's mainframe assessment.** DHL's modernisation of its mainframe COBOL inventory systems used a third-party assessment vendor that produced a catalogue of 1,800+ debt items across 12 systems. The catalogue itself was the deliverable for phase 1; the modernisation roadmap was phase 2. The two-phase split is now standard in mainframe-modernisation work [5][6].

**Gaming — Epic Games' Unreal Engine 4 → 5 transition.** Epic has talked publicly about the debt audit that preceded the UE4 → UE5 transition: a categorised inventory of breaking changes, internal API debt, and assumed-behaviour debt that took an internal team six months to produce. The catalogue's most valuable section, the team reported, was the "false debt" list — items that looked like debt to a fresh reader but were load-bearing for shipped games. Naming the false-debt items prevented well-intentioned cleanup PRs from breaking the engine [2][3].

## 6. Best Practices

- **Catalogue first, fix later, never both in the same day** — the inventory is the input to a planning conversation, not a TODO list to be burned down.
- **Capture the quadrant honestly** — the same symptom in deliberate-prudent vs inadvertent-reckless drives different conversations; the quadrant field is the hardest one to skip but the most useful one to keep.
- **Use the impact-effort matrix to triage, not to dismiss** — items in the low-impact/high-effort quadrant are still catalogued; they are just not scheduled.
- **Keep entries one screen each** — long entries discourage cataloguing; short entries get written.
- **Record discovery context** — knowing that an item was found during an AuthN walkthrough is itself information that later refactoring will use.
- **Distinguish deliberate / pedagogical debt from accidental debt in training scaffolds** — they are scheduled differently, and mixing them confuses planning.
- **Produce a one-page summary view at the end of every cataloguing session** — the catalogue's audience reads the summary; the substrate exists for those who want depth.

## 7. Hands-on Exercise

**(15 minutes — code reading + cataloguing.)** Pick any open-source repository on GitHub that you have not contributed to — a small framework, a CLI tool, a microservice example project. Spend 12 minutes scanning the repo (README, top-level structure, a handful of files chosen at random) and produce a debt catalogue of 5-7 items using the template in §3.2. Limit yourself to one minute per entry. Then take the remaining 3 minutes to:

1. Place each item in its Fowler quadrant (your best guess from outside the team).
2. Sort by severity.
3. Pick the one item you would propose as the team's next sprint's quick win.

**What good looks like.** The catalogue has 5-7 entries, each one screen. Every entry has a Location, a one-line Symptom, an Impact dimension, and a Target horizon. The quadrant guesses are stated as guesses ("best guess from outside the team"). The "next quick win" is a low-effort/high-impact item — *not* the most interesting item, which is usually high-effort. The exercise should feel rushed, not thorough; that is the point. Cataloguing rewards breadth.

## 8. Key Takeaways

- Can you place a given technical-debt item in the correct Fowler quadrant and articulate why the response differs across quadrants?
- Can you author a one-screen debt-catalogue entry that captures the minimum viable fields, including the often-skipped Quadrant and Target Horizon?
- Can you apply an impact-effort matrix to a 10+ item catalogue and identify which items belong in each of the four quadrants — including the "not actually debt" quadrant?
- Can you explain the cognitive discipline of "catalogue first, fix later" and resist the fix-it-now reflex during inventory phases?
- Can you distinguish deliberate / pedagogical debt from organic accidental debt in a brownfield codebase and schedule each appropriately?

## Sources

1. [Technical Debt Management: The Road Ahead for Successful Software Delivery — arXiv](https://arxiv.org/html/2403.06484v1) — retrieved 2026-05-26
2. [7 key challenges with brownfield codebases — Utkrusht](https://utkrusht.ai/blog/challenges-with-brownfield-development-codebases) — retrieved 2026-05-26
3. [Supporting Technical Debt Cataloging with TD-Tracker Tool — Wiley Online](https://onlinelibrary.wiley.com/doi/10.1155/2015/898514) — retrieved 2026-05-26
4. [Managing Technical Debt in 2025: Strategies for Legacy Systems — IT Convergence](https://www.itconvergence.com/blog/strategies-for-managing-technical-debt-in-legacy-software-systems/) — retrieved 2026-05-26
5. [Managing Legacy Systems and Technical Debt — Monterail](https://www.monterail.com/blog/legacy-systems-and-technical-debt-guide-to-modernization) — retrieved 2026-05-26

Last verified: 2026-05-26
