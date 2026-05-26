---
week: W05
day: Wed
topic_slug: hitl-7-auto-remediation-authority
topic_title: "HITL #7 — Auto-remediation authority, the closing touchpoint"
parent_overview: W05/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 14
sources:
  - url: https://www.unite.ai/agentic-sre-how-self-healing-infrastructure-is-redefining-enterprise-aiops-in-2026/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://rootly.com/ai-sre-guide
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.causely.ai/blog/why-your-ai-sre-agent-is-stuck-on-read-only
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://tamnoon.io/blog/building-trust-in-automated-remediation/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://venturebeat.com/ai/agent-autonomy-without-guardrails-is-an-sre-nightmare
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# HITL #7 — Auto-remediation authority, the closing touchpoint

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish between **human-in-the-loop** (approves every action) and **human-on-the-loop** (sets policy, oversees execution) trust models, and name two production-domain examples where each is appropriate.
- Describe the **four-stage graded-autonomy ladder** (read-only → advised → approval-gated → autonomous) and explain why most enterprise teams sit between stages 2 and 3.
- Name at least **five authority-boundary dimensions** an autonomy ADR must commit to (reversibility, blast-radius, audit posture, novelty, vendor coupling).
- Argue a defensible position for a specific auto-remediation case against those dimensions, rather than retreating to *"it depends"*.

## 2. Introduction

"Auto-remediation authority" is the question every modern operations platform eventually has to answer out loud: *when an alerting and diagnosis system has the technical ability to fix a problem itself, when is it allowed to?* For most of the SRE industry's history this question was hypothetical — the systems could not yet diagnose well enough to act. By 2026 the diagnosis quality has caught up, and the question is no longer hypothetical: an SRE agent often *can* restart a pod, scale a connection pool, roll back a deploy, or replay a missing audit event. The remaining question is governance, not capability.

The mainstream framing in 2026 splits trust models into two: **human-in-the-loop**, where every remediation action is gated by a human approval, and **human-on-the-loop**, where humans set policy and guardrails and the agent executes within them, with humans handling exceptions and novel cases. The 2026 *Agentic SRE* literature treats the shift from in-loop to on-loop as the defining trust evolution of autonomous operations — not because in-loop is wrong, but because it doesn't scale past a few alerts per day before approvers become the bottleneck and start rubber-stamping.

The reason this topic is the *closing* touchpoint in the seven-touchpoint HITL programme thread is that every prior touchpoint was about *one* surface: model output review, retrieval fallback, plan-day ADRs, multi-agent handoffs, LangGraph interrupts, OWASP excessive-agency framing. Each of those is a slice. Authority-to-act-on-production binds them all into one decision: *what is this system allowed to do without us, ever?* The closing touchpoint forces a position.

The full 7-touchpoint thread, named explicitly so you can place this reading in its arc:

| # | Week / Day | Touchpoint | What got committed |
|---|-----------|------------|--------------------|
| #1 | W1 Fri | LLM Engineering Essentials — explicit HITL framing | Why HITL is a first-class discipline, not a fallback |
| #2 | W2 Thu | RAG fallback — faithfulness-threshold gate, CO escalation | When retrieval fails, who decides the response shape |
| #3 | W3 Mon | Plan Day — interrupt-node-boundaries ADR | Where in the agent graph humans can/must intervene |
| #4 | W3 Wed | Multi-agent — supervisor → worker soft interrupt | How handoffs surface human review |
| #5 | W3 Thu | LangGraph hard interrupt + FAR 15.308 audit row | The legal-record artifact for each gated decision |
| #6 | W4 Wed | OWASP LLM06 Excessive Agency — authority-boundary table | Per-endpoint authority limits in application code |
| **#7** | **W5 Wed (this reading)** | **AIOps auto-remediation authority — the closing ADR** | **The authority decision generalised to autonomous incident-response actions** |

The thread is cumulative: each touchpoint commits an artifact (ADR, table, audit row) that the next one builds on. HITL #6 (W4 Wed) defined per-endpoint authority boundaries in application code — *what is this `/agent/intake-triage` endpoint allowed to do, in what scope, with what reversibility?* HITL #7 extends that exact discipline to autonomous remediation in AIOps platforms: the same authority-table shape, applied to incident-response actions (circuit-breaker trips, pod restarts, audit-event replay) instead of API endpoints. If the W4 Wed table named the model's authority over user requests, the W5 Wed ADR names the AIOps system's authority over the production substrate.

