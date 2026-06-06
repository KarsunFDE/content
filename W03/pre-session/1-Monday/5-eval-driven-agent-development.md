---
week: W03
day: Mon
topic_slug: eval-driven-agent-development
topic_title: "Eval-driven agent development — every transition has a measurable signal"
parent_overview: W03/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://hamel.dev/blog/posts/evals/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://eugeneyan.com/writing/llm-evaluators/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.smith.langchain.com/evaluation
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.evidentlyai.com/llm-guide/llm-evaluation
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.anthropic.com/research/building-effective-agents
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-06-06
---

# Eval-driven agent development — every transition has a measurable signal

> [!NOTE]
> **From earlier:** W2 Fri's eval harness gave measurable signals on RAG transitions. Today the same principle lifts to agent scope — every *agent transition* gets its own signal, not just individual LLM calls.

## 1. Learning Objectives

- Articulate the difference between test-driven and eval-driven development.
- Name the three measurement levels of an agent system (component, transition, end-to-end) and a metric for each.
- Distinguish reference-based from reference-free metrics and pick the right tool for a given signal.
- Design an offline replay set that catches regressions before a prompt edit reaches production.

## 2. Introduction

Most teams discover regression the same way: a user reports a bad output and someone says "wasn't this working last Tuesday?" Unknowable — because there is no record of what *working* meant. Eval-driven development writes it down as a dataset with expected signals *before* the next change, so you can replay after every change and watch the number move.

