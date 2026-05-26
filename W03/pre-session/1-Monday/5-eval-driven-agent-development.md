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
last_verified: 2026-05-26
---

# Eval-driven agent development — every transition has a measurable signal

## 1. Learning Objectives

By the end of this reading, the learner can:

- Articulate the difference between *test-driven* and *eval-driven* development and explain why agents need the latter.
- Identify the three levels at which an agent system should be measured (component, transition, end-to-end) and name a metric for each.
- Distinguish reference-based metrics from reference-free LLM-as-judge metrics, and pick the right tool for a given signal.
- Design an offline replay set that catches regressions before a prompt edit reaches production.
- Explain why "vibes-based" iteration on agent prompts plateaus and what offline evals replace it with.

## 2. Introduction

Most teams discover their agent has regressed the same way: a user reports a bad output, the team scrolls through traces, and someone says "wasn't this working last Tuesday?" The answer is unknowable because there is no record of what *working* meant on Tuesday. Eval-driven development is the discipline of writing down what working means — as a *data set with expected signals* — *before* you make the next change, so you can replay it after every change and watch the number move.

This is a sharper claim than "we should have tests." Tests answer pass/fail questions about deterministic code. LLM-driven systems are non-deterministic; the right unit of measurement is a *score over a population*, not a binary on a single case. The shift from tests to evals is what Hamel Husain calls *evaluation-driven development* — and his blunt summary is the right framing to start from: "Your AI product needs evals" ([Hamel Husain — Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/), retrieved 2026-05-26).

For an agent — a state machine whose transitions are LLM-decided — the principle gets richer: every transition is a measurement point. The transition into `route_to_evaluators` has a routing-accuracy signal. The transition into `score_completeness` has a scoring-correlation signal. The end-to-end run has a workflow-success signal. The system you ship is the system whose every transition has a number you can move.

## 3. Core Concepts

### 3.1 Tests, evals, and where the line is

