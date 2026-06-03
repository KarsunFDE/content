---
week: W02
day: Fri
topic_slug: eval-as-test-fixture-and-regression-framing
topic_title: "Eval-as-test-fixture + regression framing"
parent_overview: W02/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 6
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

## The GHA workflow stub

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

## Regression thresholds

| Dimension | Baseline (PR #47 example) | PR | Δ | Block if Δ |
|---|---:|---:|---:|---|
| **Faithfulness** | 0.847 | 0.792 | **-6.5%** | **>5% (blocking)** |
| **Context recall** | 0.713 | 0.594 | **-16.7%** | **>5% (blocking)** |
| Context precision | 0.681 | 0.694 | +1.9% | report-only |
| Answer relevance | 0.821 | 0.815 | -0.7% | report-only |

PR-comment shape: one comment per PR, re-edited on each push (never appended), per-dimension delta table, drill-down to `results.jsonl` one click away.

> [!IMPORTANT]
> **Block vs report split is itself an ADR.** Block on the dimensions that ship customer-visible failures (faithfulness = hallucination, context recall = retrieval coverage). Report on dimensions that signal drift but fire false positives on small QA sets (context precision = noisy, answer relevance = often a prompt fix). A high-stakes legal/medical assistant might block all four; a casual recommender might block none. **Commit the rationale alongside the code Friday EOD** — W3 Mon §0 retro consumes it.

> [!WARNING]
> **Anti-pattern: eval-as-a-job (nightly dashboard) treated as a quality gate.** Most internet eval tutorials demo a scheduled job that writes scores to a dashboard. Per the `eval-dashboard-not-pr` pattern: nobody opens the dashboard, regressions are detected hours-to-days after they land, the culprit PR is buried, and the "gate" is administrative ("please fix the dashboard") rather than technical (merge button still works). PR-gated eval is the 2026 consensus pattern. Dashboard still has a place for trend tracking — it is *not* the quality gate.

## Why 5% — and why not 1% or 10%

| Threshold | Trade-off |
|---|---|
| **1% (too tight)** | Blocks PRs that drifted within the harness's noise floor (~2-3% per-row on a 20-30 row set). Engineers learn to ignore the block. Gate dies. |
| **5% (defensible default)** | Just above the per-row noise floor; catches real regressions; tightens as QA set grows past 50 and noise drops. |
| **10% (too loose)** | Real quality regressions slip with "it's within tolerance" rationalisation. Gate provides false confidence. |

5% is the defensible default for early-stage harnesses (50-200 rows). Tighten as the harness grows; widen for slices where adversarial regressions are expected. **Commit the rationale as an ADR with explicit revisit triggers** ("revisit when QA set exceeds 200 rows OR noise floor drops below 1%").

## Brownfield + Item 12 partial close

`.github/workflows/lint.yml` ships with `disabled: true` in acquire-gov — that's debt Item 12. Today's `rag-eval.yml` is the **first CI workflow to actually run in the repo**. This is a *partial* close (rag-eval scope only); broader GHA lint stays disabled until W4 modernization. Cohort opens an OIG-style finding via `POST /api/findings` against the repo for the remaining debt — first instance of the platform managing its own technical debt as findings. Same affordance the platform offers external OIG auditors, used internally.

## Self-check

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

<details>
<summary>Cross-industry — three patterns the gate caught</summary>

- **Logistics — package routing.** A carrier shipped a re-router with nightly-job eval; six weeks later a 12% regression in address-disambiguation for a specific zip-code region was caught by a depot supervisor, not the dashboard. Post-mortem moved eval to PR-gated; subsequent regressions caught at PR time.
- **Gaming — NPC dialogue.** Studio runs 400-row eval on every PR touching prompt templates or retrieval corpus. PR comment shows persona consistency, lore faithfulness, tone-appropriateness. Block on persona 4%, report-only on tone. Harness caught two LCEL-pattern regressions an aggregate metric would have hidden.
- **Fintech — fraud-explanation.** Block on faithfulness (cannot claim the customer did something they did not) at tighter 2% threshold because failure mode is regulatory exposure. Report-only on tone.

</details>

<details>
<summary>Sources (retrieved via /web-research per D-046)</summary>

1. Tian Pan — Eval as PR comment not a job: <https://tianpan.co/blog/2026-05-01-eval-as-pull-request-comment-not-a-job> — 2026-05-26
2. FutureAGI — Prompt Regression Testing 2026: <https://futureagi.com/blog/prompt-regression-testing-2026/> — 2026-05-26
3. Braintrust — LLM evaluation guide: <https://www.braintrust.dev/articles/llm-evaluation-guide> — 2026-05-26
4. GitHub Actions — Reusing workflows: <https://docs.github.com/en/actions/sharing-automations/reusing-workflows> — 2026-05-26
5. Pockit — LLM Evaluation and Testing: <https://dev.to/pockit_tools/llm-evaluation-and-testing-how-to-build-an-eval-pipeline-that-actually-catches-failures-before-5e3n> — 2026-05-26

</details>

Last verified: 2026-06-03
