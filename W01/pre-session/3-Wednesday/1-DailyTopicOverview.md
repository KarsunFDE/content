---
week: W01
day: Wed
title: "Pre-session — Comparative-session primer + Microservices Foundation + Pair-Project init"
audience: All cohort members
time_on_task_minutes: 50
last_verified: 2026-05-26
cohort1_compression: "AM is instructor-led GalentAI vs Karsun ReDuX comparative (Galent presenter CUT per D-052; session authored from /web-research on public sources only). PM absorbs canonical Tuesday Microservices Foundation walkthrough."
---

# W1 Wed Pre-Session — Comparative AM + Microservices PM + Pair-Project init

> Read **Tuesday evening / before Wednesday 08:30**. ~50 min. Wednesday is a 3-block AM (comparative session) + dense PM (Microservices Foundation + Pair-Project repo init). The cohort lead deliberately did not bring in a Galent presenter, so the AM is **/web-research-driven on the Karsun side too**: an instructor-led tour through public sources only. Your job before Wed is to read the underlying research artifact (linked below) and bring an opinion. (Thursday's pre-session — read Wed evening — installs the LLM Engineering Essentials substrate; lives in `pre-session/4-Thursday/`.)

## 1. Wed at a glance — comparative AM + Microservices PM + Pair-Project init (5 min)

Wednesday has three shapes stacked into one day:

| Time | Block | Output |
|------|-------|--------|
| 08:30–09:00 | Final pair announcement + 30-sec pitches + class instant-runoff vote → each pair claims one of three pre-created repos | Pair + aspect + repo locked |
| 09:00–12:00 | Instructor-led GalentAI vs Karsun ReDuX comparative session (3 blocks: Galent deep-dive 50min + Karsun ReDuX deep-dive 50min + 11-dim comparative + "end-goal" framing 60min) | Cohort shared frame: what the Pair Project builds toward |
| 13:00–13:55 | Microservices Foundation walkthrough on `acquire-gov` (compressed from canonical Tuesday) | Service-mesh + containerization tour; brownfield-debt items annotated live |
| 13:55–14:10 | Brownfield-debt inventory continued | `weeks/W01/brownfield-debt.md` extended toward 7+ items |
| 14:10–14:50 | Pair-authored Comparative ADR (GalentAI vs ReDuX vs build-on-Bedrock-directly) | ADR committed to each pair's repo |
| 14:50–15:45 | Pair-Project repo init: README + ADR-0001 (aspect commitment) + `specs/week-1.md` skeleton | Three repos scaffolded; three aspects locked (no duplicates) |
| 15:45–16:00 | Instructor review, pair-by-pair | One clarifying question + one "what would make this stronger" note per pair |

**08:30 ceremony detail:** three pre-created repos sit on the KarsunFDE org — `grants-portal-modern` (Grants.gov anchor), `contract-payment-flow` (WAWF anchor), `foia-response-pipeline` (FOIA.gov anchor). Pairs pitch for their first-pick aspect; cohort + instructor vote via instant-runoff; lock order. No pair builds on `acquire-gov` itself — that's the shared training-project for modernization exercises across all six weeks. Your pair's project lives in its own repo and runs continuously Phase-1 → Phase-2.

## 2. Reading for the comparative session — what to bring an opinion on (15 min)

**Read this before Wed 09:00.** The instructor will assume you've engaged with the underlying research.

**Source-of-truth artifact (canonical for this session):**

- [`research/galentai-vs-karsun-redux-comparative-20260522.md`](https://github.com/KarsunFDE/content/blob/main/research/galentai-vs-karsun-redux-comparative-20260522.md) — ~4,000 words, 30+ cited public sources, scraped 2026-05-22.

That artifact is the source of every claim made in the AM session. Read at minimum its **§1 (GalentAI deep-dive), §2 (Karsun ReDuX deep-dive), §3 (11-dimension comparative matrix), §4 ("end-goal and capabilities" framing)**. §5 lists 15 discussion prompts the instructor may pull from based on cohort engagement; §6 is the bibliography of source URLs.

**The federal-context posture this AM teaches.** The instructor will say it explicitly Wednesday morning, but internalise it now: *"I do NOT have access to GalentAI internals. I do NOT have access to Karsun ReDuX internals. Everything is publicly available — websites, AWS Public Sector Blog, AWS Marketplace, press releases, conference talks."* This is itself the lesson. At a federal engagement you will rarely have access to a competitor's internals — modesty in claims matters, and citation discipline is the floor.

**The asymmetry to walk in with.** GalentAI is *horizontal-commercial* enterprise-AI (9 engines, 125+ agents, multi-LLM, no federal posture). Karsun ReDuX is *federal-vertical* mainframe-modernization (3-agent loop — Blueprint / Modernization / Verifier — Bedrock-only, CDAO Tradewinds Awardable Oct 2025, AWS Marketplace SaaS). Neither has FedRAMP authorization. They are not direct competitors; they solve different problems for different buyers.

**11 dimensions** the instructor will walk through (full matrix lives in research §3): Capability scope · Target customer · Deployment model · FedRAMP / ATO · LLM provider · Federal fit · Exit cost · Strengths claimed · Gaps / risks · Analyst recognition · Sales motion.

**The load-bearing pedagogical anchor (research §4).** *"You are not building a from-scratch federal-AI platform. You are building the thing that, if validated, becomes a ReDuX accelerator on your pair's specific federal-acquisitions aspect."* That's the frame Wednesday installs. Phase 1 + Phase 2 of your Pair Project map against the 3-agent ReDuX loop — for grants, Blueprint dominates; for contracts (WAWF), Modernization dominates; for FOIA, Verifier dominates.

**Pre-Wed deliverable on your side:** bring an opinion on **one** of the 15 discussion prompts in research §5. The instructor will cold-call. If you read nothing else, read §3 + §4.

Key terms: *neurosymbolic AI · CDAO Tradewinds Awardable · FedRAMP High · GovCloud · 3-agent loop (Blueprint / Modernization / Verifier) · NeuroQL / RCM / Knowledge Graph / Context Graph · Carahsoft channel · AWS Marketplace SaaS · unit-of-measure transparency · ATO (Authority to Operate).*

Reflection (write 2 sentences before Wed AM): *"If a federal CIO asked you today whether to deploy GalentAI or Karsun ReDuX for a grants-management modernization, which would you recommend and what is the strongest counter-argument to your recommendation?"*

## 3. Microservices Foundation primer — what you'll walk Wed PM (15 min)

The Wednesday PM block is the **canonical Tuesday morning Microservices Foundation walkthrough**, compressed into Wed PM because Memorial Day cost the cohort Monday. The instructor leads a 55-minute walkthrough on `acquire-gov` with `docker-compose up` running. You should walk in with `acquire-gov` clean on your machine. If the stack broke overnight, flag it at the 09:00 standup.

**The four-service mesh** (you already saw it in Tuesday's overview; this is the depth pass):

- **Angular SPA** (`frontend/`) — talks to the API gateway via JWT-attached requests. Carries the brownfield bug where one fetch hardcodes a service URL and bypasses the gateway (Tuesday flagged it; Wednesday context: this is the *kind* of debt ReDuX's Blueprint Agent surfaces at scale during legacy discovery).
- **API gateway** (`services/api-gateway/`, Spring Boot 3.x) — validates the JWT, routes to the right service. One downstream service skips claim re-validation post-gateway (deliberate-debt item 1).
- **Spring Boot services** — `solicitation-service` + `evaluation-service`. Service boundaries and bounded contexts: why these splits exist, where they would bend under federal-scale load.
- **Python/FastAPI `ai-orchestrator`** — the LLM-handling service. Thursday's first production LLM PR lands here. Today you walk its current shape: it invokes Bedrock, returns raw JSON downstream, and Spring Boot promptly hits a `KeyError` on the response (deliberate-debt — Thursday's PR fixes it).

**Auth flow end-to-end** — Angular obtains a token from the auth service, attaches it to every gateway call, gateway validates the JWT against the issuer, gateway forwards a stripped or re-signed token to the downstream service. **Watch the gap:** one of the Spring Boot services trusts the gateway-forwarded claim without re-validating it (debt item 1). In federal contexts that is a finding.

**Structured logging + correlation IDs** — every request *should* carry a correlation ID end-to-end (Angular generates, gateway propagates, services log on every line). The training repo logs are **intentionally inconsistent in format across services** (debt item 6). The Wed PM walkthrough surfaces the inconsistency; Wednesday does not fix it. The W4 modernization week comes back to this.

**Distributed-systems basics** — synchronous vs asynchronous paths, where retries live (and don't), where idempotency does and doesn't matter. The `ai-orchestrator` calls Bedrock synchronously today; Thursday's PR adds retry-with-jitter; Friday's PR adds structured-output validation gates.

**Containerization deep-dive** — multi-stage Dockerfiles (the `ai-orchestrator` pins `python:3.11-slim` correctly; the `solicitation-service` Dockerfile uses `:latest`, debt item 11), missing healthchecks on Postgres and the AI orchestrator (debt item 7) which causes cold-up cascading 5xxs, Postgres volume persistence gap (also debt item 7) where data dies on `docker-compose down -v`, Compose networking + service-discovery DNS, why Compose's service-name DNS does not survive production without a real service registry.

Key terms: *bounded context · JWT propagation · claim re-validation · correlation ID · multi-stage Dockerfile · `:latest`-tag antipattern · healthcheck · volume persistence · service-discovery DNS · idempotency.*

Reflection (write 1 sentence): *"Where in the `acquire-gov` mesh would a Karsun ReDuX deployment sit — replace `ai-orchestrator`, wrap `api-gateway`, or sit alongside as a separate modernization service? Why?"*

> [!instructor-review]
> The Wednesday PM block is sized for 55 minutes. If timing slips, the Postgres-volume deep-dive defers to Thursday AM's first 30 minutes per `PLAN.md` Special-notes line 61. Acceptable carry-over; it lands inside W1 either way.

## 4. Pair-Project repo init — what gets scaffolded Wed PM (10 min)

After the Comparative ADR (14:10–14:50), the 14:50–15:45 block scaffolds each pair's repo. The repos are **pre-created** on the KarsunFDE org — empty shells with placeholder READMEs you replace today.

**The three eligible repos for Cohort #1** (one per pair, claimed via the 08:30 vote):

| Repo | Federal-acquisitions aspect | Real-system anchor |
|------|-----------------------------|--------------------|
| [`KarsunFDE/grants-portal-modern`](https://github.com/KarsunFDE/grants-portal-modern) | Grants management | Grants.gov |
| [`KarsunFDE/contract-payment-flow`](https://github.com/KarsunFDE/contract-payment-flow) | Post-award contract administration | WAWF (Wide Area Workflow) |
| [`KarsunFDE/foia-response-pipeline`](https://github.com/KarsunFDE/foia-response-pipeline) | FOIA processing | FOIA.gov |

Aspects + anchors lock today. No two pairs can pick the same aspect. After today, the aspect is the pair's identity for the rest of the programme — the Phase-1 AI adoption work (W1 Thu → W3 Fri) and the Phase-2 modernization work (W4 → W6) both anchor to it.

**ADR-0001 (Karsun aspect commitment)** — every pair commits this today. It says:
- Which aspect (slug from `skills/scenario-design-planning/references/karsun-domain-aspects.yml`).
- Real-system anchor (Grants.gov / WAWF / FOIA.gov).
- Why this aspect + anchor — 1–2 paragraphs, honest reasoning, not a pitch.
- Two-or-three Phase-1 AI capabilities you expect to add (e.g., RAG against grantor-program rules, agentic compliance-clause check, eval harness for clause-coverage).
- Initial guess at the Phase-1 → Phase-2 modernization arc.

**Required initial repo structure** (the instructor will check at 15:45):

```
{your-claimed-repo}/
├── README.md                          ← aspect-grounded what + why
├── specs/
│   └── week-1.md                      ← Day 1 plan-spec skeleton (substance arrives Thu–Fri)
├── docs/
│   ├── adrs/
│   │   ├── 0001-karsun-aspect.md             ← committed today
│   │   └── 0002-comparative-galentai-redux.md ← committed today (pair ADR from 14:10 block)
│   └── codex-responses.md             ← Codex finding log (lands W2 onward)
├── eval/                              ← RAG eval harness arrives W2 Fri
├── src/                               ← initial scaffold per your aspect's stack needs
└── .github/workflows/                 ← CI/CD per pair's choice (lands W2+)
```

**Continuous Phase-1-through-Phase-2.** Same git history. Phase 1 (W1 Thu → W3 Fri) is AI adoption *into* the brownfield you scaffold. Phase 2 (W4 → W6) is modernization *driven by* what Phase 1 surfaced. You do not get a fresh start at W4. Today's commits matter.

Key terms: *ADR-0001 · Karsun aspect · pair-project repo · IRV (instant-runoff voting) · pre-created repo · Phase-1 → Phase-2 arc · continuous git history.*

Reflection (write 2 sentences): *"If you get your first-pick aspect today, what is the strongest Phase-1 AI capability you would commit to in ADR-0001? If you get your third-pick aspect, what is your fallback?"*

## 5. Brownfield-debt inventory — Wed PM continuation (5 min)

Tuesday started `weeks/W01/brownfield-debt.md`. Wednesday PM continues it. Goal by EOD Wed: **7+ items logged.**

Per-debt entry format:

```
ID: BFD-0NN
Location: service/path:line
Symptom: <one line>
Impact: <production / security / observability / cost / dev-experience>
Suspected fix vector: <one line — what kind of change would address it>
W-week target: <W4 modernization / W5 AIOps / etc.>
```

The Wed PM Microservices Foundation walkthrough surfaces several new items live (the instructor will name them: debt item 1 = service skipping claim re-validation; item 6 = logging-format inconsistency across services; item 7 = healthcheck + Postgres-volume gap; item 11 = `:latest`-tag antipattern). You add others as you scaffold your own repo and notice things in `acquire-gov` you didn't see Tuesday.

**Important:** do not fix the debt items. They are the W4 modernization-week material. Inventorying them now is the cohort exercise; fixing them is the W4 cohort exercise.

Reflection (1 sentence): *"Of the debt items surfaced so far, which one would you tackle first if a federal client gave you 2 weeks of solo time on `acquire-gov`?"*

## Sources

- Comparative session research artifact (canonical for Wed AM): [`research/galentai-vs-karsun-redux-comparative-20260522.md`](https://github.com/KarsunFDE/content/blob/main/research/galentai-vs-karsun-redux-comparative-20260522.md), Karsun-FDE content repo, retrieved 2026-05-26 via /web-research (scraped 2026-05-22; 30+ public-source URLs in §6 bibliography).
- AWS Public Sector Blog on Karsun ReDuX (canonical Karsun public source): https://aws.amazon.com/blogs/publicsector/karsun-solutions-builds-modernization-platform-using-amazon-bedrock/ — referenced via research artifact §2.
- Karsun ReDuX AWS Marketplace listing: https://aws.amazon.com/marketplace/pp/prodview-ewysxj3be4vra — referenced via research artifact §2.
- Karsun CDAO Tradewinds Awardable press: https://karsun-llc.com/news/karsun-redux-assessed-awardable-for-department-of-war-work-in-the-cdaos-tradewinds-solutions-marketplace/ — referenced via research artifact §2.
- GalentAI homepage: https://galent.com/ — referenced via research artifact §1.
- GalentAI "Claude Managed Agents vs Enterprise AI Platforms" (May 2026): https://galent.com/insights/blogs/claude-managed-agents-vs-enterprise-ai-platforms/ — referenced via research artifact §1.
- W1 PLAN.md (`KarsunFDE/content/W01/PLAN.md`) — Wed shape + density notes.
- W1 D3 war-room cohort brief (`KarsunFDE/content/W01/war-room/D3.md`) — 08:30 ceremony + EOD deliverables.

Recency categories for the cited material: **hot-tech (3-month window)** — GalentAI public posture, Karsun ReDuX public posture, Bedrock Claude model availability, Codex Adversarial Review tier. **Federal-regulatory (6-month window)** — CDAO Tradewinds Awardable, FedRAMP High / GovCloud posture. **Foundation-stable (12-month window)** — Spring Boot 3.x, Python 3.11, Docker Compose, JWT/OAuth2 patterns.

Last verified: 2026-05-26