The ADR you commit on this question is not a one-time artifact. It is the contract you have with the on-call rotation, with the compliance team, with the regulator, and with the customer. When the system misfires (and it will, eventually), the ADR is the document that says whether the misfire was foreseen and budgeted-for, or whether the team accidentally granted an authority nobody intended.

## 3. Core Concepts

### 3.1 In-the-loop vs on-the-loop vs out-of-the-loop

The three positions are not a spectrum but discrete operating modes:

- **In-the-loop (HITL).** The agent proposes; the human approves; the agent executes. Every action has a human-acknowledged gate. Latency: human reaction time. Throughput: human approval bandwidth.
- **On-the-loop (HOTL).** The human sets policy ahead of time (what classes of action are safe, on what conditions, with what blast radius). The agent executes within policy. The human watches summaries and handles exceptions. Latency: agent reaction time. Throughput: policy-bounded; not human-bounded.
- **Out-of-the-loop.** No human review at all. Used in narrow domains where the action is small, reversible, and high-frequency — e.g., a pod restart on OOM. The audit trail is the only oversight.

A mature platform routes different action classes to different modes. The mistake to avoid is picking one mode for the *whole platform* and applying it uniformly.

### 3.2 The graded-autonomy ladder

The 2026 AI-SRE practitioner literature has converged on a four-stage maturity ladder:

| Stage | Name | What the agent does | Where most teams sit |
|------:|------|---------------------|---------------------|
| 1 | Read-Only | Observes, correlates, summarises, explains | Initial pilots, regulated workloads |
| 2 | Advised | Recommends a remediation; produces an action plan | Most production AI-SRE deployments today |
| 3 | Approval-Gated | Drafts the fix (e.g., a PR, a runbook step) and waits for human merge/approve | Trust-mature teams on familiar action classes |
| 4 | Autonomous | Executes bounded remediation under policy, audits the action | Reserved for high-frequency, low-blast-radius classes (OOM, transient pod evictions) |

A common production posture: stage 4 for one or two well-known action classes (pod restart, connection-pool resize); stage 3 for most diff-producing actions (PRs, config changes); stage 2 for novel incidents; stage 1 for anything touching audit, billing, or auth. Teams who try to jump from stage 1 to stage 4 fail predictably — they have no observability into what the agent would have done, no track record of agreement between agent and human, and no governance backstop.

### 3.3 Authority dimensions an ADR must commit to

An auto-remediation ADR that says *"we'll be Stage 3"* and stops there is incomplete. The dimensions a real ADR commits to:

1. **Reversibility.** If the action fires wrongly, can the system un-do it? An auto-replayed missing record marked `replayed_by_agent: true` is reversible (delete the row, page a human). An auto-applied schema migration is not.
2. **Blast-radius.** What is the maximum scope of damage if the action mis-fires? Tenant-bounded? Account-bounded? Global?
3. **Audit posture.** Does the action's domain (audit logs, billing ledger, auth tables) make any non-human writer fundamentally suspect? Some controls (e.g., NIST 800-53 AU-9 protection of audit information) require strictly-limited authorisation for *any* write to the audit table — including the agent's own.
4. **Novelty.** Is this an action class the agent has executed (or proposed) successfully many times, or is this novel? Most ADRs reserve stage 4 to high-frequency known classes and route novel cases to stage 2 or 3.
5. **Vendor coupling.** Authority granted to a vendor agent is a contract with that vendor. If you grant stage-4 authority to one platform, can that authority be replicated if you switch vendors? If not, you have over-coupled.
6. **Authority delegation.** Who in the org grants this authority? The on-call engineer alone? The SRE manager? Compliance? A regulator-recognised approver? Name them; do not leave it implicit.

### 3.4 Three canonical positions on a contested action

When the question is *"can the agent do X without us?"*, the three honest positions are:

- **(A) Full Auto.** The agent executes; the action is logged; a human reviews after-the-fact within an agreed window. Defensible only when reversibility is high and blast-radius is low.
- **(B) Propose-and-Approve.** The agent drafts the fix (PR, runbook step, ticket); a human merges. The agent's authority stops at "proposing"; the human's authority is "merging". Defensible for most diff-producing actions.
- **(C) Escalate-only.** The agent observes and pages; the agent does not touch the system. Defensible when blast-radius is global, reversibility is uncertain, or the action class has not been exercised enough to build a track record.

