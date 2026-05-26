---
week: W05
day: Thu
topic_slug: deployment-planning-incident-response-runbook
topic_title: "Deployment Planning + Incident-Response Runbook for AI-on-AI Failures"
parent_overview: W05/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 14
sources:
  - url: https://sre.google/workbook/incident-response/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://martinfowler.com/bliki/CanaryRelease.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://aws.amazon.com/opensearch-service/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://sre.google/sre-book/managing-incidents/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
last_verified: 2026-05-26
---

# Deployment Planning + Incident-Response Runbook for AI-on-AI Failures

## 1. Learning Objectives

By the end of this reading, the learner can:

- Articulate the deployment artifacts a production-grade AI stack requires: alerting-as-code, log-retention policy, cutover sequencing, and rollback plan.
- Explain the difference between a generic incident-response runbook and an AI-on-AI failure runbook, and why the AI-on-AI case requires additional documented playbooks.
- Identify the four canonical AI-on-AI failure modes (false-positive cascade, drift-policy misfire, managed-service degradation, audit-of-the-audit recursion) and the response shape for each.
- Recognise the role of canary deployment, blue-green, and shadow-evaluation patterns when rolling out an AI-driven control loop.
- Map a deployment plan onto the policies and workflows already produced (drift policy, escalation workflow, audit logging contract).

## 2. Introduction

Shipping an AI-driven observability and operations stack to production differs from shipping a normal feature in two ways: the system *itself* may take actions in production (auto-remediation, auto-triage, auto-routing), and the system may *fail in ways the team has never seen before* (because the failure modes of AI-driven action are still being catalogued industry-wide). The deployment plan and the incident-response runbook are the team's defense against both.

This reading is the load-bearing artifact for Thursday's Fri-PR pack: a deployment plan that names what the cutover looks like, plus a runbook section that names what happens when the AI-on-AI control loop itself fails. The two are inseparable. Without the deployment plan, the team ships an under-considered cutover. Without the runbook, the team ships a system whose failure modes nobody has thought through.

The patterns here are foundation-stable in SRE (Google's SRE workbook on incident response, Martin Fowler's canary release pattern) with an AI-specific overlay. The AI overlay does not replace SRE practice; it adds named failure modes and documented playbooks to it.

## 3. Core Concepts

### 3.1 Deployment-plan artifacts

A production-grade deployment plan for an AI stack carries at least four artifacts:

- **Alerting-as-code.** Monitors, drift-policy rules, and escalation triggers are committed as configuration — not console-clicked. The config can be expressed in any platform-native alerting DSL (Datadog monitor JSON, Dynatrace settings 2.0, Prometheus AlertManager rules, Grafana alerting YAML, etc.) or wrapped in Terraform; the structural discipline (thresholds, severity routing, escalation paths, alert grouping) is what the review scores, not the vendor choice. The Fri PR includes the config; review of the PR is review of the alerting surface [Google SRE workbook — Incident Response, retrieved 2026-05-26].

> [!instructor-review]
> Examples in this reading use Datadog monitor JSON shorthand where a concrete syntax helps; same alerting-as-code shape applies to Dynatrace, Prometheus AlertManager, Grafana alerting. If the cohort adopts a different platform, substitute the platform's native DSL — the *structural* discipline (thresholds, escalation paths, alert grouping) is what matters.
- **Log-retention policy.** Operational log retention, audit-log retention (per regulatory minimum + incident-response reach-back margin), model-invocation log retention, drift-signal retention. Different stores often have different durabilities and different costs; the policy names each.
- **Cutover sequencing.** Which surface changes when, and in what order, with rollback gates at each step. Cutover sequencing matters disproportionately when one surface (e.g., a new circuit-breaker) protects another surface from amplifying failures.
- **Rollback plan.** Every cutover has a documented rollback. The rollback is not "redeploy the previous version" — it names the specific operational steps: traffic shift, data restoration, audit-record reconciliation, on-call notification.

### 3.2 Canary, blue-green, and shadow-evaluation for AI control loops

The standard release patterns adapt to AI deployment with small twists [Martin Fowler — Canary Release, retrieved 2026-05-26]:

- **Canary.** Route a small percentage of traffic to the new model or policy before broad rollout. For AI control loops (auto-remediation), the canary should be in a low-risk environment first (a non-prod tenant, a low-traffic region) before any prod traffic.
- **Blue-green.** Maintain two parallel deployments and switch traffic atomically. For AI policies, this means keeping the previous policy version live and routable for instant rollback if the new policy misfires.
- **Shadow-evaluation.** Run the new model or policy on the same traffic as the current one but do not act on its outputs. Compare outputs over a defined window before promoting. Shadow-evaluation is especially useful for AI control loops because it catches drift-policy misfires before any production action is taken.

A policy that says "we auto-remediate" without specifying how it canaries, blue-greens, or shadow-evaluates is missing the deployment surface; the deployment surface is where production-readiness is earned.

### 3.3 What an AI-on-AI failure runbook covers

A normal incident-response runbook covers known failure modes of the system being operated. An AI-on-AI runbook covers the named ways in which the AI-driven observation and remediation stack can itself fail. The four canonical modes:

#### 3.3.1 False-positive cascade

The anomaly detector flags a benign event (a deploy, a planned capacity change, a known maintenance window) as drift. If the policy authority allows auto-remediation, the auto-remediator starts taking action — replaying audit events, tripping circuit-breakers, rolling back deploys — that the event did not require. The cascade can amplify itself if the auto-remediation produces signals that the detector also flags.

The runbook section:

- **Kill-switch on the auto-remediation surface.** A single config flag (or equivalent) that disables auto-remediation across the stack. This must exist before the system goes live.
- **Manual rollback procedure** for whatever the auto-remediator did, with audit reconciliation. If the auto-remediator rolled back a deploy, the runbook names the steps to re-deploy with provenance.
- **Reconciliation script.** A tool that reads the audit log of auto-remediation actions taken in the false-positive window and produces a report of what to undo or accept.

#### 3.3.2 Drift-policy misfire

A drift-monitoring rule's threshold was set too aggressively, so the rule fires every few minutes during a normal traffic pattern. On-call gets paged repeatedly; alert fatigue kicks in.

The runbook section:

- **Threshold-revision procedure.** Tier 2 escalation has authority to revise the threshold; the revision requires an ADR amendment and is itself audit-logged. The revision is *not* a silent on-call action.
- **Paging-suppression window.** A documented procedure for temporarily suppressing pages from the misfire rule while the revision is in flight. The suppression window writes an audit record and has an automatic expiry.
- **Post-incident threshold-review.** Drift thresholds that fire on noise lose on-call trust; the runbook requires a quarterly threshold-review meeting where mis-firing rules are revised.

#### 3.3.3 Managed-service degradation

A managed service the AI stack depends on (Knowledge Bases, Agents, OpenSearch managed [AWS OpenSearch Service, retrieved 2026-05-26], etc.) experiences degradation. Ingestion stalls, calls 5xx, latency spikes.

The runbook section:

- **Failover to retained custom implementation.** If the migration ADR chose "hybrid," the custom implementation is still warm and can absorb load. The runbook names how to flip traffic.
- **Vendor escalation procedure.** Open the support case, attach trace IDs from the affected window, note the regulatory impact (if applicable). Vendor-side resolution can take hours; the team needs its own mitigation in parallel.
- **Incident class.** Per FedRAMP and equivalent frameworks, a managed-service-side failure with regulatory implications is an incident-response (IR-4) class event; the runbook names the regulatory-notification trigger.

#### 3.3.4 Audit-of-the-audit recursion

The AuditEvent table itself becomes the source of an AIOps signal — a watchdog rule flags unusual audit-table write rates as anomalous — that itself writes to the AuditEvent table. A feedback loop forms; the audit table grows; the rule keeps firing.

The runbook section:

- **Rate-limit on audit-table writes from AIOps actions.** The kill-switch surface, but specific to audit-relevant action recursion.
- **Debounce window.** Repeated firings of the same audit-table rule inside a window collapse to a single AuditEvent.
- **Manual audit-chain verification.** The team runs a script that walks the audit chain and identifies recursive writes, producing a reconciliation report.

