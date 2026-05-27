---
week: W01
day: Fri
topic_slug: prompt-evaluations-lifecycle-management
topic_title: "Prompt evaluations + lifecycle management"
parent_overview: W01/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 8
sources:
  - url: https://logiciel.io/blog/llm-eval-harness-internal-build-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.braintrust.dev/articles/what-is-prompt-versioning
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.kumohq.co/blog/prompt-engineering-best-practices
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.getmaxim.ai/articles/top-5-prompt-versioning-platforms-in-2026/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.digitalapplied.com/blog/prompt-library-audit-100-point-evaluation-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# Prompt evaluations + lifecycle management

## 1. Learning Objectives

By the end of this reading, the learner can:

- Treat prompts as versioned artifacts subject to the same review discipline as code.
- Design a flat-file prompt registry that scales from a single team to a multi-service deployment.
- Build a minimal eval harness with cases, scoring, regression alerts, and a documented operating cadence.
- Choose between shadow-mode, canary, and A/B rollouts for prompt changes by risk profile.
- Recognise the trap of "ad-hoc prompt tweaks" and replace it with auditable promotion through environments.

## 2. Introduction

Prompts change. Models change. The same prompt against a new model behaves differently; the same model against a tweaked prompt drifts. A production LLM system that cannot answer "which prompt was running yesterday at 3pm, and what did its eval score look like?" is one bad change away from a regression nobody can reproduce.

Industry guides published across 2026 — Logiciel, Braintrust, KumoHQ, Maxim, Digital Applied — converge on the same answer: treat prompts as code. Store them in a registry. Version them with semver. Run an eval harness on every change. Promote them through environments the same way you promote a deploy. Detect drift in production. Roll back when the eval scores regress.

This reading is short because the discipline is conceptually simple — its difficulty is in *adopting* it before scale forces it on you. Friday's PR work seeds the discipline: every prompt change committed in the cohort goes through the eval harness (which lands in W2 Fri) before merging. The infrastructure W1 sets up is intentionally minimal — flat files, git, a small Python eval script — to keep the conceptual frame clear before the tooling gets sophisticated.

## 3. Core Concepts

### 3a. Prompts as artifacts

A production prompt has three properties that pure-string thinking misses:

1. **A name and a version.** Not "the summariser prompt" but `summariser/draft-headline:v2.3.1`. Semver applies: MAJOR for format/policy changes that break downstream consumers, MINOR for behaviour improvements that remain backward-compatible, PATCH for typo or tone fixes. Braintrust and KumoHQ both recommend this exact split.
2. **A schema contract.** The prompt has named inputs (the dynamic substitution slots) and a defined output shape (the JSON schema or text format). Changes to either are breaking changes.
3. **An eval score.** Every version has a score against a held-out test set. Promotion to production requires the new version's score to meet or beat the current production version's score.

Together these make a prompt *promotable* — the same word used for code promotions — through environments (dev → staging → prod) with explicit gates at each transition.

### 3b. The prompt registry

The registry is the source of truth for "which prompt versions exist." Two implementation shapes scale at different stages:

**Flat-file (cohort default, W1–W3):**

```
prompts/
  draft-solicitation/
    v1.md         # initial release
    v2.md         # constraints update
    v2.1.md       # patch: clarify citation format
    METADATA.yaml # which version each environment points to
```

`METADATA.yaml` is the version pointer — changing a pointer (e.g., bumping prod from `v2` to `v2.1`) is the deploy. Git history is the audit trail. This shape is cheap to bootstrap, reviewable in PRs, and good enough for a six-engineer cohort building a handful of prompts.

**Hosted registry (production scale, beyond W5):** Tools like Braintrust, PromptLayer, Maxim, and LangSmith expose the same concept as a service — registry + UI + role-based access + per-environment pointer flips that don't require a code deploy. The Maxim 2026 guide documents semver branching, automatic eval runs on commit/merge, and rollout-percentage routing.

The cohort starts on the flat-file pattern. The migration path to a hosted registry is straightforward when needed; the discipline (versions, pointers, eval gates) survives the migration.

### 3c. The eval harness

An eval harness has five parts. Each is small; together they constitute a production-grade quality gate.

1. **Cases.** A held-out set of representative inputs with expected behaviour. Mix of happy-path, edge cases, adversarial inputs, regressions from past production incidents.
2. **Scoring.** Per-case scoring function. Can be deterministic (regex match, schema validity), LLM-as-judge (a separate model rates the output against a rubric), or human review (slow, expensive, the gold standard).
3. **Automation.** A script that runs cases × scoring × current-prompt-version and produces a score. Runs on every commit affecting prompts, on a schedule against production, and on-demand for ad-hoc experiments.
4. **Regression alerts.** When a prompt's eval score drops below a threshold or below the previous version's score, fail the CI run / alert the channel. Logiciel's 2026 internal-build guide names this as the load-bearing piece that distinguishes eval from "we ran some examples once."
5. **Operating cadence.** Weekly review of eval trends; monthly review of case-set coverage; quarterly review of the rubric itself. The eval harness is a system, not a one-off project.

