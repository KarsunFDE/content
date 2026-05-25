---
week: W05
title: "EOD probe questions per day — instructor coaching surface"
audience: instructor
released_at: rolling (one set per EOD)
last_verified: 2026-05-23
---

# W5 EOD Probe Questions — instructor's coaching surface

> Per `PIPELINE.md` §10 cadence, instructor closes each day with a short cohort probe + 1:1 ping for any pair signalling friction. Below are W5-specific probes per day.

## Mon EOD

1. *"Show me your W4 §0 retro write-up. Is Item 3 + Item 2 named as required W5 PR fixes?"* (Filter: did Mon §0 land?)
2. *"What's your HITL #7 starting position — A, B, or C? Reversible after Wed but you should have one."* (Filter: pair engaged the Wed question?)
3. *"Walk me through one ADR from your plan-spec. Where does it forward-reference Wed's research?"* (Filter: plan-spec coherence.)
4. *"What's the worst case if AWS managed services light up wrong this week — cost spike, blast radius, vendor lock-in?"* (Filter: risk register depth.)

## Tue EOD

1. *"Show me the Datadog APM trace view for one `POST /draft-solicitation` request. All 5 services + Bedrock visible?"* (Filter: smoke test passing.)
2. *"What `gen_ai.*` semconv attributes did you set on the Bedrock span? Why those?"* (Filter: cost-as-signal understanding.)
3. *"The AuditEvent.traceparent column — does it populate for *every* AuditEvent insert path, or did you miss one?"* (Filter: completeness for Wed HITL #7 surface.)
4. *"Did the RUM `mask-user-input` setting hold? Vendor proposal content NOT in replay?"* (Filter: LLM02 mitigation.)

## Wed EOD

1. *"Read me your HITL #7 ADR's Decision section. Is it A, B, or C? Why?"* (Filter: ADR commits, not hedges.)
2. *"What FedRAMP control did you cite — AC-5, AU-9, both? Which clause specifically?"* (Filter: regulatory grounding.)
3. *"Walk me through your 3 scenario-alternatives ADRs (W05-SA-1, -2, -3). Are they consistent with each other?"* (Filter: cross-ADR coherence — e.g., picking Datadog AI in SA-1 + Bedrock Agents in SA-3 should align with the HITL #7 choice in SA-2.)
4. *"What did you `/web-research` today that surprised you?"* (Filter: research engagement depth.)

## Thu EOD

1. *"Show me your compare-matrix from this morning. Where does Coralogix beat Datadog?"* (Filter: rigorous compare, not Datadog-loyalty.)
2. *"Your worth-it ADRs for Bedrock KB + Agents-for-Bedrock — what numbers (latency / cost / debuggability) ground them?"* (Filter: ADRs with measurements vs ADRs with stories.)
3. *"If you migrated, walk me through agency_id filter on Bedrock KB. If you didn't migrate, walk me through what you hardened instead."* (Filter: Item 10 multi-tenant preserved.)
4. *"What's left for tomorrow's PR? Show me your scope list."* (Filter: Fri readiness.)

## Fri EOD (after the Final Adversarial PR session)

1. *"Which codex finding did you defend successfully today? Walk me through your argument."* (Filter: Defense Quality dimension.)
2. *"Which codex finding did you accept and commit a fix for in-session? Show me the commit."* (Filter: engagement quality.)
3. *"What's the remediation ticket list for W6? How many P1s deferred, how many P2s, how many P3s?"* (Filter: handoff readiness.)
4. *"Looking at your tier (Distinguished / Proficient / Developing / Below threshold) — what scaffolding do you want from instructor in W6?"* (Filter: self-aware ask for support; sets W6 1:1 schedule.)
5. *"Across the 7 HITL touchpoints — which one did your pair most underestimate?"* (Programme-level reflection; feeds cohort retro.)