### 3.4 The deployment plan references the policies

A deployment plan is the last artifact in a chain. It references:

- The **authority policy** (what actions are permitted).
- The **drift-monitoring policy** (what signals are watched, at what thresholds).
- The **escalation workflow** (what tiered response runs on each signal).
- The **audit-logging contract** (what record gets written for each automated action).

The deployment plan does not duplicate these; it names them and pins their versions. When a future incident-response retro asks "what was the policy?" the deployment plan points at the answer.

## 4. Generic Implementation

A deployment-plan + runbook skeleton, vendor-neutral:

```markdown
# Deployment Plan + Runbook — <system> — <date>

## 1. Cutover sequencing
- 1.1 Pre-cutover validation (alerting-as-code merged; drift policy committed; escalation workflow on file)
- 1.2 Phase 1 — Shadow-evaluation in production (no actions taken; outputs logged)
- 1.3 Phase 2 — Canary cutover (5% of low-risk tenant traffic, 48h soak)
- 1.4 Phase 3 — Broad cutover (full traffic, with blue-green rollback warm)
- 1.5 Phase 4 — Decommission previous surface (only after 14 days of clean operation)

## 2. Rollback plan
- 2.1 Traffic shift back to previous surface (blue-green flip; estimated 60 seconds)
- 2.2 Audit-record reconciliation (script reference: tools/reconcile-audit.py)
- 2.3 On-call notification (page: oncall-llm; comms channel: #incidents)

## 3. Alerting-as-code references
- 3.1 Drift policy: drift-monitoring-policy.yaml v1
- 3.2 Escalation workflow: escalation-workflow.yaml v1
- 3.3 Authority policy: authority-policy-v3

## 4. Log retention
- 4.1 Operational logs — <observability vendor> 30 days
- 4.2 Audit logs — append-only DB, <retention> per regulatory minimum
- 4.3 Model invocation logs — <retention> for forensic reach-back
- 4.4 Drift signal logs — 90 days rolling baseline

## 5. AI-on-AI runbook sections
- 5.1 False-positive cascade — kill-switch, rollback, reconciliation
- 5.2 Drift-policy misfire — threshold-revision procedure, suppression window
- 5.3 Managed-service degradation — failover, vendor escalation, incident class
- 5.4 Audit-of-the-audit recursion — rate-limit, debounce, manual verification
```

A code reader gets a single immediate test:

```python
def emergency_kill_switch(component: str, reason: str, actor: str) -> None:
    """Disable an auto-remediation surface immediately, with full audit."""
    set_config_flag(component, "auto_remediation_enabled", False)
    write_audit_event(
        actor=actor,
        action="EMERGENCY_KILL_SWITCH",
        target=component,
        outcome="DISABLED",
        extra={"reason": reason},
    )
    page_oncall(severity="SEV-1", message=f"Kill-switch flipped on {component}: {reason}")
```

Three things to notice:

- The kill-switch is a documented, named function — not a "we'll comment out the code." The runbook references this function by name.
- The kill-switch itself produces an AuditEvent with the human actor and the reason; post-incident review can reconstruct who killed what, when, and why.
- The kill-switch pages on-call as SEV-1 — flipping the switch is itself a high-impact event because it disables an entire stack of automated protection.

## 5. Real-world Patterns

**Healthcare — deployment of an early-warning sepsis-risk model.** A hospital network rolled out their early-warning model in three phases: shadow-evaluation in one ward for two weeks (no clinical action taken; predictions logged); canary in three wards with bedside-alert-only action (no auto-paging) for one month; broad rollout with full alerting after the canary window showed no false-positive cascade. The deployment plan referenced the alerting policy and the audit-logging contract by name and version. The runbook included a kill-switch for the auto-paging surface that was exercised in two drills before broad rollout [Google SRE workbook — Incident Response, retrieved 2026-05-26].

**Fintech — managed-service degradation during a tax-season peak.** A consumer-finance company's risk-monitor depended on a managed vector store. During tax season the managed service's ingestion stalled. The runbook section for managed-service degradation pointed to the team's hybrid implementation (vector search retained on a custom Atlas-backed corpus for emergency use) and the vendor-escalation procedure. The team flipped traffic in 4 minutes and opened a sev-2 vendor ticket; total user impact was a 10-minute latency spike rather than a full outage.

