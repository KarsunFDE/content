---
checkpoint: 3
title: "QC Audit Expectations — Checkpoint 3: AI-Native SDLC + Brownfield Modernization + AI Security"
covers: ["W04 AI-Native SDLC", "W04 AI Security (OWASP LLM Top 10:2025)", "W04 Brownfield Modernization"]
delivery_date: 2026-06-22
last_verified: 2026-06-03
read_time_min: 12
audience: learner
---

# Checkpoint 3 — What to Expect (Mon 22 Jun 2026)

> Learner-facing prep brief. Covers **W4 AI-Native SDLC + AI Security + Brownfield Modernization**. Lands the Monday of W5 — collides with W5 Plan Day. **The exam runs early afternoon, not in the standard Tue slot.** Audit interviews run async across W5.

The exam-time shift (early PM instead of standard Tue) is a Cohort #1 calendar accommodation — Plan Day comes first because W5 work needs to start. Treat the Mon as a long day.

---

## What's different about C3

C1 and C2 tested AI-thinking depth. C3 tests **whether you can execute production-grade SDLC discipline on brownfield code while applying AI-security thinking on top of it.** Three threads interlock:

1. **Spec-driven dev as living discipline** — by C3 you've practiced the §0 plan retrospective 3 times. The auditor expects you to *use* that frame in answers, not recite it
2. **OWASP LLM Top 10:2025** — your pair walked the 8 ai-orchestrator endpoints against LLM01–LLM10. Be ready to map a scenario to a specific OWASP category and defend the mapping
3. **Modernization execution** — the OpenRewrite hop on `acquire-gov` (Spring Boot 2.7.18 → 3.x, Java 11 → 17 mid-week, with W4-target trajectory toward SB 4.0 + Java 21 per the latest research). The auditor will press you on the *order of operations* and *what breaks first*

**HITL #6 (Wed)** landed in W4 — *OWASP LLM06 Excessive Agency* framed as the HITL authority boundary. Be ready to defend where the human MUST sit when an agent has tool-use authority.

The Friday **Mid-Sprint Surprise** (an `acquire-gov` prod incident, Workflow 4 + Item 3 load incident) is fair game. If you ran the surprise, expect the auditor to anchor on it.

---

## Topics in scope

- [W4 PLAN — week shape](../W04/PLAN.md)

**Mon — Phase 2 framing + spec-driven dev + AI Security pre-read**
- [Phase 2 begins — modernization driven by Phase 1 discoveries](../W04/pre-session/1-Monday/2-phase-2-begins-modernization-driven-by-phase-1-discoveries.md)
- [AI backlog generation — pair-project-shaped, human-judgment-gated](../W04/pre-session/1-Monday/3-ai-backlog-generation-pair-project-shaped-human-judgment-gated.md)
- [ADR writing discipline — what Codex full-strictness looks like](../W04/pre-session/1-Monday/4-adr-writing-discipline-what-codex-full-strictness-looks-like-mon.md)
- [Iterative spec-driven dev — the living discipline](../W04/pre-session/1-Monday/5-iterative-spec-driven-dev-the-living-discipline.md)
- [Scenario-design planning artifact — the Mon graded artifact](../W04/pre-session/1-Monday/6-scenario-design-planning-artifact-the-mon-graded-artifact.md)
- [AI Security pre-read — OWASP LLM Top 10:2025 vocabulary](../W04/pre-session/1-Monday/7-ai-security-pre-read-owasp-llm-top-10-2025-vocabulary.md)

**Tue — spec-driven workshop + brownfield modernization planning**
- [Integration mapping — walking service boundaries](../W04/pre-session/2-Tuesday/2-integration-mapping-walking-service-boundaries.md)
- [API modernization patterns — Spring Cloud Gateway](../W04/pre-session/2-Tuesday/3-api-modernization-patterns-spring-cloud-gateway.md)
- [Brownfield planning & analysis (ADR)](../W04/pre-session/2-Tuesday/4-brownfield-planning-and-analysis-adr.md)
- [OpenRewrite primer — the recipe model](../W04/pre-session/2-Tuesday/5-openrewrite-primer-the-recipe-model.md)
- [Validation as spec discipline](../W04/pre-session/2-Tuesday/6-validation-as-spec-discipline.md)
- [ADR writing discipline](../W04/pre-session/2-Tuesday/7-adr-writing-discipline.md)

