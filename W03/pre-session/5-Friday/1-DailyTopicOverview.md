---
template: pre-session-reading
week: W03
day: Fri
phase: Agentic (Gate)
topic: "Phase 1 Gate prep + W4 modernization arc preview"
estimated_total_minutes: 31
last_verified: 2026-06-06
fde_situations: [1, 2, 3, 4, 5, 6, 7, 9, 10, 11, 12]
tech: [cumulative — LLM essentials + RAG + agentic]
sources_research_briefs: [research/phase-1-defense-shape.md]
author: instructor
hitl_touchpoint: null
---

# Friday Pre-Session — Phase 1 Defense + W4 Modernization Preview

> [!NOTE]
> **From earlier:** Thu wired the formal `interrupt()` boundary — the hard gate where no default, no proposal, and no auto-proceed is permitted. Today you defend the chain you built across five HITL touchpoints.

## 1. What you'll learn today

By the end of this pre-session reading you'll be able to:

- Articulate the Phase 1 Defense format: demo slot, Agency CIO Q&A, OIG Q&A, per-individual architecture defense.
- Name the five HITL touchpoints you are accountable to defend (#1 W1 Fri, #2 W2 Thu, #3 W3 Mon, #4 W3 Wed, #5 W3 Thu) and what each one protects.
- Describe how Phase-1 discoveries (what went wrong, what surprised you) become Phase-2 modernization scope on Monday morning (D-029, D-036).
- Recognize the most common defense anti-patterns — especially citation without reading and HITL hand-waving — before the panel opens.

## 2. The day at a glance

```mermaid
graph LR
    PS[Pre-session: defense prep + W4 preview]
    WR[War-room: Gate briefing + evidence pack review]
    D[Defense 09:00–12:00 - 3 pairs × 45 min]
    R[Mid-Program Retro 13:00–15:00]
    MCQ[MCQ + freeze tags + W4 briefing 15:00–16:30]
    PS --> WR --> D --> R --> MCQ
```

| Reading | Focus | Min | Why you'll need it |
|---------|-------|----:|--------------------|
| 2. W4 Modernization Arc Preview | What Phase 2 looks like — OpenRewrite, Strangler-Fig, Phase-1 discovery interlock | ~12 min | Defense answer to "what's next?" + W4 Mon §0 framing |
| 3. Defense Anti-Patterns to Avoid | Five patterns that fail the panel + the defense rubric | ~12 min | Walk in with the rubric in hand |

Two readings, 24 minutes total. Intentionally short — defense day.

## 3. Threading

- **HITL programme thread:** all 5 of 7 landed (#1 W1 Fri, #2 W2 Thu, #3 W3 Mon ADR, #4 W3 Wed multi-agent handoff, #5 W3 Thu formal `interrupt()`). Two remain: #6 W4 Wed, #7 W5 Wed.
- **Phase thread:** Phase 1 closes today. Phase 2 (Modernization) starts W4 Mon.
- **Pair-project:** each pair defends their Phase-1 repo; `phase1-freeze-pair-N-20260612` tag lands today (D-056).
- **Decision anchors:** D-029 (Plan Day §0 retro), D-036 (phase-level retro), D-039 (Phase 2 driven by Phase-1 discoveries), D-043 (HITL 7-touchpoint thread), D-044 (Karsun-aspect anchoring), D-056 (freeze tag).

## 4. Why today matters

Phase 1 is three weeks of AI adoption — LLM essentials, RAG architecture, agentic systems — built into a brownfield whose 12 debt items are waiting. The defense is not a test of memory. It is a test of whether you can make the trade-offs you made *legible* to a reviewer who was not sitting next to you when you made them.

The two panels are not symmetric. The CIO cares about cost shape, FedRAMP boundary, vendor migration, and multi-tenant scale. The OIG cares about audit completeness, HITL authority defensibility, and retrieval reproducibility. Neither will accept "we followed standard practices" without a citation.

> [!IMPORTANT]
> **You defend every ADR with a `/web-research` citation.** HITL #3 established this: every architectural decision cites a source. OIG reviewers ask "where does that practice come from?" ADRs citing only internal discussion are a finding, not a pass.

The cohort-level retro (13:00–15:00) is not an afterthought — the Phase-1 discoveries list IS the input to Monday's Plan Day §0. A weak retro produces a weak W4 plan.

> [!TIP]
> **Common failure:** hand-waving over a HITL touchpoint you didn't actually wire. "We implemented HITL" is not a defense. The question will be "show me the `interrupt()` call and the audit row it emits." If that code doesn't exist, the touchpoint doesn't exist.

## 5. How to read this

- Read file 2 (W4 Modernization Arc Preview) first — it frames what Phase 2 looks like and how today's defense feeds Monday.
- Read file 3 (Defense Anti-Patterns) second — carry the rubric table into the defense.
- Both self-checks in Sec 7 of each file: do them. They're your last dry run before the panel.
- Deeper-dives are optional today — defense day is not the time for new material.
- Total expected time: **~24 min at 100 wpm** (plus 7 min this overview = ~31 min).

> [!NOTE]
> **Prep cue.** Open your pair's Phase-1 repo + `phase1-freeze-pair-N-20260612` tag candidate in a second tab — the rubric in file 3 names exactly which artefacts the OIG reviewer will ask to see.

## 6. Two questions to walk in with tomorrow

Friday closes Phase 1. Monday opens Phase 2. Walk in with:

1. What is the one Phase-1 discovery your pair produced that most directly names a W4 modernization scope item — and how do you frame it as a numbered input to the W4 Mon §0 retro?
2. Which of your five HITL touchpoints would fail a "show me the code" challenge from the OIG reviewer, and what is your honest answer when they ask about it?

<details>
<summary>Topic-to-defense map</summary>

- File 2 (W4 preview) → Defense answer to Agency CIO "what's the Phase 2 plan?" + W4 Mon §0 retro framing
- File 3 (anti-patterns + rubric) → Per-individual architecture defense + OIG Q&A prep

</details>

<details>
<summary>Consolidated sources</summary>

- https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide — retrieved 2026-05-26
- https://spring.io/blog/2022/05/24/preparing-for-spring-boot-3-0 — retrieved 2026-05-26
- https://docs.openrewrite.org/running-recipes/popular-recipe-guides/migrate-to-spring-3 — retrieved 2026-05-26
- https://docs.openrewrite.org/recipes/java/migrate/jakarta/javaxmigrationtojakarta — retrieved 2026-05-26
- https://martinfowler.com/articles/patterns-legacy-displacement/ — retrieved 2026-05-26
- https://martinfowler.com/articles/exploring-gen-ai/i-still-care-about-the-code.html — retrieved 2026-05-26
- https://martinfowler.com/articles/exploring-gen-ai/humans-and-agents.html — retrieved 2026-05-26
- https://github.com/joelparkerhenderson/architecture-decision-record — retrieved 2026-05-26
- https://staffeng.com/guides/staff-archetypes/ — retrieved 2026-05-26
- https://lethain.com/getting-in-the-room/ — retrieved 2026-05-26

</details>

Last verified: 2026-06-06
