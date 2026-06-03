---
week: W02
day: Fri
topic_slug: llm-as-judge-for-retrieval-eval
topic_title: "LLM-as-judge for retrieval eval — bias and calibration"
parent_overview: W02/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://futureagi.com/blog/llm-as-judge-best-practices-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://futureagi.com/blog/evaluating-llm-judge-bias-mitigation-2026/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://medium.com/@adnanmasood/rubric-based-evals-llm-as-a-judge-methodologies-and-empirical-validation-in-domain-context-71936b989e80
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://docs.ragas.io/en/latest/concepts/metrics/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.anthropic.com/news/evaluating-ai-systems
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-06-03
---

# LLM-as-judge for retrieval eval — bias and calibration

> [!NOTE]
> **From topic 2:** the harness loads QA rows and grades retrieval deterministically. Generation grading is the slow part — a second LLM scores the first against a rubric. This topic is the rubric + calibration that decides whether you can trust the score.

## 1. Learning Objectives

- Write a per-band rubric with chain-of-thought and JSON-structured output.
- Name the three structural judge problems (self-preference, rubric capture, non-determinism) and the mitigation for each.
- Calibrate a judge to mean disagreement < 0.1 against a 5-row instructor pre-score, and refuse to ship below that bar.
- Score all four RAGAS dimensions with one rubric file per dimension.

## 2. Introduction

A judge LLM scoring a primary LLM is the load-bearing component of any RAG eval harness. The deterministic chunk-recall numbers from topic 2 say nothing about whether the *answer* is faithful, only whether the *retrieval* hit. Generation grading is where the judge earns its keep. Without calibration the judge confidently emits scores that may correlate with nothing an expert would say — same dashboard, same numbers, no underlying signal. The Anthropic framing is blunt: "robust evaluations are extremely difficult to develop and implement." Calibration against an instructor-pre-scored gold set is the only thing preventing the eval from lying to you.

## 3. Core Concepts

### 3.1 The per-band rubric

```python
FAITHFULNESS_RUBRIC = """
You are scoring whether an answer is faithful to its retrieved evidence.

Score bands:
- 1.0 — every factual claim is directly supported by at least one chunk.
- 0.7 — most claims grounded; exactly one claim plausible but unsupported.
- 0.3 — at least one claim contradicts or is absent from the chunks.
- 0.0 — the answer fabricates a citation or directly contradicts the chunks.

Process (chain of thought, then score):
1. List each factual claim in the answer.
2. For each claim, name the chunk that supports it, or note "unsupported".
3. Pick the score band that best fits.
4. One-sentence justification.

Return JSON: {"claims": [...], "supports": [...], "score": <band>, "justification": "..."}
"""
```

Per-band anchors + chain-of-thought are non-negotiable. A vague "rate 0-1" prompt forces the judge to invent its own definition each run; the score drifts and stddev explodes. One rubric file per dimension — don't share a mega-prompt.

### 3.2 The three structural problems

