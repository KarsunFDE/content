---
week: W06
day: Mon
topic_slug: scope-discipline-skeletons-only
topic_title: "Scope discipline — the skeletons-only rule"
parent_overview: W06/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://www.projectcubicle.com/scope-creep-and-gold-plating/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.breeze.pm/blog/gold-plating-projects
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://medium.com/rose-digital/the-two-silent-killers-of-projects-scope-creep-and-gold-plating-and-how-to-stop-them-ed49c702098c
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://allfreelancewriting.com/skeleton-outlines/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://document360.com/blog/technical-writing-process/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Scope discipline — the skeletons-only rule

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish *scope creep* (externally driven) from *gold plating* (internally driven) and recognise both in their own behaviour at the end of a project.
- Apply the "skeletons-only" rule when starting a handoff-documentation phase: produce structure-and-stubs first, prose later.
- Use a closed deliverable taxonomy as an active scope-discipline tool, not a passive list.
- Recognise the two most common failure modes at the closeout of an engagement (drafting prose too early; expanding the artefact list) and name the cost of each.

## 2. Introduction

Engineering teams sail straight at the final deadline twice: once at the implementation gate, once at the handoff gate. The implementation-gate failure modes are well known (incomplete features, broken tests, missing infra). The handoff-gate failure modes are quieter and structurally different — the work that ate the deadline was not implementation but documentation, and the team's instinct to "just write a bit more" produced exactly the wrong artefact.

Project-management practice distinguishes two distinct failure modes at the end of any engagement: **scope creep**, where external pressure (a stakeholder ask, a discovered gap) expands what the team commits to deliver; and **gold plating**, where the team itself, without anyone asking, adds polish that no one will read or use. Both eat the deadline. Both produce worse outcomes than declared honest constraint. Gold plating in particular is hard to spot because it feels virtuous — "we just made the runbook better" — until the dry-run was missed.

The countermeasure that most engineering and technical-writing practice converges on is the **skeleton-first approach**: at the start of the documentation phase, every artefact gets a structural outline with one-line stubs under each heading, and *no prose is written*. Prose lands in the second pass, after the full skeleton set is in place and the team can see the whole shape. This reading explains why, and how the discipline pays off across the final week of an engagement.

## 3. Core Concepts

### 3.1 Scope creep vs. gold plating

Project-management literature treats these as two distinct failure modes with the same outward symptom (deadline pressure).

**Scope creep** is externally driven. A stakeholder asks for a feature; the team agrees informally; the work expands without a corresponding adjustment of timeline or resourcing. Projectcubicle's treatment characterises scope creep as starting with "client input, such as new requests or revised goals." The cure is change control — a documented process for accepting or rejecting expansions.

**Gold plating** is internally driven. The team, without anyone asking, adds extras: a new dashboard panel, a polish pass on the README, a refactor that wasn't on the plan. The Rose Digital / Medium treatment calls gold plating "the quieter silent killer" precisely because it happens without discussion or approval. The cure is harder than for scope creep because the team is the source — change control has nothing to enforce against. Self-discipline (and team-level mutual enforcement) is the only countermeasure.

The two failure modes can co-occur and reinforce each other. Stakeholder pressure ("could you also add…") meets team instinct ("while we're in there, let me also clean up…") and the result is a closeout phase that misses the deadline by 40%.

### 3.2 Why the closeout phase is high-risk

Closeout is structurally high-risk for both failure modes for three reasons:

1. **Time pressure is concentrated.** Closeout is the last X days of an engagement, so any expansion compresses against a hard deadline.
2. **The team is fatigued.** Judgement about scope is worse at hour 200 of an engagement than at hour 20.
3. **The deliverables are documentation, not code.** Documentation is easy to add to incrementally ("one more paragraph won't hurt") and lacks the back-pressure that build failures and test breakages provide for code work.

The combination — high time pressure, low judgement, low back-pressure — is why the same teams that ship clean code routinely produce sprawling, late, and partly-unread handoff packages.

### 3.3 The skeletons-only rule

The technical-writing community converges on the **skeleton-first** discipline. Skeleton outlines give an overview of what you'll write before you draft the content itself; they help organise ideas and make the writing process faster (per All Freelance Writing's treatment). Document360's guide treats skeletons as "an integral part of the technical writing process" because they let the writer decide what goes where before any prose is committed.

Applied to handoff documentation, the rule has three operational parts:

1. **First pass: skeletons only.** Every artefact in the deliverable taxonomy gets a Markdown file with section headers and one-line stubs. No paragraphs. No prose. The output is *structure*, deliberately empty.
2. **Second pass: fill the skeleton.** Only after all skeletons are in place does prose authoring start. The team sees the whole shape at once and can route content to the right artefact rather than dumping it into whichever document was open.
3. **Third pass: review and trim.** The closing pass deletes content that does not earn its place. This is the pass most teams skip; without it, the artefacts grow at the expense of being read.

