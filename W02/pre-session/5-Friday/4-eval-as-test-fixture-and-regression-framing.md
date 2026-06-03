---
week: W02
day: Fri
topic_slug: eval-as-test-fixture-and-regression-framing
topic_title: "Eval-as-test-fixture + regression framing"
parent_overview: W02/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://tianpan.co/blog/2026-05-01-eval-as-pull-request-comment-not-a-job
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://futureagi.com/blog/prompt-regression-testing-2026/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.braintrust.dev/articles/llm-evaluation-guide
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://docs.github.com/en/actions/sharing-automations/reusing-workflows
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://dev.to/pockit_tools/llm-evaluation-and-testing-how-to-build-an-eval-pipeline-that-actually-catches-failures-before-5e3n
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-06-03
---

# Eval-as-test-fixture + regression framing

> [!NOTE]
> **From topic 3:** the judge emits per-dimension mean + stddev for each row. This topic is the gate — where that score lands (PR comment), what blocks merge (5% on F or CR), and how the wiring stays honest.

## 1. Learning Objectives

- Wire a GHA workflow that runs eval on every PR touching `src/`, `prompts/`, or `tests/eval/`.
- Choose blocking vs report-only dimensions and defend the choice in an ADR.
- Defend a 5% regression threshold against tighter and looser alternatives.
- Open an OIG-style finding against `acquire-gov` for the remaining disabled GHA workflows (Item 12 partial close).

## 2. Introduction

A harness that runs nightly and writes to a dashboard is administrative; a harness that runs on every PR and blocks merges is technical. The shift is the same one code coverage went through a decade earlier — dashboards get ignored; PR comments don't. PR #47 is the test case: nightly-job eval would have shipped the merge, customers would have hit the regression, postmortem would have buried the culprit three weeks deep. Eval-as-PR-fixture inverts that. The reviewer cannot scroll past the delta. The merge button is mechanically disabled. The gate is **technical, not administrative**.

## 3. Core Concepts

### 3.1 The GHA workflow

```yaml
# .github/workflows/rag-eval.yml — first CI workflow to actually run in acquire-gov
name: RAG eval

on:
  pull_request:
    paths:
      - 'src/**'
      - 'prompts/**'
      - 'tests/eval/**'

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - run: pip install -r requirements.txt
      - name: Run eval on PR branch
        run: python tests/eval/run_eval.py --output pr-results.jsonl
        env: { JUDGE_API_KEY: ${{ secrets.JUDGE_API_KEY }} }
      - name: Fetch baseline from main
        run: |
          git fetch origin main
          git show origin/main:tests/eval/baseline-results.jsonl > main-results.jsonl
      - name: Compute delta + post PR comment
        run: python tests/eval/post_pr_comment.py \
              --pr pr-results.jsonl \
              --main main-results.jsonl \
              --threshold 0.05 \
              --blocking-dimensions faithfulness,context_recall
        env: { GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
```

`post_pr_comment.py` exits non-zero on any blocking-dimension regression > threshold. Non-zero exit fails the status check → branch-protection rule disables the merge button.

### 3.2 Regression thresholds and the block/report split

