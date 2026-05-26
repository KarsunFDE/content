---
template: pre-session-reading
week: W04
day: Mon
phase: SDLC
topic: "Phase 2 framing + AI Backlog Generation + ADR Discipline + Scenario Design Planning artifact"
estimated_total_minutes: 50
last_verified: 2026-05-26
fde_situations: [3, 4, 5, 6, 10, 11]
tech: [spec-driven-dev, ADRs, OWASP-LLM-Top-10-2025, AI-backlog-generation, scenario-design-planning]
sources_research_briefs: [research/owasp-llm-top-10-20260522.md, research/spring-boot-2-7-to-4-x-20260525.md]
author: instructor
---

# W4 Mon Pre-Session — Phase 2 framing + AI Backlog Generation + ADR Discipline + Scenario Design Planning

> Read Sunday night, before W4 Mon Plan Day. ~50 min. **Phase 2 begins Monday.** This brief carries the day's substantive material — there is no separate per-topic deep-dive. The §0 retro you'll author Monday morning is the *interlock point* between Phase 1 discoveries and the 5-day modernization commitment. Verified 2026-05-26 via /web-research.

## 1. Phase 2 begins — modernization driven by Phase-1 discoveries (8 min)

Phase 1 closed Friday with your Phase 1 Defense + Mid-Program Retro. Phase 2 starts now, and it isn't "a new project" — it's the *modernization driven by what Phase 1 actually surfaced*. The W4 Mon §0 retro is the interlock point: every Phase 1 finding that hit your defense becomes either a scoped W4 ADR commit, a deferred-to-W5-with-rationale item, or an explicit cut. Nothing about Phase 1 sits unaddressed.