| Problem | Why it bites | Mitigation this week |
|---|---|---|
| **Self-preference bias** | Same-family judges rate their family's outputs ~4-8 points higher on equivalent content | Different family **OR** smaller model in same family (Haiku judging Sonnet — the size delta partially neutralises stylistic match) |
| **Rubric capture** | Judge can only grade what the rubric describes; vague rubric = noisy scores | Per-band criteria + chain-of-thought + one rubric per dimension (don't share a mega-prompt) |
| **Non-determinism** | Same prompt + same input → different scores across runs even at temp=0 | N=3 runs with seeded sampling; report mean + stddev; stddev > 0.05 = rubric too vague |

### 3.3 Calibration — the most-skipped step

Without calibration you cannot tell when the judge has silently broken. The protocol the cohort runs against the instructor-pre-scored 5 rows from Wed evening:

1. Instructor hand-scored 5 rows of the QA set.
2. Run the v1 judge prompt against the same rows.
3. Compute `mean(abs(judge_score - instructor_score))`.
4. **If mean disagreement > 0.1: iterate the prompt. Refuse to ship a judge that disagrees by more than 0.1.**
5. Re-calibrate monthly; if the bar drifts, either the data distribution shifted or the rubric aged out.

### 3.4 The four RAGAS-style dimensions

All four mandatory — per the `ragas-faithfulness-only` blocklist entry, faithfulness alone misses recall + relevance failure modes.

| Dimension | Direction | Catches |
|---|---|---|
| **Faithfulness** | response → chunks | Hallucination (claims not in chunks) |
| **Context recall** | expected chunks → retrieved | Coverage failure (right chunks missing) |
| **Context precision** | retrieved → relevant | Noise (irrelevant chunks pulled) |
| **Answer relevance** | response → query | Faithful but off-topic |

> [!IMPORTANT]
> **One rubric file per dimension.** A mega-prompt that scores all four dimensions in one judge call may save 4× judge cost but the rubric capture problem multiplies — each dimension competes for the judge's attention budget. Per-dimension rubrics + per-dimension calibration is the discipline; cost optimization via judge model selection (Haiku vs Sonnet) saves more than the multi-dim-in-one-prompt trick anyway.

## 4. Generic Implementation

```python
# tests/eval/judge.py — N=3 sampling per dimension
# Lives in acquire-gov at tests/eval/judge.py (one rubric file per dimension).
import json
import statistics
from llm_client import call_judge

def grade_faithfulness(answer, chunks, n_runs=3):
    scores = []
    for _ in range(n_runs):
        result = call_judge(
            system=FAITHFULNESS_RUBRIC,
            user=json.dumps({"answer": answer, "chunks": chunks}),
            temperature=0,  # not strictly deterministic but minimises variance
        )
        parsed = json.loads(result)
        scores.append(parsed["score"])
    return {
        "mean": statistics.mean(scores),
        "stddev": statistics.stdev(scores) if len(scores) > 1 else 0.0,
        "raw_scores": scores,
    }

def calibrate(rubric_name, instructor_scored_rows):
    """Refuse to ship judge until mean disagreement <= 0.1 vs instructor."""
    judge_means = [grade_faithfulness(r["answer"], r["chunks"])["mean"]
                   for r in instructor_scored_rows]
    instructor = [r["instructor_score"] for r in instructor_scored_rows]
    disagreement = statistics.mean(
        abs(j - i) for j, i in zip(judge_means, instructor)
    )
    if disagreement > 0.1:
        raise CalibrationFailure(
            f"{rubric_name} disagreement {disagreement:.3f} > 0.1 — iterate rubric"
        )
    return disagreement
```

Pattern repeats per dimension — one rubric file per dimension, one function per dimension, all three runs averaged. The runner from topic 2 calls these and writes mean + stddev to results. **Cost note:** 3 runs × 4 dimensions × 30 rows = 360 judge calls per eval. Pin Haiku-class for production scoring; reserve frontier models for monthly calibration anchors. Keep judge cost under 10–15% of production LLM cost; above 25% means distill or downsample.

## 5. Real-world Patterns

**E-commerce search team.** Measured ~7-point family-bias drift with single-judge (Sonnet judging Sonnet); three-judge ensemble across three families cut drift to under 2 points. Single-judge-different-family is sufficient for a starter harness; ensemble is what you graduate to. Cost delta 3×; bias delta bigger.

**Education essay-grading.** A vendor calibrates monthly against teacher-graded gold sets. Cohen's kappa dropped 0.71 → 0.58 in March 2026 after a curriculum shift; the eval team detected the drift inside two weeks because the calibration anchor was scheduled. Without it, drift would have shipped silently for a quarter.

**Medical-LLM benchmarking.** Explicitly separates safety from effectiveness; a six-model benchmark produced 57.2% effectiveness vs 54.7% safety — a gap a single-dimension judge would have hidden. The four-dimension separation is the pattern RAGAS enforces.

**Fintech compliance.** A bank calibrated their faithfulness judge against compliance-officer-scored rows monthly. When "regulatory phrasing" tightened mid-quarter, the calibration anchor flagged 0.14 disagreement on the next monthly run; the team brought it back to 0.07 before the next harness run shipped. Without the schedule, drift would have shipped 200 PRs deep.

## 6. Best Practices

- **Per-band anchors + chain-of-thought, not "rate 0-1".** Vague prompts force the judge to invent its definition each run.
- **One rubric file per dimension** — don't share a mega-prompt.
- **N=3 runs per row** with seeded sampling; stddev > 0.05 = rubric too vague.
- **Calibrate against an instructor-pre-scored gold set.** Refuse to ship if mean disagreement > 0.1.
- **Different family OR smaller model in same family** for the judge — neutralises stylistic match.
- **Monthly re-calibration** — drift is real; schedule the check, don't wait for symptoms.
- **Cost discipline:** Haiku-class for production; frontier model for monthly calibration anchor only.

> [!WARNING]
> **Anti-pattern: uncalibrated judge.** Most internet judge tutorials demo a single judge call against a hand-picked example and call it done — no comparison against a human-scored gold set, no disagreement threshold, no drift monitoring. Per `eval-judge-uncalibrated`: an uncalibrated judge confidently emits scores that may correlate with nothing. Anthropic's framing: "robust evaluations are extremely difficult to develop and implement" — calibration is the only thing preventing the eval from lying to you.

## 7. Hands-on Exercise

Write the faithfulness rubric for PR #47 before war-room: (a) start from the per-band skeleton in §3.1; (b) tighten the 0.7 and 0.3 bands using two concrete `acquire-gov` examples (one borderline-grounded, one obviously off-source); (c) run it against the 5 instructor-pre-scored rows from Wed evening; (d) compute mean disagreement; (e) iterate until ≤ 0.1. Bring both the rubric and the disagreement number to war-room block B — the team calibrates the other three dimensions (context-recall / precision / answer-relevance) together.

> [!NOTE]
> **Self-check** (30s)
>
> 1. Why use Haiku to judge Sonnet rather than Sonnet to judge Sonnet, and why is "different family entirely" the stronger move?
> 2. What does mean disagreement > 0.1 against an instructor-scored gold set actually tell you — and what's the fix?

<details>
<summary>Show answers</summary>

1. Same-family judges score their family's outputs ~4-8 points more leniently because they recognise stylistic patterns and reward them. Haiku-judging-Sonnet partially mitigates this (the size delta breaks some of the stylistic match) but doesn't eliminate it. Different family entirely (e.g., GPT or Gemini judging Claude) is the stronger move — no stylistic overlap. Today's harness uses Haiku-judging-Sonnet because it's the cheapest path with one provider; the three-judge cross-family ensemble is what you graduate to.
2. It tells you the rubric is broken — not the model, not the system under test, the **rubric**. The judge is reading the rubric one way; the instructor is reading the situation another. Fix: tighten per-band criteria (most common cause), add concrete examples per band, or tighten the chain-of-thought instructions. Re-run calibration. **Do not ship the judge until disagreement is ≤ 0.1.** If you ship anyway, the harness will confidently emit scores that mean nothing.

</details>

## 8. Key Takeaways

- Per-band rubric + chain-of-thought + JSON output — the structure is what makes the judge testable.
- Three structural problems (self-preference, rubric capture, non-determinism); each has a named mitigation.
- Calibrate against instructor pre-score; mean disagreement > 0.1 = rubric broken; refuse to ship.
- All four RAGAS dimensions, one rubric file per dimension.
- Distilled judge (Haiku-class) on the hot path; frontier model only for monthly calibration anchor.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- <https://futureagi.com/blog/llm-as-judge-best-practices-2026> — retrieved 2026-05-26 — hot-tech-3mo
- <https://futureagi.com/blog/evaluating-llm-judge-bias-mitigation-2026/> — retrieved 2026-05-26
- <https://medium.com/@adnanmasood/rubric-based-evals-llm-as-a-judge-methodologies-and-empirical-validation-in-domain-context-71936b989e80> — retrieved 2026-05-26
- <https://docs.ragas.io/en/latest/concepts/metrics/> — retrieved 2026-05-26
- <https://www.anthropic.com/news/evaluating-ai-systems> — retrieved 2026-05-26
- <https://pmc.ncbi.nlm.nih.gov/articles/PMC12855988/> — retrieved 2026-05-26

Bad-pattern blocklist: `ragas-faithfulness-only`, `eval-judge-uncalibrated`, `eval-tiny-sample-set` (last reviewed 2026-05-12).

</details>

<details>
<summary>Deeper dive — three-judge ensemble + kappa drift + cost calibration</summary>

For higher-stakes launches the 2026 recommendation has tightened toward three-judge ensembles across three families with majority or weighted vote. Single-judge-different-family is sufficient for a starter harness; ensemble is what you graduate to.

**Cross-industry validation:**

- **E-commerce search:** measured ~7-point family-bias drift with single-judge; three-judge ensemble cut measured drift to under 2 points. Cost: 3× judge calls; bias delta: bigger than the cost delta.
- **Education essay-grading:** monthly kappa-based calibration anchor caught drift from 0.71 → 0.58 inside two weeks (curriculum shift). Without scheduled re-calibration the drift would have shipped silently for a quarter.
- **Medical-LLM benchmarking:** explicitly separates safety scores from effectiveness scores; six-model benchmark produced 57.2% effectiveness vs 54.7% safety — gap a single-dimension judge would have hidden.

**Cost calibration:** 3 runs × 4 dimensions × 30 rows = 360 judge calls per eval. With Haiku-class pricing this is sub-$1 per eval run, sustainable on every PR. With Sonnet-class judges this becomes ~$5-10 per eval and PR-cost discipline matters. Frontier-judge ensembles ($20+ per eval run) are reserved for monthly calibration anchors against the master gold set.

**Drift monitoring discipline:** schedule the monthly re-calibration run as a recurring GHA job (separate from PR runs). Output goes to `tests/eval/calibration-history.jsonl` and the team retros the trend at the monthly calibration check — if disagreement creeps upward by 0.02 per month, the rubric is aging out before it breaks the 0.1 bar.

**JSON-mode discipline:** judges should emit structured JSON, not prose. JSON parses deterministically; prose requires a second parsing step that introduces its own failure mode. Anthropic + OpenAI both support structured-output modes; use them.

</details>

Last verified: 2026-06-03
