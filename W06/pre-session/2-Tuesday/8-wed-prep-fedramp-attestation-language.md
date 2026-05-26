---
week: W06
day: Tue
topic_slug: wed-prep-fedramp-attestation-language
topic_title: "Wed-prep — FedRAMP attestation language"
parent_overview: W06/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://www.fedramp.gov/rev5/baselines/
    retrieved_on: 2026-05-26
    recency_category: federal-regulatory
  - url: https://www.fedramp.gov/docs/rev5/playbook/csp/authorization/poam/
    retrieved_on: 2026-05-26
    recency_category: federal-regulatory
  - url: https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final
    retrieved_on: 2026-05-26
    recency_category: federal-regulatory
  - url: https://elevateconsult.com/insights/fedramp-controls-explained/
    retrieved_on: 2026-05-26
    recency_category: federal-regulatory
  - url: https://secureframe.com/blog/plan-of-action-and-milestones-poam
    retrieved_on: 2026-05-26
    recency_category: federal-regulatory
last_verified: 2026-05-26
---

# Wed-prep — FedRAMP attestation language

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish the three FedRAMP baselines (Low / Moderate / High) and identify which one applies to a given system based on data sensitivity.
- Recognise the three attestation states a control can be in (implemented and verified / implemented with named gap / not implemented) and write defensible language for each.
- Identify the mandatory elements of a Plan of Action and Milestones (POA&M) entry and the FedRAMP-mandated remediation timelines by risk class.
- Distinguish "control language" claims from "evidence" — and ensure every "implemented" claim has a linked artifact.
- Pre-empt the most common attestation-language failure (claiming a control is met without naming the evidence that proves it).

## 2. Introduction

FedRAMP is the U.S. federal government's standardised approach to authorising cloud services for federal use. A FedRAMP authorisation is, in practical terms, a structured attestation: the cloud service provider (CSP) lists every applicable security control from NIST SP 800-53 and states, for each one, whether it is implemented and how. The attestation is reviewed by independent assessors (3PAOs) and ultimately by a federal Authorising Official.

