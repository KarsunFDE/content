---
cohort: cohort-1-2026-05
programme: Karsun-FDE v2 (6-week intensive)
start_date: 2026-05-26  # Tue (Mon 25 May = Memorial Day, lost)
end_date: 2026-07-02    # Thu (Fri 3 Jul = Independence Day observed, lost)
weeks: 6
trainees: 6                # 3 Senior FDE + 3 Entry FDE
instructor: TBD            # populate before W1 Tue
pairs:
  - id: pair-1
    aspect: TBD            # picked W1 Wed AM from karsun-domain-aspects.yml
    senior: TBD            # name + tier
    entry: TBD
    repo: TBD              # KarsunFDE/pair-1-<aspect>
  - id: pair-2
    aspect: TBD
    senior: TBD
    entry: TBD
    repo: TBD
  - id: pair-3
    aspect: TBD
    senior: TBD
    entry: TBD
    repo: TBD
galent_presenter:
  name: TBD                # confirm 3 weeks before W1 Tue
  session_date: 2026-05-27 # Wed AM
  backup_recording: false  # set true once rehearsed
holiday_compressions:
  - date: 2026-05-25       # Mon Memorial Day
    impact: "W1 = Tue–Fri (4 days). Mon kickoff compressed into Tue morning. Tue Microservices Foundation compressed into Wed afternoon."
  - date: 2026-07-03       # Fri Independence Day observed
    impact: "W6 = Mon–Thu (4 days). Final Defense moved Fri 2 Jul → Thu 2 Jul. Cohort Retro absorbed into Thu afternoon."
checkpoints:
  - id: checkpoint-1
    date: 2026-06-09       # Tue (post-W2)
    artifact_dir: deliverables/checkpoints/checkpoint-1
  - id: checkpoint-2
    date: 2026-06-16       # Tue (post-W3 gate)
    artifact_dir: deliverables/checkpoints/checkpoint-2  # TBD authored end-of-W3
  - id: checkpoint-3
    date: 2026-06-22       # Mon (W5 start)
    artifact_dir: deliverables/checkpoints/checkpoint-3  # TBD authored end-of-W4
  - id: checkpoint-4
    date: 2026-06-29       # Mon (W6 start)
    artifact_dir: deliverables/checkpoints/checkpoint-4  # TBD authored end-of-W5
gates:
  phase_1: 2026-06-12      # W3 Fri
  phase_2: 2026-07-02      # W6 Thu (compressed from Fri)
last_updated: 2026-05-22
---

# Cohort #1 (2026-05) — Karsun-FDE v2

> **Active cohort dashboard.** This file is mutated during live delivery as pairs lock in, instructors confirm, and decisions land. Frontmatter is the canonical state; body is human-readable narrative.

## At-a-glance status

| Item | State | Notes |
|------|-------|-------|
| Galent presenter confirmed | ⏳ pending | 3-week-out deadline = Tue 5 May (passed) — needs confirmation rehearsal ASAP |
| Backup Galent recording rehearsed | ⏳ pending | Risk if presenter unavailable |
| Pair assignments | ⏳ deferred to Tue PM | Based on Tue diagnostic recast |
| Pair-N-<aspect> repos provisioned | ⏳ pending | W1 Wed AM after aspect picks |
| 3 per-pair brownfields generated | ⏳ pending | W1 Wed PM via `pair-brownfield-generator` skill |
| `acquire-gov` cross-org push | 🚦 HITL gate open | 6 commits ready at `~/KarsunFDE/acquire-gov/` |
| `assessment-ec` cross-org push | 🚦 HITL gate open | 1 commit ready at `~/KarsunFDE/assessment-ec/` |
| W1 compression remap (T03b) | ⏳ in progress | Iter-3 of autonomous loop |

## Weekly delivery calendar

| Week | Dates | Compressed? | Gate / Checkpoint |
|------|-------|-------------|-------------------|
| W1 | Tue 26 May – Fri 29 May | **Yes — 4 days** (Memorial Day) | — |
| W2 | Mon 1 Jun – Fri 5 Jun | No | First Live Defense Fri 5 Jun |
| W3 | Mon 8 Jun – Fri 12 Jun | No | **Phase 1 Gate Fri 12 Jun** + Checkpoint 1 Tue 9 Jun |
| W4 | Mon 15 Jun – Fri 19 Jun | No | Checkpoint 2 Tue 16 Jun + Mid-Sprint Surprise Fri 19 Jun |
| W5 | Mon 22 Jun – Fri 26 Jun | No | Checkpoint 3 Mon 22 Jun + Final Adversarial Review PR Fri 26 Jun |
| W6 | Mon 29 Jun – Thu 2 Jul | **Yes — 4 days** (Independence Day observed) | Checkpoint 4 Mon 29 Jun + **Final Defense Thu 2 Jul** |

## Special handling — Cohort #1

Both compression weeks share a pattern: a day is lost, content reshuffles, but every audit/exam checkpoint stays on its canonical day (Mon/Tue). Cohort cadence preserved.

- **W1 Tue absorbs Mon kickoff content** + original Tue afternoon practical. `weeks/W01/PLAN.md` documents the compression. **Iter-3 of the autonomous loop performs the file-content remap inside this cohort copy** (`weeks-cohort-202605/W01/`), leaving canonical `weeks/W01/` intact for future non-holiday cohorts.
- **W6 Final Defense moves to Thu 2 Jul.** No client showcase Friday. Defense + Cohort Retro both on Thu.

## Instructor handoff items

(Populate before W1 Tue 09:00)

- [ ] Galent presenter confirmed + tech check
- [ ] Backup recording rehearsed
- [ ] Six trainee onboarding emails sent (instructions for tools install)
- [ ] Diagnostic prompts sent to cohort the weekend before so they pre-think Tue
- [ ] AWS Bedrock model access verified per learner

## Reference

- Canonical curriculum source-of-truth: `weeks/` (unchanged across cohorts)
- This cohort's working copy: `weeks-cohort-202605/`
- Execution plan: `pipeline/COHORT1-EXECUTION-PLAN.md`
- Holiday compression rationale: `pipeline/DECISIONS.md` (D-045 area)
- Autonomous loop state: `.handoff/AUTONOMOUS-LOOP-PROMPT.md` + `.handoff/autonomous-policy.md`
