---
week: W02
day: Fri
topic_slug: eval-as-test-fixture-and-regression-framing
topic_title: "Eval-as-test-fixture + regression framing"
parent_overview: W02/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://tianpan.co/blog/2026-05-01-eval-as-pull-request-comment-not-a-job
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://futureagi.com/blog/prompt-regression-testing-2026/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.braintrust.dev/articles/llm-evaluation-guide
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.github.com/en/actions/sharing-automations/reusing-workflows
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://dev.to/pockit_tools/llm-evaluation-and-testing-how-to-build-an-eval-pipeline-that-actually-catches-failures-before-5e3n
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# Eval-as-test-fixture + regression framing

## 1. Learning Objectives

- By the end of this reading, the learner can articulate the difference between eval-as-a-job (scheduled dashboard) and eval-as-test-fixture (PR-gated), and explain why the latter actually gates quality.
- By the end of this reading, the learner can describe a reusable GitHub Actions workflow pattern that runs an eval harness on every pull request and posts a per-PR delta as a comment.
- By the end of this reading, the learner can justify a regression threshold (the "block at >5%" decision) as an ADR-grade choice rather than a magic number, and name the two failure modes thresholds trade off.
- By the end of this reading, the learner can explain which dimensions should block merges and which should report-only, and why the answer is rarely "block on all metrics".

## 2. Introduction

Most teams' first attempt at LLM evaluation looks like a nightly job that writes scores to a dashboard. Nobody opens the dashboard. Regressions accumulate. By the time a customer ticket surfaces, the culprit PR is buried three weeks deep in `git log`. The pattern that has emerged in 2026 as the actual quality gate is different: the eval runs on the pull request, the result lands as a comment on the PR diff, the reviewer cannot scroll past it, and the merge button is disabled if the delta crosses a threshold.

