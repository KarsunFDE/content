---
template: weekly-retro
week: W05
audience: cohort
authored_by: instructor (with cohort contributions)
released_at: Fri W5 14:30
duration_minutes: 60
last_verified: 2026-05-23
---

# W5 Cohort Retro — AIOps Anchor Week + Final Adversarial PR

> Authored Fri afternoon after the 3 pair retros + Final Adversarial PR scoring. Surfaces programme-level signals for skill lead + cohort lead.

## Frontmatter

```yaml
---
week: W05
phase: 2 (operationalisation)
gate: Final Adversarial Review PR
pair_scores:
  pair-1: x/40 (tier)
  pair-2: x/40 (tier)
  pair-3: x/40 (tier)
cohort_majority_tier: Distinguished | Proficient | Developing | Below threshold
authored_at: 2026-06-26T15:30:00Z
hitl_thread_closure: complete (touchpoints 1-7) / incomplete (which)
---
```

## 1. Cohort-wide rubric pattern

Where did all 3 pairs cluster? Where did they diverge?

| Dimension | Pair-1 | Pair-2 | Pair-3 | Cohort signal |
|-----------|--------|--------|--------|---------------|
| Correctness | x | x | x | ... |
| Code Quality | x | x | x | ... |
| Architectural Fit | x | x | x | ... |
| Test Coverage | x | x | x | ... |
| Brownfield Awareness | x | x | x | ... |
| PR Clarity | x | x | x | ... |
| Defense Quality | x | x | x | ... |
| Security Pass | x | x | x | ... |

## 2. AIOps anchor week landed (or didn't)

The cohort's mental model after W5:

- **Did the OTel install land cleanly?** (Tue afternoon smoke test passing across all 3 pairs = signal.)
- **Did the AI-SRE pattern vocabulary stick?** (Cohort can name + apply patterns 1-6.)
- **Did Datadog hands-on produce defensible compare-matrix work?** (Thu morning workshop output.)
- **Did the AWS managed-service migration produce honest worth-it ADRs?** (Thu afternoon output — full migration, hybrid, or no-migrate with reasoning.)

## 3. HITL #7 — the programme HITL thread closure

The 7-touchpoint HITL thread (per D-043 + D-044) closes this week:

| # | Touchpoint | Pair-1 | Pair-2 | Pair-3 |
|---|-----------|--------|--------|--------|
| 1 | W1 Fri LLM Essentials | ... | ... | ... |
| 2 | W2 Thu RAG fallback | ... | ... | ... |
| 3 | W3 Mon Plan Day ADR | ... | ... | ... |
| 4 | W3 Wed multi-agent handoff | ... | ... | ... |
| 5 | W3 Thu LangGraph deep-dive | ... | ... | ... |
| 6 | W4 Wed OWASP LLM06 | ... | ... | ... |
| **7** | **W5 Wed auto-remediation authority** | ... | ... | ... |

Cohort retro probe: **does each pair have a coherent HITL story across all 7 touchpoints, or did the thread fray somewhere?** A pair that committed Option A in W5 Wed but committed something tighter at W3 Thu has an inconsistency — name it, surface to W6 prep.

## 4. D-050 + D-060 framing — did it land?

The cohort's understanding by EOD Mon:
- D-050 deferred AWS managed services to W5.
- D-060 carved out Bedrock InvokeModel for W2-onward.
- Both decisions hold together.

If the cohort still reads D-060 as a reversal of D-050 — the decision-archaeology framing failed. Surface to skill lead for `pipeline/DECISIONS.md` tightening.

## 5. Final Adversarial PR — pattern signals

Across all 3 pairs:

- **Where codex caught the cohort by surprise** — patterns that landed for one pair but not others. Often a signal to backfill into pre-session reading for cohort #2.
- **Where codex was wrong** — findings the cohort successfully argued. Signal for the codex skill maintainer.
- **Where defense quality varied wildly** — pairs that engaged rigorously vs pairs that nodded through. Coaching surface for W6 1:1s.

## 6. AIOps platform compare-matrix — workshop output

The Thu compare-matrix outcome:

- Did the cohort defend Datadog as the right pick for `acquire-gov` cohort #1?
- Did any pair argue for a pivot? Why?
- What dimensions did pairs add to the matrix that weren't in the pre-session list?
- Is the workshop output reusable for Karsun engagement-team consumption?

## 7. AWS managed-service worth-it ADRs — pattern across pairs

Did pairs converge or diverge on the worth-it answers?

| Migration | Pair-1 | Pair-2 | Pair-3 |
|-----------|--------|--------|--------|
| Bedrock Knowledge Base for `/rag/clause-search` | migrate / hybrid / no-migrate | ... | ... |
| Agents-for-Bedrock for `/agent/intake-triage` | ... | ... | ... |

A cohort converging on "hybrid" = healthy nuance. Diverging is also fine if reasoning is documented. Cohort-wide "full migration" = signal of insufficient debuggability concern. Cohort-wide "no migration" = signal of insufficient FedRAMP boundary concern.

## 8. W6 capstone — scaffolding plan

Per Fri rubric tier interpretation:

| Pair | Tier | W6 scaffolding |
|------|------|----------------|
| Pair-1 | ... | ... |
| Pair-2 | ... | ... |
| Pair-3 | ... | ... |

Cohort-wide W6 emphasis: ... (e.g., "Defense Quality was weakest cohort-wide — W6 Tue runbook authoring + ADR catalog will lean on instructor-led defense rehearsal.")

## 9. Programme-level signals (for skill lead + cohort lead)

- **What this cohort #1 revealed about the W5 design** — is the 5–8 topic density right? Is the Wed research day landing per D-040? Is the Fri PR session the right gate format?
- **What carries forward to Cohort #2** — pre-session reading adjustments, war-room reframes, scenario-alternatives candidate-tech updates.
- **What needs `pipeline/DECISIONS.md` follow-up** — D-050 / D-060 framing tightening, HITL #7 ADR template refinement, codex Full-strictness calibration.

## 10. Honest assessment of programme arc

Phase 1 (W1–W3) → Phase 2 (W4–W6). The W4 Mid-Sprint Surprise → W5 fix backlog → W5 Fri PR arc was designed to make Phase 1 discoveries drive Phase 2 work. Did it? Where did the chain break?

- ...

## 11. Cohort retro probes (instructor-facilitated, 30 min)

1. *"What HITL touchpoint did your pair most underestimate?"*
2. *"What would you change about the W4 → W5 handoff?"*
3. *"Where did Claude help most across the 5 days? Where did it actively hurt?"*
4. *"Did you change your mind about anything from your Mon plan-spec by Fri's PR?"*
5. *"What's the one piece of feedback you have for the cohort #2 instructor running W5?"*

## 12. Output artifacts (Fri EOD)

- This cohort retro filled in.
- 3 pair retros (`weeks/W05/retros/pair-{1,2,3}-retro.md`).
- 3 pair rubric-scored outputs (`weeks/W05/assessments/W05-pair-{1,2,3}-score.md`).
- W6 capstone scaffolding plan committed.
- Remediation tickets list (any deferred Fri findings).
- `pipeline/PIPELINE.md` carry-forward notes for any programme-level changes.
