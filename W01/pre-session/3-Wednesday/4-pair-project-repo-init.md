---
week: W01
day: Wed
topic_slug: pair-project-repo-init
topic_title: "Pair-Project repo init — what gets scaffolded Wed PM"
parent_overview: W01/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://martinfowler.com/bliki/ArchitectureDecisionRecord.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://adr.github.io/adr-templates/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://aws.amazon.com/blogs/architecture/master-architecture-decision-records-adrs-best-practices-for-effective-decision-making/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://github.com/joelparkerhenderson/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://en.wikipedia.org/wiki/Instant-runoff_voting
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Pair-Project repo init — what gets scaffolded Wed PM

> Wednesday afternoon converts an empty pre-created repository into a living project: README, two ADRs, a week-1 spec skeleton, and the directory shape that will hold eval harnesses, source code, and CI workflows over the next five weeks. This generic deep-dive walks the **repository-initialisation craft** — ADR mechanics, README structure, directory conventions, voting-driven scope commitment. The overview (`1-DailyTopicOverview.md` §4) names the three Cohort #1 repos (`KarsunFDE/grants-portal-modern`, `KarsunFDE/contract-payment-flow`, `KarsunFDE/foia-response-pipeline`) and the federal-acquisitions aspects each anchors. This reading is the technique underneath.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Author an **ADR** following the Michael Nygard structure (Context / Decision / Consequences) and identify when MADR's extended structure adds value.
- Initialise a **greenfield repository** with the canonical entry-point set (README + ADR-0001 + week-1 spec + directory skeleton) without copying boilerplate from another project's history.
- Distinguish a **commitment ADR** (locks scope, hard to reverse) from a **comparison ADR** (records a choice with a defeated alternative explicit) and choose which shape fits a given decision.
- Apply **instant-runoff voting (IRV)** as a small-group decision mechanic when multiple parties have ranked preferences and a single resource must be allocated.
- Explain why **continuous git history across project phases** matters more than a clean restart — and what a "fresh start" implicitly destroys.

## 2. Introduction

The first hour of a new project is disproportionately load-bearing. The directory shape you choose, the first ADR you write, and the README you commit before any code lands all become the substrate that every subsequent change accretes onto. Teams that skip the first hour ("we'll add the README later") almost always end up with a repository whose structure has been determined by accident — by the first feature anyone landed, by the deploy pipeline a CI engineer wrote on a Friday afternoon, by the test framework that happened to be familiar.

The discipline of deliberate repo initialisation pre-dates AI engineering by decades [4]. Architecture Decision Records (ADRs), first articulated by Michael Nygard in 2011 [1], are now standard practice at AWS, Microsoft, and most well-run engineering organisations [2][3]. The MADR (Markdown ADR) template adds tradeoff analysis and decision-maker metadata [2]. The dominant convention is to keep ADRs in `docs/adr/` or `docs/adrs/` in the same repository as the code they govern, numbered sequentially (`0001-...md`, `0002-...md`) so that a directory listing tells the project's architectural story in order [4].

The forward-deployed engineer's version of repo initialisation has a particular pressure: the work the repo will hold is *committed before it is fully understood*. You commit to a scope, an architectural direction, a set of tradeoffs — and then you spend the next several weeks discovering what those commitments actually entail. ADR-0001 is rarely the most accurate ADR the project will ever have; it is the most consequential, because everything else accretes around it. The art is committing decisively while leaving honest space for revision.

This reading walks the mechanics in the generic — outside any federal-acquisitions context. The afternoon block applies it to your pair's specific aspect.

## 3. Core Concepts

### 3.1 The Nygard ADR template

Michael Nygard's 2011 form [1] is canonical and short:

```
# Title — short verb-noun phrase (e.g., "Use PostgreSQL for transactional store")

## Status
Proposed | Accepted | Deprecated | Superseded by ADR-NNNN

## Context
What is the situation that motivates this decision? Forces in play, constraints
discovered, options considered. 2-4 paragraphs, prose.

## Decision
What did we decide, in one or two clear sentences. Active voice. The decision
is the load-bearing line — if a future reader reads only one line of this ADR,
this is it.

## Consequences
What becomes easier, what becomes harder, what remains open. Honest enumeration —
including negative consequences. 2-4 bullets minimum.
```

Three properties make this template durable: (a) it is short enough that engineers actually write it, (b) the consequences section forces honesty about tradeoffs, (c) the Status field lets ADRs *retire* gracefully without being deleted (an `ADR-0023` that supersedes `ADR-0007` preserves the historical reasoning).

