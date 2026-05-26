---
week: W04
day: Fri
topic_slug: failure-handling-patterns
topic_title: "Failure handling patterns — the detect / respond / RCA / fix-or-stage cadence"
parent_overview: W04/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://sre.google/sre-book/emergency-response/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://sre.google/workbook/incident-response/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://incident.io/blog/incident-management-best-practices-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://response.pagerduty.com/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://cloudsecurityalliance.org/blog/2026/04/23/rethinking-incident-response-as-an-engineering-system-addressing-7-operational-gaps
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Failure handling patterns — the detect / respond / RCA / fix-or-stage cadence

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the four phases of the failure-handling cadence (detect → respond → root-cause → fix-or-stage) and explain why each phase is its own deliverable rather than a step on the way to the "real" work.
- Distinguish **mitigation** (buying time, restoring user-visible service) from **resolution** (finding and removing the underlying cause) and explain why production incidents prioritise mitigation first.
- Identify two common anti-patterns — conflating symptom with root cause, and skipping mitigation in favour of immediate root-cause hunting — and the engineering practices that prevent each.
- Define "generic mitigation" in the SRE sense and list three examples that work across most service architectures.
- Explain why "stage for later" is a legitimate outcome of fix-or-stage rather than a failure of the response.

## 2. Introduction

Responding to a failure is an engineering discipline with its own artifacts, measurable outcomes, and anti-patterns — not a degraded form of feature work. It is the named, repeatable cadence that turns a chaotic event into a small set of decisions made in a defensible order with a paper trail behind them.

