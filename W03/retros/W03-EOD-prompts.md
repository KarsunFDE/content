---
week: W03
purpose: "Per-day EOD reflection prompts — committed to pair-project repos at end of each day"
written_for: pairs (individual + paired reflection — 5-10 min per day)
last_verified: 2026-05-23
---

# W3 EOD Prompts — Daily Pair Reflection

> Per `pipeline/PIPELINE.md` §4, end-of-day reflection is a 5–10 minute pair-conversation closing each working day. Not assessed. Captured in the pair-project repo at `reflections/W03-{Day}.md` for the W3 weekly retro to draw on.

## Mon EOD (Plan Day) — 5 min

1. What's the riskiest ADR you committed today? Why is it the riskiest?
2. Did the §0 retro on the W2 plan change anything about how you wrote the W3 plan? Be specific.
3. Of the three HITL touchpoints landing this week, which one feels least defined right now? What will you do Tuesday morning to define it?
4. Pair-collaboration: did one of you drive the ADRs more than the other? Should it be different tomorrow?

## Tue EOD — 5 min

1. Did your single-agent intake-triage agent do something today that surprised you? (Either: it solved a case you expected to be hard, OR it failed on a case you expected to be easy.)
2. The LangSmith trace for your last triage run — did you READ it, or did you just confirm it was logged? If you didn't read it, that's the discipline that breaks Wed.
3. Was the idempotency on `route_to_evaluators` a thing you wired carefully, or a thing you wired loosely "for now"?
4. Tomorrow you transition from single-agent → multi-agent. What's the ONE thing about today's design you're worried won't survive that transition?

## Wed EOD — 8 min (longer — Wed has both incident work + scenario research + KG/CG)

1. Multi-agent shape: did the supervisor-worker pattern feel like a *natural* extension of Tuesday's single-agent, or like a *complete redesign*? What does that tell you?
2. HITL #4 (soft interrupt at supervisor → worker handoff): how did you decide the *default* behavior on the soft-gate? What did you decide auto-resume on?
3. KG/CG thinking — your pair-lead scenario (SA-1, SA-2, or SA-3): what's the ONE non-obvious thing your initial pair-look surfaced about it?
4. Item 2 (audit-log race) — did you feel it today? If yes, what was the symptom? If no, are you sure it's not silently firing in your traces?
5. Pair-collaboration: in the scenario-research slot, did you split into parallel research or stay together? Which would have been better in hindsight?

## Thu EOD — 8 min (longer — HITL #5 deep-dive day)

1. Hard `interrupt_before` at `ssa_review_ssdd`: did you wire it from a docs example, or from memory, or from first principles? How confident are you the audit row schema is right?
2. The 18-hour-resume test: did you actually run it (FastAPI restart + resume), or just unit-test the resume logic? If only unit-tested, that's a Friday-defense gap.
3. Per FAR 15.308, the SSA's judgment cannot be delegated to auto-regeneration. Did anyone in your pair *suggest* auto-regenerating the draft on resume? (Honesty check — this is a common trap.)
4. Of the five HITL touchpoints (#1-#5) that have landed cohort-wide, which ONE could you defend cold to the OIG without notes? Which ONE would you need to look up?
5. Tomorrow is the gate. What's the ONE thing you're most worried about defending? What would close that gap in 90 minutes if you had them tonight?

## Fri EOD — 10 min (Gate Day — combined personal + pair reflection)

> Run AFTER the Mid-Program Retro + AFTER the MCQ + AFTER the Phase 1 freeze tag.

1. Phase 1 cumulative: which of the three weeks (W1 LLM essentials / W2 RAG / W3 agentic) felt most natural to you? Most foreign? Why?
2. Which moment in Phase 1 most changed how you think about FDE work? (Could be a war-room save, a Codex finding that landed right, a discovery that surfaced during the multi-agent shape, an HITL ADR that took longer than expected.)
3. Phase 1 discovery you're most proud of — what did your pair surface that wasn't obvious from the W1 Tue brownfield-debt inventory?
4. Going into Phase 2: what's the ONE thing about modernization (W4-W6) you most want to learn? What's the ONE thing you most want to avoid?
5. Pair-collaboration retro for Phase 1: did the pair work? What would you change for Phase 2? (Pair stays the same per D-038; pair-collab pattern can evolve.)
6. Personal: what's one habit you adopted in Phase 1 that you want to carry into Phase 2? What's one you want to drop?

## Storage convention

Each pair commits these to `pair-N-<aspect>/reflections/W03-{Mon|Tue|Wed|Thu|Fri}.md`. Not assessed; instructor reads on Mon W4 morning before the W4 Mon 1:1s as input to the next gate-boundary coaching conversation (next 1:1 = W5 Mon per `PIPELINE.md` §4).
