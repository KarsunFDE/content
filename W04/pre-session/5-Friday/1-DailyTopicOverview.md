---
template: pre-session-reading
week: W04
day: Fri
phase: SDLC
topic: "Failure Handling Patterns — critical-fix discipline + impact assessment + amended plan-spec + tradeoff evaluation"
estimated_total_minutes: 40
last_verified: 2026-05-26
fde_situations: [1, 2, 6, 7, 8, 11]
tech: [failure-handling-patterns, impact-assessment, amended-plan-spec, Resilience4j, OpenTelemetry, W3C-traceparent]
sources_research_briefs: []
author: instructor
---

# W4 Fri Pre-Session — Failure Handling Patterns + Amended Plan-Spec Discipline

> Read Thu night, before W4 Fri. ~40 min. Tomorrow is the Mid-Sprint Surprise day — you know the date, you don't know the shape. That gap is the design (per D-049). This brief does not tell you what's coming. It loads the *discipline* you'll need regardless of shape: **detect → respond → root-cause → fix-or-stage**. Today's PDF column heading is "Failure Handling Patterns." That heading anchors every topic below. The W5 Mon Plan Day prep at §6 is deliberately short — W4 Fri is not the home for W5's AIOps anchor-week framing; tomorrow afternoon's amended plan-spec is.

## 1. Failure Handling Patterns — the discipline that holds Friday together (8 min)

The PDF column for today is **Failure Handling Patterns**. It is not "incident response" and it is not "debugging." It is the *named four-step discipline* that turns chaos into shippable engineering work, in this order:

1. **Detect** — name what's wrong, with evidence. Observability surfaces (Grafana, application logs, the `/admin/audit` page) before code. The thing you can point at IS the symptom; the thing you have to infer is the root cause. Don't conflate them.
2. **Respond** — buy time. Communicate honestly to stakeholders. Stabilise the user-visible surface (degrade gracefully, fall back, queue work for later) before chasing root cause. *The response is its own deliverable*, separate from the fix.
3. **Root-cause** — hypothesise → test → confirm. One hypothesis at a time. Cite the evidence that confirmed (or refuted) each. The 12 baseline brownfield-debt items in `acquire-gov` (per `training-project/README.md`) are a hypothesis library, not a checklist.
4. **Fix-or-stage** — decide what's shippable today vs what's staged for W5 with a documented gap. Both decisions are legitimate. *Heroically rushing a fragile fix that doesn't survive the weekend is worse than staging the work with an ADR amendment.*

**Question to bring to morning war-room:** *"For whichever step my pair stalls on tomorrow — what observability or artifact would have unstalled us 15 minutes earlier?"* You will use this question Fri EOD in `retros/Fri-post-incident-retro.md`.

> *Glossary inline:* **Detect → Respond → RCA → Fix-or-stage** is the four-step Failure Handling cadence. **Rescue branch** = a per-stage instructor-staged branch (e.g., `stage-J21`, `stage-3.0-rewrite`) you can revert to without losing Thu's work. **Amended plan-spec** = Mon's `planning/W04/` ADRs edited in-place post-incident — see §4.

## 2. Critical Fixes to Brownfield — what makes a fix shippable Friday (10 min)

From the PDF: **Critical Fixes to Brownfield**. When something breaks under load on a legacy stack, the menu of fixes is narrower than it looks. Rank them by reversibility, not by elegance:

1. **Revert to rescue branch** (always available, per D-056 single-branch design). Thu's `stage-J21` and `stage-3.0-rewrite` rescue branches exist exactly for this. Reverting is not failure — it is the cheapest correct fix when load surfaces a regression you can't characterise inside the 30-minute window.
2. **Feature-flag off** (if a flag exists). The Mon planning artifacts should have named the high-risk flips; flipping off is reversible and auditable.
3. **Hot-patch a narrow surface** — one Pydantic v2 validator on `ai-orchestrator`, one Spring Security filter on `api-gateway`, one repository method on `solicitation-service`. The narrowness IS the safety. A 12-line patch in one file is reviewable in 5 minutes; a 200-line patch across three services is not.
4. **Stage for W5 with a documented gap.** Open a ticket, write the gap into an ADR amendment, hand off honestly. *This is the W5 hand-off list in `war-room/D5.md` §3.5.*