This is the same shift code coverage went through a decade earlier. Coverage was unused until it became sticky PR comments scoped to the changed files. Evals are unused until they live where the change lives ([Eval as a Pull Request Comment, Not a Job — Tian Pan, May 2026](https://tianpan.co/blog/2026-05-01-eval-as-pull-request-comment-not-a-job), retrieved 2026-05-26).

The discipline this reading establishes — eval-as-test-fixture, regression framing, threshold rationale captured as an ADR — is the load-bearing piece. The harness from the previous readings is the data; the rubric is the score; this is the gate. Without the gate, the harness is just decoration.

## 3. Core Concepts

### 3.1 Eval-as-job vs. eval-as-test-fixture

Two patterns dominate, and they have very different effects on team behavior:

**Eval-as-a-job.** A scheduled CI job runs the eval set nightly or weekly. Results land in a dashboard, a database, or a Slack channel. Pros: easy to set up, runs against `main`, useful for long-term trend tracking. Cons: nobody looks at it, regressions are detected hours-to-days after they land, the responsible PR is often unclear, and the gate is administrative ("please go fix the dashboard") rather than technical (the merge button works).

**Eval-as-test-fixture.** The eval runs as part of the PR's CI pipeline, scoped to the changes in that PR. The result lands as a PR comment showing the delta vs. `main` for each scored dimension. A status check turns red if any blocking dimension regresses beyond a threshold. The merge button is disabled when the status check is red. The reviewer sees the delta before approving. Pros: regressions cannot ship; the gate is technical and unambiguous. Cons: requires more engineering to wire correctly; eval runtime adds to PR feedback latency.

The 2026 consensus is that the test-fixture pattern is what actually works ([Pockit — LLM Evaluation and Testing](https://dev.to/pockit_tools/llm-evaluation-and-testing-how-to-build-an-eval-pipeline-that-actually-catches-failures-before-5e3n), retrieved 2026-05-26). The dashboard still has a place — for trend tracking and historical analysis — but it is no longer the quality gate.

### 3.2 The PR comment

The comment shape that has settled into common practice:

```
## Eval delta vs. main

| Dimension          | main   | PR     | Delta  | Blocks merge?     |
| ------------------ | ------ | ------ | ------ | ----------------- |
| Faithfulness       | 0.847  | 0.792  | -6.5%  | Yes (blocking)    |
| Context recall     | 0.713  | 0.594  | -16.7% | Yes (blocking)    |
| Context precision  | 0.681  | 0.694  | +1.9%  | No (report only)  |
| Answer relevance   | 0.821  | 0.815  | -0.7%  | No (report only)  |

3 / 27 QA rows regressed. See attached results.jsonl for per-row details.
```

The discipline points:

- **Comment is re-edited on each push**, not appended. Comment stacking is noise; one comment per PR that reflects the current state of the branch.
- **Scoped to changed files** where possible — the metric delta is computed only against rows whose retrieval or generation paths touch the changed code. If the entire harness must re-run (the safe default for early-stage harnesses), the comment still notes how many rows actually changed.
- **Per-row drill-down available** as an attached artifact or expandable details block. The reviewer can investigate which rows regressed without leaving the PR.

### 3.3 The threshold

The "block at >5% regression on faithfulness or context recall" decision is the most-asked, most-misunderstood number in eval gating. It is not a magic constant; it is a justified trade-off between two failure modes:

- **False positive (too tight).** A 1% threshold blocks PRs that drifted within the harness's own noise floor. Engineers learn to ignore the block, override it, or worse — bias future curation toward rows where the system already scores well. The gate dies.
- **False negative (too loose).** A 10% threshold lets real quality regressions slip through with the rationalisation "it's within tolerance". The gate provides false confidence.

5% is the defensible default for early-stage harnesses (50–200 rows) where the per-row noise floor is around 2–3%. As the harness grows and the noise floor drops, the threshold tightens. As the QA set diversifies into adversarial rows where occasional regressions are expected, the threshold may widen for some slices.

The threshold itself is an ADR-grade decision. Commit the rationale alongside the code, with explicit triggers for revisiting ([Braintrust — LLM evaluation guide](https://www.braintrust.dev/articles/llm-evaluation-guide), retrieved 2026-05-26).

### 3.4 Block vs. report-only

Not every dimension blocks merges. The 2026 pattern that has held up:

- **Block:** faithfulness (catches hallucination — the highest-stakes failure), context recall (catches retrieval coverage — the most common silent regression).
- **Report-only:** context precision (often noisy on small QA sets — directional, useful for monitoring drift), answer relevance (often a prompt issue rather than a retrieval issue — fix it but don't block).

This split is itself an ADR — the rationale is "block on the dimensions that ship customer-visible failures, report on the dimensions that signal drift but might fire false positives." Different products will land different splits. A high-stakes legal or medical assistant might block on all four; a casual recommendation system might block on none.

### 3.5 Brownfield CI and partial closure

A common shape in real codebases: existing CI workflows are disabled (lint workflow turned off because it was broken; tests skipped because they take too long; coverage report dropped because nobody reads it). Adding eval-as-test-fixture is often the first piece of CI that actually runs in a long time. That makes it doubly important — and it creates a partial-closure pattern. The eval workflow runs; the lint workflow stays disabled; the team has now demonstrated that CI works for the high-stakes path and can graduate other workflows back online over time.

The discipline is to track the disabled workflows as work items (in some teams' tracking systems, as audit findings) so the partial closure does not become permanent ([GitHub Actions — Reusing workflows](https://docs.github.com/en/actions/sharing-automations/reusing-workflows), retrieved 2026-05-26).

## 4. Generic Implementation

A reusable GitHub Actions workflow that runs the eval harness, diffs against `main`, and posts a PR comment. The reusable-workflow pattern lets multiple repos call the same job without duplication:

```yaml
# .github/workflows/eval.yml
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

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install deps
        run: pip install -r requirements.txt

      - name: Run eval on PR branch
        run: python tests/eval/run_eval.py --output pr-results.jsonl
        env:
          JUDGE_API_KEY: ${{ secrets.JUDGE_API_KEY }}

      - name: Fetch baseline results from main
        run: |
          git fetch origin main
          git show origin/main:tests/eval/baseline-results.jsonl > main-results.jsonl

      - name: Compute delta and post PR comment
        run: python tests/eval/post_pr_comment.py \
              --pr pr-results.jsonl \
              --main main-results.jsonl \
              --threshold 0.05 \
              --blocking-dimensions faithfulness,context_recall
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

The `post_pr_comment.py` script (omitted for brevity) computes per-dimension delta, formats the table, posts via the GitHub API, and exits non-zero if any blocking dimension regresses beyond the threshold. Non-zero exit fails the status check and disables the merge button via branch-protection rules.

Notes:

- The `paths` filter scopes the workflow to relevant changes — a README-only PR does not re-run the eval.
- Baseline is the `main`-branch results checked in at the last successful merge — not regenerated on every PR (would double the eval cost).
- `secrets.JUDGE_API_KEY` is the judge model's credential; `secrets.GITHUB_TOKEN` is GitHub's auto-generated repo token for posting comments.

## 5. Real-world Patterns

**Logistics — package routing optimization.** A regional carrier shipped a re-router agent in late 2025 with eval-as-a-job (nightly run, dashboard in their internal ops portal). Within six weeks a regression to the address-disambiguation step had degraded retrieval accuracy by 12% across a specific zip-code region. Nobody noticed until a depot supervisor escalated. The post-mortem moved the eval to PR-gated; subsequent regressions of similar magnitude were caught at PR time ([Prompt Regression Testing: A Practical 2026 Guide](https://futureagi.com/blog/prompt-regression-testing-2026/), retrieved 2026-05-26).

**Gaming — NPC dialogue generation.** A studio building an LLM-driven NPC dialogue system runs a 400-row eval against every PR that touches the prompt templates or the retrieval corpus. The PR comment shows persona consistency, lore faithfulness, and tone-appropriateness scores. Block thresholds: 4% on persona consistency, report-only on tone (they accept some variance for variety). The harness has caught two LCEL-pattern regressions that an aggregate metric would have hidden ([Eval as a Pull Request Comment, Not a Job](https://tianpan.co/blog/2026-05-01-eval-as-pull-request-comment-not-a-job), retrieved 2026-05-26).

**Fintech — fraud-explanation generation.** A consumer-banking team uses eval-as-test-fixture for an agent that explains flagged transactions to customer-service reps. They block on faithfulness (cannot claim the customer did something they did not) and report on tone. The faithfulness threshold is tighter — 2% — because the failure mode is regulatory exposure, not customer satisfaction ([Braintrust — LLM evaluation guide](https://www.braintrust.dev/articles/llm-evaluation-guide), retrieved 2026-05-26).

## 6. Best Practices

- Run evals on PRs, not on schedules; the PR comment is the gate.
- Re-edit a single comment per PR; never append (comment stacking is noise).
- Pick block-vs-report dimensions deliberately and capture the choice as an ADR; not every dimension blocks merges.
- Justify the regression threshold against the harness's noise floor; 5% is a defensible default for small harnesses but is not magic.
- Use reusable GitHub Actions workflows so multiple repos share one eval pipeline.
- Track disabled CI workflows as work items so partial closure does not become permanent.
- Drill-down details (per-row results) must be one click away — reviewers need to investigate, not just see the headline.

## 7. Hands-on Exercise

**Architecture-drawing prompt (15 min):** Sketch a CI pipeline diagram for a hypothetical Q&A system over a corporate policy corpus. Show:

1. Which workflow triggers on PRs (and which trigger paths filter).
2. Where the baseline results live (and how they get updated on merge).
3. Which dimensions block merges and which report-only — justify each choice.
4. What the PR comment looks like (sketch the table).
5. Where the threshold rationale lives in the repo (as a file or as a comment) and when it would be revisited.

**What good looks like:** Your diagram has a clear single source of truth for baseline results (the `main`-branch artifact, not re-generated). Your block/report split is justified by stakes — high-stakes dimensions block, low-noise-floor dimensions report. Your PR-comment table shows per-dimension delta with explicit pass/fail markers. Your threshold rationale lives in a repo-tracked decision record (an ADR file or equivalent) and names a trigger condition for revisiting (e.g., "revisit when QA set exceeds 200 rows or noise floor drops below 1%").

## 8. Key Takeaways

- What distinguishes eval-as-a-job from eval-as-test-fixture, and why does only the latter actually gate quality?
- What does a useful PR-comment shape look like, and what discipline keeps it useful over many PRs?
- How is a regression threshold justified as an ADR rather than picked as a magic number, and what two failure modes does it trade off?
- Which dimensions should block merges, which should report-only, and what determines the split for a given product?

## Sources

1. [Eval as a Pull Request Comment, Not a Job — Tian Pan (May 2026)](https://tianpan.co/blog/2026-05-01-eval-as-pull-request-comment-not-a-job) — retrieved 2026-05-26
2. [Prompt Regression Testing: A Practical 2026 Guide — FutureAGI](https://futureagi.com/blog/prompt-regression-testing-2026/) — retrieved 2026-05-26
3. [LLM Evaluation Guide — Braintrust](https://www.braintrust.dev/articles/llm-evaluation-guide) — retrieved 2026-05-26
4. [GitHub Actions — Reusing workflows](https://docs.github.com/en/actions/sharing-automations/reusing-workflows) — retrieved 2026-05-26
5. [LLM Evaluation and Testing: How to Build an Eval Pipeline That Actually Catches Failures Before Production — Pockit/DEV](https://dev.to/pockit_tools/llm-evaluation-and-testing-how-to-build-an-eval-pipeline-that-actually-catches-failures-before-5e3n) — retrieved 2026-05-26

Last verified: 2026-05-26
