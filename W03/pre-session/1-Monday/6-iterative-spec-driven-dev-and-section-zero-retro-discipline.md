---
week: W03
day: Mon
topic_slug: iterative-spec-driven-dev-and-section-zero-retro-discipline
topic_title: "Iterative spec-driven dev + the §0 retro discipline"
parent_overview: W03/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://github.com/github/spec-kit
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.atlassian.com/team-playbook/plays/retrospective
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/bliki/ConversationalStories.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://adr.github.io/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://en.wikipedia.org/wiki/Retrospective
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-06-06
---

# Iterative spec-driven dev + the §0 retro discipline

> [!NOTE]
> **From earlier:** Topic 5 established that every ADR must name its eval signal. The §0 retro is where last cycle's signals become this cycle's carry-forwards.

## 1. Learning Objectives

- State what spec-driven development means and how it differs from "writing a doc."
- Explain why the spec retrospects on itself rather than being frozen after the first draft.
- Run a focused 15-minute §0 retro against last cycle's spec and produce three named carry-forwards.
- Distinguish a spec from an ADR and explain how the two interact.

## 2. Introduction

Spec-driven development writes down what you intend to build before you build it, then revisits that record after. The first half (spec before coding) is widely accepted and frequently skipped. The second half — revisit the spec after the cycle and update it with what actually happened — is what turns specs from upfront-design theatre into a living, falsifiable artefact.