**Wed — OWASP LLM Top 10 deep dive + HITL #6 + research slot**
- [Prompt-injection testing — stored content surfaces](../W04/pre-session/3-Wednesday/2-prompt-injection-testing-stored-content-surfaces.md)
- [PII protection — multi-tenant boundary validation](../W04/pre-session/3-Wednesday/3-pii-protection-multi-tenant-boundary-validation.md)
- [Spring Security controls — JWT-skip exploit surface](../W04/pre-session/3-Wednesday/4-spring-security-controls-jwt-skip-exploit-surface.md)
- [HITL #6 — Excessive Agency authority-boundary design](../W04/pre-session/3-Wednesday/5-hitl-6-excessive-agency-authority-boundary-design.md)
- [Security validation as automated tests](../W04/pre-session/3-Wednesday/6-security-validation-as-automated-tests.md)
- [Adversarial-review consolidation & spec-driven discipline](../W04/pre-session/3-Wednesday/7-adversarial-review-consolidation-and-spec-driven-discipline.md)
- [Research — alternative scenario tech + Bedrock Guardrails preview](../W04/pre-session/3-Wednesday/8-research-alternative-scenario-tech-and-bedrock-guardrails-preview.md)

**Thu — OpenRewrite execution day**
- [Legacy modernization — single-branch design](../W04/pre-session/4-Thursday/2-legacy-modernization-single-branch-design.md)
- [Incremental migration approaches](../W04/pre-session/4-Thursday/3-incremental-migration-approaches.md)
- [OpenRewrite hop execution](../W04/pre-session/4-Thursday/4-openrewrite-hop-execution.md)
- [Workflow augmentation vs replacement](../W04/pre-session/4-Thursday/5-workflow-augmentation-vs-replacement.md)
- [ADR review — future-hop evidence floor](../W04/pre-session/4-Thursday/6-adr-review-future-hop-evidence-floor.md)
- [Graceful degradation & rollback rehearsal](../W04/pre-session/4-Thursday/7-graceful-degradation-and-rollback-rehearsal.md)
- [Deployment planning for incremental rollouts](../W04/pre-session/4-Thursday/8-deployment-planning-for-incremental-rollouts.md)

**Fri — Mid-Sprint Surprise**
- [Failure-handling patterns](../W04/pre-session/5-Friday/2-failure-handling-patterns.md)
- [Critical fixes to brownfield](../W04/pre-session/5-Friday/3-critical-fixes-to-brownfield.md)
- [Impact assessments](../W04/pre-session/5-Friday/4-impact-assessments.md)
- [Amended plan-spec](../W04/pre-session/5-Friday/5-amended-plan-spec.md)
- [Amendment trade-off evaluations](../W04/pre-session/5-Friday/6-amendment-tradeoff-evaluations.md)

- W4 war-room incidents — `W04/war-room/`. Mon is Plan Day mode; Tue–Fri are practical/incident-driven.

### Cross-cutting

- **OWASP LLM Top 10:2025** categories — LLM01 prompt injection, LLM02 sensitive info disclosure, LLM03 supply chain, LLM05 improper output handling, LLM06 excessive agency, LLM07 system prompt leakage, LLM08 vector/embedding weaknesses, LLM09 misinformation, LLM10 unbounded consumption. You don't need to memorise numbers; you do need to recognise the *shape* and name the category in your own words
- **Spring Boot 2.7 → 3.x migration** — javax → jakarta namespace, Java 11 → 17/21 baseline, Jakarta EE upgrade path, OpenRewrite recipe authority + limits, what OpenRewrite *can't* fix
- **AWS SDK v1 → v2 migration** — builder pattern, AsyncClient, credential providers
- **Angular 17+ modern idioms** — standalone components, control-flow syntax (@if / @for / @defer), Signals
- **FastAPI** — lifespan handlers (deprecated `on_event`), Pydantic v2, async correctness
- **Spec-driven dev** as living discipline — what §0 retro looks like, what makes a plan-spec defensible
- **Mid-Sprint Surprise mechanics** — incident framing, the 12 brownfield-debt items, which surfaced under load

> **Important:** the curriculum directly teaches AI-native SDLC + AI Security + the OpenRewrite hop. It does NOT directly teach Spring Boot 3.x idioms or Angular 17 idioms in depth — but you've been working in those stacks daily on `acquire-gov` and your pair-project. **Auditors press on modernization tech you've used but not been formally taught.** Resources: `research/spring-boot-2-7-to-3-x-*.md`, `research/angular-17-plus-*.md`, `research/python-fastapi-*.md`, `research/aws-sdk-v1-to-v2-migration-*.md` in the working repo.

---

## Example scenarios (illustrative — distinct from your W4 incidents)

### Example A — Tooling strategy for a brownfield modernization

*Supporting reading:* [OpenRewrite primer — the recipe model](../W04/pre-session/2-Tuesday/5-openrewrite-primer-the-recipe-model.md) · [Incremental migration approaches](../W04/pre-session/4-Thursday/3-incremental-migration-approaches.md) · [Brownfield planning & analysis (ADR)](../W04/pre-session/2-Tuesday/4-brownfield-planning-and-analysis-adr.md)

> *"Your tech lead is debating three approaches to the next modernization hop: (1) lean on an automated codemod tool (OpenRewrite or similar) to do the heavy lifting and accept manual cleanup after; (2) hand-write a migration playbook + targeted scripts + a senior engineer driving it; (3) defer the hop and tackle smaller debt items first. Defend a choice for this cohort's stack."*

**What an opening answer should touch:**

- The decision is rarely "which tool" — it's "what does this team need to *understand* about this migration vs what can be reliable plumbing." Automated tools amplify; they don't substitute for understanding
- Frame the trade explicitly: codemod-led is fast on the 80% case but creates a long tail of edge-case manual work that can mask itself as "the tool didn't handle that"; hand-rolled is slower up front but you know exactly which code you touched and why; deferring is sometimes correct but the cost compounds — namespace migrations get harder with every new file written in the old style
- Tie it back to *what evidence you'd use to decide* — not preference. Number of touched files vs number of files that compile after the codemod vs number of tests still passing vs how much non-codified knowledge lives in the build pipeline
- The auditor wants you to land on *"it depends, and here's what I'd measure to make it not depend"* — abstract preferences don't survive the press

**Likely follow-up presses:**
- *How would you know the codemod tool *missed* something? What's your detection plan?* (Build + test + lint + a sampling review of changed files.)
- *Does this decision change if the team is staying on this stack for 6 months vs 3 years?* (Yes — investment in tooling pays back over time, not at the migration boundary.)
- *What does the *unmigrated* state cost you per week the longer you defer?* (Security patches that only ship on the new baseline; talent pool that won't work on the old one.)

### Example B — Authority boundary for an autonomous agent action

*Supporting reading:* [HITL #6 — Excessive Agency authority-boundary design](../W04/pre-session/3-Wednesday/5-hitl-6-excessive-agency-authority-boundary-design.md) · [Prompt-injection testing — stored content surfaces](../W04/pre-session/3-Wednesday/2-prompt-injection-testing-stored-content-surfaces.md) · [Security validation as automated tests](../W04/pre-session/3-Wednesday/6-security-validation-as-automated-tests.md)

> *"An agent in your pipeline has access to a write action — let's say it can post a comment, file an issue, update a record, or recommend an outcome. Some of the agent's input comes from untrusted sources (vendor-submitted content, retrieved corpus, third-party data). Walk me through how you'd reason about whether this agent should have unsupervised authority for that write action."*

**What an opening answer should touch:**

- This is an OWASP LLM06 (Excessive Agency) question, sometimes stacked with LLM01 (Prompt Injection — indirect, when the input is retrieved/processed content). The *number* matters less than the *recognition*: an action's blast radius + reversibility + observability is what determines authority, not how confident the model is
- Reversibility is the load-bearing axis. Reversible action + small blast radius + good observability = candidate for unsupervised authority. Irreversible action OR large blast radius OR weak observability = human-in-loop required, full stop. Federal-acquisitions write actions tilt heavily toward the "human required" side because FAR/DFARS provisions tie specific decisions to specific authorities (CO, SSA) that cannot be delegated to a system
- The right fix is rarely "make the model less likely to do the bad thing" — it's *remove the capability*. If the agent doesn't have the tool, no amount of clever input can invoke it
- Acknowledge the HITL touchpoint #6 framing from W4 Wed — *Excessive Agency* is the literal name of the OWASP category, and your pair already walked your service's surface against it

**Likely follow-up presses:**
- *When the agent flags a high-risk action and a human reviews it, what does the audit-log record need to look like later for an OIG audit?* (The signal that triggered the flag, the human review timestamp, the decision, the correlation ID linking back to the source input.)
- *What if the human-in-loop reviewer is overloaded and starts rubber-stamping?* (Defense-in-depth: anomaly detection on action-pattern drift, second reviewer for high-impact actions, sampling audits.)
- *How do you test this defense before shipping it?* (Adversarial eval set; synthesised injection probes; CI gate that fails the build if the agent invokes the gated action without the gate firing.)

### Example C — Modernization order-of-operations under risk

*Supporting reading:* [Legacy modernization — single-branch design](../W04/pre-session/4-Thursday/2-legacy-modernization-single-branch-design.md) · [OpenRewrite hop execution](../W04/pre-session/4-Thursday/4-openrewrite-hop-execution.md) · [Graceful degradation & rollback rehearsal](../W04/pre-session/4-Thursday/7-graceful-degradation-and-rollback-rehearsal.md)

> *"Your pair wants to bundle three changes in the same PR week: bump the language runtime baseline, swap a major dependency, and migrate to a new web-framework version. Each change touches a lot of files. Walk me through how you'd sequence these — or whether you'd sequence them at all."*

**What an opening answer should touch:**

- The instinct to bundle is wrong here. Each change has its own failure modes, its own rollback story, its own evidence-of-correctness signal. Bundling them means a CI failure tells you the bundle broke; it doesn't tell you which change broke it
- Right ordering generally lands the *least risky, most prerequisite* change first — usually the language baseline bump (because everything else may depend on it), then the framework version (which usually drives dependency-tree changes), then the major dependency swap (because by now you've cleared out a chunk of the noise)
- Each hop gets its own PR, its own CI run, its own rollback boundary. The compound-PR antipattern is one of the most common modernization mistakes
- Tie it to spec-driven discipline — was each of these three changes named in the plan-spec with its own evidence-of-done? If they all got bundled into "modernize the service", that's a plan-quality failure that bites at execution

**Likely follow-up presses:**
- *What's the rollback story if hop 2 ships and hop 3 introduces a regression?* (Atomic hops mean atomic rollbacks; you only revert the broken hop.)
- *What does CI look like during this sequencing? Three feature branches? One long-lived branch?* (Trunk-based with feature flags is generally cleaner than long-lived branches, but the right answer depends on team size and review cadence.)
- *What if a security advisory drops mid-sequence and you need to ship a patch on the *old* baseline?* (Decision: stop the sequence, ship the patch, restart — or accept the patch on the new baseline and accelerate? Names the trade.)

---

## What "meets bar" looks like

For C3 specifically:

1. **Map a scenario to an OWASP LLM category in your own words** — knowing the number is not the bar. Recognising *"this looks like an excessive-agency issue stacked with indirect prompt injection"* IS the bar
2. **Defend an OpenRewrite-recipe decision** — what it touched, what it didn't, what the next manual step is. The auditor doesn't expect you to memorise recipe DSL; they expect you to know *what OpenRewrite can and can't fix*
3. **Use the spec-driven frame in your answer** — when something failed, was the failure *in the plan* or *in the execution*? §0 retro language is fair game
4. **Talk about modernization order-of-operations** — javax → jakarta first, then Java baseline lift, then Spring Boot config classes, then config-property rename, then tests. Knowing the right *order* matters more than knowing each step

Common stuck points:

- **Treating OWASP LLM categories as a memorisation exercise** — you'll spend cycles trying to remember LLM06 vs LLM07. Skip the number; name the shape
- **Spring Boot version confusion** — there are three numbers in play this cohort (2.7.18 baseline, 3.x mid-state from OpenRewrite, 4.0 target trajectory). Know which one you're on and which one you're going to. Don't say *"we upgraded to Spring Boot"* — say which version, from which version, and which test failed first
- **Forgetting the Mid-Sprint Surprise** — if you didn't run it, study the after-action notes. The auditor knows it happened; pretending it didn't is a yellow flag
- **Vague modernization storytelling** — "we ran OpenRewrite" is not an answer. Which recipe, what it touched, what manual work it left, what the rollback story is

---

## How to prep

1. **Pull up your pair's W4 Mod Scope ADR + Threat Model ADR** — re-read both. The auditor will press on the decisions you committed, not on the option space
2. **Walk the OpenRewrite execution timeline in your head** — what you ran, what broke, what you fixed manually. If your pair didn't touch the OpenRewrite hop directly, walk the cohort's
3. **Cheat-sheet the OWASP LLM:2025 categories** — one line each, in your own words. You're not memorising; you're building recognition speed
4. **Read the W4 Mid-Sprint Surprise post-incident notes** if your pair didn't run it. Know the failure mode, the time-to-detect, the response
5. **Skim the four modernization research briefs** — Spring Boot 2.7→3.x, Angular 17+, FastAPI, AWS SDK v1→v2. Don't memorise; orient yourself to where each one lives
6. **Practise saying "this is a brownfield-debt item N" out loud** — auditors will hand you a symptom; you should be able to map back to the inventory item if it's relevant

---

## Logistics

- **Mon 22 Jun is a long day** — Plan Day AM + exam early PM. Plan your hydration + lunch. The exam is shorter (60–90 min) to account for the compressed schedule
- **Audit interview** — async across W5. Book your slot Monday afternoon, ideally for Wed or Thu rather than Tue (Tue is W5 OpenTelemetry day, which is dense)
- **No formal Plan Day artifact due** at the start of the exam — but if your pair didn't finish the W5 Plan Day work AM, expect to feel split-brain. Try to push the Plan Day artifact to mostly-done before lunch

---

*Brief last verified 2026-06-03. SB 4.0 + Java 21 trajectory is the current W4 target per the latest research refresh — re-verify against `research/spring-boot-2-7-to-3-x-20260525.md` and any newer briefs if prepping >30 days post-write.*
