---
week: W04
day: Tue
topic_slug: validation-as-spec-discipline
topic_title: "Validation-as-spec Discipline — making sure CI checks what you think"
parent_overview: W04/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://martinfowler.com/bliki/SelfTestingCode.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/articles/continuousIntegration.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.github.com/en/actions/using-jobs/using-conditions-to-control-job-execution
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/articles/patterns-legacy-displacement/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Validation-as-spec Discipline — making sure CI checks what you think

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define *validation-as-spec* and distinguish it from "we have tests" or "we have a spec."
- Identify three categories of silent CI failure where a spec ostensibly enforced is in fact unenforced.
- Recognize the `if: false` family of conditional skips in GitHub Actions as one common silent-failure pattern.
- Choose between end-to-end, contract, and domain-surface validation layers depending on what the spec claims.
- Run a self-audit of a CI pipeline to confirm that the checks named in the spec actually fire.

## 2. Introduction

A specification says what a system should do. Validation makes the spec enforceable. The discipline of *validation-as-spec* says these two artifacts must stay in sync: every claim the spec makes is paired with an executable check, and every executable check traces back to a spec claim. When they drift apart, the team loses the ability to tell whether the system is conformant.

A recurring class of brownfield discovery is "we thought CI was checking X for the last six months; it wasn't." The lint job was disabled during an emergency and never re-enabled. The integration tests were stubbed out during a flaky-test cleanup. The security scanner was added but configured to "warn" instead of "fail." A spec that *appears* enforced is not actually enforced, and nobody noticed because the green checkmark looks the way it always looks.

Martin Fowler's *Self-Testing Code* (2014) describes "self-testing code" as code where running one command executes the tests and a green light means the code is free of substantial defects. The validation-as-spec discipline extends that idea to *every* check in the pipeline. *Continuous Integration* (Fowler, 2024 update) goes further: a pipeline that silently skips checks isn't really self-testing. Brownfield modernization stresses this discipline hardest — a modernization touches every layer at once, and pre-existing validation gaps let new defects slip through alongside the legitimate changes.

## 3. Core Concepts

### 3.1 The three failure modes of silent CI

A CI pipeline can be silently broken in three structurally different ways. Each requires a different audit technique to detect.

**Skipped jobs.** A job is conditionally not run. GitHub Actions has a documented pattern (`docs.github.com`, retrieved 2026-05-26): a job with `if: <condition>` evaluating to false is *skipped*, and "a job that is skipped will report its status as 'Success'. It will not prevent a pull request from merging, even if it is a required check." This is by design — but the design becomes a trap when `if: false` or `if: github.repository == 'wrong-name'` makes a check silently not run. The check appears in the workflow file; the team thinks it's enforced; CI reports success because skipped = success.

**Inert thresholds.** A check runs but its failure threshold is set so high it never triggers. A coverage gate at 0% always passes. A security scanner configured to "report only, do not fail the build" finds problems and surfaces them in a dashboard nobody reads. A lint configured to warn rather than fail will color the log yellow forever without anyone noticing.

**Stubbed bodies.** A test job runs and reports success, but the test body has been replaced with a no-op. The most common form: an integration test suite where every test method has been replaced with `assertTrue(true)` or `@Ignore`'d during a flaky-test cleanup that never came back.

All three failure modes have the same symptom from outside: the green checkmark. The audit technique must look *inside* the workflow definitions, the threshold configurations, and the test bodies — green checks alone tell you nothing.

### 3.2 What "validation-as-spec" actually means

The phrase has a precise meaning: every assertion in the spec must trace to an executable check, and every executable check must trace back to an assertion in the spec. The relation is bidirectional.

The bidirectionality matters. *Spec → check* prevents the spec from making unenforceable promises ("the system shall not log PII" with no PII-detection check). *Check → spec* prevents checks from accumulating without anchors ("we have a coverage gate at 80% because we always had one" — but the spec never asked for coverage, and now nobody knows what 80% buys).

The discipline produces a *coverage matrix*: a table with spec assertions on one axis and CI checks on the other. Cells contain either "check X enforces this" or "no enforcement." A row with no enforcement is an unenforced assertion; the team either adds a check or removes the assertion. A column with no spec linkage is an orphaned check; the team either anchors it to a spec or removes it.

This is not the same thing as having tests. A team with 5,000 tests and a 95% line-coverage gate can still have a coverage matrix full of holes if the spec says things the tests don't check (e.g., "no PII in audit logs" — covered by zero tests). The line-coverage number is a proxy that doesn't tell you whether the *spec's claims* are validated.

### 3.3 Three layers of validation

Specifications make claims at different granularities, and validation must match the claim's layer:

- **End-to-end validation** — does the system, end to end, do what the spec says? Integration tests, acceptance tests, manual exploratory checks. Catches behavior that emerges from the composition of multiple services.
- **Contract validation** — do the service interfaces agree with the spec? Consumer-driven contract tests, OpenAPI schema validation, schema-registry compatibility checks for events. Catches drift between services without requiring all services to be running.
- **Domain-surface validation** — do the specific entry points enforce what the spec says about them? A spec claim like "every solicitation amendment cites a FAR clause" is enforceable at the `POST /amendments` endpoint via input validation. This layer catches violations at the entry point, not deep in the system.

A discipline failure is when the spec makes a claim at one layer and the team writes the validation at a different layer. "Every audit log must be tamper-evident" is an end-to-end property — verifying it requires running the audit-log writer, persisting an event, and confirming the integrity check fires. Writing a unit test for the hash function does not validate the claim; it validates a building block.

### 3.4 The audit technique — open three PRs and look

The cheapest, most informative validation-as-spec audit is to open three recent merged PRs and inspect the CI evidence for each:

1. Open the PR's GitHub Actions run (or equivalent).
2. List the jobs that ran and the jobs that were skipped.
3. For each "successful" job, click into the logs and confirm the *test body* actually executed (count of assertions, count of files linted, count of vulnerabilities scanned).
4. Cross-reference each job to a spec assertion. If you cannot find one, the job is orphaned. If a spec assertion has no job, it is unenforced.

Three PRs is enough to spot a systemic gap; the same skipped job appearing in all three runs is decisive evidence. The audit takes 30 minutes and produces a list of unenforced assertions and orphaned checks the team can address in the next iteration.

### 3.5 The relationship to spec-driven development

Validation-as-spec is the operational counterpart of spec-driven development. Spec-driven dev says "write the spec first." Validation-as-spec says "write the check that proves the spec." Without it, spec-driven dev reduces to documentation theater. Cartwright, Horn, and Lewis's *Patterns of Legacy Displacement* (2024) names the failure mode: patches accumulate "with little investment and care to keeping them healthy" until wholesale modernization is the only option. A pipeline whose validation is silently broken accelerates that slide because every patch lands without surfacing the drift.

## 4. Generic Implementation

A worked example: a generic team auditing their CI pipeline for a `notifications-service` whose spec claims (a) all PRs lint clean, (b) test coverage stays above 80%, and (c) no PII appears in log output. The team's audit technique applied to one recent merged PR.

**Step 1 — Open the workflow file.**

```yaml
# .github/workflows/ci.yml
name: ci
on: [pull_request]
jobs:
  lint:
    if: false                           # <-- silently disabled
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm install
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm install
      - run: npm test                   # runs but coverage threshold...

  pii-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./scripts/scan-pii.sh || true   # <-- never fails the job
```

Three failure modes are visible:

- **Skipped job** — `lint` has `if: false`. Per the GitHub Actions docs (retrieved 2026-05-26), a skipped job reports success, so a merge-required-check rule lets the PR through.
- **Inert threshold** — `test` runs but the coverage threshold is not configured anywhere. Tests passing tells you only that the test runner exited zero, not that coverage met the spec's 80% claim.
- **Stubbed body** — `pii-scan` has `|| true`, which makes the shell exit 0 regardless of the scanner's findings. The job is "successful" even when PII is found.

**Step 2 — Reconcile against the spec.**

| Spec assertion | Job that claims to enforce | Actually enforced? |
|---|---|---|
| All PRs lint clean | `lint` | No — `if: false` |
| Test coverage > 80% | `test` | No — threshold not configured |
| No PII in logs | `pii-scan` | No — `\|\| true` swallows failure |

Three out of three spec assertions are unenforced. CI is green on every PR.

**Step 3 — Fix the workflow.**

```yaml
name: ci
on: [pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest              # removed if: false
    steps:
      - uses: actions/checkout@v4
      - run: npm install
      - run: npm run lint               # exits non-zero on lint error

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm install
      - run: npm test -- --coverage --coverageThreshold='{"global":{"lines":80}}'

  pii-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./scripts/scan-pii.sh      # removed `|| true`
```

The same job names, the same step structure — but each job now actually enforces its assertion. The spec and the validation are aligned. Three PRs later, the team can re-audit and confirm.

**Step 4 — Document the linkage.**

Add a `spec/validation-matrix.md` file (or extend the existing spec) with the bidirectional mapping. The matrix becomes a review artifact: every spec change requires a row update; every workflow change requires a column update.

## 5. Real-world Patterns

**Aerospace supply-chain platform — five-year-old disabled check.** An aerospace parts supplier discovered their Helm-chart lint had been set `if: false` during a 2020 outage and never re-enabled. Five years of Helm changes had merged without lint; a retroactive run surfaced 73 issues across 14 charts. The post-mortem named no individual as at fault — the spec never told anyone the lint *should* fire, so nobody was watching.