This is sharper than "we should have tests." Tests answer pass/fail on deterministic code. LLM-driven systems are non-deterministic; the right unit is a *score over a population*. Hamel Husain: "Your AI product needs evals" ([Hamel Husain](https://hamel.dev/blog/posts/evals/), retrieved 2026-05-26).

For an agent — a state machine with LLM-decided transitions — every transition is a measurement point. `route_to_evaluators` has a routing-accuracy signal; `score_completeness` has a scoring-correlation signal. The system you ship is the one whose every transition has a number you can move.

## 3. Core Concepts

### 3.1 Tests vs evals — where the line is

| | Tests | Evals |
|---|---|---|
| Answers | Pass/fail on a single case | Score over a labelled population |
| Code type | Deterministic | LLM-driven, non-deterministic |
| Regression signal | Binary change | Delta from previous run |
| Evolves into | Stays as tests | Deterministic assertions as coverage matures |

Both exist simultaneously: today's eval becomes tomorrow's test as behaviour is understood ([Evidently AI](https://www.evidentlyai.com/llm-guide/llm-evaluation), retrieved 2026-05-26).

### 3.2 Three measurement levels of an agent

| Level | What it measures | Example metric |
|---|---|---|
| **Component** | Single LLM call inside a node | Faithfulness, tool-call validity, schema adherence |
| **Transition** | The choice of which edge to take | Routing accuracy, classification F1 |
| **End-to-end** | The whole workflow run | Task success rate, human override rate, time-to-resolution |

Most teams over-invest at the component level and under-invest at transition and end-to-end levels — where bugs that hurt users actually live.

> [!NOTE]
> **Operational implication:** Most pairs over-invest at the component level and skip transition-level measurement. Your HITL #3 ADR must name at least one transition-level signal — routing accuracy or override rate — not just a component faithfulness score.

### 3.3 Reference-based vs reference-free metrics

- **Reference-based** — compare output to ground truth (exact match, embedding similarity). Use when curated labels exist.
- **Reference-free** — ask another model to judge (RAGAS faithfulness, LLM-as-judge). Use when ground truth is unavailable or expensive ([Eugene Yan](https://eugeneyan.com/writing/llm-evaluators/), retrieved 2026-05-26).

LLM-as-judge failure modes: positional bias, verbosity bias, self-preference. Mitigations: pairwise comparison, position-swapping, calibration on a small human-labelled set.

### 3.4 The offline replay set — four required properties

A *replay set* is a versioned collection of input cases with expected signals; every prompt edit re-runs it and produces a delta. Four required properties:

1. **Coverage** — easy, hard, and past-failure cases.
2. **Versioning** — checked into source control alongside prompts.
3. **Stratification** — scores per slice (tenant, intent, length); averages hide minority regressions.
4. **Cost discipline** — smoke set on every PR; full set on merge ([LangSmith](https://docs.smith.langchain.com/evaluation), retrieved 2026-05-26).

> [!TIP]
> **ADR eval-signal requirement.** Each ADR must name its eval signal: *"This soft gate is right when override rate < 15% on a 50-proposal replay."* Today's ADR commits what you measure; W6's eval report is where those signals become evidence.

## 4. Generic Implementation

```python
# replay_set.jsonl — committed to source control next to the prompt
# {"input": {"subject": "Outlook won't open"}, "expected_category": "software-install", "tags": ["common"]}
# {"input": {"subject": "Forgot my pw"}, "expected_category": "password-reset", "tags": ["easy"]}
# {"input": {"subject": "screen flickers"}, "expected_category": "hardware-issue", "tags": ["hard"]}

from dataclasses import dataclass
import json
from typing import Callable

@dataclass
class EvalResult:
    n: int
    accuracy: float
    per_class: dict[str, dict[str, float]]  # precision/recall per class
    per_tag: dict[str, float]               # accuracy per slice
    cost_usd: float

def evaluate_classifier(predict_fn: Callable, replay_path: str) -> EvalResult:
    correct = 0
    per_class_counts: dict[str, dict[str, int]] = {}
    per_tag_correct: dict[str, list[int]] = {}
    total_cost = 0.0
    n = 0

    with open(replay_path) as f:
        for line in f:
            row = json.loads(line)
            n += 1
            prediction, cost = predict_fn(row["input"])
            total_cost += cost
            expected = row["expected_category"]
            is_correct = prediction == expected
            correct += int(is_correct)
            per_class_counts.setdefault(expected, {"tp": 0, "fp": 0, "fn": 0})
            per_class_counts.setdefault(prediction, {"tp": 0, "fp": 0, "fn": 0})
            if is_correct:
                per_class_counts[expected]["tp"] += 1
            else:
                per_class_counts[prediction]["fp"] += 1
                per_class_counts[expected]["fn"] += 1
            for tag in row.get("tags", []):
                per_tag_correct.setdefault(tag, []).append(int(is_correct))

    per_class = {
        cls: {
            "precision": c["tp"] / (c["tp"] + c["fp"]) if (c["tp"] + c["fp"]) else 0.0,
            "recall":    c["tp"] / (c["tp"] + c["fn"]) if (c["tp"] + c["fn"]) else 0.0,
        }
        for cls, c in per_class_counts.items()
    }
    per_tag = {tag: sum(v) / len(v) for tag, v in per_tag_correct.items()}
    return EvalResult(n=n, accuracy=correct/n, per_class=per_class,
                      per_tag=per_tag, cost_usd=total_cost)
```

The shape matters more than any single line: versioned replay set, stratified metrics (per class, per tag), cost surfaced. Runner output goes to `runs/<timestamp>.json` for future diffs.

## 5. Real-world Patterns

**Fintech — fraud-model regression suites.** Card-payment platforms gate model candidates on per-slice AUC — by merchant category, geography, dollar bucket — not just global AUC. A model that improves globally but regresses on small merchants gets rejected.

**Healthcare — clinical-decision-support metric stacks.** Modern CDS systems measure at three levels: component (medication-interaction lookup), transition (specialist referral accuracy), end-to-end (patient outcome). The three-level stack is the template for any agentic system where decisions compound ([Evidently AI](https://www.evidentlyai.com/llm-guide/llm-evaluation), retrieved 2026-05-26).

> [!TIP]
> **Cross-domain lesson:** The three-level stack (component → transition → end-to-end) applies directly to your intake-triage flow. Name one metric per level in the HITL #3 ADR today — that baseline seeds your W6 eval report.

## 6. Best Practices

- Commit the replay set alongside prompts — eval-set diffs belong in the same review as prompt diffs.
- Stratify scores by every operational dimension (tenant, intent, length); global averages hide minority regressions.
- Pair every soft-gated transition with an override-rate signal; treat sustained override > 15% as a design defect.
- Use reference-based metrics where ground truth exists; LLM-as-judge with pairwise comparison + position-swapping where it doesn't.
- Run a smoke set on every PR and the full set on merge — cost discipline keeps evals from becoming a tax teams route around.

> [!WARNING]
> **Anti-pattern: `ragas-faithfulness-only`.** RAGAS faithfulness alone misses context-recall and answer-relevance failure modes. The programme requires all four dimensions (faithfulness, context recall, context precision, answer relevance). An ADR naming only faithfulness is under-specified. See slug `ragas-faithfulness-only` in `known-bad-patterns.yml`.

## 7. Hands-on Exercise

You inherit a content-moderation agent classifying posts into `allow`, `warn`, `remove`. No eval harness exists. You have 200 labelled posts and the agent's `classify(post) -> (category, cost_usd)` function.

Write a runner that: (1) loads the dataset, (2) runs the classifier, (3) produces per-class precision/recall, (4) stratifies accuracy by `post_length_bucket` (short/medium/long), (5) reports total cost, (6) compares against `prev_run.json` and prints any class whose recall dropped > 2pp.

> [!NOTE]
> **Self-check** (30s — answer mentally before expanding)
>
> 1. Should you use reference-based or LLM-as-judge metrics here? Why?
> 2. Global accuracy improves 87% → 89%, but "long" posts recall drops 74% → 68%. Ship or hold?

<details>
<summary>Show answers</summary>

1. Reference-based — the labels are human-assigned ground truth. LLM-as-judge is for when ground truth is unavailable or too expensive to produce. Using an LLM judge when you have human labels adds cost and introduces judge-calibration uncertainty without benefit.
2. Hold. A 6-point recall drop on "long" posts is a meaningful regression even though the global number improved. Long posts are often the most complex moderation cases — the cases where errors have the highest cost. Stratification is specifically designed to surface this; a global metric would have hidden it.

</details>

## 8. Key Takeaways

- Tests answer pass/fail on deterministic code; evals score a population of LLM-driven cases.
- Three measurement levels: component (one LLM call), transition (which edge), end-to-end (workflow outcome).
- Replay set: versioned, stratified, coverage-complete, cost-disciplined — four properties, not one.
- Human override rate on soft gates is the load-bearing signal for whether a default is working.
- Every ADR in today's plan-spec must name its eval signal — that seed feeds the W6 eval report.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://hamel.dev/blog/posts/evals/ — retrieved 2026-05-26 — foundation-stable
- https://eugeneyan.com/writing/llm-evaluators/ — retrieved 2026-05-26 — foundation-stable
- https://docs.smith.langchain.com/evaluation — retrieved 2026-05-26 — hot-tech
- https://www.evidentlyai.com/llm-guide/llm-evaluation — retrieved 2026-05-26 — foundation-stable
- https://www.anthropic.com/research/building-effective-agents — retrieved 2026-05-26 — foundation-stable

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

The human override rate threshold (15% rule-of-thumb from production teams) has an important nuance: the 15% applies to a *stable population* over a sufficient sample size (typically 50+ cases). Early in deployment, override rates are often higher because the system is encountering distribution shifts it was not trained on. The right move is to watch the rate *trend* over the first two weeks, not the absolute rate on week one.

For the pair-project's intake-triage flow, a useful override-rate decomposition: measure override rate separately for `route_to_evaluators` (soft gate), `flag_for_amendment_review` (soft gate), and any hard gate (should be near zero — if the CO is overriding the hard gate to bypass it, that's an architecture problem, not a calibration problem).

LangSmith's evaluation SDK has native support for running offline evals against a dataset, tracking runs by version, and computing deltas. The `evaluate()` function accepts a `dataset_name`, an `evaluator`, and a `target` (the function under evaluation). Senior FDEs should wire this up in the pair-project's CI by W3 Thu so the Friday defense can show an actual eval dashboard, not just code.

</details>

Last verified: 2026-06-06
