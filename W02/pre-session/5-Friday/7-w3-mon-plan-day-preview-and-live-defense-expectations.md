---
week: W02
day: Fri
topic_slug: w3-mon-plan-day-preview-and-live-defense-expectations
topic_title: "W3 Mon Plan Day preview + Live Defense expectations"
parent_overview: W02/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://intent-driven.dev/blog/2026/04/29/spec-driven-development-with-adr/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://thebcms.com/blog/spec-driven-development
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://asana.com/resources/sprint-retrospective
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://juicebox.ai/blog/rubrics-for-interviews
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://dev.to/finalroundai/i-reviewed-final-round-ai-for-technical-interviews-heres-what-actually-matters-in-2026-47gd
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-06-03
---

# W3 Mon Plan Day preview + Live Defense expectations

> [!NOTE]
> **From Mon W2 plan-spec:** the §0 retro structure consumes Tue-Fri ship evidence against Monday's plan. W3 Mon is the **first formal plan-retrospective** in the programme (D-036) — today's PR #47 ship/no-ship is one of the inputs the §0 retro will retro.

## 1. Learning Objectives

- Score a Live Defense answer against the 5-dimension rubric and discriminate 5/5 from 2/5 by behavior, not vibes.
- Apply the §0 plan-retro structure to Friday's evidence — what shipped, where reality diverged, what got dropped, carries forward.
- Defend a W02 scenario-alternative ADR for 10 minutes (2 open + 5-6 probe + 2 close) with `/web-research` citation discipline.
- Preview W3's agentic systems sequence and the W2 MCQ format.

## 2. Introduction

Today is the cohort's first encounter with two artifacts they'll see five more times through W6: the Live Defense rubric and the §0 plan retro. Both are forcing functions. The rubric punishes "I read it somewhere" and rewards specific config flags + cited sources + named cost-asymmetry. The plan retro distinguishes "dropped correctly" from "dropped silently" — silent-drop is the killer signal that the plan was decorative during the week. Both build the muscle memory the Phase 1 gate (W3 Fri) and Phase 2 gate (W6 Fri) will exercise.

## 3. Core Concepts

### 3.1 Live Defense rubric (14:00 today)

10 min per defender: 2 min opener + 5-6 min probes + 2 min closer.

| Dimension | What 5/5 looks like | What 2/5 looks like |
|---|---|---|
| **Tech-stack fluency** | Names specific config flags, default behaviors, perf characteristics (e.g., Atlas `$rankFusion` weights, `numCandidates` defaults) | "I chose X because the blog post recommended it" |
| **Trade-off articulation without strawmen** | Names genuine strength of rejected alternative before explaining why it lost *for this constraint* | Presents only the weak version of the competitor |
| **Domain framing** | Connects choice to OIG / FAR / DFARS / federal-acq constraints | Generic SaaS framing; could be any product |
| **Citation discipline** | Produces `/web-research` source URL + retrieval date when asked | "I read somewhere that..." with no source |
| **Constraint awareness** | Names the conditions under which the choice would be wrong | Treats the choice as universally correct |

Scoring rubric pinned to instructor; defender does not see scores live. Scores returned in W3 Mon 1:1s.

### 3.2 W3 Mon §0 retrospective — what comes prepared

The five-category structure of a plan retro:

| § | Category | Why it matters |
|---|---|---|
| §0.1 | What the plan-spec said | Literal re-read — treat Mon spec as *contract* not aspiration |
| §0.2 | What actually shipped | PRs landed on main Mon → Fri |
| §0.3 | Where reality diverged | Plan over- / under-predicted / named the wrong thing |
| §0.4 | **What got dropped** | **Most-skipped** — items in spec Mon not in shipped work Fri |
| §0.5 | Carries forward | Explicit: still-in-scope, deferred, or cancelled? |