The codex Full strictness floor (D-034) does not relax on Friday. Whatever ships, ships with a passing adversarial review — or it doesn't ship, it stages.

**Karsun-applied framing:** "Critical fix" at a federal client means the fix has to survive the OIG observer reading the `/admin/audit` page next week. A fix that papers over a symptom without an audit-log entry IS a finding in waiting. Honesty in the audit trail is the load-bearing constraint.

> *Glossary inline:* **Codex Full strictness (D-034)** = P0/P1 findings block merge; W4 is the first week running at Full. **Pydantic v2** = `ai-orchestrator`'s input validation library (request body → model). See W4 Wed §5 for the security-validation pre-load.

## 3. Impact Assessments — naming what changed under load (8 min)

From the PDF: **Impact Assessments**. After the morning's response phase, the afternoon-shaped artifact is a one-page **impact assessment** per pair. The five named slots:

| Slot | What goes in it |
|------|-----------------|
| **Symptom** | What did users / stakeholders / observers see? Cite the surface (Grafana panel, log line, `/admin/audit` row, browser screenshot). |
| **Root-cause hypothesis** | What's the *one* explanation that accounts for every observed symptom? Name the brownfield-debt item or the modernization-stage edge it touches. |
| **Confirming/refuting evidence** | Which data point confirmed or refuted the hypothesis? *Hypothesis without evidence is a guess; evidence without hypothesis is noise.* |
| **User-visible impact** | What did the failure cost the user / the SSA / the OIG observer / the federal partner? Audit-trail gap? Timing slip? Trust erosion? Name it concretely. |
| **W5 hand-off scope** | What's staged for next week, named with a ticket-shaped identifier? |

This artifact is the substrate for Live Defense at 15:30 (`assessments/W04-live-defense.md`). Instructor probes against the impact assessment, not against Monday's clean plan-spec. The rubric scores honesty, calibration, and the chain from symptom → hypothesis → evidence — not whether the pair "fixed it."

**Question to bring to morning war-room:** *"What's the single observable signal that would have given my pair 15 more minutes?"* The answer is the W5 Mon Plan Day's §0 retro material.

## 4. Amended plan-spec — how Mon's ADRs change post-incident (8 min)

From the PDF: **Amended plan-spec**. This is the topic that ties Friday back to four weeks of spec-driven dev practice. Your pair has now run the §0 plan retrospective four times (W2 Mon / W3 Mon / W4 Mon + Tue workshop). Friday is the fifth — and the first one done under stress, against an event you didn't author.

The discipline: **amend Mon's `planning/W04/` ADRs in-place.** Do not write a new document called "post-incident notes." Open the same files, edit the same sections, leave a git history that shows the edits. Each amended decision gets a `## Amendment (Fri YYYY-MM-DD)` block underneath the original decision, naming:

1. Which Mon decision is being amended.
2. What you observed today that invalidates or refines it.
3. What the amended decision is.
4. Whether the amendment ships today or stages for W5.

**This is iterative spec-driven dev under stress.** The discipline you've practised four times in calm conditions now meets the production-shaped test. The amendment block is the artifact; the *act* of amending it (not deleting Mon's words, not pretending Mon got it right) is the discipline.

**Karsun-applied framing:** at a federal client, the original ADR + the amendment block IS the audit trail. The OIG observer's question next quarter will not be "did you predict the incident?" — it will be "when reality diverged from your plan, did you document the divergence in a way I can read?" The amended plan-spec is that document.

> *Glossary inline:* **`planning/W04/` ADRs** = Mon's six-artifact set (W4 Modernization Scope ADR + AI Security Threat Model ADR + risk register + acceptance criteria + scenario design + plan-spec). See W4 Mon `1-DailyTopicOverview.md` for the Mon authorship pass.

## 5. Amendment Tradeoff Evaluations — what gets cut, what gets shipped (6 min)

From the PDF: **Amendment Tradeoff Evaluations**. The hardest call of the afternoon is *what doesn't fit*. When the amended plan-spec has more work than the remaining hours hold, you choose. The tradeoff matrix has two axes:

|  | Low effort | High effort |
|--|-----------|-------------|
| **High severity** | Ship today (priority 1). | Stage for W5 with named ticket + ADR amendment (priority 2). |
| **Low severity** | Bundle into next clean PR; document but don't rush. | Stage for W5, deprioritise — note in the retro that the brownfield budget didn't reach it. |

The cohort-failure modes the rubric watches for:
- **Heroism** — pair tries to fix everything before 17:00; ships fragile work that the codex Full reviewer P0-blocks at 16:50. Stage instead.
- **Avoidance** — pair stages everything to W5 and ships nothing. At least one critical fix or one impact assessment must ship by EOD.
- **Spec drift** — pair fixes code but never amends Mon's ADRs. The amendment IS the deliverable.

**EOD post-incident retro prompt** (in `retros/Fri-post-incident-retro.md`): *"What changed in your pair's mental model of 'production' between 09:00 and 17:00 today?"* Don't pre-answer it. Bring it back honestly tomorrow.

The PDF also names the **Weekly Survey Feedback form** as a Friday EOD artifact — that's the per-cohort feedback channel (per the no-Slack-for-Cohort #1 toolchain, this is a Google Form the instructor circulates). 5 minutes. Honest. Friction surfaces or none of the W5 calibration is real.

## 6. W5 Mon Plan Day prep — pre-loading AIOps vocabulary (recognition tier, 8 min)

W5 is AIOps anchor week. W5 Mon's own pre-session brief carries the full framing — this section just pre-loads three vocabulary anchors so amendment ADRs that say "stage for W5" can name the right library by Monday morning. **Recognition tier only — no deep-dive here.**

- [Resilience4j — Circuit Breaker Documentation](https://resilience4j.readme.io/docs/circuitbreaker) (~5 min skim), retrieved 2026-05-23 via /web-research. The JVM circuit-breaker / rate-limiter / bulkhead / retry library. The `resilience4j-spring-boot3` starter is the W5 wiring path for services on `stage-4.0` (Resilience4j SB 4 starter is in-flight as of `last_verified`; SB 3 starter remains the canonical entry point for now — surface to instructor if a SB 4 starter has GA'd by cohort delivery). SB 4.0 ships native `@Retryable` + `@ConcurrencyLimit` annotations that cover the simple cases; Resilience4j is the answer when you need full circuit-breaker semantics + metrics.
- [OpenTelemetry — W3C Trace Context (`traceparent`)](https://opentelemetry.io/docs/specs/otel/context/api-propagators/) (~5 min skim), retrieved 2026-05-23 via /web-research. The HTTP-header standard `traceparent: 00-{trace-id}-{span-id}-{flags}` — every service in `acquire-gov` must forward + emit this by W5 Fri. Fixes debt item 6 (inconsistent correlation IDs). SB 4.0's first-party `spring-boot-starter-opentelemetry` wires OTLP export of metrics + traces out of the box.
- [Micrometer Tracing — Spring Boot Integration](https://docs.spring.io/spring-boot/reference/actuator/tracing.html) (~3 min skim), retrieved 2026-05-23 via /web-research. How `stage-4.0` services emit traces post-Thu hop. Replaces Spring Cloud Sleuth (deprecated in SB 3.x).

That's it for W5 prep tonight. The full W5 framing (Datadog AI, AIOps platforms comparative, HITL #7 = auto-remediation authority boundary) belongs in W5 Mon's pre-session — not here.

## 7. Further reading (optional, not required)

- [Resilience4j vs. Hystrix migration](https://resilience4j.readme.io/docs/migration-guide-from-hystrix) (~10 min), retrieved 2026-05-23 via /web-research. Useful if your pair will defend Resilience4j vs Hystrix-legacy in W04-SA-1's amended-ADR follow-up.
- [OWASP LLM10 Unbounded Consumption](https://genai.owasp.org/llmrisk/llm102025-unbounded-consumption/) (~10 min), retrieved 2026-05-22 via /web-research. Ties the W4 Wed AI Security work to general load-failure shapes: unbounded-consumption surfaces are everywhere in legacy stacks, not just LLM endpoints.
- [GitHub Actions OIDC to AWS](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) (~15 min), retrieved 2026-05-23 via /web-research. W5 absorbs AWS deploy via CI/CD onboarding (per D-050). Skip if you've used GHA OIDC before.

---

Last verified: 2026-05-26
