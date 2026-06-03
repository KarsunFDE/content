---
week: W02
day: Fri
phase: Foundation
title: "Pre-session — RAG eval harness + first Live Defense + W3 Mon preview"
audience: All cohort members
estimated_total_minutes: 76
last_verified: 2026-06-03
fde_situations: [1, 3, 6, 7, 9, 11, 12]
tech: [ragas, llm-as-judge, eval-harness, regression-testing, github-actions, langchain-v1, multi-agent-preview, prompt-injection-probes]
sources_research_briefs:
  - research/langchain-v1-20260522.md
  - research/bedrock-claude-catalog-20260522.md
author: instructor
---

# W2 Fri Pre-Session — Eval harness + first Live Defense + W3 Mon preview

> [!NOTE]
> **From Thu (D4):** HITL #2 wired with the **0.85 conjunction gate** (faithfulness ≥ 0.85 AND relevance ≥ 0.85) and a 4hr CO queue SLA. Today's harness is what *measures* whether a PR still holds that gate — Thu wired the runtime, Fri wires the regression test.

## 1. What you'll learn today

By the end of war-room you'll be able to:

- Build a flat-file `qa.jsonl` harness covering all four RAGAS dimensions against a 20-30 row curated set.
- Calibrate an LLM-as-judge to mean disagreement < 0.1 against a 5-row instructor pre-score, and refuse to ship below that bar.
- Wire eval-as-PR-comment with a 5% blocking threshold on faithfulness + context-recall via GHA.
- Diagnose cross-dimension drift and walk back fixes that move one dimension at the cost of another.
- Defend a W02 ADR in 10 minutes against the live-defense rubric — citation discipline as load-bearing.

## 2. The day at a glance

```mermaid
graph LR
    T2[Topic 2: Harness] --> T3[Topic 3: Judge calibration]
    T3 --> T4[Topic 4: PR gate]
    T4 --> T5[Topic 5: Improvement loops]
    T5 --> T6[Topic 6: Security probes]
    T6 --> T7[Topic 7: W3 Mon preview]
    T7 --> WR[War-room: build + ship]
```

| Topic | Focus | Reading min | Why you'll need it |
|-------|-------|------------:|--------------------|
| 2. Eval harness from scratch | Flat-file `qa.jsonl` + runner | ~11 min | The harness IS the AM build |
| 3. LLM-as-judge | Per-band rubric + N=3 + 0.1 calibration | ~11 min | Judge without calibration = the harness lies |
| 4. Eval-as-PR-fixture | GHA + 5% block gate + branch protection | ~11 min | The wiring that makes PR #47 a no-merge |
| 5. Improvement loops | One variable at a time + metric→layer | ~11 min | Monday's candidate fix without three rounds of guessing |
| 6. Security probes | LLM01 probe row + conservative voting | ~11 min | Starter set for the security harness; full W4 Wed |
| 7. W3 Mon + Live Defense | §0 retro + 5-dim rubric + agentic preview | ~11 min | Today's PM at 14:00 |

## 3. Threading

- **HITL programme thread:** Thu's #2 wired runtime → today *measures* it; W3 Mon #3 ADR consumes today's PR #47 ship/no-ship as evidence.
- **Phase thread:** Phase 1 (AI Adoption) — week-close anchor. Eval discipline carries into W3 agentic systems.
- **Pair-project:** the harness ports verbatim into pair-project repos; each pair's QA set covers their anchor (grants / FOIA / post-award) by Friday EOD.
- **Decision anchors:** D-031 (LangSmith deferred to W5), D-033 (LangChain v1.0 posture), D-034 (Codex Adversarial Ramping), D-036 (W3 Mon §0 retro), D-040 (Wed-PM research slot), D-043 + D-044 (HITL thread).

## 4. Why today matters

PR #47 is real. It claims a 30% latency win via a chunking tweak — but yesterday's HITL #2 gate only catches issues at runtime. Without a regression harness, the team is one PR away from quietly losing the faithfulness floor they spent Thursday wiring. Today builds the discipline that turns "we have a runtime gate" into "we have a runtime gate AND we know whether each PR holds it." Evidence not vibes.

The day stacks three things — harness build, first Live Defense, first two-tier MCQ. Live Defense is the cohort's first encounter with the rubric they'll see five more times through W6. Citation discipline is rated explicitly — defending a pattern as "industry standard" without `/web-research` source is exactly the failure mode the rubric is built to catch. The MCQ format (10 senior applied-reasoning + 10 entry applied-recognition, 30 min, open-book) is the standard format for W3–W6.

