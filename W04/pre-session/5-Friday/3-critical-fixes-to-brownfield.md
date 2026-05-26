---
week: W04
day: Fri
topic_slug: critical-fixes-to-brownfield
topic_title: "Critical fixes to brownfield — ranking by reversibility, not by elegance"
parent_overview: W04/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://tianpan.co/blog/2026-04-12-brownfield-ai-integrating-llm-features-into-legacy-codebases
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://martinfowler.com/bliki/StranglerFigApplication.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-decomposing-monoliths/strangler-fig.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://www.flagsmith.com/blog/deployment-strategies
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://designrevision.com/blog/feature-flags-best-practices
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Critical fixes to brownfield — ranking by reversibility, not by elegance

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define the **fix-rank-by-reversibility** discipline and explain why it inverts the usual instinct to rank fixes by elegance or completeness.
- Name the four standard brownfield fix categories — revert, feature-flag off, narrow hot-patch, stage for later — and the reversibility cost of each.
- Explain why a 12-line patch in one file is structurally safer in a brownfield than a 200-line patch across three services, even when both fix the same root cause.
- Identify the production conditions under which "stage for later" is the *correct* fix, not the cowardly one.
- Articulate why the Strangler Fig pattern's "keep the monolith for rollback" rule applies during incidents, not just during planned migrations.

## 2. Introduction

A brownfield codebase is structurally different from a greenfield under incident pressure in one specific way: **the cost of "I think this is right" goes up sharply.** In a greenfield, you wrote the tests, the validators, and know which paths the change touches. In a brownfield, all three claims are usually false. The system has paths the original authors did not document, validators that do things their names do not suggest, and tests that assert the wrong invariants.

This asymmetry drives the discipline of critical fixes in brownfields. The question "is this fix correct?" cannot be answered with greenfield confidence in the incident window. So the discipline shifts: instead of optimising for **correctness** (which you cannot verify in time), you optimise for **reversibility** (which you *can* verify — reversibility is a property of the change shape, not of the code under the change).

The principle: **rank the available fixes by reversibility, not by elegance.** A small reversible patch that mostly works is safer than a clean irreversible patch that fully works, because in a brownfield you cannot prove "fully works" inside the incident window.

The legacy-modernization literature converges on this. Martin Fowler's [Strangler Fig pattern](https://martinfowler.com/bliki/StranglerFigApplication.html) preserves the legacy path "for rollback" even after the new path is shipped. [AWS Strangler Fig guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-decomposing-monoliths/strangler-fig.html) phrases it as "keep the monolith application for rollback." Brownfield-AI literature in 2026 ([Tian Pan, April 2026](https://tianpan.co/blog/2026-04-12-brownfield-ai-integrating-llm-features-into-legacy-codebases)) frames it as "if the new path degrades quality, flip the route back to legacy." Keep the old path alive; make the new path reversible.

## 3. Core Concepts

### 3.1 The reversibility-first ranking

Rank the available fixes by **how cheaply you can undo this if you are wrong**, not by how elegantly the fix expresses the underlying intent. From most reversible to least:

| Rank | Fix category | Reversibility cost | Time to revert | When to use |
|------|-------------|--------------------|----------------|-------------|
| 1 | **Revert to known-good** | Lowest — one git op, one deploy | < 5 minutes | Load surfaces a regression you cannot characterise in the window |
| 2 | **Feature-flag off** | Lowest — one config flip, no deploy | < 1 minute | A flag exists for the affected surface |
| 3 | **Narrow hot-patch** | Medium — must deploy the revert | 5–15 minutes | Single file, single validator, single repository method |
| 4 | **Cross-cutting patch** | High — multi-file revert, possibly multi-service | 30+ minutes | Almost never under incident pressure — stage instead |
| 5 | **Stage for later** | N/A (no deploy today) | N/A | When the fix's reversibility cost exceeds residual user harm |

The discipline is to **walk this ranking top-down**, not bottom-up. The instinct of a strong engineer is often to reach for rank 3 or 4 — the "real" fix that addresses the underlying cause — because that is what good engineering looks like in calm conditions. Under incident pressure that instinct is the anti-pattern. Start at rank 1; only move down when rank 1 is unavailable for principled reasons.

### 3.2 Revert — the structural safety net

