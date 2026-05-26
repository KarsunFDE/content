---
week: W05
day: Mon
topic_slug: ops-governance-spec
topic_title: "Ops Governance Spec — the operating contract for a live AIOps stack"
parent_overview: W05/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://cubeapm.com/blog/enterprise-observability-strategy/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.cloudraft.io/blog/implement-compliance-first-observability-opentelemetry
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://platformengineering.org/blog/self-service-observability
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.augmentcode.com/guides/ai-sre-incident-management
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://blogs.oracle.com/ai-and-datascience/runtime-governance-enterprise-agentic-ai
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Ops Governance Spec — the operating contract for a live AIOps stack

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define what an "Ops Governance Spec" is and how it differs from a platform-choice ADR, a runbook, or a compliance authorization document.
- Enumerate the five canonical sections of an Ops Governance Spec (scope, alerting policy, on-call surface, compliance boundary, audit-of-the-audit).
- Articulate why a governance spec must be written *before* an AIOps stack is turned on, not after.
- Recognise the failure modes of skipping the governance spec (silent alert sprawl, undefined on-call ownership, compliance-boundary drift).

## 2. Introduction

An Ops Governance Spec is the document that names *how an AIOps or observability stack is governed once it's live*. It's the operating contract between the team that owns the stack and the consumers of its signals — incident responders, security reviewers, audit, finance, the customers whose telemetry it ingests.

The category isn't new — large enterprises have long maintained monitoring-policy documents — but the AIOps era has reshaped it for three reasons ([CubeAPM enterprise observability strategy, 2026-05-26](https://cubeapm.com/blog/enterprise-observability-strategy/)):

1. **Automated remediation has authority** — a platform that auto-restarts services, auto-rolls-back deploys, or auto-files incidents is now a policy enforcer, not just a sensor. Governance must precede the authority.
2. **AI features cost real money per request** — token-cost telemetry per tenant is now load-bearing for SaaS economics; governance has to name how that telemetry is sampled, retained, and exposed.
3. **Telemetry now contains content** — LLM prompts and outputs carry PII, IP, and sometimes regulated data; the governance spec is where redaction and retention policy lands.

Without a governance spec, the AIOps stack runs anyway — but on tacit norms that everyone interprets differently. The result is the failure modes engineers have always seen: alert sprawl, missed signal in a noisy dashboard, ownership ambiguity, retention drift past compliance boundaries, no clean answer to "who turned that automation on?"

This reading walks the spec generically. The day's overview ties it to the cohort's stack; here we look at the shape the document takes across industries.

## 3. Core Concepts

### 3.1 What the spec is not

- *Not a platform-choice ADR.* That decision lives in `doc/adr/`. The governance spec assumes a platform has been chosen and names how it's operated.
- *Not a runbook.* Runbooks tell responders what to do during incidents; the governance spec defines what *qualifies* as an incident and who owns the response shape.
- *Not a FedRAMP or SOC 2 authorization document.* Those are downstream artifacts that may reference the governance spec; the spec itself is the team's operating contract, not the auditor's evidence.
- *Not optional.* A team can stand the platform up without writing the spec, but the implicit version of it gets enforced anyway — just by accident.

### 3.2 The five canonical sections

#### Section 1: Integration scope

What surfaces the AIOps stack monitors — listed at the *service* and *endpoint* level, not the platform-feature level. The discipline is precision: "we monitor service A's /healthz, /api, and /metrics endpoints with OTel auto-instrumentation, service B's batch jobs via custom span emission, and the messaging tier via the Kafka exporter."

Scope discipline is the bedrock. A scope of "the whole platform" is not a scope; it's an aspiration. Self-service observability literature consistently flags this as where governance fails first — telemetry sprawls because no one ever said no ([Platform Engineering — self-service observability, 2026-05-26](https://platformengineering.org/blog/self-service-observability)).

#### Section 2: Alerting policy

What fires an alert, what severity, what destination. The format that works across industries:

| Signal | Severity | Threshold | Destination | Suppression conditions |
|--------|----------|-----------|-------------|------------------------|
| `error_rate{service=A} > 5%` | SEV-2 | 5min window | on-call rotation | suppressed during scheduled deploys |
| `llm_token_cost{tenant=*} > $X/hr` | SEV-3 | 1hr window | finance + tenant ops | suppressed if tenant has burst-credit |

The discipline is *each row is justified*. A row that the team can't explain isn't governance — it's a candidate for retirement.

#### Section 3: On-call surface

Who gets paged, on what severity, with what runbook entry. Names roles, not people (people churn; the role abstraction outlives the org chart). Escalation paths are explicit: primary on-call → secondary → team lead → incident commander.

The governance spec is where on-call expectations are *committed*; the runbook is where they're *operationalised*. The two must agree.

#### Section 4: Compliance boundary

What data flows where. Critical fields:

- **Regulated data flows** — which signals carry PII / PHI / regulated content, and what redaction is applied at ingest.
- **Data-residency boundary** — which signals leave a regulated region; which platform endpoint they cross.
- **Retention policy** — how long each signal class is retained, where it's archived, and how it's destroyed at the end of the window.
- **Tenant partitioning** — how multi-tenant signals are isolated in the observability backend.

For federal-modernization, regulated-industry, or any FedRAMP-touching engagement, this section is also the input the authorization documentation cites — the governance spec is the source of truth for "where the data goes," and the FedRAMP boundary document references it ([CloudRaft compliance-first observability, 2026-05-26](https://www.cloudraft.io/blog/implement-compliance-first-observability-opentelemetry)).

#### Section 5: Audit-of-the-audit

What audit row the platform itself writes when it fires an automated action — a remediation, an auto-scale, a circuit-open, a rollback. This is the meta-audit: a record of the AIOps stack's own actions, separate from the application audit log.

The pattern emerged because automated-remediation features moved from "useful" to "now mandatory to govern." A team that turns on auto-remediation without audit-of-the-audit has handed authority to a black box ([Oracle runtime governance for enterprise agentic AI, 2026-05-26](https://blogs.oracle.com/ai-and-datascience/runtime-governance-enterprise-agentic-ai)).

### 3.3 Why before, not after

The governance spec must be committed *before* the stack is turned on, for three reasons:

- **Default behaviour calcifies fast.** Once an alert policy is live and people are paged from it, retiring it becomes a political act, not a technical one.
- **Boundary calls are harder retroactively.** Naming the compliance boundary before data flows is a paragraph; renaming it after is a migration.
- **Audit-of-the-audit can't be retrofitted cleanly.** If the platform's automated actions weren't logged from day one, the team can't reconstruct what authority was exercised when.

The discipline is not about completeness — early specs are short and provisional. The discipline is that the document exists before the stack is taking actions in production.

## 4. Generic Implementation

A generic governance spec for a hypothetical e-commerce SaaS standing up an AIOps stack. The structure carries to any domain.

```markdown
# Ops Governance Spec — checkout-service AIOps

## 1. Integration scope
- frontend-spa (Angular)        → RUM, real-user metrics
- order-service (Java)          → OTel Java auto-agent, all endpoints
- payment-service (Java)        → OTel Java auto-agent, all endpoints
- recommender-service (Python)  → OTel Python auto-instrumentation + gen_ai.* spans
- kafka                         → Kafka exporter, lag + consumer metrics
- postgres                      → postgres-exporter, slow queries + connection pool

OUT of scope this iteration:
- internal batch ETL jobs (separate spec, q3 work)
- staging environment metrics (kept on dev tier, no on-call)

## 2. Alerting policy
- order error_rate > 2% / 5min      → SEV-2, primary on-call
- payment 5xx > 0.5% / 5min          → SEV-1, primary + secondary
- recommender token_cost > $200/hr   → SEV-3, finance + product
- kafka consumer lag > 10k msgs      → SEV-2, primary on-call
(All thresholds reviewed monthly; suppression windows around deploys are
documented in the runbook, not here.)

## 3. On-call surface
- Primary on-call: checkout-platform rotation (weekly handover Mon 10:00)
- Secondary: payments rotation (paged on SEV-1 only)
- Escalation: team lead → on-duty IC for any SEV-1 unresolved at 30 min

## 4. Compliance boundary
- PCI-DSS in-scope signals: payment-service spans, redacted at the OTel
  Collector (no PAN, no CVV, no full cardholder name in span attributes).
- Data residency: all telemetry stays in EU-west region; cross-region export
  is not enabled.
- Retention: traces 30 days, metrics 13 months, logs 7 days hot + 1 year cold.
- Tenant partitioning: spans carry `tenant_id` tag; backend dashboards
  restricted by tag-based RBAC.

## 5. Audit-of-the-audit
The AIOps stack's automated actions emit a record to the `aiops-audit` topic:
- Watchdog-fired anomaly alert: timestamp, signal, threshold, action taken.
- Auto-suppression of duplicate alerts: timestamp, suppressed alert ID,
  duration, justification rule.
- (No remediation actions enabled in this iteration; if added, audit row
  must precede activation.)
```

The spec is short on purpose. It commits the operating contract; the runbook expands it; the ADRs cite it.

## 5. Real-world Patterns

**Healthcare — pharmaceutical R&D telemetry boundary**. A pharma R&D group running drug-discovery agents wrote a governance spec whose compliance section was the load-bearing part: every signal class carrying molecular-screening data was tagged at ingest, redacted at the Collector, and retained on a separate backend tier that never left the regulated region. The spec preceded any dashboard build. The result was that when the audit team arrived six months later, the question "where does this data go?" had a one-page answer instead of a forensic exercise ([CloudRaft compliance-first observability, 2026-05-26](https://www.cloudraft.io/blog/implement-compliance-first-observability-opentelemetry)).

**Fintech — alerting policy retirement discipline**. A payments platform that had been running on tacit alerting rules for three years committed to writing a retroactive governance spec. The first month produced no improvement — they just documented what existed. The second month they used the spec as a *retirement* gate: every row that the team couldn't justify by signal, threshold, and destination was retired. Alert volume dropped 60%, MTTR for the alerts that survived dropped meaningfully because responders learned to trust the surviving ones ([AIOps on-call fatigue reduction, 2026-05-26](https://devops.com/aiops-for-sre-using-ai-to-reduce-on-call-fatigue-and-improve-reliability/)).

**Logistics — on-call surface clarity at scale**. A logistics platform with 14 sub-teams found that the governance spec's on-call surface section solved a recurring problem: when a multi-service incident hit, no one knew which rotation owned the response. The spec named one *incident commander* role across all 14 rotations, with the IC's authority pre-committed. The clarity wasn't a technical change — it was a contract everyone had read before the bad day ([AI SRE incident management, 2026-05-26](https://www.augmentcode.com/guides/ai-sre-incident-management)).

**Gaming — audit-of-the-audit for live-ops automation**. A free-to-play title with aggressive live-ops automation (auto-scale on event spikes, auto-disable on payment-service degradation) committed audit-of-the-audit from day one: every automated action wrote a row to a separate audit stream that ops, finance, and customer-support all consumed. When a live-event incident triggered an auto-disable that affected a partner promotion, the audit-of-the-audit was the artifact that resolved blame within hours instead of weeks. The discipline is cheap to maintain and ruinous to skip ([Oracle runtime governance, 2026-05-26](https://blogs.oracle.com/ai-and-datascience/runtime-governance-enterprise-agentic-ai)).

## 6. Best Practices

- **Write the spec before turning the stack on.** Provisional and short beats absent and elaborate. The first version names what is being monitored, what fires alerts, who responds, where data goes, and what the platform itself audits.
- **Name roles, not people, in the on-call surface.** Role abstractions outlive the org chart; people-named on-call surfaces break the moment someone changes teams.
- **Make each alerting row justified.** A row whose existence the team can't explain is a retirement candidate, not a precedent.
- **Treat the compliance boundary as a separate concern from the alerting policy.** Combining them in one section is where governance specs become unreadable; keep them as Section 4 (boundary) and Section 2 (alerting) and let each evolve at its own cadence.
- **Audit-of-the-audit precedes any automated remediation.** If you can't commit the meta-audit row, don't turn on the action.
- **Review the spec on a fixed cadence**, not when something breaks. Monthly or quarterly is typical; the cadence is what catches drift before it becomes an incident.
- **Keep the spec versioned and dated.** A governance spec that doesn't show its age is one nobody trusts; date the document and version-bump on substantive change.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** Pick a service you've worked on. On a single page, draft its Ops Governance Spec using the five-section structure from §3:

1. Integration scope (services + endpoints in scope, what's out of scope).
2. Alerting policy (3–5 rows with signal, severity, threshold, destination).
3. On-call surface (rotations, escalation path).
4. Compliance boundary (regulated data, residency, retention, tenant partitioning).
5. Audit-of-the-audit (what record is written when the platform itself takes an automated action; if none, say so).

**What good looks like.** A finished page reads like a contract — each section is specific enough that two engineers reading it would respond identically to the same incident. If any row in the alerting policy can't be justified ("why does this fire? who reads it? what do they do?") flag it for retirement. If the compliance section says "we follow company policy," the spec is shelfware; replace with concrete fields.

## 8. Key Takeaways

- *What does an Ops Governance Spec commit?* The operating contract for a live AIOps stack — scope, alerting, on-call, compliance boundary, audit-of-the-audit.
- *How does it differ from an ADR or a runbook?* The ADR makes the platform choice; the runbook tells responders what to do during incidents; the governance spec defines what *qualifies* as an incident and who owns the response shape.
- *Why must it be written before the stack goes live?* Default behaviour calcifies fast; boundary calls are harder retroactively; audit-of-the-audit cannot be retrofitted cleanly.
- *What's the cheapest failure mode the spec prevents?* Alert sprawl — rows that fire, nobody owns, nobody retires, and that ultimately train responders to ignore the entire surface.

## Sources

1. [Enterprise Observability Strategy in 2026 — practical framework (CubeAPM)](https://cubeapm.com/blog/enterprise-observability-strategy/) — retrieved 2026-05-26
2. [Implementing Compliance-first Observability with OpenTelemetry (CloudRaft)](https://www.cloudraft.io/blog/implement-compliance-first-observability-opentelemetry) — retrieved 2026-05-26
3. [Self-service observability — consistency, cost control, scale (Platform Engineering)](https://platformengineering.org/blog/self-service-observability) — retrieved 2026-05-26
4. [AI SRE in Incident Management — on-call (Augment Code)](https://www.augmentcode.com/guides/ai-sre-incident-management) — retrieved 2026-05-26
5. [Runtime Governance for Enterprise Agentic AI (Oracle)](https://blogs.oracle.com/ai-and-datascience/runtime-governance-enterprise-agentic-ai) — retrieved 2026-05-26

Last verified: 2026-05-26
