---
week: W01
title: "Enterprise Engineering & LLM Engineering Essentials"
phase: "Onboarding (Tue–Wed) + Phase 1 begins Thu"
gate: "—"
density_target: "8–12 topics/day"
last_verified: 2026-05-22
cohort1_compression: "Mon 25 May = Memorial Day (US federal holiday). W1 runs Tue–Fri (4 days) for Cohort #1. Original Mon kickoff content compressed into Tue morning; original Tue microservices content compressed into Wed afternoon."
---

# W01 PLAN — Tue–Fri at a glance

> Master plan for Week 1 per `RevaturePro_FDE-Karsun_v2.pdf`, **compressed for Cohort #1's loss of Mon 25 May (Memorial Day).** Tue–Wed are embedded Week 0 onboarding. Thu–Fri begins Phase 1 of the pair Project. Future cohorts without the holiday clash will revert to the canonical 5-day W1 shape; see git history for the pre-compression version.

## Tue — Cohort Kickoff + Full Stack Engineering Refresh I + Angular Tour *(11 topics)*

*Absorbs the original Mon kickoff content + the original Tue afternoon onboarding practical. War-room slot runs in Tour mode (no Incident-mode scenario yet; cohort lacks architectural vocabulary on day 1).*

**Morning (war-room, Tour mode — `war-room/D2.md`)** — Cohort kickoff. Claude Code install + auth + plugin/MCP intro. Custom-skill tour. Instructor walks the training repo + models Claude-Code-assisted onboarding patterns live. **Angular SPA tour — routing, components, services, the brownfield gaps (hardcoded service URL bypassing the gateway, missing form validation, dead routes).**

**Afternoon (practical)** — Each learner picks weakest sub-stack and onboards via Claude Code into the real training repo. `docker-compose up` exploration. CI/CD walked via a real PR through GHA. AWS Bedrock model-access verification. **Brownfield-debt inventory STARTED in `weeks/W01/brownfield-debt.md`.** Onboarding-patterns playbook seeded in `weeks/W01/onboarding-patterns.md`.

**Conceptual (`pre-session/D2.md`)** — FDE programme overview + 12 FDE situations primer. Comparative framing for Wed's instructor-led GalentAI vs Karsun ReDuX comparative session (Galent presenter CUT per D-052; session authored from /web-research on public sources only) + the AWS Public Sector Blog post on Karsun ReDuX + Galent's "Claude Managed Agents vs Enterprise AI Platforms" post (pre-reading for Wed).

**Diagnostic recast (EOD):** *"what did you ask Claude that helped you understand the codebase fastest?"* feeds pair-assignment proposal. **Originally Mon EOD; now Tue EOD due to Memorial Day compression.** Mitigation for the shortened observation window: diagnostic instructions sent to cohort the weekend before so learners can pre-think their answers.

## Wed — Instructor-led Comparative Half-Day + Microservices Foundation + Pair Project Repo Init *(12 topics, at cap)*

*Pair assignments formally announced 08:30 (Wed morning); 30-second pitch + class instant-runoff vote claims pre-created pair-project repos. Afternoon absorbs the original Tue morning Microservices Foundation deep-dive.*

**Morning (3 hr — `war-room/D3.md`)** — Instructor-led GalentAI vs Karsun ReDuX comparative session (Galent presenter CUT per D-052). Grounded in `research/galentai-vs-karsun-redux-comparative-20260522.md` (4,000 words, 30+ cited public sources). Three blocks: GalentAI deep-dive (50min) + Karsun ReDuX deep-dive (50min, post-break) + 11-dimension comparative matrix + "end-goal and capabilities" framing + cohort discussion (60min).

**Afternoon (practical, dense)** — **Microservices Foundation walkthrough** (compressed from original Tue morning): microservices architecture of the training repo (Angular SPA → Spring Boot service mesh → Python/FastAPI AI microservices), service boundaries + bounded contexts, OAuth2 + JWT auth flow end-to-end, structured logging + correlation IDs (intentionally inconsistent), distributed-systems basics. **Containerization deep-dive — multi-stage Dockerfiles, `:latest`-tag antipattern (brownfield-debt item 11), missing healthchecks (item 7), Postgres volume persistence gap (item 7), Compose networking + service-discovery DNS.** **Brownfield-debt inventory CONTINUED.** Pair-authored Comparative ADR (`templates/scenario-alternatives.md` shape). **Pair Project repo initialization** — each pair scaffolds their continuous Phase-1-through-Phase-2 build (`pair-N-<aspect>` under the KarsunFDE org per the locked aspect from `skills/scenario-design-planning/references/karsun-domain-aspects.yml`).

