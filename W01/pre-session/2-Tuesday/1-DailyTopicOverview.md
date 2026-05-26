---
week: W01
day: Tue
title: "Day-1 orientation reading — programme arc, Claude Code, acquire-gov first-look, brownfield-debt framing, Wed comparative prep"
audience: All cohort members
time_on_task_minutes: 76
read_when: Tuesday morning, alongside the live session as setup runs in the background
last_verified: 2026-05-26
cohort1_compression: "Densest day of W01 — absorbs canonical Mon kickoff (Memorial Day loss) plus canonical Tue conceptual block. 16 raw PDF items compressed into 7 applied-framing topics. The truly heavy Microservices Foundation walkthrough slides to D3 Wed PM (separate pre-session)."
---

# W1 Tue Orientation Reading — Programme arc + Claude Code + acquire-gov + Wed comparative prep

> Tuesday is the densest day of W1. The morning is live in the room (cohort kickoff, Claude Code orientation, `acquire-gov` first-look, Angular tour with three brownfield gaps named). This reading is your **applied frame** — what each block is for, why it matters at a federal client, what to walk in carrying.
>
> **Cohort #1 note:** W1 runs Tue–Fri (4 days). Mon 25 May = Memorial Day, no class. Tuesday absorbs the original Mon kickoff content + the original Tue conceptual block. The heavy Microservices Foundation deep-dive moves to Wed PM — don't try to cram it into Tuesday.
>
> Read this before 09:00 if you can. If your environment isn't all-the-way provisioned yet (Claude Code accounts may not be live for some learners on Day 1), this reading + the repo is the work for any in-between window.

## 1. What this intensive is — programme arc + the 12 FDE situations (10 min)

The Forward Deployed Engineer (FDE) programme prepares software engineers to lead AI-enabled engagements at federal-modernization clients. This six-week intensive (Karsun-FDE v2) is **Phase 2** of the broader arc — Phase 1 is foundational learning before the cohort starts; Phase 3 is six months of on-the-job development after W6 with monthly coach check-ins.

Each day carries up to 12 topics. The shape is **3hr war-room → 3hr practical → 2hr conceptual** — mornings drop you into a real codebase under a constraint, afternoons build, evenings preview tomorrow. It mirrors how an FDE actually works at a client: walk into an unfamiliar system, comprehend it under pressure, ship work that holds up to scrutiny.

The cohort runs against `acquire-gov` — a federal-acquisitions training project with 12 deliberate brownfield-debt items seeded into it. You modernise it across the six weeks. Each pair *also* owns a separate continuous **Pair Project** repo anchored to a real federal-acquisitions aspect (grants management, FOIA, post-award admin) claimed Wed AM. The Pair Project runs both phases: AI Adoption in W1 Thu → W3 Fri, then Modernization W4 → W6, same git history across both.

**The 12 FDE situations** are the vocabulary the next six weeks operate in. You'll see them named every day — read them now so the names don't surprise you:

1. **Data comprehension** — what is this data, who owns it, what's stale, what's missing?
2. **Tech-stack comprehension** — what is this codebase, why is it shaped this way, where are the seams?
3. **Requirements synthesis** — what does the client actually need (vs what they asked for)?
4. **Estimation** — what's this going to cost in time, money, complexity?
5. **6R and system mapping** — Retain / Rehost / Replatform / Refactor / Repurchase / Retire. Knowledge-graph + Context-graph thinking (W3).
6. **Non-functional requirements** — performance, security, observability, compliance, accessibility.
7. **Cloud thinking** — what does production look like on AWS?
8. **DevOps thinking** — how does code get from a PR into production safely?
9. **Database thinking** — relational, document, vector — which when?
10. **Security thinking** — OWASP, OWASP LLM Top 10, multi-tenant boundaries, secret management.
11. **Architecture thinking** — ADRs, trade-off framing, defending decisions under probing.
12. **Design thinking** — user outcomes, not feature lists; what does the contracting officer actually need?

**Two gates and a final defence** anchor the six weeks: **W3 Fri 12 Jun** Phase-1 Defense + Mid-Programme Retro; **W5 Fri 26 Jun** Final Adversarial Review PR; **W6 Thu 2 Jul** Deployment Gate + Client Showcase + Cohort Retro (originally Fri 3 Jul; moved up because Independence Day observed lands on the canonical date).