Revert has the lowest reversibility cost because you re-revert: the original code is back, no understanding of the bug required, only the prior deploy artifact. The Strangler Fig pattern is the architectural expression — keep the legacy path running for the entire migration so "revert to legacy" is always available ([2026 practitioner writeup](https://theartofcto.com/frameworks/2026-02-06-fig-tree-strangler-pattern-replace-legacy-without-big-bang): "keep both systems running until the old one has no traffic"). For incident response, the deploy discipline is: every deploy leaves the previous artifact tagged and one-command-reachable. If reverting requires hunting through CI history, the cost is already too high to count as rank 1.

### 3.3 Feature-flag off — the no-deploy revert

Flags have **lower revert cost than revert itself** — a flag flip takes milliseconds and requires no deploy. 2026 practitioner guidance ([Flagsmith](https://www.flagsmith.com/blog/deployment-strategies); [DesignRevision](https://designrevision.com/blog/feature-flags-best-practices)) emphasises flag flips should be **tested in both directions** in CI, so the off-state under incident pressure is not a configuration the team has never exercised.

The corollary is **flag hygiene**: a flag "permanent" for six months has typically diverged from its off-state in ways nobody documented. Treat flags as having a half-life; periodically exercise them in CI or accept they are no longer rank-2 fixes.

### 3.4 Narrow hot-patch — the 12-line rule

A "narrow hot-patch" touches **one file**, **one method**, **one validator**, or **one repository call**. The narrowness is the safety:

- A 12-line patch in one file is reviewable in five minutes. A 200-line patch across three services is not.
- A 12-line patch has a small surface for the test suite to miss; a 200-line patch has many.
- A 12-line patch has a deterministic blast radius; a 200-line patch can have emergent interactions you discover after deploy.
- A 12-line patch is reverted by reverting one file; a wide patch by reverting many files in many repos.

Narrow-hot-patch examples by stack: one Spring Security filter, one Pydantic validator, one Angular guard, one SQL index addition. *Not* "refactor the security configuration," "rework the validation layer," etc. The discipline is to **explicitly constrain the diff shape before writing it**. If the natural shape is wider than one file, that is a signal to stage — not a license to widen.

### 3.5 Stage for later — the criterion

"Stage for later" is correct when the fix's reversibility cost exceeds the residual user harm of leaving the symptom present (or of leaving the rank-1/rank-2 mitigation running). The criterion: **stage when the fix cannot be made narrow, but the symptom can be made tolerable.**

Standard shape of stage: (1) confirm the mitigation is stable, (2) open a follow-up ticket with named identifier, (3) write an ADR amendment in the same file as the original ADR, (4) hand off in writing.

Stage is *not* "we did nothing." It is "we shipped a documented decision to defer a non-trivial fix while a stable mitigation runs." The audit trail has the same shape as a shipped fix; only the timeline differs. Regulated industries treat the staged ticket plus ADR amendment as the compliance artifact.

### 3.6 The review floor does not relax under stress

The narrow hot-patch shape exists precisely so a fix can pass a strict review under time pressure: small enough that a reviewer verifies it in five minutes. A wide patch that "needs to ship" without review is not a fix; it is a second incident waiting for the weekend. Review effort scales with diff size, not with urgency.

## 4. Generic Implementation

A reversibility-first decision checklist, framework-agnostic, looks like the following. The example uses generic terms and applies regardless of stack.

```yaml
# Critical-fix decision checklist (run top-down; stop at first YES)

decision_log:
  incident_id: "{ticket id}"
  symptom: "{one-sentence symptom}"
  mitigation_in_place: "{name of generic mitigation currently running}"

  rank_1_revert:
    available: yes | no
    last_good_artifact_id: "{commit SHA or release tag}"
    revert_validated_in_ci: yes | no
    chosen: yes | no

  rank_2_feature_flag:
    flag_exists_for_surface: yes | no
    flag_off_state_exercised_recently: yes | no
    chosen: yes | no

  rank_3_narrow_hot_patch:
    affected_files: 1                                # MUST be 1
    affected_services: 1                             # MUST be 1
    affected_lines_estimate: "<= 30"
    test_path_for_change_exists: yes | no
    reviewer_available_for_5_min_review: yes | no
    chosen: yes | no

  rank_4_cross_cutting_patch:
    # Under incident pressure, this rank is almost always WRONG.
    # If the natural fix shape requires this rank, stage instead.
    chosen: no

  rank_5_stage:
    follow_up_ticket_id: "{ticket id}"
    adr_amendment_path: "{path/to/ADR.md}"
    mitigation_to_keep_running: "{name}"
    owner_of_staged_work: "{name}"
    chosen: yes | no
```

The deliberate property of the checklist: the team is forced to walk the ranking in order, and each rank has a falsifiable "available?" question. A rank-3 narrow hot-patch is *not available* if the natural shape of the fix is three services; the checklist forces that fact to be written down before the team starts coding. The checklist becomes the audit artifact when the postmortem is written.

> [!instructor-review]
> **Reversibility-cost claims should be re-verified per-stack.** The 30-line "narrow hot-patch" threshold is a rule of thumb from the Spring Boot / FastAPI / Angular sources; teams on different stacks may want to recalibrate (Go services tend to write longer error-handling blocks, Rust patches are often slightly larger because of explicit lifetimes, etc.). Instructor: confirm the cohort's stack-specific norm during the morning war-room rather than treating 30 as universal.

## 5. Real-world Patterns

**Fintech — payment-gateway timeout regression.** A consumer-fintech platform deployed a new payment-gateway client library at 14:00 UTC; by 14:15, p99 latency on the checkout endpoint had tripled. The on-call's first instinct was to investigate the timeout configuration (rank 3). The incident commander overrode: revert the deploy (rank 1). Revert took four minutes; p99 returned to baseline within eight. The team spent two business days investigating the timeout, then shipped the proper fix on Tuesday with regression tests. *The rank-1 revert preserved the weekend; the rank-3 investigation produced the durable fix on the next business day.*

**Gaming — matchmaker corruption.** A multiplayer studio shipped a new matchmaking algorithm behind a feature flag at 5% of traffic. Within 30 minutes, the 5% cohort was reporting elevated match-crash rates. The team flipped the flag to 0% (rank 2) — instantaneous, no deploy. Investigation in calm conditions revealed a corrupted asset bundle that one CDN edge was serving. The fix shipped a week later, properly tested. *Without the flag, the team would have had to revert under pressure — which would also have reverted three unrelated improvements in the same release.*

A second cluster of cases — healthcare validator hot-patch, e-commerce search-index drift, and the audit-trail story under regulated-industry compliance — appears in the [further-reading appendix](./3-critical-fixes-to-brownfield-further-reading.md).

## 6. Best Practices

- **Tag every deploy artifact so revert is one command.** "Revert to known-good" stops being rank-1 if it requires hunting through CI history.
- **Exercise feature-flag off-states in CI.** A flag that has not been flipped in six months is a flag whose off-state may no longer work.
- **Constrain the narrow-hot-patch diff explicitly.** Before writing the patch, name the one file and one method that will change. If the natural shape exceeds that, escalate to stage rather than widen.
- **Treat the 5-minute review window as a precondition.** A narrow patch passes review in five minutes; if your reviewer needs more, the patch is too wide.
- **Write the ADR amendment in the same file as the original ADR.** The audit trail wants the original and the amendment in one place — not a separate "post-incident" document.
- **Stage with a named owner, not a TBD.** A staged ticket without an owner has a half-life of three sprints before it is forgotten.
- **Run strict review even on the narrow hot-patch.** The narrowness exists so review can be fast, not so review can be skipped.

## 7. Hands-on Exercise

**Task (paper or markdown sketch, 10–15 min):** You are the on-call for a SaaS product in an industry of your choice (NOT federal acquisitions — pick fintech, healthcare, gaming, e-commerce, or logistics). Your monitoring shows that since the 14:00 deploy, your primary write endpoint is silently dropping ~3% of requests. The deploy included a refactor of the request-validation layer (touched four files in three services).

Walk through the reversibility-first ranking:

1. **Rank 1 — revert.** State the precondition that would let you choose this rank in your scenario. If unavailable, state why.
2. **Rank 2 — feature flag.** Did the refactor go behind a flag? If yes, choose this rank. If no, note the operational change you would make for next time.
3. **Rank 3 — narrow hot-patch.** Identify a 30-line, single-file shape that would address the symptom. Name the file and the method.
4. **Rank 4 — cross-cutting patch.** State the reason this is *not* the right rank under incident pressure.
5. **Rank 5 — stage.** Draft the one-paragraph follow-up ticket you would open if the staged path is correct. Include symptom, mitigation in place, proposed fix shape, and named owner.

**What good looks like.** A correct sketch walks the ranking top-down, names what would have to be true for each rank to be chosen, and reaches a defensible decision. A common mistake is jumping to rank 3 or 4 because that's what "real engineering" looks like; the discipline is writing down why rank 1 and rank 2 were not available first. Experienced answers often default to rank 1 under pressure and treat rank-3 as a Tuesday-morning option once the mitigation is stable.

## 8. Key Takeaways

- **Why does reversibility outrank elegance in brownfield incident response?** Elegance requires correctness, correctness requires verification, and verification is what you cannot fully do in the incident window. Reversibility is a property of the change shape, verifiable without running the code.
- **What is the standard ranking?** Revert (1) → feature-flag off (2) → narrow hot-patch (3) → cross-cutting patch (4, almost always wrong under pressure) → stage (5).
- **Why is a 12-line patch in one file safer than a 200-line patch across three services?** Smaller test-miss surface, smaller review diff, smaller blast radius, smaller revert cost.
- **When is "stage for later" correct?** When a stable mitigation runs, residual user harm is bounded, and the fix's natural shape exceeds narrow-hot-patch. Staged with named owner + ADR amendment is a documented engineering decision.
- **Why does the strict review bar not relax under pressure?** The narrow-hot-patch shape exists precisely so review completes in five minutes; skipping review on a wide patch produces the next incident.

## Sources

1. [Brownfield AI — Integrating LLM Features into Legacy Codebases (Tian Pan, April 2026)](https://tianpan.co/blog/2026-04-12-brownfield-ai-integrating-llm-features-into-legacy-codebases) — retrieved 2026-05-26
2. [Martin Fowler — Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html) — retrieved 2026-05-26
3. [AWS Prescriptive Guidance — Strangler Fig Pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-decomposing-monoliths/strangler-fig.html) — retrieved 2026-05-26
4. [Flagsmith — 8 Feature Flag Deployment Strategies](https://www.flagsmith.com/blog/deployment-strategies) — retrieved 2026-05-26
5. [DesignRevision — Feature Flags: 12 Best Practices](https://designrevision.com/blog/feature-flags-best-practices) — retrieved 2026-05-26

Last verified: 2026-05-26