### 3d. A/B and shadow rollouts

Promoting a new prompt to production is not a binary decision. The 2026 prompt-versioning literature documents three rollout strategies, each appropriate for different risk profiles:

- **Shadow mode.** Run the candidate prompt in parallel with the live prompt, return only the live prompt's output, but score both. Zero user impact; full eval signal. Use for high-risk changes where you cannot afford an in-production regression.
- **Canary.** Route a small percentage of traffic (1–5%) to the new prompt. Monitor real-world metrics (latency, cost, downstream conversion). Ramp up if metrics hold. Use for changes confident in shadow but with real-world variables (cost shape, latency) the shadow can't measure.
- **A/B test.** Split traffic 50/50 with random assignment. Measure differential impact on a business metric. Use for changes where the answer depends on user behaviour you cannot test offline.

Braintrust's recommendation: pick the rollout strategy at the same time you tag the version. The metadata of the prompt includes its planned rollout shape; the registry tooling enforces the rollout policy at deploy time.

### 3e. The eval pyramid

A useful frame from the broader software-testing world maps onto prompt evals:

```
              [ slow, expensive ]
                 human review
                /              \
              /                  \
            /     LLM-as-judge    \
          /                        \
        /   deterministic checks    \
      /______________________________\
              [ fast, cheap ]
```

Run the deterministic checks on every commit (schema validity, regex matches, simple invariants). Run LLM-as-judge nightly or on PR (semantic faithfulness, instruction adherence). Run human review weekly on a curated subset (the cases where automation fails or judgement is required). Each layer catches different failure modes; treating them as substitutes is a mistake.

## 4. Generic Implementation

A minimal eval harness fits in ~50 lines of Python. Worked example for a healthcare-summarisation prompt used by a medical-records vendor (no federal-acquisitions terminology):

```python
import json
import yaml
from pathlib import Path
from typing import Callable

# Each case: input + expected behaviour
EvalCase = dict     # {"id": ..., "input": ..., "checks": [...]}

def run_eval(
    prompt_text: str,
    cases: list[EvalCase],
    model_invoke: Callable[[str, str], str],
    scorers: dict[str, Callable[[str, EvalCase], float]],
) -> dict:
    scores = {name: [] for name in scorers}
    for case in cases:
        output = model_invoke(prompt_text, case["input"])
        for scorer_name, scorer_fn in scorers.items():
            scores[scorer_name].append(scorer_fn(output, case))
    return {
        name: sum(values) / len(values) for name, values in scores.items()
    }

# Deterministic scorer: does the output parse as the expected schema?
def schema_valid(output: str, case: EvalCase) -> float:
    try:
        parsed = json.loads(output)
        expected_keys = set(case["checks"]["required_keys"])
        return 1.0 if expected_keys.issubset(parsed.keys()) else 0.0
    except json.JSONDecodeError:
        return 0.0

# LLM-as-judge scorer: prompt a separate model to rate semantic accuracy
def semantic_faithful(output: str, case: EvalCase) -> float:
    judge_prompt = (
        f"Reference: {case['checks']['reference_text']}\n"
        f"Candidate: {output}\n"
        "On a 0-1 scale, is the candidate faithful to the reference? "
        "Return only a decimal number."
    )
    rating = judge_model_invoke(judge_prompt).strip()
    return float(rating)

# Wire it up
cases = yaml.safe_load(Path("evals/summariser_cases.yaml").read_text())
prompt = Path("prompts/summariser/v2.md").read_text()
scores = run_eval(
    prompt_text=prompt,
    cases=cases,
    model_invoke=bedrock_invoke,
    scorers={"schema_valid": schema_valid, "faithful": semantic_faithful},
)

# Gate: fail CI if any score drops below threshold
THRESHOLDS = {"schema_valid": 0.98, "faithful": 0.85}
for name, score in scores.items():
    if score < THRESHOLDS[name]:
        raise SystemExit(f"eval {name} dropped to {score:.3f} < {THRESHOLDS[name]}")
```

The whole apparatus fits in one file. The discipline is not the code — it's *running it on every prompt change* and *failing the merge when scores regress*.

## 5. Real-world Patterns

**E-commerce — search-relevance prompt iteration:** A retail platform iterates on a query-classification prompt that routes user queries between catalogue search and a customer-support fallback. The team holds a 2,000-case eval set drawn from production traffic, scored on deterministic intent-match. Every prompt PR runs the full eval; canary rollouts watch live click-through-rate. When a quarterly model update silently changed model behaviour, the eval harness caught the regression before the canary even rolled — exactly the early-warning signal the harness exists to provide.

