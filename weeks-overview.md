# /weeks/

One folder per programme week. **Karsun-FDE v2 is a 6-week shape** (finalized 2026-05-18 per D-044). Week 0 is embedded in Week 1 (Mon–Wed); Weeks 1–6 are the active programme weeks.

For cohort-specific instantiation, copy this folder to `weeks-cohort-<YYYYMM>/` and work in that copy — per the cohort instantiation checklist in `pipeline/PIPELINE.md` §17.

## Week index (Karsun-FDE v2)

| Week | Topic | Phase | Gate |
|------|-------|-------|------|
| W01 (Mon–Wed) | Embedded Week 0 — onboarding, Claude-Code-assisted PEP refresh, brownfield-debt inventory (Tue), **Galent comparative demo Wed + Pair Project repo initialization** | Onboarding | — |
| W01 (Thu–Fri) | **LLM Engineering Essentials** — Bedrock invocation, structured output, HITL Fri (1st of 7 HITL touchpoints) | Phase 1 begins Thu | — |
| W02 | **RAG Architecture & Agent Integration** (pure RAG depth; agentic content lands W3) | Phase 1 — AI Adoption | — |
| W03 | **Agentic Systems** — single + multi-agent + LangGraph + HITL deep-dive + KG/CG | Phase 1 → Gate | **Phase 1 Presentation & Defense + Mid-Program Retro** (Fri) |
| W04 | **AI Native SDLC & Brownfield Modernization** — spec-driven dev deep-dive, AI Security Engineering Wed, Mid-Sprint Surprise Fri | Phase 2 begins | — |
| W05 | **AI Ops: Governance, Compliance & Deployment Readiness** — full-week AIOps anchor (was v1 W6; was a 1-day anchor in v1) | Phase 2 — operationalisation | **Final Adversarial Review PR** (Fri — v2's replacement for v1 Exit Technical Exam) |
| W06 | **Client Deliverability** — runbook, ADR catalog, eval report, handoff, security attestation, HITL authority boundaries (Wed); Final Defense + Client Showcase + Cohort Retro (Fri) | Phase 2 → Gate | **Deployment Gate / Final Defense** (Fri) |

Every week folder ships with a `README.md` that names the artifact subfolders. The artifacts themselves are produced by the skills listed in `pipeline/PIPELINE.md` §14, populated during each weekly content-authoring pass per §6.

For the mapping from the prior v0.1 10-week shape (W00..W10) to the v1 7-week shape to the v2 6-week shape, see `pipeline/CHANGELOG.md`.

## Archived weeks

- `_archived-v1-W05/` — v1's Production AI / Observability + Validation + Security week. Substance redistributed across v2 W4 + W5; folder kept for audit trail. See its README for the distribution map.