GitHub's *Spec-Kit*: the spec is "the lingua franca" — the artefact humans and AI agents both consume to drive implementation ([GitHub spec-kit](https://github.com/github/spec-kit), retrieved 2026-05-26). Fowler's framing: requirements revised by the conversation, not frozen by it ([Martin Fowler](https://martinfowler.com/bliki/ConversationalStories.html), retrieved 2026-05-26).

Today is the **second §0 retro** of the programme (D-036) — already practised, not novel. It runs in the PM plan-spec block before the three required ADRs.

## 3. Core Concepts

### 3.1 What a spec contains — six sections

Drawing on GitHub Spec-Kit ([GitHub spec-kit](https://github.com/github/spec-kit), retrieved 2026-05-26):

1. **Context** — situation, stakeholders, prior state.
2. **Goals** — success in measurable terms.
3. **Non-goals** — what this cycle does not do.
4. **Approach** — architecture sketch, trade-offs, sub-ADRs.
5. **Risks and unknowns** — what could go wrong; what is not yet known.
6. **Acceptance signals** — evidence the cycle is done.

### 3.2 Spec vs ADR — different artefacts, different lifetimes

- **Spec** — describes *what we are building this cycle*. Bounded, time-boxed, superseded by the next cycle's spec.
- **ADR** — records a *single architecturally significant decision*: context, options, choice, consequences ([adr.github.io](https://adr.github.io/), retrieved 2026-05-26). Durable, atomic, accumulated.

A spec *references* ADRs ("we chose supervisor-worker — see ADR-0011"). ADRs accumulate; specs are superseded.

> [!NOTE]
> **Operational implication:** ADRs and specs age differently — do not let them drift into the same document. Today's plan-spec *references* the three ADRs; it does not contain them. Each ADR lives in `docs/adr/` so it can be superseded independently.

### 3.3 What "iterative" means — §0 before §1

The load-bearing rule: **§0 before §1** — do not write the new spec until you have re-read the old one against what actually shipped.

```
§0  read Cycle N-1 spec + what shipped → name carry-forwards
§1  write Cycle N spec on top of carry-forwards
§2  build; ship; capture deltas → feeds next §0
```

This stops fictional-baseline drift: each new spec built on assumptions the previous cycle invalidated but nobody recorded.

### 3.4 The §0 anatomy — three questions, 10–15 minutes

Borrowing from agile retrospective practice ([Atlassian](https://www.atlassian.com/team-playbook/plays/retrospective), retrieved 2026-05-26):

1. **What did the previous spec under-estimate?** (Work thought small that turned out large; constraint thought loose that turned out binding.)
2. **What did the previous spec over-engineer?** (Affordance built but unused; gate added but never tripped.)
3. **Which decision turned out to be load-bearing?** (The assumption everyone is now implicitly carrying.)

Each answer produces a carry-forward — a sentence in §1 (Context) or §5 (Risks). The §0 is calibration, not blame.

> [!IMPORTANT]
> **W3 §0 anchors** (instructor confirms in Mon 1:1): (1) What did the W2 RAG plan underestimate? Most pairs: multi-tenant filtering on the retrieval boundary (Item 10). (2) What did W2 over-engineer? Coaching candidate: hybrid retrieval + reranker on Day 1 when basic kNN sufficed. (3) Which W2 ADR turned out to be load-bearing? Carry it explicitly into W3.

## 4. Generic Implementation

```markdown
## §0 Retrospective on Cycle N (W2 RAG week)

### What did the previous spec under-estimate?
The data-quality risk (§5) materialised harder than expected.
Multi-tenant retrieval boundary had 35% of cases with missing
tenant-scope filters — we patched inline but it is fragile.

### What did the previous spec over-engineer?
The hybrid retrieval + reranker pipeline was more complex than
needed at this scale. Basic kNN handled the load; the reranker
added latency without measurable recall improvement.

### Which decision turned out to be load-bearing?
ADR-0007 (PostgreSQL pgvector over Qdrant for the prototype).
Every subsequent indexing decision assumes it. Do not revisit
this cycle.

## §1 Context (W3 — Agentic week)
Carrying forward from W2: (1) multi-tenant retrieval boundary
gets an explicit filter in every agent tool call wrapping the
RAG layer; (2) reranker removed from W3 agent tool schema;
(3) ADR-0007 is locked — pgvector is the retrieval backend.
```

The shape: §0 names three things, §1 carries them forward, §2+ builds on them. The whole §0 fits on one page.

## 5. Real-world Patterns

**Software PM — Atlassian, GitLab, Linear cycle-boundary retros.** Modern product teams use the three-question shape with the discipline that retro output lands as concrete next-cycle commitments — not bullet points filed away ([Atlassian](https://www.atlassian.com/team-playbook/plays/retrospective), retrieved 2026-05-26).

**Hardware engineering — NASA lessons-learned databases.** Mature hardware programmes open each design review with "lessons carrying forward from the previous build." The cost of forgetting a cycle's failure in hardware is measured in lives. Software borrowed the form; the discipline is the same.

> [!TIP]
> **Cross-domain lesson:** Every mature engineering discipline has a formal carry-forward mechanism. The §0 retro is yours. Three one-sentence carry-forwards in 15 minutes is the minimum viable version — it prevents fictional-baseline drift without turning Plan Day into an all-morning ceremony.

## 6. Best Practices

- Run §0 *before* writing §1 — reversing the order defeats the purpose.
- Time-box §0 to 10–15 minutes: one document against one set of outcomes, not a generic retrospective.
- Capture carry-forwards as one-sentence claims; if you cannot say it in a sentence, you have not understood the delta.
- ADRs are the durable layer; specs are the time-boxed planning layer — do not conflate them.
- Resist the urge to skip §0 on a "small" cycle — small cycles drift fastest.

> [!WARNING]
> **Anti-pattern: `spec-as-afterthought`.** Writing the spec *after* shipping produces a fictional baseline the next cycle plans against — the §0 retro cannot correct a spec that was never honest. An ADR without a `/web-research` citation has the same failure mode: a decision without evidence is vibes in writing. Both must be written before implementation. See slug `adr-without-citation` (Topic 7).

## 7. Hands-on Exercise

Pick a W2 feature your pair shipped. Write the §0 retrospective — three questions, one paragraph each, ending with three one-sentence carry-forwards.

Expected output: (1) a concrete under-estimation with evidence, (2) one over-engineering with the specific affordance that did not pay off, (3) one load-bearing decision and why revisiting it would be disruptive, (4) three "Carry forward" sentences — each falsifiable, not aspirational.

> [!NOTE]
> **Self-check** (30s — answer mentally before expanding)
>
> 1. What is the failure mode if you write the new spec (§1) before running the §0 retro on the previous spec?
> 2. An ADR records a single architectural decision; a spec covers an entire cycle. If they conflict (the spec says "use pgvector" but ADR-0007 says "use Qdrant"), which wins?

<details>
<summary>Show answers</summary>

1. Fictional baselines — you build the new cycle's plan on assumptions the previous cycle invalidated but nobody named. The §0 is specifically the moment that forces the question "what is actually true today, against what the spec says." Skipping it means the new spec inherits the old spec's errors silently.
2. The ADR wins — it is the durable, atomic record of a decision with explicit context and consequences. The spec is a time-boxed planning document. When they conflict, the resolution is to update the spec to reference the ADR correctly, not to override the ADR. If the ADR itself needs to change, write a new superseding ADR first.

</details>

## 8. Key Takeaways

- §0 (retro on last cycle) must precede §1 (write new cycle) — reversing is the failure mode.
- §0 produces three carry-forwards: under-estimation, over-engineering, load-bearing decision. Each is one falsifiable sentence.
- Specs are time-boxed and superseded; ADRs are durable and accumulated. Neither replaces the other.
- Four failure modes §0 corrects: silent drift, frozen assumptions, fictional baselines, goal creep.
- Today's §0 names the W2 carry-forwards that shape the three W3 ADRs.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://github.com/github/spec-kit — retrieved 2026-05-26 — hot-tech
- https://www.atlassian.com/team-playbook/plays/retrospective — retrieved 2026-05-26 — foundation-stable
- https://martinfowler.com/bliki/ConversationalStories.html — retrieved 2026-05-26 — foundation-stable
- https://adr.github.io/ — retrieved 2026-05-26 — foundation-stable
- https://en.wikipedia.org/wiki/Retrospective — retrieved 2026-05-26 — foundation-stable

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

The §0 retro has a known failure mode in senior-led teams: the retro becomes a victory lap on what worked and skips the over-engineering question. Senior FDEs tend to defend architectural choices rather than interrogate them. The corrective: assign the over-engineering question to the *junior* team member to answer first, then the senior responds. The junior is more likely to name what was confusing or unnecessary; the senior is more likely to defend the intent. The tension produces a more honest §0.

The "fictional baselines" failure mode compounds across weeks. By W4, a pair that has been skipping §0 retros will be planning against metrics from W1 that were never measured correctly. The instructor's gate-boundary 1:1 on W3 Mon is specifically designed to surface this: the instructor asks "what does your W2 spec say your eval signal was for the RAG routing step?" and checks whether the pair can answer. A pair that says "we didn't track that" is experiencing fictional-baseline drift in real time.

GitHub Spec-Kit's "executable spec" framing is worth exploring for senior FDEs: a spec that can be parsed by an LLM agent and turned into a task list is qualitatively different from a spec that is just documentation. The programme's `templates/week-plan-spec.md` is designed to be partially executable — the ADR sections drive the HITL gate implementation, and the eval-signal sections drive the LangSmith eval configuration.

</details>

Last verified: 2026-06-06