A useful working distinction ([Evidently AI — LLM evaluation guide](https://www.evidentlyai.com/llm-guide/llm-evaluation), retrieved 2026-05-26):

- **Tests** are deterministic pass/fail checks against a known answer. `assert sum(2, 2) == 4`. Required, but insufficient for LLM-driven code.
- **Evals** are *statistical* measurements over a *labelled or judged dataset*. They produce a score per dimension per dataset, and the meaningful unit of decision is whether the score moved compared to the previous version of the prompt or model.
- The line moves over time: today's eval (LLM-as-judge over 200 examples) becomes tomorrow's test (deterministic regex assertion over the 10 examples whose behaviour is fully understood). Both exist simultaneously.

### 3.2 The three measurement levels of an agent

A serviceable mental model for where to attach evals:

| Level | What it measures | Example metric |
|---|---|---|
| **Component** | A single LLM call inside a node | Faithfulness, tool-call validity, schema adherence |
| **Transition** | The choice of which edge to take | Routing accuracy, classification F1 |
| **End-to-end** | The whole workflow run | Task success, time-to-resolution, human override rate |

Most teams over-invest at the component level (because individual LLM calls are easy to test in isolation) and under-invest at the transition and end-to-end levels (where the bugs that hurt users actually live).

### 3.3 Reference-based vs reference-free metrics

- **Reference-based** — you have a ground-truth answer for each example, and you compare the system's output to it (e.g., exact match, BLEU, ROUGE, embedding similarity). Right when you have curated labels.
- **Reference-free** — no ground truth; you ask another model (or a rubric) to judge the output (e.g., RAGAS faithfulness, LLM-as-judge correctness). Right when ground truth is unavailable or expensive ([Eugene Yan — LLM Evaluators](https://eugeneyan.com/writing/llm-evaluators/), retrieved 2026-05-26).

LLM-as-judge has known failure modes (positional bias, verbosity bias, self-preference). Mitigations: pairwise comparison instead of absolute scoring, position-swapping to detect bias, calibration against a small human-labelled set. Without the mitigations, LLM-judge scores drift.

### 3.4 The "offline replay set" pattern

The load-bearing artefact in an eval-driven workflow is an *offline replay set* — a curated collection of input cases, each with the expected signal (label, judge-rubric, or trace property). Every prompt edit, every model swap, every architectural change re-runs the replay set and produces a delta against the previous run.

A serviceable replay set has four properties:

1. **Coverage** — it includes the easy cases the system already gets right, the hard cases it sometimes gets wrong, and the cases that have failed in production.
2. **Versioning** — the set is checked into source control alongside the prompts, so a diff to the prompt and a diff to the eval set live in the same review.
3. **Stratification** — scores are reported per slice (e.g., per intent category, per tenant, per length bucket), not just as a single global number — averages hide regressions on minority slices.
4. **Cost discipline** — running the full set on every PR is expensive; common practice is a small smoke set on every PR + the full set on merge to main ([LangSmith — Evaluation concepts](https://docs.smith.langchain.com/evaluation), retrieved 2026-05-26).

### 3.5 Metrics for the three levels of an agentic flow

A non-exhaustive but serviceable starter kit:

**Component-level (one node, one LLM call):**

- *Faithfulness* — is the output grounded in retrieved context? (RAGAS faithfulness, LLM-judge against context.)
- *Tool-call validity* — when the model emitted a tool call, did it have the right name, schema, and arguments?
- *Schema adherence* — when the model was supposed to emit JSON with N fields, did it?

**Transition-level (which edge did we take):**

- *Routing accuracy* — when the supervisor routed to worker A, was A the right choice? (Confusion matrix over labelled examples.)
- *Classification F1* — for transitions driven by a classifier, per-class precision/recall.
- *Calibration* — when the model said "85% confident," was the empirical accuracy near 85%?

**End-to-end:**

- *Task success rate* — over a labelled dataset, what fraction of runs reached the correct terminal state?
- *Human override rate* — what fraction of soft-gated transitions did the human override? (A leading indicator of a misfiring auto-route; see §3.6.)
- *Time-to-resolution* — for workflows that should complete quickly, the wall-clock cost of a run.

### 3.6 Human override rate as the load-bearing operational signal

For agents that include soft gates (suggest a default, allow a human to override), the *override rate* is the most useful operational signal: it tells you whether the default is good enough to keep, or whether the human is constantly overriding it and the default should be retired. Anthropic's *Building Effective Agents* essay makes the same point in operational terms: the value of human review is in the *delta* between the system's suggestion and the human's correction — that delta *is* the next training signal ([Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents), retrieved 2026-05-26).

A useful rule-of-thumb threshold from production teams: if a soft gate's human override rate exceeds 15% over a stable population, the default is wrong and needs to be revisited.

### 3.7 What to do when evals disagree with vibes

A common pattern: an engineer "feels" the new prompt is better, but the eval set says the score went down. The right move is *not* to discard the evals. The right move is to find the cases the evals say got worse and look at them — either the evals' rubric is wrong (revise the rubric or add a missing slice) or your intuition is wrong (the new prompt is better on the visible cases but worse on the invisible ones). The eval-driven discipline is the corrective for sample-of-one human review.

## 4. Generic Implementation

A worked example outside federal-acquisitions: an email-triage agent for an IT helpdesk. Tickets arrive, the agent classifies them (`password-reset`, `software-install`, `hardware-issue`, `other`) and either routes to the right queue or escalates to a human.

```python
# replay_set.jsonl  (committed to source control next to the prompt)
# {"input": {"subject": "Outlook won't open", "body": "..."}, "expected_category": "software-install", "tags": ["common"]}
# {"input": {"subject": "Forgot my pw", "body": "..."}, "expected_category": "password-reset", "tags": ["easy"]}
# {"input": {"subject": "screen flickers", "body": "..."}, "expected_category": "hardware-issue", "tags": ["hard"]}

from dataclasses import dataclass
import json
from typing import Callable

@dataclass
class EvalResult:
    n: int
    accuracy: float
    per_class: dict[str, dict[str, float]]   # precision/recall per class
    per_tag: dict[str, float]                # accuracy per slice
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

            # Confusion-matrix bookkeeping
            per_class_counts.setdefault(expected, {"tp": 0, "fp": 0, "fn": 0})
            per_class_counts.setdefault(prediction, {"tp": 0, "fp": 0, "fn": 0})
            if is_correct:
                per_class_counts[expected]["tp"] += 1
            else:
                per_class_counts[prediction]["fp"] += 1
                per_class_counts[expected]["fn"] += 1

            # Per-slice accuracy
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

    return EvalResult(
        n=n,
        accuracy=correct / n,
        per_class=per_class,
        per_tag=per_tag,
        cost_usd=total_cost,
    )
```

The shape of the runner is what matters more than any single line: it loads a *versioned* replay set, runs the classifier under test, reports *stratified* metrics (per class, per tag), and surfaces the cost so you can decide if the size of the set is sustainable on every PR.

> [!instructor-review]
> **Do not use RAGAS faithfulness as the only metric for a RAG step.** The cohort will see RAGAS again in W3 Tue (agentic RAG). The known-bad pattern is "RAGAS faithfulness is sufficient" — faithfulness alone misses context-recall and answer-relevance failures. The programme spec uses all four RAGAS dimensions (faithfulness, context recall, context precision, answer relevance). See `~/fde-10-week/skills/tech-research/references/known-bad-patterns.yml` entry `ragas-faithfulness-only`.

## 5. Real-world Patterns

**Fintech — fraud-model regression suites.** Card-payment platforms run a labelled set of flagged-and-cleared transactions through every model candidate before promotion. The gating signal is not just global AUC but per-slice AUC — stratified by merchant category, geography, dollar bucket. A model that improves global numbers but regresses on small-business merchants gets rejected. Stratify your eval set the way you stratify your users.

**Healthcare — clinical-decision-support metric stacks.** Modern CDS systems are measured at three levels on every model update: component (does the medication-interaction lookup return the right pairs?), transition (when the system recommends a specialist referral, was that the right referral?), end-to-end (did the patient's outcome improve in the studied population?). The three-level stack is the right template for any agentic system where decisions compound ([Evidently AI — LLM evaluation guide](https://www.evidentlyai.com/llm-guide/llm-evaluation), retrieved 2026-05-26).

**E-commerce — A/B harnesses for search-and-recommend.** Large retail search platforms ship behind A/B harnesses with offline eval sets *and* online metrics. Offline scores gate which candidates ship to A/B at all; online metrics decide whether the A/B promotes. Offline evals are not a replacement for online experiments; they are the *gate* that decides what is worth experimenting on.

**Gaming — toxicity-classifier replay sets.** Competitive-gaming platforms run a labelled replay set of historical chat. New classifier versions must score within a tight band of the previous version's per-class recall before they can ship; every false-positive that a user appeals gets added to the set. The continuous expansion of the set is the discipline that keeps the system improving instead of drifting.

## 6. Best Practices

- Commit the replay set into source control alongside prompts and graph definitions — diffs to the eval set belong in the same review as diffs to the system under evaluation.
- Stratify scores by every dimension you care about operationally (tenant, intent, length, language) — global averages hide minority-slice regressions.
- Pair every soft-gated transition with an override-rate dashboard; treat sustained override > 15% as a design defect, not a user-training problem.
- Use reference-based metrics where ground truth is available and LLM-as-judge with pairwise comparison + position-swapping where it is not.
- Run a small smoke set on every PR and the full set on merge — the cost discipline is what keeps evals from becoming a tax that teams route around.
- Treat eval results that disagree with intuition as a finding, not an inconvenience — inspect the cases that moved, fix either the rubric or the prompt.
- Add every production failure to the replay set; the set should grow monotonically.

## 7. Hands-on Exercise

**Code task (12–15 minutes).** You inherit a content-moderation agent that classifies forum posts into `allow`, `warn`, `remove`. It has no eval harness. You have:

- A directory of 200 historical posts, each with a label assigned by a senior moderator.
- The agent's `classify(post) -> (category, model_cost_usd)` function.

Write a runner that:

1. Loads the labelled dataset.
2. Runs the classifier on each post.
3. Produces per-class precision/recall.
4. Stratifies accuracy by `post_length_bucket` (short / medium / long).
5. Reports total cost.
6. Compares the result against the previous run (stored in `prev_run.json`) and prints any class whose recall dropped by more than 2 percentage points.

**What good looks like.** Your runner is < 80 lines. It writes results to `runs/<timestamp>.json` so future runs can diff against it. The diff against the previous run highlights regressions explicitly (you do not require a human to eyeball two JSON files). Stratification surfaces a per-length-bucket recall — and you would notice if "long" posts regressed even while the global accuracy improved. You explicitly avoid LLM-as-judge here (the labels are human; reference-based is the right tool).

## 8. Key Takeaways

- Can you state the difference between a test and an eval in one sentence?
- Can you name the three measurement levels of an agentic flow (component, transition, end-to-end) and a metric for each?
- Can you justify when to reach for reference-based metrics vs LLM-as-judge, and what bias mitigations LLM-as-judge requires?
- Can you describe the offline replay set pattern and the four properties a serviceable replay set has (coverage, versioning, stratification, cost discipline)?
- Can you explain why human override rate is the load-bearing operational signal for any agent with soft gates?

## Sources

1. [Your AI Product Needs Evals — Hamel Husain](https://hamel.dev/blog/posts/evals/) — retrieved 2026-05-26
2. [Evaluating the Effectiveness of LLM-Evaluators — Eugene Yan](https://eugeneyan.com/writing/llm-evaluators/) — retrieved 2026-05-26
3. [Evaluation concepts — LangSmith Docs](https://docs.smith.langchain.com/evaluation) — retrieved 2026-05-26
4. [LLM evaluation: a beginner's guide — Evidently AI](https://www.evidentlyai.com/llm-guide/llm-evaluation) — retrieved 2026-05-26
5. [Building Effective AI Agents — Anthropic](https://www.anthropic.com/research/building-effective-agents) — retrieved 2026-05-26

Last verified: 2026-05-26