### 3.2 MADR's extensions and when they earn their place

MADR (Markdown ADR) adds [2]:

- **Decision-makers** — who actually signed off, by role.
- **Considered options** — the explicit list of alternatives, with one-line summaries.
- **Pros and cons of each option** — the tradeoff analysis, not just the chosen path.
- **Decision outcome** — the choice, with reasoning that references the pros/cons table.

MADR earns its place when the decision has multiple defeated alternatives that would each have been defensible. A "Use PostgreSQL" decision in a project where only PostgreSQL was ever seriously considered does not benefit from MADR's full structure — Nygard is enough. A "Use AWS Bedrock vs OpenAI direct vs self-hosted Llama" decision where all three were live options benefits enormously from MADR, because the defeated alternatives' tradeoffs will be re-litigated within months.

### 3.3 Commitment ADRs vs comparison ADRs

A useful distinction in practice:

- **Commitment ADRs** lock scope. "This project will own the X workflow end-to-end." There is no comparison being made — the decision is to *claim a scope*. The Context section explains why this scope matters; the Decision section names it crisply; the Consequences section is honest about what the project will *not* address.
- **Comparison ADRs** choose among alternatives. "Use Postgres vs MongoDB vs DynamoDB for the primary store." The Context names the options; the Decision names the choice; the Consequences section includes the dimensions on which the defeated alternatives would have been better.

Both shapes are valid. The error is using a comparison form for a commitment decision (it makes the decision look more contestable than it is) or using a commitment form for a comparison decision (it hides the alternatives, making the choice look obvious when it was not).

A new project typically commits its first ADR (ADR-0001 — what is this project, what scope does it own) and its second ADR (ADR-0002 — the first significant comparison the project had to make). The two together set the substrate.

### 3.4 The minimal viable directory skeleton

A greenfield project benefits from committing the *empty* shape of its directory tree before any feature code lands. The pattern that works across most stacks:

```
{repo-name}/
├── README.md                ← human entrypoint; what + why in one screen
├── LICENSE                  ← legal posture
├── .gitignore               ← language-appropriate
├── docs/
│   ├── adrs/                ← all ADRs, numbered sequentially
│   │   ├── 0001-...md       ← commitment ADR
│   │   └── 0002-...md       ← first significant comparison
│   └── decisions.md         ← optional human-readable index of ADRs (or use a tool)
├── specs/                   ← per-iteration plans + spec docs
│   └── week-1.md            ← (or sprint-1, milestone-1 — match the team's cadence)
├── src/                     ← all source code (language-organised below this)
├── tests/                   ← all tests (mirror the src/ shape)
├── eval/                    ← (for AI projects) eval harness lives here from day one
└── .github/
    └── workflows/           ← CI lives here
```

Committing the empty skeleton with placeholder `.gitkeep` files or one-line stub READMEs is intentional. It signals to every contributor (and to every code-generation tool) where new artifacts go. A repository that grows organically from no shape ends up shaped by accident; one that starts with a deliberate empty shape converges faster.

### 3.5 Instant-runoff voting as a small-group allocation mechanic

When multiple parties want the same scarce resource and ranking is meaningful, **instant-runoff voting (IRV)** [5] is a defensible default. The mechanic:

1. Each voter ranks the options 1st, 2nd, 3rd, …
2. Count first-place votes. If any option has a majority, it wins.
3. If no majority, eliminate the option with the fewest first-place votes; redistribute those voters' next-ranked choice.
4. Repeat until a majority emerges.

IRV is well-suited to small-group scope commitment (e.g., three pairs choosing from three options) because it (a) lets every voter express full preferences rather than a single binary, (b) reduces strategic-voting incentives compared to first-past-the-post, (c) terminates predictably in a small number of rounds. Tie-handling in IRV uses second-place counts as the standard tiebreaker [5]; a final tie is resolved by an explicit tiebreaker function agreed beforehand (coin flip, facilitator decides, draw lots — the discipline is committing to the tiebreaker *before* the vote, not after).

The alternative — first-past-the-post — works for two-option choices but produces poor outcomes when three or more options exist, because it ignores the second-best preference structure.

### 3.6 Continuous git history vs the "fresh start" temptation

A pattern worth resisting: when a project transitions from one phase to the next (prototype → production, MVP → v2, exploration → commitment), the temptation is to start a new repository "to leave the experiment behind". This destroys two things: (a) the commit-by-commit history that explains *why* the project arrived at its current shape, (b) the blame trail that lets future contributors trace any line of code back to the context in which it was written.