Each position is defensible in some context and indefensible in another. The ADR's job is to name *which context* and to argue *why this context is one of those.*

## 4. Generic Implementation

Outside Karsun's federal domain, the cleanest worked example is a fintech payment-platform engineering team building an AI-SRE policy.

**Action class:** auto-scaling the read-replica connection pool when latency on a checkout endpoint exceeds 95th-percentile threshold for ten minutes.

**Dimensions:**

- *Reversibility:* high — the pool size is a configmap; reverting takes one apply.
- *Blast-radius:* tenant-shared but bounded — the checkout cluster is its own infrastructure.
- *Audit posture:* the change writes to the platform's deploy audit log via the configmap webhook; no special compliance regime.
- *Novelty:* the agent has proposed this fix 47 times in the last two quarters; a human approved 44 of those and rejected 3 (all rejected because the root cause was a noisy-neighbour write workload, not a connection-pool issue).
- *Vendor coupling:* the action is a `kubectl apply`; portable across SRE vendors.
- *Authority delegation:* SRE manager approves the policy; on-call SRE owns rollback.

**Decision:** Stage 4 (autonomous) for pool sizes ≤ 2× current; Stage 3 (approval-gated) for pool sizes > 2× or when CPU on the upstream DB exceeds 70%. Novel root causes (the 3 rejected cases) escalate to a human via the existing on-call channel.

A short policy snippet (illustrative, not advocacy of a specific tool):

```yaml
# policy.connection-pool.yaml
action_class: scale_read_replica_pool
applies_when:
  metric: checkout_p95_latency_ms
  threshold: 1200
  duration: 10m
autonomy:
  - if: target_size <= current_size * 2 AND upstream_db_cpu < 70
    mode: autonomous
    blast_radius: cluster
    reversibility: high
  - else:
    mode: approval_gated
    approver_group: on_call_sre
audit:
  emit_event_to: deploy_audit_log
  fields: [actor, action, before, after, reason, trace_id]
post_action:
  human_review_window: 24h
```

The shape is what matters: action class, applies-when, autonomy modes branched by precondition, blast-radius declared, audit emission required, human review window named.

> [!instructor-review]
> The policy snippet above is illustrative and avoids advocating a specific vendor or SDK. If the cohort adopts a concrete platform (Datadog, Dynatrace, etc.), substitute the platform's native policy syntax — but keep the dimensions explicit; do not let a vendor's defaults swallow the ADR's structure.

## 5. Real-world Patterns

**Fintech (high-frequency trading platform, 2025–2026).** The platform's AI-SRE is granted stage-4 authority for read-replica scaling and circuit-breaker resets, but stage-1 (read-only) for anything touching the trading engine itself or the position-keeping database. The boundary is set by reversibility: a mis-scaled read replica is recoverable; a mis-written position is not. The platform's incident reviews specifically commend the *narrowness* of the agent's autonomy as the reason the team trusts it ([Tamnoon: Building Trust in Automated Remediation](https://tamnoon.io/blog/building-trust-in-automated-remediation/), retrieved 2026-05-26).

**Healthcare (large hospital network EHR vendor).** The EHR's operations team runs the AI-SRE in stage-2 (advised) across the board. The reasoning, captured in their public engineering blog: HIPAA and the hospital's own incident-review process require *named human actors* on every change touching patient-data systems. Stage 3 was piloted and rolled back after the compliance team flagged that an agent-merged PR did not have a defensible "human actor" trail for the regulator. The team's stage-3 pilot is now scoped to non-patient-data infrastructure only.

**E-commerce (retail platform, holiday-season scaling).** Different action classes sit at different stages: stage-4 for autoscaling stateless front-end pods, stage-3 for cache-warming jobs and CDN rule changes, stage-2 for database failover, stage-1 (read-only) for payments. The platform's published 2026 post-mortem of a Black Friday incident attributes the *avoidance* of a wider outage to stage-1 being honoured on payments — the agent flagged a fraud-service degradation but did not touch the payment path; a human on-call routed traffic ([Rootly AI SRE Guide](https://rootly.com/ai-sre-guide), retrieved 2026-05-26).