The Rev 5 (Revision 5) baseline is the current version as of 2026, derived from NIST SP 800-53 Revision 5 ([NIST SP 800-53 Rev. 5, NIST CSRC, retrieved 2026-05-26](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final); [FedRAMP Controls Explained 2026, Elevate Consult, retrieved 2026-05-26](https://elevateconsult.com/insights/fedramp-controls-explained/)). The Moderate baseline contains 323 controls; the Low baseline ~156; the High baseline considerably more. Each control has a defined parameter set, an enhancement family, and FedRAMP-specific guidance.

The *language* of the attestation is load-bearing. Reviewers read attestations for years, across hundreds of CSPs, and develop a strong sense of what defensible language looks like. A control written up as "we have implemented this" with no evidence linkage reads very differently from "we have implemented this; see `artifacts/auth-policy-v3.pdf` §4 for the configuration baseline and `logs/audit-202602.gz` for an example month of audit events." The second is what makes assessment efficient; the first generates information requests that stall the review.

This reading covers the discipline of attestation language generically. The day's overview applies it to the W6 Wed authoring of `docs/security-attestation.md`; this reading defines the underlying vocabulary and structure.

## 3. Core Concepts

### 3.1 The three baselines

FedRAMP Rev 5 has three primary baselines plus a tailored low-impact (LI-SaaS) baseline:

- **Low** — for systems holding low-impact federal data (no PII, no controlled unclassified information). ~156 controls.
- **Moderate** — for systems holding controlled unclassified information (CUI), employee PII, and "key mission data." 323 controls. The most common baseline for FedRAMP authorisations.
- **High** — for systems holding data whose loss would cause "severe or catastrophic" impact. Considerably more controls and stricter parameter values.
- **LI-SaaS** — a tailored variant of Low that retains the 156 control set but reduces independent-testing burden, with 90 controls addressable via CSP attestation rather than 3PAO testing.

Pick the baseline that matches the data the system holds, not the baseline that's easiest to attest against. Choosing too-low a baseline is a structural defect that surfaces during review; choosing too-high a baseline buries the team in evidence work that the data sensitivity doesn't justify. For systems handling typical federal CUI or employee PII without classified data, Moderate is the right baseline ([Vanta breakdown of FedRAMP baselines, retrieved 2026-05-26](https://www.vanta.com/collection/fedramp/fedramp-levels-baselines)).

### 3.2 The three attestation states

For each applicable control, the attestation must state one of three positions:

1. **Implemented and verified.** "Implemented. Evidence: see `<artifact-path>`." The control is in place and the artifact named demonstrates it.
2. **Implemented with named gap.** "Implemented with named gap. Gap: `<one-sentence>`. Compensating control: `<description>` (if any). Tracked in POA&M as `<POA&M ID>`." The control is operational but a specific limitation is acknowledged.
3. **Not implemented (blocker).** "Not implemented. This is a blocker for authorisation. Remediation owner: `<role>`. ETA: `<date>`. Tracked in POA&M as `<POA&M ID>`." The control is absent; the system cannot pass review until it is implemented.

A common fourth state — "not applicable" — is legitimate but requires a justification. "Not applicable; the system does not perform `<function>` to which this control applies, so the control does not apply." The justification must be defensible — claiming non-applicability for a control that genuinely applies is a red flag for reviewers.

The structural rule: **never claim "implemented" without naming the evidence.** Reviewers learn quickly to distrust attestations that assert without evidencing. The evidence does not have to be in the attestation document itself — a pointer to a separately maintained artifact is fine — but the pointer must exist.

### 3.3 The Plan of Action and Milestones (POA&M)

Every gap declared in the attestation (state 2 or state 3 above) must have a corresponding POA&M entry. The POA&M is the live tracker of remediation work; it is updated monthly during continuous monitoring ([FedRAMP POA&M Playbook, retrieved 2026-05-26](https://www.fedramp.gov/docs/rev5/playbook/csp/authorization/poam/); [Paramify on POA&Ms, retrieved 2026-05-26](https://www.paramify.com/blog/what-are-fedramp-poams-plan-of-actions-and-milestones-explained)).

Mandatory POA&M entry elements (the FedRAMP template defines 30+ columns; the most load-bearing are):

- **POA&M ID** — unique identifier.
- **Affected control(s)** — the NIST 800-53 control(s) the gap relates to.
- **Weakness description** — what is broken or missing, in user-visible terms.
- **Severity** — Low / Moderate / High / Critical.
- **Detection date** — when the gap was found.
- **Scheduled completion date** — when remediation will finish.
- **Milestones** — interim steps toward remediation, with status dates.
- **Responsible party** — the named owner.
- **Status** — Open / On-track / Delayed / Closed.

### 3.4 FedRAMP remediation timelines

FedRAMP enforces specific remediation windows by risk class:

- **Critical risks:** remediated within **30 days** of discovery.
- **High risks:** within **30 days** of discovery.
- **Moderate risks:** within **90 days** of discovery.
- **Low risks:** within **180 days** of discovery.

([Paramify, retrieved 2026-05-26](https://www.paramify.com/blog/what-are-fedramp-poams-plan-of-actions-and-milestones-explained); [Secureframe on POA&M, retrieved 2026-05-26](https://secureframe.com/blog/plan-of-action-and-milestones-poam))

These windows are not aspirational — they are enforced through the continuous-monitoring review. A POA&M item that exceeds its window without re-baselining is itself a finding. A "named gap" disclosed honestly with a remediation date inside the window is the *expected* state; the question is whether the team is meeting its own dates.

### 3.5 OWASP LLM Top 10 cross-mapping

For AI-assisted systems, the security attestation typically also addresses OWASP LLM Top 10 categories ([OWASP LLM Top 10 2025 v2.0](https://genai.owasp.org/llm-top-10/), confirmed canonical via the curriculum research brief at `research/owasp-llm-top-10-20260522.md`). Each LLM-specific risk maps to one or more FedRAMP controls (often AC-3 access enforcement, AC-4 information flow, SI-10 input validation, AU-2 audit events). The attestation language is the same shape: implemented + evidence / implemented with gap / not implemented.

The cross-mapping prevents a common failure mode: an attestation that fully covers FedRAMP controls but misses AI-specific risks because no FedRAMP control was *named* for them. Bringing the OWASP LLM categories into the attestation surface ensures every AI-risk category is explicitly addressed.

### 3.6 Honesty rule

The most important discipline: never claim "implemented" without a linked artifact. Five honest "implemented with named gap" entries with POA&M IDs and remediation dates land much better with reviewers than fifty unqualified "implemented" claims. The reviewer's job is to find the gaps; declaring them yourself converts the engagement from adversarial to collaborative.

This is the same tradeoff-honesty discipline (see the corresponding reading) applied to security controls. A *named* gap is half-closed; a *hidden* gap is whole-open and, if found by the reviewer, is a finding *plus* a transparency concern.

## 4. Generic Implementation

A worked example outside federal acquisitions: a healthcare SaaS preparing a HITRUST CSF attestation. The structure parallels FedRAMP closely. Control under attestation: **HITRUST CSF 01.b — Identification and Authentication of Users.**

**Poor (state 1, no evidence):**

> "We authenticate all users with strong authentication and follow industry best practices."

"Strong" is undefined; "industry best practices" is unverifiable; no artifact. A reviewer immediately generates information requests.

**Better (state 1, with evidence):**

> "**Implemented.** Authentication via SAML 2.0 SSO against the customer IdP, mandatory MFA enforced at the IdP via SAML AuthnContext. Service accounts use mutual TLS with 90-day cert rotation. Evidence: `artifacts/sso-config-v3.yaml`; `policies/auth/mfa-required.md`; `evidence/svc-acct-cert-rotations-2026Q2.csv`. Last reviewed 2026-05-15 by Security Lead. Tested in `reports/pentest-2026-Q1.pdf` §3.4."

Specific protocol, enforcement mechanism, service-account approach; three artifact pointers; last-review date; independent test cross-reference. Verifiable element by element.

**State 2 — implemented with named gap:**

> "**Implemented with named gap.** Standard user auth via SAML 2.0 SSO + MFA. Gap: 3 legacy break-glass admin accounts use static credentials with quarterly rotation. Compensating control: jump-host-only access (`policies/network/jumphost-acl.md`), real-time Security alerting on use, Security-Lead approval required (`runbooks/break-glass.md`). POA&M-2026-031, Moderate, scheduled 2026-08-30 (within 90-day window from detection 2026-05-26). Owner: Identity team lead."

**State 3 — not implemented:**

> "**Not implemented.** Customer-portal user SSO not yet in place; portal users authenticate via username/password with optional TOTP. Blocker for HITRUST Level 3+ on the customer-portal scope. POA&M-2026-009, High, scheduled 2026-06-20 (within 30-day window from detection 2026-05-26). Owner: Customer-portal engineering lead. Compensating control during gap: lockout after 5 failed attempts (`policies/auth/lockout.md`), brute-force monitoring (`monitoring/customer-portal-bruteforce.yaml`)."

Same structural pattern across three states. Honest, bounded, verifiable.

## 5. Real-world Patterns

**Healthcare — HITRUST CSF and HIPAA attestation.** Healthcare cloud providers use HITRUST CSF with strong structural parallels to FedRAMP: defined control families, parameter values, mandatory evidence linkage, and a POA&M-equivalent. The three-state pattern is identical; only the control catalog differs.

**Finance — SOC 2 Type II and ISO 27001.** Sales-driven attestation regimes use the same language pattern: "control implemented; evidence: `<artifact>`." SOC 2 Type II is evidence-heavy — every claim must be backed by an artifact from a defined operating period (typically 6–12 months). Remediation tracking shows up as the "management response" section per finding.

**Aviation — ED-202A / DO-326A security certification.** Civil aviation systems under FAA / EASA security regimes attest against ED-202A / DO-326A catalogs. Same language pattern; the gap tracker is called a "Security Risk Action Plan."

**Automotive — UN R155 CSMS.** Vehicle manufacturers attesting to UN R155 use control-implementation language ("CSMS-required process is implemented as described in `<reference>`, with evidence at `<artifact>`"). The 30/90/180 day remediation pattern appears in adjacent automotive frameworks as severity-tiered SLAs.

## 6. Best Practices

- **Pick the baseline that matches the data sensitivity** (Low / Moderate / High / LI-SaaS). Wrong-baseline choice is a structural defect that surfaces during review.
- **Use the three explicit states** (implemented / implemented with gap / not implemented) per control. Avoid vague language like "mostly implemented" or "partially."
- **Always link evidence for "implemented" claims.** An "implemented" claim without an evidence pointer reads as theatre to experienced reviewers.
- **Open POA&M entries for every gap** declared in the attestation. The POA&M and the attestation must agree.
- **Stay inside FedRAMP remediation windows** by risk class (30/30/90/180 days for Critical/High/Moderate/Low). A POA&M item that exceeds its window without re-baselining is itself a finding.
- **Cross-map AI-specific risks** (OWASP LLM Top 10) to FedRAMP controls so no AI risk goes unaddressed.
- **Prefer five honest "implemented with gap" entries to fifty unqualified "implemented" claims.** Reviewer trust is built on accurate self-reporting.

## 7. Hands-on Exercise

**Time:** 15 minutes. **Format:** rewriting + drafting.

You are writing the attestation entry for a generic control: **AC-2 Account Management** — the control that requires the organisation to identify and manage privileged accounts, including specifying authorised users, account types, and a review cycle.

**Steps:**

1. **Draft a state-1 (implemented and verified) entry** for a system where account management *is* mature. Include at least three named artifacts and a most-recent-review date. Aim for 80–150 words.
2. **Draft a state-2 (implemented with named gap) entry** for the same system where the gap is "service accounts above the threshold are not yet in the quarterly review cycle." Include the compensating control, POA&M ID, severity (you pick — justify with one sentence), scheduled completion within the appropriate remediation window, and owner.
3. **Draft a state-3 (not implemented) entry** for an adjacent control AC-2(13) (Disable Accounts for High-Risk Individuals) where the automation does not yet exist. Include severity, the 30-day window math (use today's date as detection), and a compensating control for the gap window.

**Self-check:**

- Does each entry use the exact state language ("Implemented" / "Implemented with named gap" / "Not implemented")?
- Does every "implemented" or "compensating control" claim have an artifact pointer (even if you invented the path)?
- Does each POA&M entry have an ID, severity, scheduled completion within window, and named owner?
- Did you pre-compute the 30/30/90/180 day window math correctly for the severity you chose?

**What good looks like:** Three entries that read as attestation prose, not marketing copy. Each entry leads with the state label, supports it with specifics, names artifacts, and (for states 2 and 3) carries a POA&M ID + severity + ETA. If your state-1 entry uses words like "robust", "strong", "industry best practice", or "comprehensive" without naming a specific protocol or artifact, you've reproduced the failure mode this reading warns against. Rewrite with specifics.

## 8. Key Takeaways

- Can I name the baseline (Low / Moderate / High / LI-SaaS) that matches my system's data sensitivity?
- For every control I claim is "implemented," can I name the evidence artifact?
- For every named gap, have I opened a POA&M entry with ID, severity, schedule, owner, and milestones?
- Am I inside the FedRAMP remediation windows (30/30/90/180 days for Critical/High/Moderate/Low)?
- Have I cross-mapped AI-specific risks (OWASP LLM Top 10) to FedRAMP controls so no AI risk goes unaddressed?

## Sources

1. [FedRAMP Rev 5 Authorization Baselines (FedRAMP.gov)](https://www.fedramp.gov/rev5/baselines/) — retrieved 2026-05-26
2. [Plan of Action and Milestones (POA&M) — FedRAMP Documentation](https://www.fedramp.gov/docs/rev5/playbook/csp/authorization/poam/) — retrieved 2026-05-26
3. [NIST SP 800-53 Rev. 5 (NIST CSRC)](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final) — retrieved 2026-05-26
4. [FedRAMP Controls Explained: 2026 Compliance Guide (Elevate Consult)](https://elevateconsult.com/insights/fedramp-controls-explained/) — retrieved 2026-05-26
5. [Understanding the Plan of Action and Milestones (Secureframe)](https://secureframe.com/blog/plan-of-action-and-milestones-poam) — retrieved 2026-05-26

Last verified: 2026-05-26