The discipline is unintuitive because the first pass produces output that *looks* incomplete. The team's instinct, especially under deadline pressure, is to start writing real prose immediately. The skeleton discipline trades that early sense of progress for the structural correctness of the whole.

### 3.4 The taxonomy as an active discipline tool

A closed deliverable taxonomy (the previous reading) becomes an *active* scope-discipline tool when used correctly:

- Any candidate content the team thinks of has to be routed to one of the named artefacts.
- If no artefact fits, the candidate is either out of scope or signals a real taxonomy gap (a rare event after the taxonomy is declared).
- The default action for "this doesn't fit anywhere" is *reject*, not *add a slot*.

Without this discipline the team will absorb every candidate content item, and the taxonomy becomes wallpaper instead of a fence.

### 3.5 Naming trade-offs is scope discipline, not failure

A trade-off honestly named in the known-weaknesses inventory is scope discipline. A trade-off silently fixed at the last minute is gold plating — and one of the most expensive forms, because it usually trades the dry-run, the rest, or the next-item-on-the-list for a fix nobody asked for and nobody will review.

The mental shift: "we left this known weakness because we deliberately chose X over Y under constraint Z" is a *finding*, not a *failure*. It belongs in the known-weaknesses inventory with a re-evaluation trigger. The instinct to "just fix it real quick" is exactly the failure mode the discipline guards against.

## 4. Generic Implementation

Below is a generic Day-1-of-closeout checklist for a small engineering team starting handoff documentation. The example is a **regional-bank loan-origination platform handoff** (fintech), entirely outside federal acquisitions.

```
DAY-1 CLOSEOUT CHECKLIST — Loan-Origination Platform Handoff
============================================================

Goal of today: ship empty skeletons for all 7 handoff artefacts.
NOT goal of today: prose. Resist.

Hour 1 (9:00–10:00) — Taxonomy review
  [ ] Open handoff-taxonomy.yml
  [ ] Walk the 7 artefacts as a team; confirm scope of each
  [ ] Confirm the taxonomy is closed (no candidate 8th artefact accepted today)
  [ ] If any artefact's scope is unclear, resolve in the meeting, not later

Hour 2–4 (10:00–13:00) — Skeleton creation
  [ ] One file per artefact, in the agreed path
  [ ] Section headers per artefact (use the template in docs/templates/)
  [ ] Under each header: one-line stub of what content belongs there
  [ ] NO PROSE. If you find yourself writing a paragraph, STOP and convert to a one-line stub.
  [ ] No code snippets yet
  [ ] No diagrams yet

Hour 5 (14:00–15:00) — Peer skim
  [ ] Each engineer skims a teammate's skeleton (15 min each, pairs swap)
  [ ] Reviewer asks: "Is the structure right? Are the stubs the right scope?"
  [ ] Reviewer does NOT ask: "Is the content right?" — there is no content yet
  [ ] Capture structural issues as comments; do not rewrite

Hour 6 (15:00–16:00) — Fix structural issues
  [ ] Resolve every comment from the peer skim
  [ ] Re-verify the taxonomy is closed; reject any 8th-artefact suggestion

Hour 7 (16:00–17:00) — Tomorrow's plan
  [ ] List the 3 highest-priority artefacts for tomorrow's prose pass
  [ ] List the dry-run/rehearsal items that depend on the artefacts being done
  [ ] Stop. Do not start prose tonight.

End-of-day check
  [ ] 7 files exist at the right paths
  [ ] Each file has the agreed section structure
  [ ] Each section has a one-line stub
  [ ] No file is over 2 pages of stubs
  [ ] No prose, no diagrams, no code snippets
```

Annotations:

- The **goal/non-goal pair at the top** is the most load-bearing line of the checklist. Without it, the team will start prose at hour 2.
- The **peer-skim** explicitly restricts reviewers to structural feedback. This is what prevents the skim from becoming a prose-revision pass.
- The **stop signal at hour 7** is non-optional. The discipline is fragile; one engineer who starts prose at 22:00 collapses the whole team's pacing.

## 5. Real-world Patterns

**Book-writing in publishing.** Trade-book authors (technical and non-technical) routinely use a skeleton-first approach: chapter-level outlines, then scene-level beats, then prose. Editors at major publishing houses use the outline to negotiate scope with the author *before* any prose is written, because revisions at the outline level are cheap and revisions at the prose level are expensive. The same economics apply to handoff documentation.

**News editorial in newsrooms.** Investigative-journalism teams operate on a "nut graf + section heads + bullet stubs" outline before any reporter writes prose. The editor reviews the outline; if the structure is wrong, the writer doesn't waste a day on prose that gets cut. Outlets like ProPublica and the New York Times' investigative desk run this discipline tightly. The structural-review-before-prose pattern transplants directly to engineering documentation.