| Dimension | Baseline (PR #47 example) | PR | Δ | Block if Δ |
|---|---:|---:|---:|---|
| **Faithfulness** | 0.847 | 0.792 | **-6.5%** | **>5% (blocking)** |
| **Context recall** | 0.713 | 0.594 | **-16.7%** | **>5% (blocking)** |
| Context precision | 0.681 | 0.694 | +1.9% | report-only |
| Answer relevance | 0.821 | 0.815 | -0.7% | report-only |

PR-comment shape: one comment per PR, re-edited on each push (never appended), per-dimension delta table, drill-down to `results.jsonl` one click away. Block on dimensions that ship customer-visible failures (faithfulness = hallucination, context recall = retrieval coverage). Report on dimensions that signal drift but fire false positives on small QA sets (context precision = noisy, answer relevance = often a prompt fix).

### 3.3 Why 5% — and why not 1% or 10%

| Threshold | Trade-off |
|---|---|
| **1% (too tight)** | Blocks PRs that drifted within the harness's noise floor (~2-3% per-row on a 20-30 row set). Engineers learn to ignore the block. Gate dies. |
| **5% (defensible default)** | Just above the per-row noise floor; catches real regressions; tightens as QA set grows past 50 and noise drops. |
| **10% (too loose)** | Real quality regressions slip with "it's within tolerance" rationalisation. Gate provides false confidence. |

5% is the defensible default for early-stage harnesses (50-200 rows). Tighten as the harness grows; widen for slices where adversarial regressions are expected. **Commit the rationale as an ADR with explicit revisit triggers** ("revisit when QA set exceeds 200 rows OR noise floor drops below 1%").

> [!IMPORTANT]
> **Block vs report split is itself an ADR.** A high-stakes legal/medical assistant might block all four; a casual recommender might block none; federal-acq blocks F + CR. **Commit the rationale alongside the code Friday EOD** — W3 Mon §0 retro consumes it. The dashboard still has a place for trend tracking; it is *not* the quality gate.

## 4. Generic Implementation

```python
# tests/eval/post_pr_comment.py — block + report delta against main
# Lives in acquire-gov at tests/eval/post_pr_comment.py
import json
import sys
import os
import requests
from collections import defaultdict

BLOCKING = set(os.environ.get("BLOCKING_DIMENSIONS",
                              "faithfulness,context_recall").split(","))
THRESHOLD = float(os.environ.get("THRESHOLD", "0.05"))

def aggregate(path):
    sums, counts = defaultdict(float), defaultdict(int)
    with open(path) as f:
        for line in f:
            row = json.loads(line)
            for dim, score in row["scores"].items():
                sums[dim]   += score
                counts[dim] += 1
    return {dim: sums[dim] / counts[dim] for dim in sums}

def main(pr_path, main_path, pr_number):
    pr   = aggregate(pr_path)
    base = aggregate(main_path)
    rows, fail = [], False
    for dim in sorted(set(pr) | set(base)):
        delta = pr[dim] - base[dim]
        rel   = delta / base[dim] if base[dim] else 0
        block = dim in BLOCKING and rel < -THRESHOLD
        rows.append((dim, base[dim], pr[dim], rel, block))
        fail = fail or block
    post_comment_to_pr(pr_number, render(rows))
    sys.exit(1 if fail else 0)
```

Non-zero exit fails the status check → branch-protection disables the merge button. PR-comment edits in-place on every push so the most recent delta is always at the top.

## 5. Real-world Patterns

**Logistics — package routing.** A carrier shipped a re-router with nightly-job eval; six weeks later a 12% regression in address-disambiguation for a specific zip-code region was caught by a depot supervisor, not the dashboard. Post-mortem moved eval to PR-gated; subsequent regressions caught at PR time. The depot supervisor is the customer who eventually opens the ticket — by the time they see it, the culprit PR is buried three weeks deep.

**Gaming — NPC dialogue.** A studio runs 400-row eval on every PR touching prompt templates or retrieval corpus. PR comment shows persona consistency, lore faithfulness, tone-appropriateness. Block on persona 4%, report-only on tone. Harness caught two LCEL-pattern regressions an aggregate metric would have hidden — both were dimension-specific patterns where one dimension drifted while the aggregate stayed flat.

**Fintech — fraud-explanation.** Blocks on faithfulness (cannot claim the customer did something they did not) at a tighter 2% threshold because failure mode is regulatory exposure. Report-only on tone. The threshold tightness mirrors the cost-asymmetry framework from Thursday — regulated environments tune blocking dimensions tight; consumer-facing UX tunes them looser with sampling.

**SaaS — support assistant.** A B2B SaaS support team adopted PR-gated eval and within two months saw "did we break anything" hallway conversations replaced by PR-comment scrolling. Behavioural shift: reviewers started reading the eval delta *before* the code diff because the delta surfaces customer impact while the diff surfaces implementation detail.

## 6. Best Practices

- **PR comment, not dashboard.** The reviewer cannot scroll past the delta; the merge button mechanically disables.
- **Edit the PR comment in place on each push** — never append. The most recent delta is always at the top.
- **Block on customer-visible failure modes** (faithfulness, recall in most domains); report on noise-prone dimensions.
- **5% threshold defensible default** for 50-200 row harnesses; tighten as QA set grows and noise drops.
- **Commit the block/report split as an ADR** with explicit revisit triggers.
- **Branch protection enforces it**, not goodwill. Goodwill regresses; CI rules don't.
- **Use the dashboard for trends, not the gate.** Trend-tracking is a different question than per-PR regression.

> [!WARNING]
> **Anti-pattern: eval-as-a-job (nightly dashboard) treated as a quality gate.** Most internet eval tutorials demo a scheduled job that writes scores to a dashboard. Per `eval-dashboard-not-pr`: nobody opens the dashboard, regressions are detected hours-to-days after they land, the culprit PR is buried, and the "gate" is administrative ("please fix the dashboard") rather than technical (merge button still works). PR-gated eval is the 2026 consensus pattern. Dashboard still has a place for trend tracking — it is *not* the quality gate.

## 7. Hands-on Exercise

Wire the GHA workflow against PR #47 in your fork before war-room: (a) copy the YAML from §3.1; (b) commit `tests/eval/baseline-results.jsonl` from main first so the diff has a baseline; (c) trigger the workflow on PR #47 and verify the status check goes red on faithfulness or context recall; (d) write the 2-3 sentence ADR rationale for "why we block on F+CR but report on CP+AR." War-room block C wires branch protection and opens the Item 12 OIG finding for the remaining disabled workflows in `acquire-gov`.

> [!NOTE]
> **Self-check** (30s)
>
> 1. Why is "eval as a nightly job with a dashboard" the wrong pattern, and what does eval-as-test-fixture actually fix?
> 2. Why does report-only ≠ "we don't care" for context precision and answer relevance?

<details>
<summary>Show answers</summary>

1. Nobody opens the dashboard. By the time a customer ticket surfaces, the culprit PR is buried three weeks deep. The "gate" is administrative ("please fix the dashboard") rather than technical (merge button still works). Eval-as-test-fixture fixes it by running on the PR, posting the delta as a PR comment, and turning the status check red on blocking-dimension regressions — the reviewer cannot scroll past the delta, the merge button is mechanically disabled. Same shift code coverage went through a decade earlier.
2. They're the early-warning system for cross-dimension drift (topic 5). A PR that improves faithfulness 8% but silently degrades answer relevance 4% is not a win — it's a shape-shift, and reporting non-blocking dimensions makes the shape visible. They don't block because their noise floor is higher on small QA sets, but they get read at PR review time, and a clear drift across many PRs is itself a signal to revisit the gate split.

</details>

## 8. Key Takeaways

- PR comment + branch-protected status check, not a dashboard.
- Block on customer-visible failure dimensions (F + CR for federal-acq); report on noise-prone dimensions (CP + AR).
- 5% threshold defensible for 50-200 row harnesses; tighter as the harness grows.
- The block/report split is an ADR with revisit triggers — commit it.
- Item 12 partial close — `rag-eval.yml` is the first CI workflow to actually run in `acquire-gov`; the rest remain debt the cohort tracks as OIG-style findings.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- <https://tianpan.co/blog/2026-05-01-eval-as-pull-request-comment-not-a-job> — retrieved 2026-05-26 — hot-tech-3mo
- <https://futureagi.com/blog/prompt-regression-testing-2026/> — retrieved 2026-05-26
- <https://www.braintrust.dev/articles/llm-evaluation-guide> — retrieved 2026-05-26
- <https://docs.github.com/en/actions/sharing-automations/reusing-workflows> — retrieved 2026-05-26 — foundation-stable
- <https://dev.to/pockit_tools/llm-evaluation-and-testing-how-to-build-an-eval-pipeline-that-actually-catches-failures-before-5e3n> — retrieved 2026-05-26

</details>

<details>
<summary>Deeper dive — Item 12 partial close + cross-industry threshold tightness</summary>

**Brownfield + Item 12 partial close.** `.github/workflows/lint.yml` ships with `disabled: true` in `acquire-gov` — that's debt Item 12. Today's `rag-eval.yml` is the **first CI workflow to actually run in the repo**. This is a *partial* close (rag-eval scope only); broader GHA lint stays disabled until W4 modernization. Cohort opens an OIG-style finding via `POST /api/findings` against the repo for the remaining debt — first instance of the platform managing its own technical debt as findings. Same affordance the platform offers external OIG auditors, used internally.

**OIG-style finding meta-loop:** the platform's own technical debt being tracked as a finding it produces is the meta-pattern. Auditors who see a system using its own audit affordance against itself are more likely to trust the affordance against external corpora. The cohort experiences this discipline from both sides in Phase 2.

**Cross-industry threshold tightness:**

- Healthcare clinical decision support — block all four dimensions at 3%. Failure mode is patient harm; cost asymmetry maximally favours false-positives.
- Fintech consumer chat — block F at 2%, CR at 5%, report CP + AR. Failure mode is regulatory exposure on F; CR matters but less acutely.
- Legal-tech contract Q&A — block F + CR at 3%, report CP + AR. Per the legal-tech 2026 evaluation, 90% F and 90% CP are production targets; below 70% on either blocks entirely.
- Casual recommendation systems — report all four, no blocking. The dashboard *is* the right answer here because the failure mode is "user sees a slightly worse recommendation," not customer harm.

**The PR-comment vs status-check pairing:** the status check provides the technical block; the PR comment provides the human-readable explanation. Both are necessary. A status check alone forces engineers to dig into CI logs; a PR comment alone is ignorable. Together they shift the conversation upstream — the reviewer reads the delta first, then the diff.

</details>

Last verified: 2026-06-03