**Conceptual (`pre-session/D3.md`)** — Week 1 Thu LLM engineering essentials pre-reading.

## Thu — LLM Engineering Essentials (Production Mindset) *(10 topics)*

**Morning (first FDE war-room — `war-room/D4.md`)** — LLM engineering scenario in the training repo: instructor frames the production-quality solicitation-drafting endpoint.

**Afternoon (practical)** — Pair Project + training-project: layer production-quality LLM integration into the Spring Boot endpoint that calls the Python service. LLMs as engineering systems; hallucination failure modes; Bedrock invocation; streaming; retry logic; cost structure.

**Conceptual (`pre-session/D4.md`)** — Structured outputs + production API integration.

## Fri — LLM Engineering Continued + **Explicit HITL** + First PR Adversarial Review *(12 topics)*

**Morning (`war-room/D5.md`)** — Cost + reliability for LLM endpoints.

**Afternoon (practical)** — Structured JSON output (Pydantic); output validation gates (Pydantic + Bean Validation contract tests); context engineering; dynamic context assembly; context compression; prompt evaluations + lifecycle management; **explicit HITL framing inside LLM Engineering Essentials** (1st of 7 programme touchpoints per D-043+D-044); async FastAPI integration; idempotency + retry-strategy depth; **Pair Project Phase 1 Day 1 commit**; **First PR Adversarial Review of Plans** (Codex Light per D-034) on each pair's Day-1 plan-spec; Scenario-alternatives prompt "Bedrock vs OpenAI direct" + Light W1 MCQ.

**Conceptual (`pre-session/D5.md`)** — Pre-reading drop for W2 Mon Plan Day.

## Special notes for W1 (Cohort #1)

- **Mon 25 May = Memorial Day, no class.** W1 runs Tue–Fri (4 days). Original Mon kickoff compressed into Tue morning; original Tue Microservices Foundation deep-dive compressed into Wed afternoon. No `D1.md` artifacts authored for Cohort #1's W1 (D2..D5 only).
- **W1 Wed AM = instructor-led GalentAI vs Karsun ReDuX comparative session** (Galent presenter CUT per D-052). Session grounded in `research/galentai-vs-karsun-redux-comparative-20260522.md`. Instructor authors from public sources only; cohort internalises federal-context modesty ("we don't have access to internals").
- **Pair assignment** deferred to **Tue afternoon** based on Tue diagnostic; pairs formally announced Wed 08:30 (then 30-sec pitch + class instant-runoff vote for pre-created repo claim). Less observation window (1 day vs 2 in canonical) — mitigated by sending diagnostic prompts weekend-before so learners pre-think.
- **First Live Defense does NOT run Fri W1** — scenarios won't yet have ADRs. First Live Defense = Fri W2.
- **Codex Adversarial Review starts at Light strictness per D-034** — pairs see findings but most are framed as coaching, not blocking.
- **HITL becomes a programme thread Fri** — explicit framing inside LLM Engineering Essentials. Each pair commits *one* HITL decision in their Phase 1 plan-spec by EOD Fri.
- **Density risk Wed:** at the 12-topic cap. If afternoon Microservices Foundation runs over, defer "Postgres volume persistence gap" deep-dive to Thu morning's first 30 minutes (acceptable carry-over; it lands inside W1 either way).

## Assets in this folder

- `PLAN.md` — this file.
- `pre-session/D2.md` through `D5.md` — pre-session reading material (one per day; **no D1.md for Cohort #1** due to Memorial Day).
- `war-room/D2.md` through `D5.md` — morning war-room scenario (one per day; D3 AM is the instructor-led GalentAI vs Karsun ReDuX comparative session).
- `scenarios/` — scenario-alternatives prompts released this week.
- `assessments/` — Light MCQ Fri + first Codex Adversarial Reviews on PRs.
- `codex-reviews/` — Codex review outputs (populated by `codex-adversarial-review` skill against PR diffs).
- `retros/` — End-of-week retros (3 pair + 1 cohort).
- `brownfield-debt.md` — cohort-authored brownfield-debt inventory (Tue afternoon + Wed afternoon).
- `onboarding-patterns.md` — cohort-owned living doc of Claude-Code onboarding patterns that work.