> [!IMPORTANT]
> **War-room scene:** 9:00 AM PR #47 sits in review claiming +30% latency on `/answer-qa`. The chunking tweak split FAR/DFARS sections at 512-token boundaries — fast, but DFARS 215.371-4 timing exceptions now span two chunks. Without a harness, the team's instinct says "merge it, retro the win." Today's eval harness settles it with evidence by 12:00. PM 14:00 = first Live Defense (one defender per pair, 10 min, W02 scenario-alternative ADR). PM 16:30 = first two-tier W2 MCQ.

## 5. How to read this

- Read topics 2-7 in order. Topic 2 builds the harness, topic 3 makes its judge trustworthy, topic 4 wires it to PRs, topic 5 turns red signals into fixes, topic 6 adds the security layer, topic 7 previews W3 Mon §0 and the Live Defense rubric.
- Self-checks at the end of each topic — 30s mental answer before expanding.
- Deeper-dives optional but recommended for senior FDEs (three-judge ensembles, kappa drift, parent-child indexing).
- Total expected time: **~76 min at 100 wpm**.

> [!CAUTION]
> **Cross-topic anti-pattern:** Internet "RAG eval in 90 min" tutorials commonly demo **5-row QA sets** + **faithfulness-only scoring** + **uncalibrated judges** + **eval-as-dashboard**. All four are anti-patterns today's curriculum contradicts directly. Per `known-bad-patterns.yml` slugs `eval-tiny-sample-set`, `ragas-faithfulness-only`, `eval-judge-uncalibrated`, `eval-dashboard-not-pr`.

## 6. Two questions to walk in with tomorrow

1. Why is "eval as a nightly dashboard" the wrong pattern, and what specifically does eval-as-PR-fixture fix that the dashboard doesn't?
2. A defender says "I picked Atlas Vector Search because it's industry standard." Which two rubric dimensions does that answer score lowest on, and what does a 4/5 answer look like?

<details>
<summary>Topic-to-war-room map</summary>

- Topic 2 → War-room AM block (~40 min): build `qa.jsonl` + runner against PR #47.
- Topic 3 → War-room AM block (~25 min): calibrate judge prompt against 5 instructor-pre-scored rows.
- Topic 4 → War-room AM block (~25 min): wire GHA workflow + branch protection; status check goes red on PR #47.
- Topic 5 → War-room AM block (~15 min): per-row diff on PR #47; name the regressed layer; commit candidate fix for Monday.
- Topic 6 → War-room AM block (~15 min): seed 3-5 probes into `qa-security.jsonl`.
- Topic 7 → PM 14:00 (Live Defense 60 min) + PM 16:30 (MCQ 30 min) + EOD retros.

</details>

<details>
<summary>Tonight's reading map + thread context</summary>

- Pre-read budget Fri: ~76 min at 100 wpm across 6 topic files + this overview.
- HITL thread continues: W2 Thu (#2 wired today gets *measured*) → W3 Mon (#3, Plan-Day ADR — §0 retro consumes this week's evidence) → W3 Wed (#4) → W3 Thu (#5, LangGraph `interrupt()`) → W4 Wed (#6, LLM06) → W5 Wed (#7).
- LangSmith deferred to **W5 per D-031** — Fri's harness is flat-file + GHA on purpose. The discipline (write QA set first, then the system) is the muscle memory W3 inherits and W5 productionises.
- Wed-PM dedicated research slot (D-040) feeds the W02 scenario-alternative being defended at 14:00.
- W3 Mon §0 retro (D-036) is the **first formal plan-retrospective in the programme** — your honest retro inputs Fri EOD become its data.

</details>

<details>
<summary>Consolidated sources (all retrieved via /web-research per D-046)</summary>

- RAGAS metrics overview: <https://docs.ragas.io/en/latest/concepts/metrics/> — 2026-05-26
- Anthropic — Evaluating AI systems: <https://www.anthropic.com/news/evaluating-ai-systems> — 2026-05-26
- GitHub Actions — Reusable workflows: <https://docs.github.com/en/actions/sharing-automations/reusing-workflows> — 2026-05-26
- Bedrock Claude catalog: <https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html> — 2026-05-26
- LangSmith eval (DO NOT IMPLEMENT W2 — D-031): <https://docs.smith.langchain.com/evaluation> — 2026-05-26
- LangChain v1.0 posture (D-033): <https://docs.langchain.com/oss/python/releases/langchain-v1> — 2026-05-22

Research briefs: `research/langchain-v1-20260522.md`, `research/bedrock-claude-catalog-20260522.md`.

</details>

Last verified: 2026-06-03
