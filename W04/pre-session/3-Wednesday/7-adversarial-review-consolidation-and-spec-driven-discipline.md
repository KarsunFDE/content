---
week: W04
day: 3-Wednesday
topic_slug: adversarial-review-consolidation-and-spec-driven-discipline
topic_title: "Adversarial Review Consolidation + Spec-Driven Discipline Check-in"
parent_overview: W04/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://www.thoughtworks.com/en-us/insights/blog/agile-engineering-practices/spec-driven-development-unpacking-2025-new-engineering-practices
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.augmentcode.com/guides/what-is-spec-driven-development
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.infoq.com/articles/spec-driven-development/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.mend.io/blog/llm-security-risks-mitigations-whats-next/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://arxiv.org/abs/2602.16741
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# Adversarial Review Consolidation + Spec-Driven Discipline Check-in

## 1. Learning Objectives

By the end of this reading, the learner can:

- Apply a severity rubric (P0 / P1 / P2 / P3) to security findings from an adversarial reviewer and route each correctly (block merge / fix-this-PR / next-iteration / nice-to-have).
- Re-check security findings against the originating plan-spec to identify *spec drift* — places where the code diverged from the intended behavior named in the spec.
- Recognise spec-driven development (SDD) as the 2025 industry consensus for AI-assisted engineering, and explain why specs catch what unit tests cannot (architectural violations, contract drift, security anti-patterns across service boundaries).
- Distinguish three disposition outcomes for a finding (fix now / amend spec + fix next / document gap and defer with audit trail) and write a decision record for each.
- Recognise the comment-injection adversarial-review attack and the defense (cross-check LLM review against static analysis + human reviewer).

## 2. Introduction