The continuous-history alternative is to keep the same repository across phases, mark phase transitions with **tags** (`v0-prototype`, `v1-production`), and use **branches** for risky experiments that may not survive. The git history then becomes the architectural narrative of the project — readable in `git log --oneline` order, queryable with `git blame`, and traceable through `git log --follow` even across file renames.

## 4. Generic Implementation

A generic example: a small team is initialising a new project to build an AI-powered customer-support routing system. The team has just voted (via IRV) to claim the *email-triage* workflow over the alternatives of *chat-routing* and *voice-transcript-summarisation*. The first 60 minutes of repo work look like this.

**Step 1 — Commit the empty skeleton (10 minutes):**

```bash
$ git init
$ mkdir -p docs/adrs specs src tests eval .github/workflows
$ cat > README.md <<'EOF'
# email-triage

AI-powered email triage for high-volume customer-support inboxes. This project
owns the email-triage workflow end-to-end: ingestion, classification, routing,
and the human-review surface for low-confidence cases.

## Status

Week-1 scaffolding. ADR-0001 commits the project's scope. ADR-0002 records the
LLM-provider comparison. The first functional code lands in week 2.

## Layout

- docs/adrs/ — architecture decisions, numbered
- specs/ — per-week plans
- src/ — application code
- tests/ — test suites
- eval/ — eval harness (lands week 3)
- .github/workflows/ — CI

## How to run

(arrives in week 2 with the first PR)
EOF
$ touch specs/week-1.md src/.gitkeep tests/.gitkeep eval/.gitkeep
$ git add -A
$ git commit -m "Initial scaffold: directory skeleton + README"
```

**Step 2 — Author ADR-0001 (commitment, 15 minutes):**

```markdown
# ADR-0001: Scope — Email Triage

## Status
Accepted (2026-05-XX)

## Context
The team chose between three customer-support workflows in scope: email triage,
live-chat routing, and voice-transcript summarisation. Email triage was selected
via instant-runoff vote: two first-place votes, one second-place that
redistributed in the second round.

The email-triage workflow has the highest ratio of (volume) × (mis-routing cost)
of the three. Existing tooling at the host organisation handles classification
poorly — 38% of tickets are routed to the wrong queue on the first pass per the
2025 internal audit.

## Decision
This project owns the email-triage workflow end-to-end: inbound parsing,
classification, queue routing, and the human-review surface for cases where the
classifier confidence falls below threshold.

## Consequences
- Live-chat routing and voice-transcript summarisation are explicitly OUT of scope.
  A future project may extend the classifier substrate, but this team commits to
  email-triage as the surface for the rest of the engagement.
- The classifier must support a tunable confidence threshold from week 1; the
  human-review surface is not optional.
- Eval data will be drawn from the past 12 months of routed-and-corrected emails,
  not from a synthetic corpus — recorded in ADR-0003 once data access is confirmed.
```

**Step 3 — Author ADR-0002 (comparison, 15 minutes):**

ADR-0002 records the first significant comparison the project has to make — for an AI project, often the LLM-provider choice. The team uses the MADR shape because the alternatives are all live (Anthropic via cloud, OpenAI direct, self-hosted Llama). The Decision section names the choice; the Consequences section is honest about the dimensions on which the defeated alternatives would have been better (cost, latency, fine-tuning flexibility, etc.).

**Step 4 — Author the week-1 spec skeleton (15 minutes):**

The `specs/week-1.md` file is the team's plan for the coming week. It is *not* a feature spec — it is a plan: what gets built, what gets researched, what gets deferred. The skeleton names the deliverables, the unknowns, and the gates for "week 1 complete". Substance arrives as the week progresses.

**Step 5 — First push (5 minutes):**

```bash
$ git add -A
$ git commit -m "ADR-0001 scope commitment; ADR-0002 LLM provider; week-1 spec"
$ git push -u origin main
```

The push is small (five files), but the substrate is in place. Every subsequent commit accretes onto a deliberate shape.

## 5. Real-world Patterns

**Healthcare AI — Hugging Face's medical-imaging starters.** Hugging Face's recommended layout for medical-imaging projects ships with an explicit `docs/adrs/` directory and an ADR-0001 template that the team is expected to fill in before any model-training code lands. The discipline came from regulatory pressure: medical-imaging code must be traceable to the architectural decisions that produced it [3][4].

