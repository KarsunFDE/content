# W01 · Enterprise Engineering & LLM Engineering Essentials

*Phase:* Onboarding (Mon–Wed) + Phase 1 AI Adoption begins Thu
*Gate:* -
*Project phase:* Pair Project Phase 1 begins Thursday
*Weekly assessments:* Comparative ADR (Wed, un-graded) + MCQ (light, Fri) + Codex on PRs from Thu (Light per D-034)

## v2 day-by-day (per `RevaturePro_FDE-Karsun_v2.pdf`)

| Day | Headline | Anchor topics |
|-----|----------|---------------|
| Mon | Full Stack Engineering Refresh I + Cohort Kickoff + Claude Code install/auth/plugin/skill tour | Training repo clone+build+run; Claude-Code-assisted onboarding patterns; Project container utilization; CI/CD GHA workflows; practical "weakest sub-stack onboard with Claude" |
| Tue | Full Stack Engineering Refresh II | Microservice Project Definition; OAuth2 + JWT auth overview; Structured Logging + Correlation IDs; Distributed systems basics; **Brownfield Debt Inventory** (feeds W4); AWS Bedrock Model Access Verification; Schema Reading for Project Prep for Karsun×GalentAI |
| **Wed** | **GalentAI/Redux Walkthrough** | Architecture mapping of GalentAI; live demo of GalentAI against realistic scenario; GalentAI vs Karsun Redux Comparative ADR; **Pair Project Repo Initialization & Planning** |
| Thu | Project Phase 1 — LLM Engineering Essentials | LLMs as Engineering Systems; Hallucination Failure Modes; Model Selection Criteria; AWS Bedrock Model Invocation; Streaming Responses; Retry Logic & Strategies; Cost Structure of LLMs |
| Fri | Project Phase 1 — First PR Adversarial Review of Plans | Structured JSON Output; Output Validation Gates; Context Engineering; Dynamic Context Assembly; Context Compression Patterns; Prompt Evaluations & Lifecycle Management; **Human In The Loop (HITL)** (1st of 7 programme touchpoints — per D-043 + D-044); Async FastAPI Integrations w/ Spring Boot |

## How this folder gets filled

When the instructor instantiates a cohort, this folder is the destination for all
week-W01 content artifacts. Each subfolder corresponds to an artifact type from
`pipeline/PIPELINE.md` §5.

Daily pre-session reading lives at `pre-session/<N-DayName>/1-DailyTopicOverview.md` — one per teaching day, capped at 12 topics/day per `PIPELINE.md` §4. Authored by `pre-session-author`. See `weeks-overview.md` for which days each week has.

| Subfolder | Artifact type | Skill that authors it |
|-----------|---------------|------------------------|
| `war-room/D1..D5.md` | Morning war-room scenario | `war-room-scenario` |
| `scenarios/W##-SA-#.md` | Alternative-tech scenario (3–5 per week) | `scenario-alternatives` |
| `assessments/` | MCQ (senior + entry), Live Defense, Scenario Design Planning | `mcq-generator`, `live-defense-rubric`, `scenario-design-planning` |
| `codex-reviews/` | Codex Adversarial Review on Karsun PRs | `codex-adversarial-review` |
| `retros/` | End-of-week retros (3 pair + 1 cohort) | `weekly-retro` |
| `PLAN.md` | One-page week plan (Mon–Fri grid) | Instructor (hand-authored, reviewed weekly) |

## Special notes for Week 1

- **Mon–Wed = embedded Week 0** (Claude-Code-assisted onboarding into training repo + microservices foundations Tue + Galent comparative demo Wed morning). See `pipeline/PIPELINE.md` §10.
- **W1 Wed = Galent presenter slot.** Comparative ADR (GalentAI vs Karsun ReDuX) written in the afternoon. **Pair Project repos initialised this same afternoon** (per v2 PDF).
- **Thu–Fri** begins the FDE-content run with LLM Engineering Essentials. Phase 1 of the Pair Project starts Thursday.
- **W1 Fri explicit HITL** is the first of seven HITL programme touchpoints (per D-043 + D-044 narrative). Make it a named callout in the conceptual block.
- **Pair assignment is deferred to W1 Mon afternoon** after the diagnostic — record final pairs in `/weeks-cohort-YYYYMM/COHORT.md`.
- Topic density: 8–12/day (Foundation density). W1 Fri sits near the upper cap with 8 conceptual items + First PR Adversarial Review — pre-session reading scoped tight.