**Gaming (live-service multiplayer platform).** Stage-4 for matchmaking-queue rebalancing (high-frequency, low-blast-radius); stage-3 for region-failover; stage-1 for store/microtransaction systems. The studio's published 2026 SRE talk emphasises that the *boundary between stage-3 and stage-4 is not a function of how smart the agent is — it's a function of how reversible the action is.*

## 6. Best Practices

- **Pick a position per action class, not per platform.** Uniform autonomy modes always end up either too lax (compliance failures) or too strict (no actual scale benefit).
- **Make reversibility a precondition for stage-4, not a hope.** If the un-do path requires more than a single command, the action is not yet stage-4-eligible.
- **Keep a track record before you raise the stage.** A stage-4 promotion is justified by N successful stage-3 executions with human agreement, not by aspiration.
- **Audit the agent's own writes the same way you audit a human's.** The agent is an actor; its actions are first-class log entries with `actor=agent`, `action`, `trace_id`, and a human-readable reason field.
- **Set a human review window for stage-4 actions.** Even fully-autonomous actions get reviewed within 24 hours by a named approver — not to undo them, but to keep the policy honest.
- **Name the delegating authority in the ADR.** "We grant the agent stage-3 authority on this class" is incomplete; "the SRE manager grants the agent stage-3 on this class; the compliance team has reviewed the policy; the on-call engineer holds rollback" is complete.
- **Treat vendor lock-in on autonomy as a real cost.** If the same authority cannot be replicated on a second platform, you are coupling more than you think.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** Pick *any* one action class on a system you've recently shipped or operated — pod restart, cache flush, retry-after-failure, deploy rollback, audit event replay, schema migration, etc. Do not pick something from federal acquisitions; use a system from your prior work.

For that action class, fill in:

1. Reversibility (high / medium / low — with one sentence why).
2. Blast-radius (per-request / per-tenant / per-cluster / global).
3. Audit posture (is there a regulator or compliance regime that constrains *any* non-human writer?).
4. Novelty (how many times has the action been executed in the last year? Was the agent's proposal ever wrong?).
5. Vendor coupling (one sentence — would the same authority work on a second platform?).
6. Authority delegation (who in the org grants this? who holds rollback?).

Then commit a one-sentence position: *"On this class, the agent operates at Stage N because of dimension X."*

**What good looks like.** A good answer names a Stage (1, 2, 3, or 4), names the *one or two* dimensions that drove the choice (not all six — most cases are dominated by one or two), and names an escalation rule for the case the dimensions do not cover. *"It depends"* is not an answer; *"Stage 3 on this class because the action is diff-producing and the regulatory regime requires a named human merger; escalate to stage 2 if the diff touches the audit schema"* is.

## 8. Key Takeaways

- *What is the difference between in-the-loop and on-the-loop, and why does it matter at scale?*
- *What are the four stages of the graded-autonomy ladder, and which stages does the typical enterprise SRE deployment occupy?*
- *Which dimensions does an auto-remediation ADR have to commit to, and why is reversibility usually the dominant one?*
- *Why is "it depends" a failed answer to an autonomy question, and what does a defensible position look like instead?*

## Sources

1. [Agentic SRE: How Self-Healing Infrastructure Is Redefining Enterprise AIOps in 2026 — Unite.AI](https://www.unite.ai/agentic-sre-how-self-healing-infrastructure-is-redefining-enterprise-aiops-in-2026/) — retrieved 2026-05-26
2. [What is an AI SRE? The complete AI SRE Guide in 2026 — Rootly](https://rootly.com/ai-sre-guide) — retrieved 2026-05-26
3. [Why Your AI SRE Agent Is Stuck on Read-Only — Causely](https://www.causely.ai/blog/why-your-ai-sre-agent-is-stuck-on-read-only) — retrieved 2026-05-26
4. [How to Build Trust in AI-Powered Security Remediation — Tamnoon](https://tamnoon.io/blog/building-trust-in-automated-remediation/) — retrieved 2026-05-26
5. [Agent autonomy without guardrails is an SRE nightmare — VentureBeat](https://venturebeat.com/ai/agent-autonomy-without-guardrails-is-an-sre-nightmare) — retrieved 2026-05-26

Last verified: 2026-05-26