**Fintech — fraud-explanation prompt:** A fintech compliance team generates plain-English explanations for flagged transactions. The eval rubric is compliance-rated by humans on a sample (10 cases / week) plus LLM-as-judge across 200 cases for semantic faithfulness. Prompt versions are tagged with the rubric version they were measured against — when the rubric tightens, all prompts are re-scored, and any that drop below threshold are flagged for revision.

**Gaming — narrative-generation prompts:** A live-service game uses LLMs to generate quest descriptions. Three prompt variants are A/B-tested with random traffic assignment; the eval metric is player-engagement (quest-acceptance rate) measured over a 72-hour window. The team treats engagement as the only metric that matters, but uses LLM-as-judge for style consistency as a guardrail — a quest description with high engagement and low style score gets surfaced for review.

**Logistics — routing-instruction prompt:** A last-mile delivery service generates driver-readable routing instructions from raw route data. The eval harness scores against a held-out set of human-written reference instructions (BLEU plus a semantic-overlap score). Shadow mode runs new prompts against live traffic for two weeks before any canary; the cost of a bad routing instruction (a missed delivery) is high enough that no in-production regression is acceptable.

## 6. Best Practices

- Version every prompt with semver — MAJOR for format/contract breaks, MINOR for behaviour changes, PATCH for typos.
- Keep the registry close to code in the cohort phase (flat files + git); migrate to a hosted registry only when the team or surface area demands it.
- Build the eval harness *before* you need it — adding it after the first regression is more painful than building it cheaply on day one.
- Mix scorer types — deterministic for cheap signal, LLM-as-judge for semantic, human for the load-bearing rubric calibration.
- Pick the rollout strategy at version-tag time, not at deploy time — the metadata travels with the version.
- Run evals on a schedule against production sampled traffic, not just on commits — model drift happens without prompt changes.
- Never bypass the eval gate "just this once" for a hotfix; the hotfix is the case the harness was built to catch.

## 7. Hands-on Exercise

**Task (10–15 min):** Build a minimal eval-harness skeleton for the W1 Thu summariser prompt:

1. Create `prompts/summariser/v1.md` with a placeholder system prompt.
2. Create `evals/summariser_cases.yaml` with 3 hand-written cases, each carrying `input`, `required_keys`, and `reference_text`.
3. Write a Python script that loads the prompt, iterates the cases, calls a stub `model_invoke` (returns canned output for now), runs two scorers (schema_valid and a stub faithful=1.0), and prints scores.
4. Wire it to fail with a non-zero exit code if `schema_valid < 0.9`.

**What good looks like:** The script can be re-run after a prompt edit and the score recomputes. The CI failure on a deliberately broken prompt (return non-JSON) is loud and names the offending scorer. The case file is small enough to read in one screen but structured enough that adding the 4th case is obvious. The directory shape matches the registry pattern from §3b — when you add `v2.md` later, the script still points at the right file via `METADATA.yaml`.

## 8. Key Takeaways

- *What's the difference between a prompt and a versioned prompt artifact?* The artifact has a name, a semver version, an input/output contract, and a published eval score.
- *Why a registry instead of inline prompt strings?* Promotion through environments, audit trail in git, ability to roll back without a code deploy.
- *What are the five parts of an eval harness?* Cases, scoring, automation, regression alerts, operating cadence — and the operating cadence is the one teams skip.
- *Which rollout strategy fits a high-risk prompt change?* Shadow mode first, then canary, then full A/B only if the change depends on real user behaviour.
- *What does "treat prompts as code" actually mean operationally?* Same review discipline, same versioning, same CI gates, same rollback playbook — applied to a markdown file instead of a Python file.

## Sources

1. [Evaluating LLM Outputs: Internal Eval Harness for 2026 (Logiciel)](https://logiciel.io/blog/llm-eval-harness-internal-build-2026) — retrieved 2026-05-26
2. [What is prompt versioning? Best practices for iteration without breaking production (Braintrust)](https://www.braintrust.dev/articles/what-is-prompt-versioning) — retrieved 2026-05-26
3. [Prompt Engineering Best Practices Testing & Versioning Framework for 2026 (KumoHQ)](https://www.kumohq.co/blog/prompt-engineering-best-practices) — retrieved 2026-05-26
4. [Top 5 Prompt Versioning Platforms in 2026 (Maxim)](https://www.getmaxim.ai/articles/top-5-prompt-versioning-platforms-in-2026/) — retrieved 2026-05-26
5. [Prompt Library Audit: 100-Point Evaluation Framework 2026 (Digital Applied)](https://www.digitalapplied.com/blog/prompt-library-audit-100-point-evaluation-2026) — retrieved 2026-05-26

Last verified: 2026-05-26
