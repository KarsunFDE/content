---
week: W03
day: Mon
topic_slug: hitl-interrupt-node-boundaries-adr
topic_title: "HITL #3 — interrupt-node boundaries ADR"
parent_overview: W03/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://docs.langchain.com/oss/python/langgraph/interrupts
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/oss/python/langgraph/persistence
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://adr.github.io/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://github.com/joelparkerhenderson/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.anthropic.com/research/building-effective-agents
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-06-06
---

# HITL #3 — interrupt-node boundaries ADR

> [!NOTE]
> **From earlier:** Topics 2–6 built the justification chain — irreversibility, state-machine model, memory scope, eval signal, §0 carry-forwards. This topic is the deliverable: write the ADR that commits it all by 17:00.

## 1. Learning Objectives

- Classify any agent transition as no-gate / soft-gate / hard-gate with a one-sentence justification.
- Write an ADR for a single HITL gate covering all six required sections.
- Distinguish the implementation primitive (static `interrupt_before`, dynamic `interrupt()`, tool-layer approval token) from the policy it enforces.
- Apply the "one hard gate per pair" heuristic and recognise over-gating and under-gating.

## 2. Introduction

HITL #3 is where human-in-the-loop becomes an *architectural commitment*. Each transition gets a classification (no / soft / hard gate), a justification, and an implementation primitive — written before any code.

Teams that ship successfully defend each gate in one sentence. Teams that struggle either gate everything (alert fatigue) or gate nothing (irreversible action slips through). Anthropic's *Building Effective Agents*: human checkpoints are part of the agent-computer interface, not exception handlers bolted on later ([Anthropic](https://www.anthropic.com/research/building-effective-agents), retrieved 2026-05-26).

## 3. Core Concepts

### 3.1 The three gate types

| Type | Behaviour | When to use |
|---|---|---|
| **No gate** | Auto-execute, log, move on | Reversible action; downstream gate already covers it |
| **Soft gate** | Suggest a default; human can approve / edit / reject | Reversible-but-risky; routing choices that benefit from sanity checks |
| **Hard gate** | Block until a human resumes the graph; no auto-resume timeout | Irreversible action; regulatory requirement; externalising boundary |

Classification follows the irreversibility principle (Topic 2): hard gates where actions are irreversible; soft gates where the default benefits from review but the action can still be undone.

### 3.2 Six sections of a defensible HITL ADR

