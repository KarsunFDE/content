---
week: W02
day: Fri
topic_slug: llm-as-judge-for-retrieval-eval
topic_title: "LLM-as-judge for retrieval eval — bias and calibration"
parent_overview: W02/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://futureagi.com/blog/llm-as-judge-best-practices-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://futureagi.com/blog/evaluating-llm-judge-bias-mitigation-2026/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://medium.com/@adnanmasood/rubric-based-evals-llm-as-a-judge-methodologies-and-empirical-validation-in-domain-context-71936b989e80
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.ragas.io/en/latest/concepts/metrics/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.anthropic.com/news/evaluating-ai-systems
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# LLM-as-judge for retrieval eval — bias and calibration

## 1. Learning Objectives

- By the end of this reading, the learner can describe the LLM-as-judge pattern, its inputs, and the three structural problems it carries (self-preference bias, rubric capture, non-determinism).
- By the end of this reading, the learner can write a rubric-style judge prompt with explicit per-band scoring criteria and explain why freeform "rate 0–10" prompts are unreliable.
- By the end of this reading, the learner can describe a calibration protocol that compares judge scores against human-scored examples and articulate what disagreement threshold means "the judge is broken".
- By the end of this reading, the learner can explain why a multi-dimensional rubric (faithfulness, context recall, context precision, answer relevance) is required and why faithfulness alone is insufficient.

## 2. Introduction

LLM-as-judge — a second language model scoring the first language model's output against a rubric — has become the default solution for grading open-ended generation. Substring-match and BLEU/ROUGE break the moment the system can answer the same question two valid ways. Human scoring works but does not scale to the hundreds of evals a serious team runs per week. LLM-as-judge is the compromise that everyone has settled on, and it is structurally unreliable in ways that take real discipline to control.

The pattern looks deceptively simple — call a model, give it the question, the system's answer, and a rubric, ask for a score. The actual engineering challenge is making that score stable enough to gate a merge. Three problems are non-negotiable: judges from the same family as the system tend to rate their family's outputs more leniently, vague rubrics produce noisy scores that drift across runs, and even temperature-zero judges give different scores on different runs because of sampling artifacts in the inference stack.

This reading walks the structural problems, the rubric design patterns that mitigate them, and the calibration protocol that catches when the judge has silently broken. It is paired with the harness reading — the harness gives you the data, the judge gives you a score, calibration tells you whether to trust the score. All three are required.

## 3. Core Concepts

### 3.1 The pattern

A judge call takes three inputs:

- The original query the system was asked.
- The system's output (answer string, plus optionally the retrieved chunks).
- A rubric describing what "good" means for this evaluation dimension.

The output is a score (often a band: 0.0, 0.3, 0.7, 1.0, or a continuous 0–1 number) and, in the best implementations, a free-text justification. Single-dimension rubrics ("score faithfulness from 0 to 1") are weaker than multi-dimension rubrics ("score faithfulness, context recall, context precision, and answer relevance independently") because the dimensions fail for different reasons and need different fix paths.

### 3.2 Self-preference bias