**Mobile banking app — coverage gate masked deteriorating quality.** A retail bank had an 80% coverage gate that always passed. An audit found coverage was computed against the *whole* codebase, but new modules were excluded from measurement. Three years of new modules added with zero tests had not moved the global percentage. The fix was to scope coverage to *changed files in the PR*, not the global codebase.

**Logistics SaaS — PII scanner found in "report only" mode after a leak.** A logistics SaaS discovered through a customer complaint that a microservice was logging recipient phone numbers. Their PII scanner wrote findings to a dashboard but did not fail builds; nobody had been assigned to read the dashboard. The remediation was twofold: fail builds on critical findings, and add a weekly dashboard-review ritual.

**Healthcare EHR vendor — orphaned contract caused production breakage.** An EHR vendor had been running consumer-driven contract tests for years. One 2024 partner submitted a contract with a typo'd partner ID; the contract sat in the repo unreferenced by CI. The integration shipped; a year later, schema drift caused cascading failures. The fix was a CI script that fails the build if any contract file is unreferenced — making "every contract is enforced" itself an enforced spec assertion.

## 6. Best Practices

- **Run a validation-as-spec audit at the start of every modernization** — modernizations are too expensive to run on top of silent CI; surface gaps before they compound.
- **Make "skipped = success" visible in dashboards** — distinguish "passed" from "skipped" in your PR-status UI; teams that only look for red checkboxes will never see skipped jobs.
- **Pair every spec assertion with a CI evidence link** — when a spec says X, the spec should link to the job or workflow file that enforces X.
- **Reject PRs that disable a check without an ADR** — a check being disabled is a decision; decisions get ADRs; ADRs get reviewed. Casual `if: false` should be impossible.
- **Audit thresholds quarterly** — coverage gates, severity gates, and rate-limits drift. A quarterly re-read keeps them anchored.
- **Treat orphaned checks as failures too** — a job nobody can map to a spec is a job with no clear failure mode; either anchor it or delete it.
- **Run the audit yourself on your own pipeline this week** — the technique is cheap; the cost of skipping it is high.

## 7. Hands-on Exercise

**Exercise: Audit one PR for validation-as-spec gaps (20 min).**

Pick any recently merged PR in any project (work, open source, a class project). Without consulting the author:

1. List the spec assertions the project makes (look at README, CONTRIBUTING, any docs that say "we require X").
2. Open the PR's CI run.
3. List the jobs that ran and the jobs that were skipped.
4. For each job, open the logs and identify whether the body actually exercised what its name suggests (lint job actually linted, test job actually tested, security job actually scanned).
5. Map jobs to spec assertions in a two-column table.
6. Identify at least one unenforced assertion (row with no job) and at least one orphaned job (column with no assertion).

**What good looks like:** the audit produces a one-page artifact (mapping table + summary). The summary names at least one validation gap. If you cannot find any gaps, either the project's pipeline is unusually clean or — more likely — you have not read deeply enough into the job logs. Stubbed bodies hide well; look for short job durations, suspiciously round assertion counts, and silent dependencies on environment variables that may not be set in PR runs.

A common surprise the first time: the project's README claims to enforce something the CI does not check at all. That's the discovery the exercise is for.

## 8. Key Takeaways

- *What does validation-as-spec mean?* — Every spec assertion has an enforced check, and every check anchors to a spec assertion. Bidirectional.
- *What are the three silent-failure modes?* — Skipped jobs (where skip = success), inert thresholds (gates that never fire), and stubbed bodies (tests that don't test).
- *Why is GitHub Actions' "skipped = success" a trap?* — Because the green checkmark looks identical to a job that actually ran and passed; only opening the workflow file reveals the skip.
- *Which validation layer matches which spec claim?* — End-to-end for emergent behavior, contract for service interfaces, domain-surface for entry-point claims.
- *How do you actually audit?* — Open three recent merged PRs, list ran vs skipped vs stubbed jobs, map each to the spec, and look for unenforced rows and orphaned columns.

## Sources

1. [Self-Testing Code](https://martinfowler.com/bliki/SelfTestingCode.html) — retrieved 2026-05-26
2. [Continuous Integration (Martin Fowler, 2024 update)](https://martinfowler.com/articles/continuousIntegration.html) — retrieved 2026-05-26
3. [GitHub Actions — Using conditions to control job execution](https://docs.github.com/en/actions/using-jobs/using-conditions-to-control-job-execution) — retrieved 2026-05-26
4. [Patterns of Legacy Displacement](https://martinfowler.com/articles/patterns-legacy-displacement/) — retrieved 2026-05-26

Last verified: 2026-05-26
