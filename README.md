# KarsunFDE — Content

Cohort-facing async content for the **Karsun-FDE 6-Week Intensive** programme. This repo is what trainees read between live sessions: pre-session primers, war-room scenario structures, weekly assessments, alternatives prompts.

> **Cohort #1 starts Tue 26 May 2026.** Week 1 lives in `W01/`. Subsequent weeks (`W02/` through `W06/`) are added at end-of-each-Friday during cohort #1 delivery.

---

## Programme shape (6 weeks)

| Week | Focus | Phase | Gate |
|------|-------|-------|------|
| W1 (Tue 26 May – Fri 29 May) | Onboarding + LLM Engineering Essentials | Onboarding → Phase 1 begins Thu | — |
| W2 | RAG Architecture (pure RAG) | Phase 1 | — |
| W3 | Agentic Systems (single + multi-agent + HITL + KG/CG) | Phase 1 → Gate | **Phase 1 Gate Fri 12 Jun** |
| W4 | AI-Native SDLC + AI Security + Brownfield Modernization | Phase 2 begins | Mid-Sprint Surprise Fri |
| W5 | AIOps anchor week | Phase 2 | **Final Adversarial PR Fri 26 Jun** |
| W6 (Mon 29 Jun – Thu 2 Jul) | Client Deliverability + HITL Authority Boundaries | Phase 2 → Gate | **Final Defense Thu 2 Jul** |

**Cohort #1 calendar exceptions** — Mon 25 May (Memorial Day) and Fri 3 Jul (Independence Day observed) are lost. W1 runs Tue–Fri (4 days); W6 runs Mon–Thu (4 days); Final Defense moves Fri 3 Jul → **Thu 2 Jul**. All checkpoints unaffected.

---

## How to read this repo

Each `Wnn/` folder contains everything you need for that week:

- **`PLAN.md`** — the week-at-a-glance: topics per day, density notes, special-day callouts.
- **`pre-session/`** — daily reading material. Read each day's `D<n>.md` the evening before that day's live session (or that morning at the latest).
- **`war-room/`** — the morning war-room structure. Don't read this in advance — your instructor walks you through it live. Reference after-the-fact to recall what you covered.
- **`scenarios/`** — Scenario-Alternatives prompts released that week. Pairs respond per `templates/scenario-alternatives.md` shape.
- **`assessments/`** — the week's graded artifact (rotated across MCQ, Codex adversarial review, Live Defense, Scenario-Design plan).
- **`README.md`** — pointer + week-specific notes.

The `COHORT.md` at the repo root has cohort-level constants: pair assignments, instructor, schedule.

---

## What's NOT here (and where to find it)

| Topic | Where it lives |
|-------|----------------|
| Training-project source (Angular + Spring + Python + Bedrock) | [`KarsunFDE/acquire-gov`](https://github.com/KarsunFDE/acquire-gov) (public) |
| Your pair-project source | `KarsunFDE/pair-N-<aspect>` (public, generated W1 Wed) |
| Federal-acquisitions domain references (FAR/DFARS, OIG/OMB) | [`KarsunFDE/domain-knowledge`](https://github.com/KarsunFDE/domain-knowledge) (public) |
| Checkpoint exam materials + audit rubrics | `KarsunFDE/assessment-ec` (**private — not associate-visible**) |
| Curriculum production pipeline (skills, decisions, roadmap) | `KarsunFDE` internal working repo (not associate-visible) |

---

## Refresh cadence

This repo updates **every Friday end-of-day** during cohort #1 delivery — that's when the next week's content lands. If you see a week folder that doesn't have an `assessments/` subdirectory, the assessment artifact is still being authored and will land before the live session that week.

`last_verified` frontmatter on each file shows the most recent content review date. If `last_verified` is more than 30 days old on a hot-tech topic (Bedrock, LangChain, Atlas Vector Search, OWASP LLM Top 10), flag your instructor — federal-AI moves fast and references may have drifted.

---

## Programme contacts

| Role | Contact |
|------|---------|
| Programme lead / instructor | (set at cohort kickoff) |
| Galent presenter (W1 Wed) | (confirmed pre-cohort per programme spec §17) |
| Karsun ReDuX framing source | AWS Public Sector Blog + AWS Marketplace (public-only; ReDuX internals are proprietary) |

---

*Cohort #1 of Karsun-FDE 6-Week Intensive. Programme v2.0 (2026-05-18). This repo seeded 2026-05-22.*