The cadence has many names across the industry. Google SRE writes it as Prepare → Detect → Respond → Recover → Learn ([Google SRE Workbook](https://sre.google/workbook/incident-response/)). PagerDuty teaches Detect → Triage → Diagnose → Remediate → Continuous Learning. NIST SP 800-61 names Preparation → Detection/Analysis → Containment/Eradication/Recovery → Post-Incident Activity. 2026 practitioner writeups collapse the verbose names into four operational steps that travel between teams — **detect, respond, root-cause, fix-or-stage** ([incident.io 2026](https://incident.io/blog/incident-management-best-practices-2026)).

The vocabulary varies; the underlying claim does not. Each phase exists because skipping it produces a specific class of harm. Skip detection and you don't know what's broken. Skip response and users absorb the cost while engineers chase a root cause. Skip root-cause and the same incident returns next week. Skip fix-or-stage and you ship a fragile Friday patch that breaks Saturday. The discipline is doing the phases *in order*, and recognising when one phase has reached the point where the next can begin.

This reading covers the cadence as a generic engineering discipline. Industry-specific framing appears in the patterns section; application to your specific architecture is the work of your overview document.

## 3. Core Concepts

### 3.1 Detect — naming what's wrong with evidence

Detection converts a vague signal ("the dashboard looks off," "support is getting calls") into a specific, evidence-backed statement of the symptom. The output is a sentence with verb, noun, and measurable surface: *"checkout-service is returning 5xx at 14% of requests, started 14:32 UTC, currently still climbing."*

Google SRE frames detection as the discipline of making the system "tell us when it's broken, or perhaps when it's about to break" ([Google SRE Emergency Response](https://sre.google/sre-book/emergency-response/)). The two ingredients are an alerting mechanism and an oncall process; both are preconditions, neither is the detection itself. Detection is the moment between the page firing and the responder writing down what the symptom *is*.

The most common detection anti-pattern is **conflating symptom and root cause**. A 14% 5xx rate is a symptom; "the database is slow" is a hypothesis. Calling the hypothesis the symptom locks the team into a tunnel — they investigate the database and miss the actual cause (a deploy thirty seconds earlier, a downstream dependency, a feature-flag flip). The detect deliverable is the symptom in evidentiary terms; hypotheses are not detection.

### 3.2 Respond — buying time before chasing root cause

The second phase has the most aggressive prioritisation rule in the cadence: **mitigate before you diagnose**. Google states it explicitly: "always aim to first stop the impact of an incident, and then find the root cause" ([Google SRE Workbook](https://sre.google/workbook/incident-response/)). Incident.io's 2026 guide is sharper: "Mitigation (service restoration) is explicitly prioritized over root cause investigation during active incidents."

The structural reason is asymmetric cost. Every minute of an ongoing incident compounds user harm, audit-log gaps, and responder cognitive load. Root-cause analysis can wait an hour or a day; user-visible impact cannot. The response phase generates a **mitigation action** that reduces the symptom's blast radius — not a description.

SRE literature names a small set of **generic mitigations** that work across most architectures without root-cause understanding:

1. **Roll back the most recent deploy.** If the incident correlates with a release, revert. Reversible; analysis comes later.
2. **Drain or re-route traffic.** Move users away from the affected region/AZ/instance.
3. **Flip a feature flag off.** Auditable, reversible, instant.
4. **Restart the affected process.** Last resort; masks leaks and races, but buys time.

The anti-pattern is the engineer who refuses to mitigate until they "understand what's wrong." Understanding is the *next* phase's deliverable. Mitigating without understanding is the textbook response.

### 3.3 Root-cause — one hypothesis at a time, with evidence

The root-cause phase begins once user-visible impact is contained. Its discipline is **hypothesis-and-evidence**, in that order: state the one explanation that accounts for every observed symptom, then test it against a data point that would have to be true if the hypothesis is correct.

The Cloud Security Alliance's 2026 framing ([CSA — Rethinking Incident Response](https://cloudsecurityalliance.org/blog/2026/04/23/rethinking-incident-response-as-an-engineering-system-addressing-7-operational-gaps)) names two recurring failure modes:

- **Plural hypotheses, no tests.** The team brainstorms five things it "might be," fixes one, declares victory, and never confirms which mattered. The same incident returns three weeks later.
- **Evidence without hypothesis.** The team accumulates log lines without ever stating the explanatory hypothesis. Data becomes noise; the postmortem reads as forensic narrative rather than causal explanation.

The discipline is **one hypothesis at a time, named explicitly, with the confirming or refuting data point cited.**

### 3.4 Fix-or-stage — what ships now, what gets a ticket

The fourth phase converts the root cause into a decision: ship the fix now, or stage for a later sprint with a documented gap. Both are legitimate outcomes; both are deliverables. The phase exists because *not* making the explicit decision is the most common production anti-pattern in the cadence.

Decision criteria are reversibility, blast radius, and time-of-day:

- **Ship now** when the fix is small (single file, narrow surface), reversible, and validated by an existing test path.
- **Stage** when the fix touches multiple services, when the validation surface is unclear, when business hours are ending, or when responder fatigue is high.

"Stage" is not "do nothing." It is "open a ticket with a named identifier, write the gap into a follow-up artifact, and hand off honestly to whoever owns next sprint." The audit trail of the staged decision has the same shape as the shipped decision; only the timeline differs.

The anti-pattern is **heroism** — trying to ship every fix regardless of hour, blast radius, or fatigue. The cost of a heroic Friday-afternoon push is the Saturday-morning failure that lands on an exhausted team. Stage is explicit deferral; heroism is implicit collapse.

## 4. Generic Implementation

A failure-handling runbook is a written artifact that names the four phases, the artifacts each phase produces, and the gates between phases. Below is a generic single-page template that any team can adopt and adapt. Domain-specific framing — the actual symptoms, the actual mitigations, the actual rollback steps — fills in once a team owns the template.

```markdown
# Failure handling runbook — {service or product name}

## Phase 1 — Detect
Symptom (one sentence, with surface + measurement):
  e.g., "{service} returning {error class} at {rate}, started {time}, still {trend}"
Severity (1-4, by your team's SEV definitions): SEV-?
Initial responder (named human, with backup):

## Phase 2 — Respond
Generic mitigations attempted (check all that apply):
  [ ] Roll back most recent deploy            (reversibility: high)
  [ ] Drain or re-route traffic               (reversibility: high)
  [ ] Flip feature flag off                   (reversibility: high)
  [ ] Restart affected process                (reversibility: medium)
  [ ] Degrade gracefully (fallback path)      (reversibility: high)
Mitigation outcome (one sentence): {symptom rate after mitigation}
Time-to-mitigate: {minutes from detect to user-visible recovery}

## Phase 3 — Root-cause
Working hypothesis (one sentence): "{the one explanation}"
Confirming evidence: {data point or log line, with timestamp}
Refuting evidence considered and ruled out:
  - {alternative hypothesis}: {why ruled out}

## Phase 4 — Fix-or-stage
Decision: SHIP NOW | STAGE FOR {sprint identifier}
Rationale (criteria): reversibility / blast radius / time-of-day / responder fatigue
Artifact updated:
  [ ] ADR amendment             at {path}
  [ ] Runbook update            at {path}
  [ ] Follow-up ticket          {ticket id}
Owner of staged work (if staged): {name}
```

The deliberate design property: each phase ends with a written artifact that the next phase can consume. The handoff is paper, not memory. When the responder hands the incident to a daytime team for root-cause analysis, the daytime team reads the runbook instead of asking "what happened?" When the team that drafts the fix-or-stage decision hands the staged work to next sprint, the sprint planner reads the rationale rather than re-litigating the call.

## 5. Real-world Patterns

**E-commerce — Black Friday checkout degradation.** A large online retailer's checkout service degraded under peak load. The first responder spent twelve minutes inspecting the database before the incident commander redirected to mitigation: a region-drain moved 30% of traffic to a healthy AZ within four minutes, dropping the 5xx rate from 18% to 2%. The root-cause hunt then ran in parallel with normal traffic and identified an N+1 query on a rarely-exercised promo-code path. The fix-or-stage decision was *stage*: the team feature-flagged the promo-code path off for the remainder of Black Friday weekend and shipped the proper fix on Tuesday with regression tests. The retailer's published postmortem cited the twelve-minute diagnosis-first delay as the lesson: "mitigate first, diagnose with the traffic still flowing."

**Logistics — last-mile dispatcher overload.** A delivery platform's dispatcher service silently stopped acknowledging route-assignment requests during a Sunday traffic spike. Detection lagged because the dashboard showed "requests received" climbing — the issue was that the *acknowledgement* path had silently broken on a recent deploy. Once the symptom was named correctly (acknowledgement rate, not request rate), the response was an immediate rollback. The root-cause investigation found a validator that silently coerced a `null` field to an empty string, dropping the downstream confirmation. The fix-or-stage decision was *ship* (a one-line validator change), with a staged runbook update to add "acknowledgement rate" to the standing dashboard so future detections would not lag.

A second cluster of real-world patterns — gaming matchmaker latency and healthcare PACS upload failures — appears in the [further-reading appendix](./2-failure-handling-patterns-further-reading.md), with attention to the regulated-industry case where "stage" is structurally unavailable.

## 6. Best Practices

- **Always write the symptom down before forming hypotheses.** The act of writing distinguishes "what we see" from "what we think it might be" and prevents tunnelling on a wrong hypothesis.
- **Mitigate first; diagnose with the traffic still flowing.** Generic mitigations (rollback, traffic drain, feature flag off, restart) are domain-independent and almost always available. Use one.
- **State exactly one root-cause hypothesis at a time, with one confirming data point.** Plural hypotheses without tests produce no understanding; evidence without hypothesis produces no causal claim.
- **Treat "stage for later" as a legitimate, documented outcome.** A staged fix with a ticket, a named owner, and an ADR amendment is a deliverable; a heroically-rushed Friday patch that breaks Saturday is not.
- **Hand off in writing, not in memory.** The runbook is the handoff artifact between phases and between shifts; populate it as the incident progresses.
- **Run a blameless postmortem when the incident closes.** The postmortem is the fifth (Learn) phase. Skipping it means the next identical incident is solved from scratch.

## 7. Hands-on Exercise

**Task (whiteboard or markdown sketch, 10–15 min):** You are the on-call for a small SaaS product in an industry of your choice (NOT federal acquisitions — pick fintech, healthcare, gaming, e-commerce, or logistics). At 14:30 your dashboard shows the primary API returning 5xx at 9% of requests. The most recent deploy was at 14:21.

Walk through the four-phase cadence on paper:

1. **Detect.** Write the symptom in one sentence with surface, measurement, and timestamp. State explicitly what the symptom is *not* (e.g., "this is not a database alert; the database panel is green").
2. **Respond.** Choose one generic mitigation and write the one-sentence rationale. State the expected time-to-effect.
3. **Root-cause.** State the one working hypothesis you would test first, and the single data point that would confirm or refute it.
4. **Fix-or-stage.** Assume root cause is identified as a missing null-check on a request validator that was changed in the 14:21 deploy. Decide: ship the fix now, or stage. Justify in one sentence against reversibility, blast radius, and time-of-day.

**What good looks like.** A correct sketch makes the symptom evidentiary (rate + timestamp + non-symptom), picks a generic mitigation rather than waiting on root-cause clarity (rollback is the obvious choice here), states *one* hypothesis with *one* test (not five hypotheses with vague hunches), and reaches a defensible fix-or-stage decision tied to the criteria rather than to vibes. A common mistake is to skip phase 2 entirely and jump to "investigate the deploy" — note that as the anti-pattern the cadence exists to prevent.

## 8. Key Takeaways

- **What are the four phases of the failure-handling cadence, in order?** Detect, respond, root-cause, fix-or-stage. Each phase produces its own artifact; each artifact is the input to the next phase.
- **Why does respond come before root-cause?** Because user-visible impact compounds while diagnosis runs; mitigation is asymmetrically cheaper than the cost of letting the symptom continue. Generic mitigations (rollback, drain, flag-off, restart) work without root-cause understanding.
- **What is a generic mitigation, and name three?** A domain-independent action that reduces blast radius without requiring root-cause understanding. Three: rollback most recent deploy, drain/re-route traffic, flip feature flag off.
- **Why is "stage for later" a legitimate outcome of fix-or-stage?** Because heroically shipping a fragile fix produces a worse next incident; explicit deferral with a named ticket and ADR amendment is a documented engineering decision, not a failure of response.
- **What is the most common anti-pattern in root-cause analysis?** Either plural hypotheses with no confirming tests (vague brainstorm), or evidence accumulation without a stated hypothesis (forensic narrative without causation). The discipline is one-hypothesis-one-test, named explicitly.

## Sources

1. [Google SRE — Emergency Response](https://sre.google/sre-book/emergency-response/) — retrieved 2026-05-26
2. [Google SRE Workbook — Incident Response](https://sre.google/workbook/incident-response/) — retrieved 2026-05-26
3. [incident.io — Incident Management Best Practices 2026](https://incident.io/blog/incident-management-best-practices-2026) — retrieved 2026-05-26
4. [PagerDuty — Incident Response Documentation](https://response.pagerduty.com/) — retrieved 2026-05-26
5. [Cloud Security Alliance — Rethinking Incident Response as an Engineering System (April 2026)](https://cloudsecurityalliance.org/blog/2026/04/23/rethinking-incident-response-as-an-engineering-system-addressing-7-operational-gaps) — retrieved 2026-05-26

Last verified: 2026-05-26