**Reflection question:** which of the 12 situations are you most comfortable with from prior work, and which are most unfamiliar? Hold that answer for Tue EOD diagnostic.

## 2. Claude Code as comprehension accelerant — install, auth, plugins, custom skills (12 min)

The instructor will demo Claude Code live this morning. The point is **not** that Claude writes the code for you — the point is that you stop grepping. You ask the codebase questions, in English, and verify the answer against the actual file.

That's the muscle. Verifying Claude's output against the source is the FDE discipline. Claude diverging from the README isn't a bug — it's information. Either Claude is wrong **or the README is stale**. Both happen in federal codebases. At a client, you're often the first person to read a system in two years.

**What you'll see this morning:**

- **`.mcp.json` at the repo root** — the MCP (Model Context Protocol) layer. MCP is how Claude Code talks to external systems (Jira, Confluence, Google Drive, Atlassian Rovo). They show up as `mcp__<server>__<tool>` in Claude's tool list. Different from a custom skill (which is a markdown file you write), different from a built-in tool (Read, Edit, Bash). **At a federal client, Karsun's MCP investment is where the leverage is** — Karsun engineers integrate Karsun-owned MCP servers into client work.
- **`.claude/skills/`** — custom skills. Each is a markdown file with YAML frontmatter (`name`, `description`, `triggers`) plus a SKILL.md body that tells Claude what to do, what to inspect, what it must never do. The instructor will walk 2–3 example skills. Skills can invoke tools, spawn sub-agents, write files, run bash, and gate other tool calls. **They're programs, not prompts** — that's the misconception to pre-empt.
- **`.claude/hooks/`** — pre/post-tool guards. They can block sensitive reads (`.env`, `*.pem`) or commits containing AWS keys. You'll trip one or two this week; that's instructive, not an error.
- **`.claude/statusline/`** — the line at the top of the Claude Code interface. Useful for context-budget tracking later in the week.

**Key terms to internalise:** *MCP server* (external-system bridge), *custom skill* (in-repo workflow as markdown), *built-in tool* (Read/Edit/Bash, model-native), *hook* (pre/post-tool guard).

**Claude Code accounts may not be live for everyone on Day 1.** If yours isn't, don't worry — read the repo as-is with your editor, take notes from the projector demo, and we'll bring your account online by Wed AM at latest. The Day-1 hard prerequisite is **Docker Desktop + Git + an editor**, not Claude Code.

**Sub-rule for those whose accounts ARE live:** the cohort default is `/caveman` mode (per D-047 — terse, low-token responses by default). If you're burning your daily quota fast, the first thing the instructor will check is whether your caveman is off.

**Reflection question:** when you walk into an unfamiliar codebase at a client, what's the first three questions you'd ask Claude? Hold that answer — Tue EOD diagnostic asks for the exact prompts.