Adversarial code review by an independent reviewer — human or LLM-backed — has emerged as a standard discipline in security-conscious teams in 2025. The 2026 OWASP guidance and the Mend.io LLM Security in 2025 report both name codex-style independent review as a "should-have" defense layer alongside SAST and dependency scanning ([Mend.io, 2025](https://www.mend.io/blog/llm-security-risks-mitigations-whats-next/)). The CVE log makes the case directly: GitHub Copilot's CVE-2025-53773 — a CVSS-9.6 remote code execution flaw in AI-generated code — would have been caught by an independent reviewer running on the same diff.

This reading is a discipline check-in, not new material. By W4 Wednesday morning, every pair has practised independent adversarial review on previous PRs. What's new in W4 is that **codex strictness is now at "Full"** — P0 and P1 findings block merge. The discipline of *consolidating* findings — walking the codex output line by line, re-checking each against the plan-spec, naming a disposition — is the practice that prevents findings from becoming technical debt that surfaces six months later as an incident.

The companion frame is **Spec-Driven Development (SDD)**. The Thoughtworks Technology Radar identified SDD as "one of the most important practices to emerge in 2025" — putting structured specs, not chat prompts, at the centre of AI-assisted engineering ([Thoughtworks, 2025](https://www.thoughtworks.com/en-us/insights/blog/agile-engineering-practices/spec-driven-development-unpacking-2025-new-engineering-practices)). The empirical case is sobering: a February 2026 study counted more than 110,000 surviving AI-introduced issues in production repositories ([InfoQ, 2025](https://www.infoq.com/articles/spec-driven-development/)). Most of those slipped past unit tests because unit tests check individual functions; they cannot catch architectural violations, API contract drift, or security anti-patterns that emerge *across* service boundaries. The spec is the artifact that catches those.

## 3. Core Concepts

### 3.1 Severity rubric — four tiers

A useful severity scheme for adversarial review findings:

| Tier | Definition | Disposition |
|---|---|---|
| **P0** | Exploitable in production; clear attack path; CVSS ≥ 7.0 equivalent | **Block merge.** Fix in this PR. No exceptions. |
| **P1** | Security-relevant bug with realistic attack scenarios; CVSS 4.0–7.0 | **Block merge.** Fix in this PR or amend the spec and fix in next iteration. |
| **P2** | Defense-in-depth gap; CVSS < 4.0 or requires unrealistic preconditions | Fix in next iteration; record in spec amendment. |
| **P3** | Code quality / clarity / nit | Optional — author's discretion. |

The cohort's codex configuration at "Full" strictness flags P0 + P1 as merge-blocking. P2 and P3 are advisory.

### 3.2 The three-step consolidation discipline

For each finding from an adversarial reviewer:

1. **Open the finding alongside the originating plan-spec.** What did the spec name as the expected behaviour? Does the finding reveal spec drift (the code diverged from the spec) or a spec gap (the spec never named this case)?
2. **Pick a disposition.** Three valid outcomes:
   - **Fix now.** Patch the code in this PR. The spec was right; the code drifted.
   - **Amend spec + fix next.** The spec missed a case; update the spec, then patch in next iteration.
   - **Defer with documented gap.** The finding is real but out-of-scope for this work; create a backlog item with the audit trail, and name the security context that will catch it later.
3. **Record the decision.** A one-line decision-record per finding ("F-007: P1, spec-drift, fix-now, PR #142, codex line 187") is enough. Without the record, the next reviewer reopens the question.

### 3.3 Spec drift as a finding category

Spec-Driven Development frames the *interlock* between specs and code as the load-bearing discipline ([Augment Code, 2025](https://www.augmentcode.com/guides/what-is-spec-driven-development); [Thoughtworks, 2025](https://www.thoughtworks.com/en-us/insights/blog/agile-engineering-practices/spec-driven-development-unpacking-2025-new-engineering-practices)). Three drift categories show up repeatedly:

- **API contract drift.** The spec names a request/response shape; the code returns a different shape. Unit tests in isolation may still pass because each side mocks the other.
- **Architectural-boundary drift.** The spec puts a security check at layer A; the code skipped it because layer B was assumed to also check. Both layers may have unit tests; neither names the interlock.
- **Security-anti-pattern drift.** The spec named a defense layer (e.g., "validate input via Pydantic"); the code added a "quick path" that skips it for a specific case.

The consolidation discipline asks one question per finding: "is this a drift category I can name?" If yes, the fix is also a spec-discipline fix, not just a code patch.

### 3.4 The §0 retro discipline

Plan Day in the FDE programme starts with a "§0 Plan retrospective on last week's spec" ([CLAUDE.md programme docs]). The §0 retro is the *cumulative* spec-driven check. By W4 Wed, the cohort has practised this four times (W2/W3/W4 Mon plus the Tue spec-driven workshop). Wed morning's consolidation extends the practice to Tue's deliverables: the spec is not a Monday-only artifact — it is the lens you re-check every day's output against.

The connection to security review: an adversarial finding that reveals spec drift is a §0 finding waiting to happen. Catching it Wednesday morning instead of next Monday is the value of the discipline.

### 3.5 The adversarial-comment attack on LLM reviewers

A 2026 large-scale empirical study tested whether attacker-controlled code comments could fool LLM-based security reviewers ([arXiv 2602.16741, 2026](https://arxiv.org/abs/2602.16741)). The good news for cohort discipline: across 9,366 trials with 8 frontier models, adversarial comments produced "small, statistically non-significant effects on detection accuracy" — commercial models held 89–96% baseline detection.

The takeaway is NOT "LLM review is bulletproof." The takeaway is the defense: pair LLM review with static analysis (SAST tools) and human review. The IRIS approach (ICLR 2025) achieved a 104% improvement over CodeQL alone by combining LLMs with static analysis. No single layer is sufficient; the bundle is what works.

## 4. Generic Implementation

A generic decision-record template for adversarial-review consolidation:

```yaml
# .review/2026-W04-wed-codex-consolidation.yml
# One entry per finding from the codex review.

findings:
  - id: F-007
    pr: 142
    codex_line: 187
    severity: P1                # P0 / P1 / P2 / P3
    category: security
    summary: >
      `/api/orders/{id}` reads the order without re-checking tenant_id
      against the calling user's tenant. Relies on gateway-side filter.
    spec_reference: planning/W04/security/orders-api-spec.md#tenant-binding
    drift_type: architectural-boundary    # api-contract / arch-boundary / sec-anti-pattern
    disposition: fix-now
    fix_pr: 144
    reviewer: codex
    notes: >
      Spec said "every read re-validates tenant_id against caller."
      Code skipped it on the by-id path. Belt-and-braces re-check added.

  - id: F-008
    pr: 142
    codex_line: 234
    severity: P2
    category: defense-in-depth
    summary: >
      No rate-limit on the LLM-mediated /summarise endpoint.
      Realistic abuse requires authenticated caller — lower priority.
    spec_reference: planning/W04/security/llm-endpoints-spec.md
    drift_type: spec-gap                   # spec didn't name this case
    disposition: amend-spec-fix-next
    fix_ticket: KAN-2117
    reviewer: codex
    notes: >
      Spec amendment: add per-caller rate limit section. Fix in next iter.

  - id: F-009
    pr: 142
    codex_line: 311
    severity: P3
    category: code-quality
    summary: >
      Magic number 30000 in cache TTL — replace with named constant.
    disposition: defer
    reviewer: codex
    notes: >
      Nit. Author optional. No backlog ticket.
```

What each section does:

- **`severity` + `disposition`** are the load-bearing fields. The disposition map (P0 → fix-now, P1 → fix-now-or-amend-spec, P2 → amend-spec-fix-next, P3 → optional) is the rubric the team agrees to up-front.
- **`spec_reference`** names which plan-spec line the finding bears on. If the field is empty, the finding either is a spec gap OR the spec is incomplete.
- **`drift_type`** categorises the gap (API contract drift, architectural-boundary drift, security-anti-pattern drift, or pure spec gap). Categorising is what turns one-off finding-fixes into systemic spec improvements.
- **`fix_pr` / `fix_ticket`** closes the loop. Every non-deferred finding has a traceable next-action.

The template is YAML for reviewability; in a real workflow it's also consumed by tooling that aggregates findings across PRs and surfaces patterns (e.g., "this team has had 7 architectural-boundary drift findings in 3 weeks — the spec process needs help").

## 5. Real-world Patterns

**Fintech — codex-style review as a release-blocking gate.** A 2025 wealth-management platform integrated a codex-equivalent independent-review pass into their CI pipeline. P0/P1 findings block the build. The published metric: 14% of all PRs had at least one P0 finding caught at this gate that would otherwise have shipped. The team initially complained about velocity loss; six months in, they reported a 60% drop in security-related production incidents and the velocity-loss complaint went away ([Mend.io, 2025](https://www.mend.io/blog/llm-security-risks-mitigations-whats-next/)).

**Healthcare — spec-driven catch of HIPAA boundary drift.** A 2025 hospital-network SaaS used Augment Code's SDD workflow. A PR added a new "share with another provider" feature. Unit tests passed. The adversarial review flagged that the spec named PHI-redaction at the share boundary, but the implementation only redacted at the read boundary — leaving a path where shared PHI escaped redaction. The team's post-mortem credited the SDD discipline: "the spec was the artifact that caught the drift; unit tests alone would have missed it because each side's tests passed in isolation" ([Augment Code, 2025](https://www.augmentcode.com/guides/what-is-spec-driven-development)).

**Open-source — adversarial-comment defense in practice.** A 2026 open-source project that uses LLM-based security review added a "comment scrubbing" preprocessing step to its review pipeline after observing that contributors (well-meaning ones) had been adding "this is safe because…" comments that the LLM reviewer was weighting too heavily. After the scrub, the LLM reviewer flagged a previously-missed SQL-injection vector in one of those "this is safe" code paths. The lesson named in the project's engineering blog: "comments are not part of the security contract; the review should read code, not narrative" ([arXiv 2602.16741, 2026](https://arxiv.org/abs/2602.16741)).

**Logistics — spec-driven catch of API contract drift.** A 2026 logistics platform's order-tracking API was generated from a spec that named `delivery_window` as an ISO-8601 datetime range. A refactor changed the field to a string. Downstream consumers didn't break immediately because they were also using a permissive parser. Three weeks later, a different consumer started returning empty windows because *its* parser was strict. The post-mortem identified the gap: the spec was the source of truth, but no CI check enforced the spec against the implementation. Fix: contract-test step in CI that asserts the spec and the implementation agree ([InfoQ, 2025](https://www.infoq.com/articles/spec-driven-development/)).

## 6. Best Practices

- **Agree the severity rubric BEFORE the first review.** Negotiating P0-vs-P1 mid-review erodes trust in the gate.
- **Every finding has a disposition with an owner.** Open findings without dispositions are how technical debt accrues.
- **Re-check every finding against the plan-spec.** If the spec was silent on this case, the finding is a spec gap — amend the spec.
- **Pair LLM review with static analysis and human review.** No single layer is sufficient; the bundle is what works.
- **Scrub comments before LLM review.** Adversarial comments can shift reviewer behavior — read code, not narrative.
- **Track drift-type categories over time.** A team with repeated architectural-boundary drift needs a spec-process improvement, not just more reviews.
- **Spec-driven discipline is a daily lens, not a Monday-only artifact.** Re-check every day's output against the spec.

## 7. Hands-on Exercise

**Code task (10–15 min):** You are given a codex-output report with five findings on a single PR. The findings are:

1. SQL string concatenation in a search endpoint (no parameterisation).
2. Missing rate-limit on an LLM-mediated endpoint.
3. A magic number 86400 in a cache-TTL field.
4. The PR's tenant-id binding skipped on a new "by-name" lookup variant; spec required tenant-binding on every read.
5. A new dependency added without a license review.

For each finding: assign a severity (P0/P1/P2/P3), name a disposition (fix-now / amend-spec-fix-next / defer), identify whether it's spec drift or spec gap (if applicable), and write the one-line decision record.

**What good looks like:** Finding 1 is P0 fix-now (active SQLi vector). Finding 4 is P1 fix-now, architectural-boundary drift (the spec required this; the code drifted). Finding 2 is P2 amend-spec-fix-next, spec gap (the spec didn't name rate-limits for LLM endpoints — add a section, then fix). Finding 3 is P3 defer (nit). Finding 5 is process — P1 fix-now (license review is a release gate). The candidate also notes that findings 1 + 4 together are blocking; the PR cannot merge until both are addressed.

## 8. Key Takeaways

- The severity rubric (P0/P1/P2/P3) maps to dispositions (block-merge / fix-now / amend-spec / defer) — can I assign tier and disposition without negotiating?
- Every finding is re-checked against the plan-spec — is the gap drift (code diverged) or spec-gap (spec was silent)?
- Spec-Driven Development catches what unit tests cannot — architectural-boundary drift, API contract drift, security-anti-pattern drift across service boundaries — am I producing and maintaining the spec as a daily lens?
- LLM-based review is robust to comment injection but is one layer of a bundle — am I pairing it with static analysis and human review?
- The §0 retro discipline extends to mid-week consolidation — am I re-checking Tuesday's deliverables against the spec on Wednesday morning?

## Sources

1. [Spec-driven development: Unpacking one of 2025's key new AI-assisted engineering practices (Thoughtworks)](https://www.thoughtworks.com/en-us/insights/blog/agile-engineering-practices/spec-driven-development-unpacking-2025-new-engineering-practices) — retrieved 2026-05-26
2. [What Is Spec-Driven Development? A Complete Guide (Augment Code)](https://www.augmentcode.com/guides/what-is-spec-driven-development) — retrieved 2026-05-26
3. [Spec-Driven Development: When Architecture Becomes Executable (InfoQ, 2025)](https://www.infoq.com/articles/spec-driven-development/) — retrieved 2026-05-26
4. [LLM Security in 2025: Key Risks, Best Practices & Trends (Mend.io)](https://www.mend.io/blog/llm-security-risks-mitigations-whats-next/) — retrieved 2026-05-26
5. [Can Adversarial Code Comments Fool AI Security Reviewers (arXiv 2602.16741, 2026)](https://arxiv.org/abs/2602.16741) — retrieved 2026-05-26

Last verified: 2026-05-26