**W2 dimensions to come prepared to retro:** chunking decision (PR #47 is the evidence), vector store choice (Atlas vs Wed's alternatives), HITL #2 threshold landed where Mon predicted, eval harness scope (Mon's last open question — shipped Fri or under-scoped?).

### 3.3 W3 also introduces agentic systems + W2 MCQ logistics

W3 = single-agent Mon-Tue (`POST /agent/intake-triage`), then multi-agent + LangGraph + HITL #5 `interrupt()` Wed-Fri (Evaluator → consensus → SSA). W3 Mon pre-session lives in next week's folder.

**W2 MCQ 16:30** — 10 senior (applied reasoning) + 10 entry (applied recognition), 30 min, **open-book**, honor system on the answers themselves. Format you'll see W3-W6.

> [!TIP]
> **Defender common failure-mode to dodge:** defending a pattern as "industry standard" without a citation. The rubric weights **citation discipline as load-bearing, not decorative** — "I read it somewhere" gets scored down. Bring the `/web-research` URL + retrieval date in your head, or be prepared to look it up live. Same bar Codex Adversarial Review applies to PRs.

## 4. Generic Implementation

```markdown
# Plan retrospective — Plan-Spec [WXX-Mon, dated YYYY-MM-DD]
# Lives at docs/retros/WXX-mon-plan-retro.md in your pair-project repo

## §0 What the plan said
[Quote original plan-spec verbatim — goals, named work items, open questions, success criteria.]

## §1 What shipped
[List with PR links: what landed on main Mon → Fri.]

## §2 Where reality diverged
| Plan item | What actually happened | Why |
| --- | --- | --- |

## §3 What got dropped
| Item | Sub-case (correctly / silently / under-pressure) | Carry-forward decision |
| --- | --- | --- |

## §4 Carries forward into next plan-spec
[Bulleted, with next-spec section each will appear in.]

## §5 Planning heuristic updates
[What did we learn about how to write the next plan? Concrete, prescriptive.]
```

Template is a forcing function — cannot be filled with hand-waving. Empty cells are themselves a signal ("we have no evidence about X — that's a gap"). The §3 sub-case column is where the discipline lives: dropped correctly vs silently vs under-pressure.

## 5. Real-world Patterns

**Healthcare — CDS plan retros.** Adopted spec-first plan-retros for clinical-decision-support changes. Six weeks caught a pattern: the team over-scoped corpus-extraction, under-scoped prompt-engineering. Visible only because the retro tracked predicted vs actual per work item. §5 heuristic: scope extraction at 1.5×, prompt work at 2×.

**Fintech — structured-defense hiring.** Adopted structured-defense interview format after measuring 3× predictive-validity improvement over unstructured. The behavioral-anchor discipline (5/5 + 2/5 named) closed the validity gap. Without anchors, raters cluster at 3.

**Engineering grad programs — rubric evolution.** Added "AI infrastructure literacy" as a dimension — distributed-systems thinking, cost awareness, monitoring, fallback strategies. The old rubric scored graduates highly on theory while industry flagged them as weak on operational AI — the rubric was measuring the wrong thing.

**Karsun-FDE — live-defense recurrence.** The 5-dim rubric appears 6 times (W2 Fri, W3 Fri Phase 1, W4 Fri, W5 Wed AIOps, W6 Wed Capstone, W6 Thu Final). Same dimensions, escalating stakes. The rubric isn't introduced at the gate — it's the gauge all along.

## 6. Best Practices

- **Treat Mon plan-spec as a contract**, not aspiration — §0 retro grades you against it.
- **Distinguish dropped correctly / silently / under-pressure** — silent drop is the failure case.
- **Cite `/web-research` URL + retrieval date** — "industry standard" loses citation + trade-off scores.
- **Name the genuine strength of the rejected alternative** before explaining why it lost.
- **Frame in federal-acq context** — generic SaaS framing reads as "could ship anywhere."
- **Name conditions under which the choice would be wrong** — senior-tier constraint awareness.
- **Single-rater default**, named explicitly; multi-rater calibration when borderline.

> [!WARNING]
> **Anti-pattern: rubric-no-anchors.** Most internet defense-rubric examples use vague adjectives ("strong articulation", "good awareness") without behavioral anchors at every band. Per the `rubric-no-anchors` pattern: raters cluster scores at 3 because they have no observable definition to discriminate between bands. Signal is destroyed. Today's rubric specifies what 5/5 *and* 2/5 look like with concrete behavior; expect probes that test the exact distinction. The senior-Live-Defense rubric (W3+) tightens these anchors further; same shape, more discrimination.

## 7. Hands-on Exercise

Prep for your defender slot at 14:00: (a) pick the W02 scenario-alternative ADR (Wed-PM research slot output); (b) write a 90-second opener naming the choice + specific config / flag / version / cost-asymmetry; (c) prep three `/web-research` URLs with retrieval dates; (d) name the genuine strength of the rejected alternative; (e) name conditions under which your choice would be wrong. EOD: 3 pair retros + 1 cohort retro feeding W3 Mon §0.

> [!NOTE]
> **Self-check** (30s)
>
> 1. The most-skipped category in a plan retro is "what got dropped." Why does it expose planning skill more directly than "what went well/badly"?
> 2. A defender says "I picked Atlas Vector Search because it's industry standard." Which two rubric dimensions does that answer score lowest on, and what would a 4/5 answer look like?

<details>
<summary>Show answers</summary>

1. Because the plan-spec is a *committed artifact* — every item that was in it Monday is reviewable evidence. A "what went well/badly" retro is about process feelings. "What got dropped" forces a direct comparison: spec at Mon vs ship at Fri, item by item, with a *sub-case* attached (dropped correctly / silently / under pressure). The silent-drop sub-case is the killer signal — it means the team didn't consult the plan during the week (spec was decorative) OR wasn't honest about progress. Process retros can't surface that; plan retros must.
2. Lowest on **citation discipline** ("industry standard" has no source) and **trade-off articulation without strawmen** (no alternative named, no genuine strength of competitors acknowledged). A 4/5 answer: "Atlas because the Vector Search and BM25 legs share one query path (`$rankFusion` from the Aug 2025 GA per [/web-research URL, retrieved 2026-05-26]), removing a dual-pipeline failure mode we'd hit with Pinecone + Elastic. Pinecone has stronger pure-vector latency at our scale; we lose that to win operational simplicity. We'd flip if QPS exceeded ~500/s where Pinecone's latency edge dominates the simplicity win."

</details>

## 8. Key Takeaways

- Live Defense rubric: 5 dimensions, anchors at 5/5 and 2/5; citation discipline load-bearing.
- §0 retro: dropped correctly / silently / under-pressure — silent drop is the failure case.
- Template is a forcing function — empty cells signal gaps.
- W3 = single-agent Mon-Tue, multi-agent + LangGraph + HITL #5 Wed-Fri.
- W2 MCQ (10 senior + 10 entry, 30 min, open-book) recurs through W6.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- <https://intent-driven.dev/blog/2026/04/29/spec-driven-development-with-adr/> — retrieved 2026-05-26 — hot-tech-3mo
- <https://thebcms.com/blog/spec-driven-development> — retrieved 2026-05-26
- <https://asana.com/resources/sprint-retrospective> — retrieved 2026-05-26 — foundation-stable
- <https://juicebox.ai/blog/rubrics-for-interviews> — retrieved 2026-05-26 — foundation-stable
- <https://dev.to/finalroundai/i-reviewed-final-round-ai-for-technical-interviews-heres-what-actually-matters-in-2026-47gd> — retrieved 2026-05-26

</details>

<details>
<summary>Deeper dive — dropped sub-cases + bias mitigation + EOD logistics</summary>

**Dropped sub-cases the retro must distinguish:**

| Sub-case | Signal |
|---|---|
| Dropped correctly | Mid-week realisation scope was wrong; planning heuristic updates ("scope research-heavy items at half-day not full") |
| Dropped silently | Item not noticed until retro — **failure case**. Plan was decorative OR team wasn't honest about progress |
| Dropped under pressure | Knew item was at risk, trade-off was deliberate, retro surfaces *what was traded against what* |

**Bias mitigation in oral defense.** Oral assessment is susceptible to charisma effects, halo bias, central-tendency clustering. Mitigations:

- **Behavioral anchors at every band.** 5/5 has a specific definition ("articulates complex ideas with exceptional clarity AND proactively structures answers"). Without anchors, raters cluster at 3.
- **Evidence-based scoring.** Score paired with specific observation: "scored 4 on trade-off because the defender named the genuine strength of Pinecone before explaining why Atlas was the better fit for this constraint."
- **Multi-rater calibration** when possible; disagreements >2 bands trigger discussion. Single-rater is sometimes unavoidable but should be named.
- **Scores not returned live.** Protects defender from in-flight defensiveness; protects rater from charisma-driven score creep.

**Cross-industry validation:** healthcare network adopted spec-first plan-retros for clinical-decision-support changes — 6 weeks of retros caught the pattern that the team consistently over-scoped corpus-extraction and under-scoped prompt-engineering. Fintech adopted structured-defense interview format after measuring 3× predictive-validity improvement over unstructured. Engineering grad programs evolved oral defense rubrics to add "AI infrastructure literacy" — distributed-systems thinking, cost awareness, monitoring, fallback strategies, clear grip on what AI can't guarantee.

**EOD logistics: 3 pair retros + 1 cohort retro** via `templates/weekly-retro.md`. **Don't bury negative signals.** Mon's plan-spec wasn't perfect; Tue/Wed/Thu showed gaps; W3 Mon §0 NEEDS the honest input. If your pair is short on negative signals during the retro, seed prompts:

- "Mon's plan-spec said X. Tue's reality showed Y. Where was the gap?"
- "One HITL touchpoint you'd wire differently this week?"
- "What does PR #47's no-merge tell you about your own future PR shapes?"

Codex Adversarial Review (Ramping per D-034) runs on every Fri PR: harness PR + Thu's `legacy_chain.py` migration PR + cleanup PRs. P1 findings block.

</details>

Last verified: 2026-06-03
