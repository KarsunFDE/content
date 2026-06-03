---
week: W02
day: Fri
topic_slug: llm-as-judge-for-retrieval-eval
topic_title: "LLM-as-judge for retrieval eval — bias and calibration"
parent_overview: W02/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 6
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

## The judge prompt skeleton

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

Per-band anchors + chain-of-thought are non-negotiable. A vague "rate 0-1" prompt forces the judge to invent its own definition each run; the score drifts and stddev explodes.

## The three structural problems

| Problem | Why it bites | Mitigation this week |
|---|---|---|
| **Self-preference bias** | Same-family judges rate their family's outputs ~4-8 points higher on equivalent content | Different family **OR** smaller model in same family (Haiku judging Sonnet — the size delta partially neutralises stylistic match) |
| **Rubric capture** | Judge can only grade what the rubric describes; vague rubric = noisy scores | Per-band criteria + chain-of-thought + one rubric per dimension (don't share a mega-prompt) |
| **Non-determinism** | Same prompt + same input → different scores across runs even at temp=0 | N=3 runs with seeded sampling; report mean + stddev; stddev > 0.05 = rubric too vague |

## Calibration — the most-skipped step

Without calibration you cannot tell when the judge has silently broken. Dashboard looks fine, scores correlate with nothing an expert would say.

The protocol the cohort runs against the instructor-pre-scored 5 rows from Wed evening:

1. Instructor hand-scored 5 rows of the QA set.
2. Run the v1 judge prompt against the same rows.
3. Compute `mean(abs(judge_score - instructor_score))`.
4. **If mean disagreement > 0.1: iterate the prompt. Refuse to ship a judge that disagrees by more than 0.1.**
5. Re-calibrate monthly; if the bar drifts, either the data distribution shifted or the rubric aged out.

> [!WARNING]
> **Anti-pattern: ship the judge prompt uncalibrated.** Most internet judge tutorials demo a single judge call against a hand-picked example and call it done — no comparison against a human-scored gold set, no disagreement threshold, no drift monitoring. Per the `eval-judge-uncalibrated` pattern. An uncalibrated judge confidently emits scores that may correlate with nothing. The Anthropic framing is blunt: "robust evaluations are extremely difficult to develop and implement" — calibration is the only thing preventing the eval from lying to you.

## The four RAGAS-style dimensions

All four mandatory — per the `ragas-faithfulness-only` blocklist entry, faithfulness alone misses recall + relevance failure modes.

| Dimension | Direction | Catches |
|---|---|---|
| **Faithfulness** | response → chunks | Hallucination (claims not in chunks) |
| **Context recall** | expected chunks → retrieved | Coverage failure (right chunks missing) |
| **Context precision** | retrieved → relevant | Noise (irrelevant chunks pulled) |
| **Answer relevance** | response → query | Faithful but off-topic |

## Self-check

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

<details>
<summary>Generic Python — judge call with N=3 sampling</summary>

```python
# tests/eval/judge.py
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
```

Pattern repeats per dimension — one rubric file per dimension, one function per dimension, all three runs averaged. The runner from topic 2 calls these and writes mean + stddev to results.

**Cost note:** 3 runs × 4 dimensions × 30 rows = 360 judge calls per eval. Pin Haiku-class for production scoring; reserve frontier models for monthly calibration anchors. Keep judge cost under 10-15% of production LLM cost; if it creeps above 25%, distill or downsample.

</details>

<details>
<summary>Cross-industry — three-judge ensemble + monthly kappa drift</summary>

For higher-stakes launches the 2026 recommendation has tightened toward three-judge ensembles across three families with majority or weighted vote. Single-judge-different-family is sufficient for a starter harness; ensemble is what you graduate to.

E-commerce search team measured ~7-point family-bias drift with single-judge; the three-judge ensemble cut measured drift to under 2 points. Education essay-grading vendor calibrates monthly against teacher-graded gold sets — when kappa dropped from 0.71 to 0.58 in March 2026 (after a curriculum shift), the eval team detected the drift inside two weeks. Medical-LLM benchmarking explicitly separates safety scores from effectiveness scores; a six-model benchmark produced 57.2% effectiveness vs 54.7% safety — gap a single-dimension judge would have hidden.

</details>

<details>
<summary>Sources (retrieved via /web-research per D-046)</summary>

1. FutureAGI — LLM-as-Judge Best Practices 2026: <https://futureagi.com/blog/llm-as-judge-best-practices-2026> — 2026-05-26
2. FutureAGI — LLM-Judge Bias Mitigation 2026: <https://futureagi.com/blog/evaluating-llm-judge-bias-mitigation-2026/> — 2026-05-26
3. Rubric-Based Evals (Adnan Masood, Apr 2026): <https://medium.com/@adnanmasood/rubric-based-evals-llm-as-a-judge-methodologies-and-empirical-validation-in-domain-context-71936b989e80> — 2026-05-26
4. RAGAS metrics: <https://docs.ragas.io/en/latest/concepts/metrics/> — 2026-05-26
5. Anthropic — Evaluating AI systems: <https://www.anthropic.com/news/evaluating-ai-systems> — 2026-05-26
6. PMC 12855988 — medical-LLM benchmark: <https://pmc.ncbi.nlm.nih.gov/articles/PMC12855988/> — 2026-05-26

Bad-pattern blocklist: `ragas-faithfulness-only`, `eval-judge-uncalibrated`, `eval-tiny-sample-set` (last reviewed 2026-05-12).

</details>

Last verified: 2026-06-03