**Healthcare clinical-trial reporting.** Clinical-trial reports (the ICH E3 structured report standard) are produced skeleton-first because the regulatory audience expects an exact section structure. Trial sponsors generate the entire skeleton — every section, every subsection, every table-shell — before any narrative is written. This is the most extreme form of the discipline: the regulatory shape *is* the deliverable, and the prose is filler. The lesson for engineering: when the structure of the deliverable is known in advance, generate it all first.

**Game-studio production schedules.** Larger game studios run "vertical-slice-then-fill" production: ship a tiny, end-to-end playable slice first, then expand. The analogue in documentation is the skeleton — a minimum viable structure of the whole, then fill. Studios that skip the slice find themselves with three deeply-polished levels and seven that don't exist when the deadline hits. Documentation teams that skip the skeleton find themselves with one deeply-prose runbook and six artefacts that don't exist.

## 6. Best Practices

- **Declare today's goal AND non-goal at the start of the closeout day.** "Today we ship empty skeletons; today we do NOT write prose." The non-goal is the load-bearing part.
- **Use the deliverable taxonomy as a fence, not wallpaper.** If a candidate piece of content does not fit a named artefact, the default is *reject*.
- **Treat known weaknesses as findings, not failures.** Name them honestly in the inventory with a re-evaluation trigger; do not silently fix.
- **Make peer review structural before it is prose-level.** Review skeletons against structure; review prose against content. Conflating them produces neither.
- **Stop at the end of the day even if you feel "I could just finish this section."** Pacing is the discipline; one engineer working late breaks the team's pacing.
- **Reject the eighth artefact.** Nine-times-out-of-ten the missing slot is gold plating dressed as a gap.
- **Run a dry-run / rehearsal against the artefacts before the gate.** A dry-run that catches a missing artefact at 36 hours out is a save; a dry-run skipped to "finish polish" is the failure mode.

## 7. Hands-on Exercise

**Whiteboarding prompt (10 min).** You are starting Day 1 of a 4-day closeout phase for an engagement that has 7 handoff artefacts to ship. The team's instinct (and yours) is to start writing the runbook immediately because "the runbook is the one we know best."

Draft a one-page **Day-1 plan** that resists this instinct.

Expected components:

- One sentence naming the day's goal.
- One sentence naming the day's non-goal (this is the load-bearing line).
- An hour-by-hour breakdown for the 7 working hours.
- A peer-review block restricted to structural feedback (with one sentence stating what reviewers MUST NOT do).
- A stop signal at end-of-day stated as a non-negotiable, not a suggestion.
- One sentence on how the plan handles "I just want to write the runbook real quick" pressure from a teammate.

**What good looks like.** The non-goal line should be unambiguous enough that an engineer at hour 4 who has started writing prose can be redirected with a one-sentence reference to the plan. The peer-review block should pre-empt the most common drift (reviewing content where structure was being reviewed). The "real quick" pressure response should be specific: name the artefact the engineer was going to start, point them at the skeleton-first rule, and reassign their hour to a structural task.

## 8. Key Takeaways

After this reading the learner should be able to answer:

- What is the difference between scope creep and gold plating, and which is harder to defend against at the closeout of an engagement? *(maps to LO 1)*
- What does the "skeletons-only" rule mean operationally, and what is the cost of breaking it on Day 1? *(maps to LO 2)*
- How does a closed deliverable taxonomy become an active scope-discipline tool, and what does it do that an open documentation backlog does not? *(maps to LO 3)*
- Why is "honestly naming a trade-off in the known-weaknesses inventory" scope discipline rather than failure? *(maps to LO 1, 4)*
- What two failure modes most commonly eat the deadline at engagement closeout, and what is the named countermeasure for each? *(maps to LO 4)*

## Sources

1. [Scope Creep and Gold Plating in Project Management — Projectcubicle](https://www.projectcubicle.com/scope-creep-and-gold-plating/) — retrieved 2026-05-26
2. [What is gold plating in project management? [2025] — Breeze](https://www.breeze.pm/blog/gold-plating-projects) — retrieved 2026-05-26
3. [The Two Silent Killers of Projects: Scope Creep and Gold-Plating — Frank Vidal on Medium](https://medium.com/rose-digital/the-two-silent-killers-of-projects-scope-creep-and-gold-plating-and-how-to-stop-them-ed49c702098c) — retrieved 2026-05-26
4. [Skeleton Outlines for Writers — All Freelance Writing](https://allfreelancewriting.com/skeleton-outlines/) — retrieved 2026-05-26
5. [A Complete Guide to the Technical Writing Process — Document360](https://document360.com/blog/technical-writing-process/) — retrieved 2026-05-26

Last verified: 2026-05-26