**E-commerce — false-positive cascade on a Black Friday eve.** A retailer's anomaly detector flagged the start-of-Black-Friday traffic surge as anomalous (because the model had learnt a baseline that did not include peak-season traffic). The auto-remediation rolled back a deploy and tripped a circuit-breaker; revenue dropped for 12 minutes. The post-incident review produced a new runbook entry: "scheduled high-traffic events require the detector to enter shadow-only mode for the window." The runbook revision was itself committed and audit-logged [Martin Fowler — Canary Release, retrieved 2026-05-26].

**Gaming — audit-recursion at launch.** A studio's anti-cheat audit pipeline saw audit-table write rates spike at game-launch time. A separate AIOps rule flagged the audit-table write rate as anomalous and wrote its own audit row — which the rule then re-flagged. The recursion was caught in canary because the canary window included a simulated launch-day traffic shape. The runbook entry that resulted: "audit-table-write-rate alerts must be rate-limited to one firing per hour and must exempt audit-rule-self-writes." The launch went out with the rule revised [Google SRE workbook — Incident Response, retrieved 2026-05-26].

## 6. Best Practices

- Commit alerting-as-code; never ship policies that exist only in the observability vendor's console.
- Use shadow-evaluation for any AI control loop before canary; canary tests user-impact, shadow tests output-correctness.
- Maintain a warm rollback path for at least 14 days post-cutover; instant blue-green flip is the most valuable artifact a deployment plan produces.
- Build the four named AI-on-AI runbook sections (false-positive, misfire, managed-service, audit-recursion); these are foreseeable failure modes whose responses you should not be inventing during an incident.
- Run incident drills against the runbook before production cutover; runbooks that have never been exercised do not survive contact with real incidents.
- Reference policies by ID and version in the deployment plan; never duplicate policy content into the plan.
- Treat the kill-switch as a first-class operational tool: named, audited, paged, and rehearsed.
- Reconciliation scripts (for false-positive cascades, audit-recursion) ship with the deployment plan, not after the first incident.

## 7. Hands-on Exercise

Pick an AI-driven control loop you know or design one (an auto-scaler that uses ML predictions, an automated content-moderation system, a fraud-detection pipeline with auto-block, a build-pipeline guardrail). In 15 minutes:

1. Draft the cutover sequencing for it (shadow → canary → broad → decommission), with at least one validation gate at each phase.
2. Pick two of the four AI-on-AI failure modes (false-positive cascade, misfire, managed-service degradation, audit recursion). For each, write three steps the runbook would prescribe.
3. Identify the kill-switch surface (what config flag, on what component, with what audit record).

**What good looks like.** The cutover sequencing has named validation gates ("Phase 1 complete when X metric observed in Y window") rather than vague handoffs. The two runbook sections are specific to the chosen system — they reference the components and the data flows by name. The kill-switch is a documented function (with a name) that produces an AuditEvent, not a hand-wave.

## 8. Key Takeaways

- What four artifacts does a production-grade AI-stack deployment plan require, and why is each one load-bearing?
- How do canary, blue-green, and shadow-evaluation adapt for AI control loops, and which one is uniquely useful for AI?
- What are the four canonical AI-on-AI failure modes, and what is the documented response shape for each?
- Why does the deployment plan reference upstream policies by ID rather than duplicate their content?
- How does the kill-switch surface serve as the team's defence against failure modes the runbook did not anticipate?

## Sources

1. [Google SRE Workbook — Chapter 9: Incident Response](https://sre.google/workbook/incident-response/) — retrieved 2026-05-26
2. [Martin Fowler — Canary Release](https://martinfowler.com/bliki/CanaryRelease.html) — retrieved 2026-05-26
3. [Amazon OpenSearch Service](https://aws.amazon.com/opensearch-service/) — retrieved 2026-05-26
4. [Google SRE Book — Managing Incidents](https://sre.google/sre-book/managing-incidents/) — retrieved 2026-05-26

Last verified: 2026-05-26