**Read / watch:**
- [Claude Code docs](https://docs.claude.com/en/docs/claude-code) — install + auth path. Skim only.
- `.mcp.json` and `.claude/skills/` in the `acquire-gov` repo — you'll see these projected; open them yourself if your account is live.

## 3. The acquire-gov training repo — clone, compose, the 4-service mesh (12 min)

`acquire-gov` is your shared training codebase. Think of it as a federal-acquisitions microservices system with deliberate scars — the modernization work across W4–W5 is fixing those scars.

**Four services, three runtimes, two data stores:**

| Layer | Service | Stack | Port |
|-------|---------|-------|------|
| Frontend | Angular SPA | Angular 17 (target: v20/v21 in W4 modernization) | 4200 |
| Edge | `api-gateway` | Spring Boot 2.7.18 + Java 11 + javax (target: SB 3.5+, Java 17/21, jakarta) | 8080 |
| Domain | `solicitation-service` + `evaluation-service` | Spring Boot 2.7.18 + Java 11 | 8081, 8082 |
| AI orchestration | `ai-orchestrator` | Python 3.11 + FastAPI + LangChain v1.0 + LangGraph | 8000 |
| Persistence | Postgres (relational + audit) | 16.x | 5432 |
| Persistence | MongoDB Atlas (Vector Search for RAG, W2+) | Atlas baseline | — |
| LLM provider | AWS Bedrock (Claude) | Sonnet 4.5 / 4.6, Opus 4.5 / 4.6, Haiku 4.5 | — |

**The mesh in one sentence:** Angular → `api-gateway` (JWT validation, tenant routing, rate limiting) → Spring Boot domain services → Python `ai-orchestrator` → Bedrock. Postgres carries domain state + audit; Atlas Vector Search comes online for RAG in W2.

**Why microservices, not a monolith?** Federal procurements increasingly require **6R-mappable** systems — you need to be able to point at one service and say "we're Replatforming this one to ECS Fargate while Retaining that one on EC2 for now." A monolith makes that conversation a year-long migration; a microservices mesh makes it a two-week pull request. The mesh shape *is* the architectural decision the cohort is learning to defend.

**Distributed-systems basics you'll see surface this morning:**
- *Service discovery* — services find each other by DNS name on the Compose network (`api-gateway:8080`, not `localhost:8080`).
- *Health checks* — every service exposes `/actuator/health` (Spring) or `/health` (FastAPI). One service is **missing** its healthcheck on purpose (W4 inventory item).
- *Boot ordering* — Compose's `depends_on` waits for container start, not service-ready. Postgres takes ~5 seconds longer than Spring boot expects; the gateway retries.
- *Logs interleave* — `docker compose logs -f` is messy. Correlation IDs (topic 4) are how you untangle them.

**Container-first.** Every service runs in its own container with its own pinned runtime. **You do not need to install Java / Python / Node on your host.** `docker-compose up` is the truth source. Optional host installs (JDK 21 via SDKMAN, Python 3.11 via pyenv, Node 20 via nvm) are for editor autocomplete + debugger only, not for running the stack.

**Required on your host:**
- **Docker Desktop** — the one hard prerequisite.
- **Git** — clone the repo.
- **VS Code or your editor of choice** — read + edit code.

**Morning preview:** the instructor will run `docker compose ps` showing every service green, then walk through what each service does and where the service-to-service calls live. You'll follow on your own machine. If your stack doesn't come up overnight, that's fine — we triage in the room.

**Read / watch:**
- `training-project/README.md` in the `acquire-gov` repo — architecture diagram + the canonical 12 brownfield-debt items.
- `docker-compose.yml` at the repo root — the service definitions you'll see boot live.

## 4. Auth + observability foundations — OAuth2/JWT + structured logging + correlation IDs (10 min)

Two foundations every federal-acquisitions system needs, both touched today, both deepened Wed PM in the Microservices Foundation walkthrough.

**OAuth2 + JWT — how a request flows through the mesh.**

A user logs into the Angular SPA. The SPA gets back an **OAuth2 access token** (a signed JWT — JSON Web Token). Every subsequent request carries that JWT in an `Authorization: Bearer <token>` header. The `api-gateway` is where validation happens:

1. **Signature check** — gateway pulls the issuer's JWKS (JSON Web Key Set) and verifies the JWT signature.
2. **Claims extraction** — the JWT carries `sub` (subject — who is this user), `tenant_id` (which tenant), `roles` (what can they do), `exp` (when does this expire).
3. **Tenant resolution** — gateway routes the request based on `tenant_id` claim. A multi-tenant federal system can't trust the body of the request; the JWT claim is the load-bearing fact.
4. **Forward** — gateway forwards to the domain service with the JWT still attached (or a downscoped service token, depending on architecture).

**The deliberate gap in acquire-gov:** the Angular SPA has a hardcoded service URL bypassing the gateway (`solicitation.service.ts` calls `solicitation-service:8081` directly, not `api-gateway:8080`). That bypass breaks tenant routing, JWT signature validation, and rate limiting — all of which live at the gateway. The instructor surfaces this live in the Angular tour. **You don't fix it today.** It's inventory item 8 of 12.

**Structured logging + correlation IDs — how you debug a distributed system.**

When a request hits the SPA, gets a JWT, traverses the gateway, hits two domain services, calls the AI orchestrator, and the AI orchestrator calls Bedrock — that's six log streams interleaved in `docker compose logs`. **You cannot untangle this with grep.**

The pattern: every request gets a **correlation ID** at the gateway (UUID). The gateway adds it to the request headers (`X-Correlation-ID`). Every downstream service reads it from the header, adds it to its own log context (via MDC in Java's SLF4J, via `contextvars` in Python), and emits it on every log line. Now `grep <correlation-id>` across all six streams gives you the full request trace.

**Structured logging** is the format. Each log line is **JSON** (`{"ts": "...", "level": "INFO", "service": "api-gateway", "correlation_id": "abc-123", "msg": "..."}`), not human-readable text. JSON lets you ship logs to CloudWatch / Datadog / Grafana and query them by field. The cohort will see *intentionally inconsistent* structured logging across acquire-gov's four services — that inconsistency is one of the deliberate debt items, deepened W4 modernization + W5 AIOps.

**Key terms to internalise:** *OAuth2 access token* (the bearer credential), *JWT claims* (`sub`, `tenant_id`, `roles`, `exp`), *JWKS* (the public-key set used to verify signatures), *correlation ID* (the per-request UUID stitching distributed logs together), *MDC / contextvars* (the language-level mechanism for propagating correlation IDs into log lines).

**Reflection question:** the api-gateway is the only place JWT signature validation happens. What's the security consequence of a bypass route around it? (Hold this for the Angular tour — gap 1 lands live.)

**Read / watch:**
- The `api-gateway/SecurityConfig.java` file in acquire-gov (Spring Security 5.x style on the 2.7.18 baseline — uses the now-removed `WebSecurityConfigurerAdapter`; the W4 modernization migrates to the component-based `SecurityFilterChain` bean). [Spring Boot 3.x research brief](../../../research/spring-boot-2-7-to-3-x-20260525.md) §3 documents the migration.
- The `ai-orchestrator/main.py` middleware that reads `X-Correlation-ID` from the request and pushes it into Python's `contextvars`.

## 5. CI/CD + Bedrock prerequisites — GHA workflows, AWS SSO, model access verification (8 min)

Every PR you open this week runs through GitHub Actions. Every Claude call eventually lands on Bedrock. Both need a quick check.

**GitHub Actions (GHA) — the CI pipeline.**

`acquire-gov` has three workflows under `.github/workflows/`:

- `ci.yml` — runs on every PR. Builds all four services. Runs unit tests. **Currently has a deliberate `--skip-tests` flag hiding broken Angular routes** (debt item 12). You'll see this surfaced live in the Angular tour.
- `debt-enforcement.yml` — runs the brownfield-debt inventory check. Used in W4 modernization to gate "did we actually fix this?" PRs.
- `deploy.yml` — deploys to AWS via OIDC (more below).

**OIDC-to-AWS — what it means.**

Older CI pipelines store long-lived AWS access keys as GitHub secrets. **That's the antipattern.** Long-lived secrets get leaked, get committed by mistake, get used by ex-employees. The modern pattern is **OIDC (OpenID Connect)**: GitHub Actions presents a short-lived JWT to AWS STS, AWS verifies the JWT's `repo:<org>/<repo>` claim against an IAM trust policy, AWS hands back a short-lived role token. **No long-lived keys.** `acquire-gov`'s `deploy.yml` uses OIDC. Karsun's federal posture depends on this pattern.

**AWS SSO — your account access during the cohort.**

We'll walk you through `aws configure sso` Tuesday afternoon. Per D-050, Karsun pays for a single shared AWS account for the cohort — AWS access goes live in **W5** (the AIOps anchor week), not W1. Tuesday's AWS exposure is **read-only Bedrock model-access verification** (the smoke test below) — no full SSO required for that on Day 1; the instructor will run it on the projector with the shared cohort credentials.

**Bedrock model-access verification (the smoke test).**

`aws bedrock list-foundation-models --region us-east-1` should return a JSON list including Claude Sonnet 4.5 / 4.6, Opus 4.5 / 4.6, Haiku 4.5. If your account isn't yet entitled to Claude on Bedrock, the response will be empty for Anthropic models — that's a model-access ticket in the AWS console, not a code problem. The instructor will run this live; you don't run it on your own machine today.

**Pinned model IDs** (current as of 2026-05-22 — see [Bedrock Claude catalog research brief](../../../research/bedrock-claude-catalog-20260522.md) for the full list):

- Claude Opus 4.5 — `anthropic.claude-opus-4-5-20251101-v1:0`
- Claude Sonnet 4.5 — `anthropic.claude-sonnet-4-5-20251101-v1:0`
- Claude Haiku 4.5 — `anthropic.claude-haiku-4-5-20251101-v1:0`

> [!instructor-review]
> If a learner's reading turns up tutorials citing `anthropic.claude-v2` or `anthropic.claude-instant`, those are **stale model IDs** — the known-bad-patterns flag for `bedrock-old-model-ids` applies. Surface in the room.

**Key terms to internalise:** *GHA* (GitHub Actions), *OIDC-to-AWS* (short-lived federated credentials, no long-lived keys), *AWS SSO* (your cohort account access path), *Bedrock model access* (per-model entitlement at the AWS-account level, not a code-level concern).

**Read / watch:**
- `.github/workflows/ci.yml` in acquire-gov — the `--skip-tests` line is the one to flag.
- `infra/github-actions/deploy.yml` — the OIDC trust block. You don't need to understand every IAM policy line; you need to see that no AWS secret is checked in.

## 6. Brownfield debt — what you start surfacing today (12 min)

The 12 deliberate brownfield-debt items are seeded into `acquire-gov` on purpose. **You don't fix them today.** Surfacing is the work. Fixing is W4 modernization. The Tuesday afternoon practical is **starting** the inventory; Wed PM continues it during the Microservices Foundation walkthrough; by EOD Wed you target 12 items captured in `weeks-cohort-202605/W01/brownfield-debt.md`.

**Three items surface live in the Tuesday Angular tour.** They're not abstract — the instructor opens the actual file and runs the actual bad code:

**Gap 1: Hardcoded service URL bypassing the gateway** *(item 8 of 12)*
- File: `frontend/src/app/services/solicitation.service.ts`
- The Angular service calls `solicitation-service:8081` directly. The api-gateway is bypassed entirely.
- Consequence: no JWT validation, no tenant routing, no rate limiting. (See topic 4.)
- *Radioactive question to ask the codebase:* where else is a backend URL hardcoded? Any service-to-service call in the SPA? Any direct service URL in `docker-compose.yml`?

**Gap 2: Missing form validation** *(item ~6 of 12)*
- File: `frontend/src/app/solicitation-new/`
- The form has no `Validators.required` on any control. Empty solicitations post through end-to-end. The backend accepts them. The AI orchestrator calls Bedrock and generates "a solicitation about nothing."
- Consequence: real Bedrock token spend on inputs that should have been rejected client-side. **Cost-attribution is W5 AIOps territory** — today, you just see it.
- *Cohort prompt to hold:* what does this cost in real dollars if a user mashes the submit button 100 times?

**Gap 3: Dead routes shipping past CI** *(item 12 of 12)*
- File: `frontend/src/app/app.routes.ts`
- Two routes (`/reports`, `/admin/audit`) point to components that no longer exist. Compiled errors are **hidden by the `--skip-tests` flag** in `.github/workflows/ci.yml` (see topic 5).
- Consequence: a broken app shipped to staging on every merge. Federal audit-trail concern (clicking "Reports" produces a blank page, not an error message).
- *Cohort prompt to hold:* what does it cost to skip tests in CI? Not just bugs — trust, audit-trail, the federal-CIO conversation.

**The other 9 debt items** (you'll find these across Tue PM + Wed PM):

| # | Item (one-liner) | Surfaces |
|---|------------------|----------|
| 1 | JWT signature validation skipped if header malformed | api-gateway |
| 2 | Spring Boot 2.7.18 / Java 11 / `javax.*` (OSS-unsupported since Jun 2023) | every Spring service |
| 3 | LangChain pinned to `0.1.x` in `requirements.txt` | ai-orchestrator |
| 4 | AWS SDK v1 in build dependencies (general support ended 31 Dec 2023) | every Spring service |
| 5 | LangChain `chain.run()` / `Chain` class usage (removed in v1.0) | ai-orchestrator |
| 6 | Missing form validation (Gap 2 above) | Angular SPA |
| 7 | Missing healthcheck + Postgres volume persistence gap | `docker-compose.yml` |
| 8 | Hardcoded service URL bypassing gateway (Gap 1 above) | Angular SPA |
| 9 | Inconsistent structured logging — JSON in some services, text in others | every service |
| 10 | No correlation-ID propagation across service boundaries | every service |
| 11 | `:latest` Docker image tags (no version pinning) | `docker-compose.yml` |
| 12 | `--skip-tests` flag in CI hides broken routes (Gap 3 above) | `.github/workflows/ci.yml` |

> [!instructor-review]
> Item 3 (LangChain pinned to `0.1.x`) and item 5 (`Chain` class / `chain.run()`) match the `langchain-chain-class` and `langchain-lcel-fundamental` known-bad-patterns. They are **intentionally** in the repo — they're the W2 RAG-week LangChain v1.0 migration material. Don't let a learner "helpfully" upgrade them Tuesday.

**Schema-reading exercise (afternoon practical).** Open the Postgres schema (`infra/postgres/init.sql` or the equivalent Spring JPA entities) + the Mongo Atlas Vector Search configuration (W2 stub). Five minutes of reading the schema tells you what the system thinks the world looks like — which tables have audit columns, which foreign keys are missing, which fields are nullable when they shouldn't be. Schema-reading is **how an FDE comprehends data semantics fast** at a client.

**Each learner names their weakest sub-stack** during the morning's stack tour (Angular 17 / Spring Boot 2.7 / Python+FastAPI / Bedrock+LangChain / Postgres+Atlas / Compose+GHA-OIDC). That naming feeds the pair-assignment diagnostic.

**Morning preview:** the instructor adds items to `brownfield-debt.md` live as the cohort surfaces them. You'll watch the doc grow. By EOD you should have 5+ items captured + the three Angular gaps named.

**Read / watch:**
- `training-project/README.md` in `acquire-gov` — the canonical 12-item list (don't read in advance if you want to discover them live; skim if you want the map first).
- [Spring Boot 3.x research brief](../../../research/spring-boot-2-7-to-3-x-20260525.md) — context on items 2 + 4 (the 2.7.18 → 3.5+ jump + the `javax` → `jakarta` namespace migration).
- [Angular 17+ research brief](../../../research/angular-17-plus-20260525.md) — context on the Angular 17 baseline + post-17 evolution (standalone components, signals, `@if`/`@for` control flow).
- [AWS SDK v1 → v2 migration research brief](../../../research/aws-sdk-v1-to-v2-migration-20260525.md) — context on item 4.

## 7. Prep for Wed — Karsun x GalentAI public-source reading list (12 min)

Wednesday morning is a **3-hour instructor-led GalentAI vs Karsun ReDuX comparative session**. Per D-052, the Galent presenter was cut for Cohort #1 — the instructor authors the session from publicly-available material only. The discipline lesson is itself part of the curriculum: at a federal client, you often work with what's public + your own observations, not with internal vendor decks.

**Three sources to skim before Wed AM:**

1. **[AWS Public Sector Blog — Karsun Solutions builds modernization platform using Amazon Bedrock](https://aws.amazon.com/blogs/publicsector/karsun-solutions-builds-modernization-platform-using-amazon-bedrock/)** (~10 min)
   - What ReDuX is, what it does, why it's on Bedrock. Karsun's federal posture (FedRAMP, ATO precedent).
   - Note: Karsun's actual ReDuX internals are proprietary. Everything publicly knowable comes from this post, the AWS Marketplace listing, and press releases.

2. **[Galent — Claude Managed Agents vs Enterprise AI Platforms](https://galent.com/insights/blogs/claude-managed-agents-vs-enterprise-ai-platforms/)** (~10 min)
   - Galent's positioning of GalentAI as an "EveryOps" platform.
   - How Galent frames itself against hyperscaler-native managed-agent runtimes.

3. **[Galent — homepage / engine listing](https://galent.com/)** (~2 min)
   - 9 proprietary "engines" listed publicly. The architecture lesson is the *horizontal multi-engine* shape vs Karsun's *federal-vertical* shape.

**Don't pre-read** the full research artifact at [`research/galentai-vs-karsun-redux-comparative-20260522.md`](../../../../KarsunFDE/content/research/galentai-vs-karsun-redux-comparative-20260522.md) (~4,000 words, 30+ public sources) — that's the **instructor's** source-of-truth for Wed AM, and your Wed pre-session will walk you to it. Skim §3 (comparative matrix) only if you have appetite.

**The framing to walk in with — six dimensions to bring an opinion on:**

| Dimension | What to bring an opinion on |
|-----------|----------------------------|
| Capability scope | What does each platform actually *do*? AIOps? Modernization? Knowledge management? All three? |
| Market positioning | Federal CIOs? Commercial enterprise? Both? |
| Deployment model | SaaS-only? Self-hostable? FedRAMP authorization status? AWS Marketplace listing? |
| LLM provider | Bedrock-only? Multi-provider? |
| Federal fit | FedRAMP posture? ATO precedent? GovCloud availability? CDAO Tradewinds? |
| Pricing transparency | Who publishes rates? Who doesn't? What does that tell you about their target buyer? |

The Wed pre-session expands this to the 11-dimension matrix the instructor will work through Wed AM — don't try to memorise 11 dimensions tonight; the six above are the load-bearing ones.

**Why the Galent presenter was cut (the discipline lesson).** Per D-052, relying on a vendor presenter for a comparative session is the antipattern — the vendor will say "we are better." The FDE move is **researching the vendor against public sources, building your own comparative**, and bringing a defensible opinion to the client conversation. Wednesday morning models that move. You'll be expected to do it yourself at engagement.

**Morning preview for Wed:** 08:30 final pair announcements + 30-second pair-project pitches + class instant-runoff vote claiming pre-created repos (`grants-portal-modern` / `contract-payment-flow` / `foia-response-pipeline`). Then 09:00–12:00 comparative session. Then 13:00 Microservices Foundation walkthrough (the heavy Tue content that compressed to Wed). Then 14:50 Pair Project repo init.

**Reflection question:** the Galent blog frames managed-agents as "easy entry, no enterprise scale." Bring an opinion on whether that's true *for a federal client* specifically. Hold it for Wed AM.

**Read / watch:**
- The three public links above. Skim, don't deep-read.
- (Optional, if you finish the above before Tuesday evening) The deep-dive at [`KarsunFDE/content/research/galentai-vs-karsun-redux-comparative-20260522.md`](../../../../KarsunFDE/content/research/galentai-vs-karsun-redux-comparative-20260522.md) §1 + §3 only.

## What W1 Tue actually looks like

Cohort kickoff is 09:00 sharp. **Bring your laptop and your curiosity** — Docker Desktop, Git, an editor of your choice, that's the hard requirement. Everything else (Java/Python/Node toolchains, Claude Code auth, AWS SSO) we walk you through live. **No slides Tue morning** — the instructor walks the `acquire-gov` codebase live and models how an FDE comprehends an unfamiliar system. Take notes; your *living* `weeks-cohort-202605/W01/onboarding-patterns.md` doc will reference these patterns through W6.

**Pair assignment is not fixed yet.** Tuesday afternoon's diagnostic + Wed-morning observations feed the pair proposal. Tue EOD posts a proposal; Wed 08:30 announces final pairs.

**Tuesday afternoon practical:** each learner picks their weakest sub-stack and starts orienting in the training project (with Claude Code if accounts are live; with the repo + your editor + this orientation doc if not), plus first pass at the brownfield-debt inventory.

**EOD Tue deliverable (16:00):**
1. *Diagnostic answer* — one paragraph: *"What did you ask Claude (or, if not yet provisioned, would you have asked) that would help you understand the codebase fastest?"* Cite the exact prompt.
2. *Weakest sub-stack named* — one sub-stack, from the morning's stack tour. Feeds pair-assignment.
3. *One thing you want to be better at by W6* — single sentence. Goes in your private 1:1 doc.

If you have time between session blocks and don't yet have an environment ready, **just read the repo.** `CLAUDE.md` at the root, then `pipeline/PIPELINE.md`, then `training-project/README.md`. That reading IS the work for those windows — no homework guilt.

## Sources

- Karsun ReDuX context — AWS Public Sector Blog post (cited in topic 7), retrieved 2026-05-22 via /web-research.
- GalentAI positioning + engine catalog — galent.com homepage + Insights blog (cited in topic 7), retrieved 2026-05-22 via /web-research.
- Spring Boot 3.x current state — [`research/spring-boot-2-7-to-3-x-20260525.md`](../../../research/spring-boot-2-7-to-3-x-20260525.md), retrieved 2026-05-25 via /web-research.
- Angular 17 baseline + post-17 ecosystem state — [`research/angular-17-plus-20260525.md`](../../../research/angular-17-plus-20260525.md), retrieved 2026-05-25 via /web-research.
- AWS Bedrock Claude model catalog — [`research/bedrock-claude-catalog-20260522.md`](../../../research/bedrock-claude-catalog-20260522.md), retrieved 2026-05-22 via /web-research (WebSearch+WebFetch fallback).
- AWS SDK v1 → v2 migration — [`research/aws-sdk-v1-to-v2-migration-20260525.md`](../../../research/aws-sdk-v1-to-v2-migration-20260525.md), retrieved 2026-05-25 via /web-research.
- GalentAI vs Karsun ReDuX comparative deep-dive — [`KarsunFDE/content/research/galentai-vs-karsun-redux-comparative-20260522.md`](../../../../KarsunFDE/content/research/galentai-vs-karsun-redux-comparative-20260522.md), retrieved 2026-05-22 via /web-research.

Last verified: 2026-05-26