**Fintech — Wise's open-source repo conventions.** Wise (formerly TransferWise) has published its internal repo conventions, including a mandatory ADR for any service that touches the payment graph. The first ADR in any such repo commits the scope, the second records the consistency model (eventual vs strong vs hybrid), and the third names the audit posture. The pattern produces remarkably consistent repos across hundreds of services [1][3].

**Gaming — open-source game engine ADRs.** The Godot game engine project maintains a public ADR archive that includes the original 2018 decision to migrate from Python-based scripting to GDScript. The ADR is still readable, still cited in discussions years later, and still has its Consequences section visibly accurate — the proof of a well-written ADR is whether the consequences turn out to be true [1].

**Logistics — Convoy Inc.'s commitment-vs-comparison pattern.** Convoy, a freight-tech company, has talked publicly about the distinction between commitment ADRs and comparison ADRs in their engineering blog. They report that commitment ADRs that try to compare alternatives (when there are none) introduce false contestability and slow the team down; comparison ADRs that hide the alternatives produce confusion in retrospect [2].

## 6. Best Practices

- **Commit ADR-0001 before any feature code lands** — it is the project's first promise to its future self.
- **Use Nygard's short template for commitment ADRs and MADR's extended template for comparison ADRs** — match form to function.
- **Keep ADRs in `docs/adrs/` in the same repo as the code they govern** — distributed knowledge that drifts from the code is worse than no knowledge at all.
- **Commit the empty directory skeleton in the first commit** — `.gitkeep` files are cheap and they signal where new artifacts go.
- **Choose the IRV tiebreaker function before the vote, not after** — disputes about how to resolve a tie are intractable if the rule wasn't agreed in advance.
- **Tag phase transitions; don't restart repositories** — `git tag v1-end` is the right way to mark the end of a phase, not `git init` in a new directory.
- **Write the Consequences section honestly, including negatives** — an ADR with no downsides listed is an ADR no one will revisit, and the downsides are the most useful part to revisit.

## 7. Hands-on Exercise

**(Repository init, 15 minutes.)** Pick a small project idea you have not built — a personal todo manager, a recipe scaler, a workout tracker, anything generic — and initialise a repository following the steps above. Specifically:

1. Create the directory skeleton with `.gitkeep` files.
2. Author a one-screen README naming the project's scope and the layout.
3. Write ADR-0001 (commitment) using Nygard's template — Context, Decision, Consequences.
4. Write ADR-0002 (comparison) using MADR's extended template — at least three considered options, decision outcome with reasoning.
5. Commit the whole thing in two commits ("initial scaffold" and "ADRs 0001–0002").

**What good looks like.** The README is one screen, names the project's scope in a sentence, and lists the directory layout. ADR-0001's Decision section is one or two crisp sentences; its Consequences section enumerates at least two honest downsides. ADR-0002 names three options with one-line summaries and pros/cons; the Decision Outcome cites the pros/cons table directly. The git log reads cleanly: two commits, descriptive messages, no `WIP` or `fix typo` clutter at this stage.

## 8. Key Takeaways

- Can you describe the Nygard ADR template and explain why the Consequences section is the load-bearing part?
- Can you distinguish a commitment ADR from a comparison ADR and choose the right form for a given decision?
- Can you initialise a greenfield repository with a deliberate directory skeleton, a one-screen README, and the first two ADRs in under an hour?
- Can you run an IRV process for a small-group resource-allocation decision and articulate why you chose the tiebreaker function you did?
- Can you explain what is lost when a project "starts fresh" with a new repo at a phase transition, and what the continuous-history alternative looks like?

## Sources

1. [bliki: Architecture Decision Record — Martin Fowler](https://martinfowler.com/bliki/ArchitectureDecisionRecord.html) — retrieved 2026-05-26
2. [ADR Templates — adr.github.io](https://adr.github.io/adr-templates/) — retrieved 2026-05-26
3. [Master architecture decision records (ADRs) — AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/master-architecture-decision-records-adrs-best-practices-for-effective-decision-making/) — retrieved 2026-05-26
4. [architecture-decision-record — joelparkerhenderson on GitHub](https://github.com/joelparkerhenderson/architecture-decision-record) — retrieved 2026-05-26
5. [Instant-runoff voting — Wikipedia](https://en.wikipedia.org/wiki/Instant-runoff_voting) — retrieved 2026-05-26

Last verified: 2026-05-26