A judge from the same model family as the system under test tends to score that family's outputs higher than equivalent outputs from a different family. The mechanism is plausibly stylistic — the judge recognises its own family's phrasing patterns and rewards them. Documented gaps of 4–8 points on equivalent content have appeared when GPT-4 judges GPT-4 versus when GPT-4 judges Claude on the same outputs ([FutureAGI — LLM-Judge Bias Mitigation 2026](https://futureagi.com/blog/evaluating-llm-judge-bias-mitigation-2026/), retrieved 2026-05-26).

Mitigation: use a different model family for the judge than for the system. If your system uses Claude, judge with GPT or Gemini, or vice versa. If you must judge with the same family, use a smaller model in that family as the judge (e.g., Haiku judging Sonnet) — the size difference partially neutralises the stylistic match.

For higher-stakes launches, the recommendation has tightened in 2026 toward three-judge ensembles across three families, aggregated by majority or weighted vote ([FutureAGI — LLM-as-Judge Best Practices 2026](https://futureagi.com/blog/llm-as-judge-best-practices-2026), retrieved 2026-05-26). For a starting harness, the single-judge-different-family pattern is sufficient; the ensemble pattern is what you graduate to.

### 3.3 Rubric capture

A judge can only grade what its rubric describes. A vague rubric ("rate faithfulness from 0 to 1") forces the judge to invent its own definition, and different runs invent different definitions. The mitigation is explicit per-band criteria:

> 1.0 — every factual claim in the answer is directly supported by a retrieved chunk.
> 0.7 — most claims are grounded, one claim is plausible but not in the chunks.
> 0.3 — at least one claim is contradicted by, or absent from, the chunks.
> 0.0 — the answer directly contradicts the chunks or fabricates a citation.

Plus a chain-of-thought instruction: "First identify each factual claim in the answer. Then check each claim against the retrieved chunks. Then assign a band score with one sentence of justification." Chain-of-thought is worth the tokens — the judge is forced to do the work step by step, and the justification serves as audit evidence for why a particular score landed.

The same pattern applies to retrieval dimensions (context recall, context precision) and to answer-quality dimensions (answer relevance). Each dimension gets its own rubric file with its own per-band criteria. Sharing a single mega-prompt across dimensions tends to bleed criteria across dimensions ([Rubric-Based Evaluations & LLM-as-a-Judge — Adnan Masood, Apr 2026](https://medium.com/@adnanmasood/rubric-based-evals-llm-as-a-judge-methodologies-and-empirical-validation-in-domain-context-71936b989e80), retrieved 2026-05-26).

### 3.4 Non-determinism

LLM-as-judge is non-deterministic even at temperature 0. The same prompt with the same input can produce different scores across runs because of sampling artifacts in the inference stack — the judge is a sampling problem, not a deterministic one. The mitigation that has settled into 2026 practice: N=3 runs with seeded sampling, report mean and standard deviation. Standard deviation above ~0.05 on any dimension is a signal the rubric is too vague — tighten it.

This also matters for cost. Each judge call costs tokens; three calls per row per dimension across a 250-row set across four dimensions is 3,000 judge calls per eval run. Pin a cheap, fast judge model (Haiku-class or equivalent) for production scoring, and reserve frontier models for monthly calibration anchors.

### 3.5 Calibration

The single most-skipped step. Without calibration, you cannot tell when the judge has silently broken — the dashboard shows scores, the scores look fine, and you have no idea whether they correlate with what an expert would say.

The protocol:

1. Hand-score 5–10% of the QA set with a domain expert (or two experts for adjudication on disagreements).
2. Run the judge against the same rows.
3. Compute mean absolute disagreement: `mean(abs(judge_score - human_score))`. Or compute Cohen's kappa for categorical agreement.
4. If mean disagreement exceeds the threshold (a common bar is 0.1 for continuous scores, or kappa < 0.6 for categorical), the rubric is broken — iterate the prompt.
5. Re-run monthly. If kappa drifts downward over time, either the data distribution has shifted (new corpus, new query patterns) or the rubric has aged out.

The Anthropic guidance frames the same idea differently: "robust evaluations are extremely difficult to develop and implement" — the implication is that calibration is the only thing that prevents the eval from confidently lying to you ([Anthropic — Evaluating AI systems](https://www.anthropic.com/news/evaluating-ai-systems), retrieved 2026-05-26).

### 3.6 The four dimensions

A multi-dimensional rubric is the difference between a harness that catches real regressions and one that ships false confidence. The canonical four ([RAGAS — Overview of available metrics](https://docs.ragas.io/en/latest/concepts/metrics/), retrieved 2026-05-26):

- **Faithfulness** — Do the answer's claims appear in the retrieved chunks? Catches hallucination.
- **Context recall** — Of the chunks that *should* have been retrieved to answer this query, how many were? Catches retrieval coverage failures.
- **Context precision** — Of the chunks that *were* retrieved, how many are actually relevant? Catches retrieval noise.
- **Answer relevance** — Does the answer address what the query actually asked? Catches the case where the system produced a faithful but off-topic response.

> [!instructor-review]
> Some sources advocate "faithfulness alone is sufficient" for shipping RAG. This is the `ragas-faithfulness-only` blocklist entry. Faithfulness alone misses context-recall (the system did not retrieve the right chunks) and answer-relevance (the system answered a different question than the one asked) failure modes. The harness must score all four dimensions.

## 4. Generic Implementation

A rubric-driven judge call, in plain Python. No `Chain` class, no `|` pipe composition — just function calls:

```python
# tests/eval/judge.py
import json
import statistics
from llm_client import call_judge  # thin wrapper over your judge model

FAITHFULNESS_RUBRIC = """
You are scoring whether an answer is faithful to its retrieved evidence.

Score bands:
- 1.0: every factual claim is directly supported by at least one chunk.
- 0.7: most claims grounded; exactly one claim plausible but unsupported.
- 0.3: at least one claim contradicts or is absent from the chunks.
- 0.0: the answer fabricates a citation or directly contradicts the chunks.

Process (chain of thought, then score):
1. List each factual claim in the answer.
2. For each claim, name the chunk that supports it, or note "unsupported".
3. Pick the score band that best fits.
4. One-sentence justification.

Return JSON: {"claims": [...], "supports": [...], "score": <band>, "justification": "..."}
"""

def grade_faithfulness(answer, chunks, n_runs=3):
    scores = []
    for _ in range(n_runs):
        result = call_judge(
            system=FAITHFULNESS_RUBRIC,
            user=json.dumps({"answer": answer, "chunks": chunks}),
            temperature=0,  # not strictly deterministic, but minimises variance
        )
        parsed = json.loads(result)
        scores.append(parsed["score"])
    return {
        "mean": statistics.mean(scores),
        "stddev": statistics.stdev(scores) if len(scores) > 1 else 0.0,
        "raw_scores": scores,
    }
```

The pattern repeats for each dimension — one rubric per dimension, one function per dimension, all three runs averaged. The runner from the previous reading calls these and writes mean + stddev for each dimension to the results file.

## 5. Real-world Patterns

**Healthcare — medical-LLM benchmarking.** Researchers building safety-and-effectiveness benchmarks for medical LLMs explicitly separate safety scores from effectiveness scores rather than collapsing into a single number. A six-model benchmark produced average total scores of 57.2% but safety scores of only 54.7% — the kind of gap a single-dimension judge would have hidden ([A novel evaluation benchmark for medical LLMs, PMC 12855988](https://pmc.ncbi.nlm.nih.gov/articles/PMC12855988/), retrieved 2026-05-26).

**E-commerce — search relevance ranking.** A large marketplace's search team ran a paired-comparison judge ensemble (three frontier models, majority vote) to grade re-ranker changes. Single-judge attempts had shown ~7-point family-bias drift; the ensemble cut measured drift to under 2 points ([FutureAGI — LLM-as-Judge Best Practices 2026](https://futureagi.com/blog/llm-as-judge-best-practices-2026), retrieved 2026-05-26).

**Education — automated essay grading.** A K-12 vendor running automated rubric-based scoring on student essays calibrates monthly against teacher-graded gold sets. When kappa dropped from 0.71 to 0.58 in March 2026 (after a curriculum shift introduced new essay prompts), the eval team detected the drift inside two weeks and re-grounded the rubric on the new prompt distribution ([Rubric-Based Evaluations & LLM-as-a-Judge — Adnan Masood](https://medium.com/@adnanmasood/rubric-based-evals-llm-as-a-judge-methodologies-and-empirical-validation-in-domain-context-71936b989e80), retrieved 2026-05-26).

## 6. Best Practices

- Use a different model family for the judge than for the system; if you must share a family, use a smaller model as the judge.
- Write per-band rubric criteria for every dimension, with a chain-of-thought instruction; refuse vague "rate 0–10" prompts.
- Run each judge call N=3 times with seeded sampling; report mean and standard deviation; stddev > 0.05 means the rubric is too vague.
- Calibrate against a human-scored gold set monthly; track mean absolute disagreement (continuous) or Cohen's kappa (categorical); alert if it drifts.
- Use a multi-dimensional rubric (at minimum: faithfulness, context recall, context precision, answer relevance); never ship faithfulness-only.
- Keep judge cost under 10–15% of production LLM cost; if it creeps above 25%, distill or downsample.
- Log the judge's chain-of-thought justification alongside the score — it is your audit trail when someone disputes a result.

## 7. Hands-on Exercise

**Code task (15 min):** Take the `grade_faithfulness` function in Section 4 and modify it for a different dimension: **answer relevance** (does the answer address the question the user actually asked?). Write the rubric prompt (per-band criteria + chain-of-thought instruction), then sketch the function signature and one or two test rows that should hit different bands.

**What good looks like:** Your rubric clearly distinguishes "the answer is faithful but off-topic" (which should score low on relevance but possibly high on faithfulness) from "the answer is on-topic but unsupported" (which should score high on relevance but low on faithfulness). The chain-of-thought instruction asks the judge to first restate the question's intent in its own words before scoring. Your test rows include at least one row designed to score 1.0 and one designed to score 0.0; the band boundaries between them are unambiguous.

## 8. Key Takeaways

- What are the three structural problems with LLM-as-judge, and what is the standard mitigation for each?
- Why does a rubric need per-band criteria and chain-of-thought, rather than a freeform "rate 0–10" prompt?
- What calibration protocol catches when the judge has silently broken, and what threshold means "broken"?
- Why is a multi-dimensional rubric required, and what does each of the four canonical RAG dimensions catch that the others miss?

## Sources

1. [LLM-as-Judge Best Practices in 2026: Calibration, Bias, and Cost — FutureAGI](https://futureagi.com/blog/llm-as-judge-best-practices-2026) — retrieved 2026-05-26
2. [LLM-Judge Bias Mitigation (2026): Detect, Measure, Fix — FutureAGI](https://futureagi.com/blog/evaluating-llm-judge-bias-mitigation-2026/) — retrieved 2026-05-26
3. [Rubric-Based Evaluations & LLM-as-a-Judge — Adnan Masood (Medium, Apr 2026)](https://medium.com/@adnanmasood/rubric-based-evals-llm-as-a-judge-methodologies-and-empirical-validation-in-domain-context-71936b989e80) — retrieved 2026-05-26
4. [RAGAS — Overview of available metrics](https://docs.ragas.io/en/latest/concepts/metrics/) — retrieved 2026-05-26
5. [Anthropic — Evaluating AI systems](https://www.anthropic.com/news/evaluating-ai-systems) — retrieved 2026-05-26
6. [A novel evaluation benchmark for medical LLMs — PMC 12855988](https://pmc.ncbi.nlm.nih.gov/articles/PMC12855988/) — retrieved 2026-05-26

Last verified: 2026-05-26