Per [adr.github.io](https://adr.github.io/) (retrieved 2026-05-26):

1. **Title + status** — `ADR-NNNN: HITL gate on transition X` (proposed | accepted | superseded).
2. **Context** — what is the transition; what does it produce; who is the stakeholder.
3. **Decision** — no gate / soft gate / hard gate. Named explicitly; resist hedging.
4. **Justification** — regulation citation OR reversibility argument. Never "felt risky."
5. **Implementation primitive** — `interrupt_before=[node]`, `interrupt()`, or tool-layer approval token.
6. **Consequences** — cost (latency, on-call); what it pre-empts (audit finding, incident).

> [!NOTE]
> **Operational implication:** The six-section ADR format is the defence skeleton. At Phase-1 Gate on Friday, the instructor asks "walk me through your HITL #3 ADR" and expects each section in under 60 seconds. Write it in that shape today.

### 3.3 The justification rule — regulation OR reversibility, never risk

- **Regulation-anchored** — name the clause (FAR 15.206, HIPAA §164.508, PCI-DSS Req 8.3). If it changes, the ADR is where you revisit.
- **Reversibility-anchored** — name the externalisation (public registry publish, SMS send, partner-consumed write).

If you cannot complete either sentence, the gate is probably wrong.

### 3.4 Implementation primitives — static vs dynamic

LangGraph exposes two hard-gate primitives and one soft-gate pattern ([LangGraph Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts), retrieved 2026-05-26):

- **Hard gate, static** — `interrupt_before=[node_name]` at compile time. Inspectable at build; resume requires `Command(resume=...)`. W3's canonical wiring for FAR-anchored gates.
- **Hard gate, dynamic** — `interrupt(payload)` inside a node. Pause conditional on runtime state. Cohort meets it Thu (HITL #5).
- **Soft gate** — tool-layer approval token or queue-with-claim-check; ordinary async APIs, not graph interrupts.

> [!TIP]
> **"One hard gate per pair" coaching heuristic.** Zero hard gates: irreversibility argument not internalised. One: typically correct — sits at the externalising boundary (`escalate_to_co`, `publish_amendment`, `release_funds`). Two: defensible only with two genuinely independent irreversible actions. Three+: over-gating. Two to four soft gates is normal; soft gates should outnumber hard gates 2–4×.

## 4. Generic Implementation

```markdown
# ADR-0011 — HITL gate on `issue_credit` transition

## Status
Accepted (2026-05-26)

## Context
The shipment-exceptions agent triages damaged-shipment claims. After
triage, `prepare_credit_proposal` drafts a credit object internally
(reversible). The next transition is `issue_credit` (Stripe refund API
— money leaves the platform; downstream consumers fire) or
`escalate_to_ops_lead` (routes to a human queue).

CFO constraint: "Credits over $500 require my explicit sign-off.
Below $500, act — but give me an audit trail I can read every Monday."

## Decision
- `issue_credit` when credit ≤ $500:   **no gate** (auto-execute)
- `issue_credit` when credit > $500:   **hard gate** (block until ops lead resumes)
- `escalate_to_ops_lead`:              **soft gate** (agent's routing suggestion
  surfaced to ops lead with option to redirect)

## Justification
- ≤$500 auto-execute: reversibility-with-acceptable-cost — dollar amount
  below CFO's defined threshold AND below reversal-by-dispute feasibility.
- >$500 hard gate: CFO's stated constraint. Citation: internal finance
  policy FIN-POL-2026-03 §4.2.
- Soft gate on escalation: ops leads have historically corrected the
  agent's queue choice ~12% of the time; soft gate surfaces choice before
  committing.

## Implementation primitives
- ≤$500: ordinary edge; audit row written by `issue_credit` before Stripe call.
- >$500: `interrupt_before=["issue_credit"]` at compile time. Resume requires
  `Command(resume={"approved": True, "approver_id": ...})`.
- Escalation: agent emits suggested queue in state; ops-lead UI surfaces it;
  override via normal API call (no graph interrupt).

## Consequences
- CFO gets weekly audit-readable row per refund.
- Ops-lead on-call bounded by >$500 case rate (~3/wk).
- Alert fatigue avoided — soft gate is non-blocking.
- Threshold tweaks require only the router-edge conditional to change.
```

The shape matters more than any single field: a real decision, a real justification, named alternatives, named consequences. A future engineer can re-derive whether the decision still holds.

## 5. Real-world Patterns

**Healthcare — CDS sign-off tiers.** CDS systems classify suggestions into three tiers: system acts, system suggests, clinician must act. Every tier transition gets an explicit policy — the analog of a HITL ADR. The FDA expects AI/ML devices to document where the human is in the loop ([Anthropic](https://www.anthropic.com/research/building-effective-agents), retrieved 2026-05-26).

**Fintech — payment-platform action policies.** Stripe-style platforms classify every endpoint: auto-execute, requires-admin, requires-compliance-review. Each policy is a HITL ADR — externalisation is the API call, justification is regulation (KYC, AML, sanctions).

> [!NOTE]
> **Cross-domain lesson:** Every regulated domain enforces a tiered action policy — CDS sign-off tiers, payment compliance reviews, FAR authority ceilings. Your ADR is the federal version. The structure travels; only the regulation citation changes.

## 6. Best Practices

- Write the ADR before implementation — defend the decision in prose first.
- One ADR per gate, not per topology — small, atomic ADRs age better and supersede cleanly.
- Cite the specific regulation clause or externalisation; vague justifications produce ADRs no-one trusts in six months.
- Default to *fewer* hard gates than intuition suggests; irreversibility is sharper than risk.
- Pair every hard gate with three artefacts: audit row (before the call), notification, externalisation marker.
- Re-audit gates every cycle (§0 retro, Topic 6) — gates accumulate and many become theatre.

> [!WARNING]
> **Anti-pattern: `adr-without-citation`.** A gate justified by "because it could be risky" is vibes-based architecture — it will not survive a Phase-1 Defense or OIG audit. Every gate must complete one sentence: "Required because FAR clause X" OR "because the outside world observes the action via Y." Can't complete either? The gate does not belong. See slug `adr-without-citation`.

## 7. Hands-on Exercise

You are designing an AI-assisted hiring platform. The team lead says: *"The system can rank candidates, draft outreach, and schedule screens. But no rejection email goes out without my approval, and no offer extends without two recruiters signing off."*

For each transition, write an abbreviated ADR (title, decision, justification, primitive):
1. `rank_candidates` — orders applicants by predicted fit.
2. `draft_outreach_email` — generates a draft.
3. `send_outreach_email` — externalises via SendGrid.
4. `schedule_screen` — books a calendar slot.
5. `send_rejection_email` — externalises via SendGrid.
6. `extend_offer` — emails a formal offer letter.

> [!NOTE]
> **Self-check** (30s — answer mentally before expanding)
>
> 1. `send_outreach_email` externalises to a vendor's email system. Should it be a hard gate or a soft gate, and what is the one-sentence justification?
> 2. `extend_offer` needs two recruiters to sign off. Is this implemented as one hard gate with a special two-actor flag, or as two separate gates?

<details>
<summary>Show answers</summary>

1. Soft gate. The team lead has *delegated* outreach to the system — the constraint is only on rejections and offers. `send_outreach_email` is irreversible (email leaves the relay) but the stakeholder did not mandate a hard gate on it. A soft gate surfaces the draft and lets the team lead redirect if something looks wrong, without blocking every outreach. Justification: "team lead has delegated outreach but benefits from a final-draft review before externalisation."
2. Two separate gates — both must pass, sequentially. A single hard gate with a "two-actor flag" is a single point of failure (one reviewer can accidentally clear it). Two sequential soft gates (or one soft + one hard) that each require a distinct `approver_id` mean both reviewers must actively act. The resulting audit trail shows two separate approvals with two timestamps and two actor IDs — defensible to any compliance review.

</details>

## 8. Key Takeaways

- Three gate types: no gate, soft gate, hard gate — classification follows irreversibility, not risk.
- Six-section ADR: title/status, context, decision, justification, implementation primitive, consequences.
- Justification is regulation citation OR reversibility argument — never "because it felt risky."
- One hard gate per pair is typically correct; soft gates should outnumber hard gates 2–4×.
- Every hard gate must produce three artefacts: audit row (before the call), notification, externalisation marker.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://docs.langchain.com/oss/python/langgraph/interrupts — retrieved 2026-05-26 — hot-tech
- https://docs.langchain.com/oss/python/langgraph/persistence — retrieved 2026-05-26 — hot-tech
- https://adr.github.io/ — retrieved 2026-05-26 — foundation-stable
- https://github.com/joelparkerhenderson/architecture-decision-record — retrieved 2026-05-26 — foundation-stable
- https://www.anthropic.com/research/building-effective-agents — retrieved 2026-05-26 — foundation-stable

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

The "alternatives considered" section in an ADR is underused and often omitted. For HITL gate ADRs, the most useful alternative to name is always "no gate" — forcing the author to articulate why the default (auto-execute) was rejected. If the author cannot make a crisp case against "no gate," the gate probably should not exist.

For the pair-project's intake-triage flow, the canonical three-ADR set for W3 Mon is:
1. **Single-vs-multi-agent** — why the Mon–Tue phase is single-agent ReAct rather than supervisor-worker. Justification: dynamic direction not yet required at intake scope; supervisor added Wed once the evaluator topology is defined.
2. **LangGraph as framework** — why LangGraph over raw LangChain or a custom orchestrator. Justification: native interrupt primitives, durable checkpointer, graph-state inspection API, D-033 posture.
3. **HITL #3 interrupt-node boundaries** — the gate ADR itself, per this topic.

Senior FDEs should also consider: what happens to the ADR when FAR 15.206 is amended in the next FAC cycle? The ADR's justification cites the clause; if the clause changes, the ADR is where you return. This is why "specific clause" beats "the FAR generally" as a citation — a general cite cannot trigger a revision when the specific rule changes.

LangGraph's `get_state_history(config)` returns every step the thread has taken. For regulated workflows, this API is the audit-replay surface — you can iterate state history, find the checkpoint just before the hard gate fired, and verify the audit row was written correctly before the externalisation. Senior FDEs should wire this into the pair-project's test suite as an integration test on the HITL gate.

</details>

Last verified: 2026-06-06