Concretely, your pair walks into Plan Day morning with three artifacts in hand: the W3 retro file (`weeks/W03/retros/`), the Phase 1 Defense feedback the instructor wrote up, and the 12-debt-item inventory from W1 Tue (`training-project/README.md`). By 17:00 Monday those three feed two ADRs: the **W4 Modernization Scope ADR** (which 2–3 debt items + per-pair-unique work you commit to) and the **AI Security Threat Model ADR** (which 3–4 OWASP LLM:2025 entries you'll exercise Wed). Scope, not heroism.

**Question to bring to morning war-room:** *Which single Phase 1 finding most narrows your W4 modernization scope — and which Phase 1 discovery did your pair NOT yet make?* That second half is the harder question. Plan Day's §0 retro forces it onto paper.

> **Glossary:** **§0 retro** — first section of every Plan Day artifact per D-029 + D-036: "what did last week's spec get wrong, and what does that mean for this week's scope?" Living discipline, not a ceremony. This is the 3rd practice — W2/W3/W4 Mon.

## 2. AI Backlog Generation — pair-project-shaped, human-judgment-gated (10 min)

The Mon AM exercise is **AI Backlog Generation**: your pair feeds Phase 1 Defense findings + the 12-debt-item baseline + your pair-project repo's slice into an LLM-assisted backlog drafting prompt, gets a candidate W4 backlog back, and then **prunes manually** to the 2–3 debt items + per-pair-unique work that fits this week. The LLM proposes; the pair commits. The human-judgment step is the load-bearing discipline, not an afterthought.

What makes this exercise non-trivial is that the LLM will happily generate a 15-item backlog full of plausible work — every item correctly traced to a Phase 1 finding, every item realistically scoped. Your job is to recognise that a plausible 15-item backlog is *worse than a thoughtful 3-item backlog*, because Week 4 has only one execution day (Thu) and one incident-response day (Fri). The prune isn't bookkeeping; it's a forecast of where Friday's surprise will eat the budget.

Frame for the pair: the LLM is a *backlog candidate generator*, not a planner. You are the planner. The ADR is your audit trail of which candidates you accepted, which you cut, and *why*. Codex Full strictness will read those ADRs Thu — see §3.

## 3. ADR-Writing Discipline — what Codex Full strictness looks like Mon (10 min)

Two ADRs land Mon as draft commits in `planning/W04/adrs/`: **01-w4-modernization-scope.md** and **02-ai-security-threat-model.md**. Both will be read by codex adversarial review at Full strictness per D-034 — P0 and P1 findings block merge starting Thu. "ADR Writing Discipline" this week means writing ADRs that survive contact with an adversarial reviewer.

Three things make an ADR survive:

1. **Evidence floor.** Every named decision cites the artifact it rests on. "We're modernizing solicitation-service first" cites the Phase 1 finding that named it. "LLM07 + LLM08 apply to the JWT-skip surface" cites the endpoint in `feature-inventory-target.md`. No bare assertions.
2. **Alternatives considered.** Each ADR names at least one alternative the pair *didn't* pick, and why. The codex reviewer's first complaint about a thin ADR is "no alternatives analysis" — pre-empt it.
3. **Rollback plan.** What does undoing this decision look like Friday afternoon at 16:30 when the Mid-Sprint Surprise has eaten your buffer? Name the rollback explicitly. If there isn't one, that's an ADR finding, not a missing section.

Codex Full strictness floor isn't punitive — it's the W4 calibration of the same review surface the cohort has worked through Light (W1), Ramping (W2), and Near-full (W3). By Thu the cohort knows the floor; Mon you write the ADRs that meet it.

> **Glossary:** **Modernization scope ADR** — the architecturally-irreversible commit a pair makes Monday to say *which* legacy debt items they're executing this week. Not all 12 — pick 2–3 + per-pair-unique. **HITL authority boundary** (previewed for Wed): for an AI endpoint, who can call it, what authority the AI exercises, what gate requires a human, what rollback looks like. The LLM06 Excessive Agency translation into FedRAMP language.

## 4. Iterative Spec-Driven Dev — the living discipline you've now practised 3× (8 min)

You wrote `planning/W02/` on W2 Mon. You wrote `planning/W03/` on W3 Mon. Today you write `planning/W04/`. The *artifact* isn't the point — the §0 retro on last week's spec is the point. Spec-driven dev's value isn't "we wrote it down." It's "we go back and ask what the spec got wrong, then we change scope." That's a living discipline, not a Monday-only document.

Tuesday's workshop will deepen this: cohort opens their own merged W2–W3 PRs and discovers the GitHub Actions lint job is `if: false`-disabled (debt item 12 — the OIG-Findings-Tracker meta-joke per inventory line 387). The pair's PRs were never actually linted. **That discovery IS the discipline working.** Spec-driven dev that doesn't notice its own gaps is just paperwork. Mon's ADRs will state "we ship against a linted CI"; Tuesday's workshop tests whether that claim is true.

Continuous Validation Principle: the spec isn't done when it's written; it's done when the next day's reality has been compared against it and the spec has either survived or been amended. The §0 retro is *yesterday's spec, today*. You'll meet a much more violent version of this Friday afternoon — see W4 Fri's pre-session.

## 5. Scenario Design Planning Artifact — the Mon graded artifact (8 min)

Mon EOD-due, graded: the **Scenario Design Planning** six-artifact set. Rubric in `assessments/W04-scenario-design-planning.md`. This is the W4 cadence of a Plan-Day artifact the cohort runs three times in the programme — W2 Mon, W4 Mon, W6 Mon (per D-029). It's the formal output of Plan Day.

The six artifacts:

1. **§0 retro** on W3's spec (the file, not just the conversation).
2. **W4 Modernization Scope ADR** (per §3 above).
3. **AI Security Threat Model ADR** (per §3 above, scoped to Wed's exercises).
4. **Pair-project slice** — which of the 3 pair-project repos owns which slice of the work, and how the per-pair-unique item interacts with the shared `acquire-gov` debt.
5. **Test/validation plan** — what proves Thu's OpenRewrite hop landed cleanly? What proves Wed's Pydantic validators actually fire?
6. **Risk + rollback log** — pair's named "if X breaks, we do Y" rollback steps. Friday's surprise will test at least one of them.

Grading is against the Mon ADR drafts: an ADR can be thin and the brief still passes (W4 codex Full is what scores ADR depth), but the *six artifacts must exist and cohere*. Missing artifacts ≠ graded artifact.

## 6. AI Security pre-read — OWASP LLM Top 10:2025 vocabulary (6 min)

Bridging to Wed, you need recognition-tier vocabulary for OWASP LLM Top 10:2025. Don't go deep tonight — Wed is the depth day. Tonight, three things:

- [OWASP Top 10 for LLM Applications — project page](https://owasp.org/www-project-top-10-for-large-language-model-applications/) (~10 min read), retrieved 2026-05-22 via /web-research. The canonical landing — **note the current revision is 2025 (v2.0), published 2025-03**. Skim the ten IDs.
- [OWASP GenAI Security Project — LLM Top 10:2025 archive](https://genai.owasp.org/llm-top-10/) (~15 min read), retrieved 2026-05-22 via /web-research. Entry-by-entry detail. **Focus on LLM01, LLM06, LLM07, LLM08** — the four you'll exercise Wed.

**Question to bring to morning war-room:** *For each of the 8 ai-orchestrator endpoints in `feature-inventory-target.md`, which OWASP LLM:2025 entries apply most directly?* This question drives the AI Security Threat Model ADR draft Mon afternoon.

> **Glossary:** **OWASP LLM Top 10:2025 (v2.0)** — current canonical revision (published 2025-03, verified 2026-05-22). **Do not cite 2023 IDs.** 2025 LLM07 is *System Prompt Leakage*, not 2023's "Insecure Plugin Design". **Legacy stack** — per D-056 single-branch design, `acquire-gov` main IS SB 2.7.18 + Java 11 + `javax.*` + Spring Security 5 + AWS SDK v1. You modernize forward Thursday.

## 7. Further reading (optional)

- [Spring Boot 2.7 to 3.0 Migration Guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide) (~15 min read), retrieved 2026-05-23 via /web-research. Skim only — Thursday is when this matters; tonight is recognition-tier. Cross-reference `research/spring-boot-2-7-to-4-x-20260525.md` for the cohort's pinned version posture (3.5.14 current 3.x; 4.0.6 current major; 2.7.18 OSS-EoL since 30 Jun 2023).
- [NIST AI RMF 1.0](https://www.nist.gov/itl/ai-risk-management-framework) (~30 min) — companion framework to OWASP. Less prescriptive, but federal-modernization-shaped. Retrieved 2026-05-23 via /web-research. Useful background for W6 security attestation, not W4 Wed.
- *(optional)* [The Lifecycle of a Code AI Tool — Spec-driven dev framing](https://lethain.com/spec-driven-development/) (~10 min read), retrieved 2026-05-23 via /web-research. *Skip if you already have a strong mental model of spec-driven-as-discipline.*

**Looking ahead this week:** Tue deepens spec-driven dev (the lint-disabled discovery). Wed = AI Security depth + **HITL #6** (LLM06 Excessive Agency as authority-boundary design) — Mon's threat-model ADR is the seed. Thu = OpenRewrite hop on `acquire-gov` main per D-056 (the legacy stack is what you're modernizing forward — no sibling branch). Fri = **expect a disruption**; the cohort knows the date, not the shape.

Last verified: 2026-05-26
