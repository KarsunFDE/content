---
week: W03
day: Mon
topic_slug: hitl-interrupt-node-boundaries-adr
topic_title: "HITL #3 — interrupt-node boundaries ADR"
parent_overview: W03/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 14
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
last_verified: 2026-05-26
---

# HITL #3 — interrupt-node boundaries ADR

## 1. Learning Objectives

By the end of this reading, the learner can:

- Classify any transition in an agent's state machine as no-gate / soft-gate / hard-gate, with a defensible argument.
- Write an Architecture Decision Record (ADR) for a single HITL gate, including context, options, decision, and consequences.
- Distinguish the *implementation primitive* for a gate (static `interrupt_before`, dynamic `interrupt()`, tool-layer approval token) from the *policy* the gate enforces.
- Apply the "one hard gate per pair" coaching heuristic and recognise the over-gating and under-gating failure modes.
- Cite a regulation or a reversibility argument as the justification for each gate, never "because it felt risky."

## 2. Introduction

HITL #3 is where human-in-the-loop becomes an *architectural commitment* rather than a discussion point. Each transition in the topology gets a classification (no/soft/hard gate), a justification, and an implementation primitive.

Every agentic system shipping into a regulated, safety-critical, or money-handling context faces the same question: *which transitions get a human gate?* Teams that ship successfully are not the ones who add the most gates — they are the ones who can defend each gate to a senior reviewer with a one-sentence justification. Teams that struggle either gate everything (alert fatigue) or gate nothing (the irreversible action slips through). The ADR exists to force the defence.

Anthropic's *Building Effective Agents* frames it well: agentic systems should treat the agent-computer interface with the same care as a human-computer interface — human checkpoints are part of the ACI, not exception handlers bolted onto an autonomous loop ([Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents), retrieved 2026-05-26).

## 3. Core Concepts

### 3.1 The three gate types

| Type | Behaviour | When to use |
|---|---|---|
| **No gate** | Auto-execute, log, move on | Reversible action; low-risk; or an action a downstream gate already covers |
| **Soft gate** | Suggest a default, surface to the human, allow approve/edit/reject | Reversible-but-risky; routing choices that benefit from sanity checks; ambiguous classifications |
| **Hard gate** | Block until a human resumes the graph; no auto-resume timeout | Irreversible action; regulatory requirement; externalising boundary |

The classification follows directly from the irreversibility principle (Topic 2): hard gates exist where actions are irreversible, soft gates where the system's default benefits from human review but the action can still be undone.

### 3.2 What an ADR for a HITL gate must contain

A defensible HITL ADR has six sections, following the standard ADR shape ([adr.github.io](https://adr.github.io/), retrieved 2026-05-26; [Joel Parker Henderson — ADR examples](https://github.com/joelparkerhenderson/architecture-decision-record), retrieved 2026-05-26):

1. **Title and status** — `ADR-NNNN: HITL gate on transition X` (status: proposed | accepted | superseded).
2. **Context** — what is the transition? What action does it produce? Who is the stakeholder constraint?
3. **Decision** — one of {no gate, soft gate, hard gate}, named explicitly. Resist hedging language.
4. **Justification** — either a regulation citation OR a reversibility argument. Not "because it felt risky."
5. **Implementation primitive** — the technical mechanism: `interrupt_before=[node]`, dynamic `interrupt()`, tool-layer approval token, queue-with-claim-check, etc.
6. **Consequences** — what does this gate cost (latency, on-call burden, alert volume)? What does it pre-empt (audit findings, incidents)?

A useful sub-section to add at the bottom is **alternatives considered** — even one sentence on why the chosen gate type beat the others helps future readers re-derive the decision.

### 3.3 The justification rule — regulation OR reversibility, never "risk"

Every gate must be backed by *either* a specific regulation citation *or* a reversibility argument from Topic 2. "Because it felt risky" is a feeling, not a justification.

- **Regulation-anchored gate** — name the clause (FAR 15.206, HIPAA §164.508, PCI-DSS Req 8.3, SOX §404). The clause is the durable artefact; if it changes, the ADR is where you revisit.
- **Reversibility-anchored gate** — name the externalisation (publishes to a public registry, sends an SMS via Twilio, writes to a system of record consumed by partners).

If you cannot complete either sentence, the gate is probably wrong — the action is reversible, or no regulation actually mandates the gate.

### 3.4 The implementation primitives

LangGraph's current surface offers two primitives for hard gates and one for soft gates ([LangGraph Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts), retrieved 2026-05-26):

- **Hard gate, static** — `interrupt_before=[node_name]` at compile time. Resume requires `Command(resume=...)` from outside.
- **Hard gate, dynamic** — `interrupt(payload)` inside a node. Pause conditional on runtime state. Cohort meets it Thu.
- **Soft gate** — tool-layer approval token or queue-with-claim-check. Often implemented as ordinary async APIs, not interrupts.

> [!instructor-review]
> **Static `interrupt_before=[...]` vs dynamic `interrupt()` — canonical story for W3.** Both primitives are valid in LangGraph v1.x. The programme's canonical wiring Mon→Tue→Wed→Thu is **static `interrupt_before`** for the named, FAR-anchored gate boundaries — simpler, inspectable at compile time, matches the war-room walkthroughs, and what production codebases everywhere still use. The LangGraph docs (retrieved 2026-05-26) describe **dynamic `interrupt()`** as the docs-canonical alternative when the *pause condition itself* is runtime-decided (e.g., "interrupt only when proposed refund > $500"). Today's ADR exercise names which primitive each gate in the pair's topology uses, with a defensible justification. Surface the nuance — do not let a learner read the LangChain blog and conclude the curriculum is stale.

### 3.5 The "one hard gate per pair" coaching heuristic

A pair-project's first intake-triage topology should typically have *one* hard gate (irreversible escalation) and *one or two* soft gates (routing decisions):

- **Zero hard gates** — the team has not internalised the irreversibility argument.
- **One hard gate** — typically correct. Sits at the externalising boundary (`escalate_to_human_queue`, `publish_to_registry`, `release_funds`).
- **Two hard gates** — defensible only with two genuinely independent irreversible actions. Most pairs conflate "high-risk" with "irreversible."
- **Three or more** — over-gating. Coach down.

Two to four soft gates is normal; they should outnumber hard gates by 2–4×.

### 3.6 Avoiding the future-gate trap

A common Plan-Day pitfall: importing a gate from a *future* part of the workflow into today's topology. Designing today's intake gates against tomorrow's regulation produces incoherent ADRs that defend the wrong action with the wrong clause. Be explicit about scope: this ADR covers transitions T1..Tn in *this* topology; other transitions get their own ADRs when they enter the topology.

### 3.7 The three artefacts the gate's implementation must produce

A hard gate that fires must leave three artefacts (cross-referencing Topic 2):

1. **Audit row** — actor, timestamp, intent, inputs, outputs. Written before the side effect.
2. **Notification** — back to authoriser or wider audience, confirming the action happened.
3. **Externalisation marker** — outside-world proof (HTTP 200, receipt ID, signed callback).

A graph with `get_state_history(thread_id)` gives you (1) for free — every checkpoint is an audit row ([LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence), retrieved 2026-05-26). The ADR should name which artefact the gate is responsible for producing and which the surrounding infrastructure handles.

## 4. Generic Implementation

A worked ADR outside federal-acquisitions: a logistics platform building an agentic shipment-exceptions handler. Inbound shipments occasionally arrive damaged; the agent triages the case, drafts a customer reply, and either auto-issues a small credit (≤ $50) or escalates to a human ops lead.

```markdown
# ADR-0011 — HITL gate on `issue_credit` transition

## Status
Accepted (2026-05-26)

## Context
The shipment-exceptions agent (`exceptions_graph_v2`) handles inbound damaged-shipment
claims. After triage, it reaches `prepare_credit_proposal` (reversible — drafts a credit
object in our internal DB but does not externalise). The next transition is one of:

- `issue_credit` — externalises via Stripe refund API. Money leaves the platform's
   merchant account; the customer's card is credited. The Stripe webhook fires
   downstream consumers (CRM, finance reconciliation, customer email).
- `escalate_to_ops_lead` — routes to a human queue with full context.

The CFO's stated constraint: "I — and only I — sign off on credits over $500. Below $500,
I want the system to act, but with an audit trail I can read every Monday."

## Decision

- Transition `issue_credit` when proposed credit ≤ $500:    **no gate** (auto-execute).
- Transition `issue_credit` when proposed credit > $500:    **hard gate** (block until
  ops lead resumes the graph with explicit approval).
- Transition `escalate_to_ops_lead`:                          **soft gate** (the agent's
  routing suggestion is surfaced to the ops lead with the option to redirect to a
  different queue).

## Justification

- The ≤$500 auto-execute path is justified by reversibility-with-acceptable-cost: refunds
  *are* irreversible in the regulatory sense, but the dollar amount is below the CFO's
  defined acceptable-loss threshold AND below the threshold where reversal-by-charge-dispute
  is operationally feasible.
- The >$500 hard gate is justified by the CFO's stated constraint. Citation: internal
  finance policy FIN-POL-2026-03 §4.2. (Not a public regulation; an internal policy with
  the same force.)
- The soft gate on `escalate_to_ops_lead` is justified by routing-quality: ops leads have
  historically corrected the agent's queue choice ~12% of the time; the soft gate
  surfaces the choice before committing.

## Implementation primitives

- `≤$500` auto-execute: ordinary edge in the graph. Audit row written by the
  `issue_credit` node before the Stripe API call.
- `>$500` hard gate: `interrupt_before=["issue_credit"]` at compile time. Resume requires
  `Command(resume={"approved": True, "approver_id": ...})` from the ops-lead UI.
- `escalate_to_ops_lead` soft gate: the agent emits the suggested queue as part of state;
  the ops-lead UI surfaces it and accepts an override; the actual queue assignment happens
  after the human's confirmation via a normal API call (no graph interrupt).

## Consequences

- The CFO gets a weekly audit-readable row per refund.
- Ops-lead on-call burden bounded by >$500 case rate (~3/wk).
- Alert fatigue avoided because the soft gate is non-blocking — click, not wait.
- Threshold tweaks (e.g., $250) require only the router-edge conditional to change.

## Alternatives considered

- Hard gate on every `issue_credit`: rejected; blocks ~95% of delegated cases.
- No gate: rejected; violates the CFO's stated constraint.
- Three-tier gating ($0–50/$50–500/$500+): rejected as over-engineering at current volume.
```

The shape of the ADR is what matters: a real decision with a real justification, named alternatives, and named consequences. A future engineer can pick this up and re-derive whether the decision still holds.

## 5. Real-world Patterns

**Healthcare — clinical-decision-support sign-off boundaries.** Modern CDS systems classify suggestions into "system can act" (e.g., adjust an alert threshold), "system suggests, clinician acts" (e.g., propose a medication dose), and "clinician must act, system records" (e.g., final treatment plan). Every transition between tiers gets an explicit policy — the analog of an ADR. The discipline is regulator-driven: the FDA expects AI/ML medical devices to document where the human is in the loop ([Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents), retrieved 2026-05-26).

**Fintech — payment-platform action policies.** Stripe-style platforms have internal "action policies" classifying every API endpoint into auto-execute, requires-platform-admin, requires-compliance-review. Each policy has the shape of a HITL ADR — externalisation is the API call, gate is the approval workflow, justification is regulation (KYC, AML, sanctions screening). Any platform with a money-movement boundary has a HITL gate there.

**Aviation — fly-by-wire authority modes.** Modern aircraft expose explicit "authority modes" (autopilot, autothrottle, manual, direct law). Transitions between modes are first-class architectural events with handover protocols. Avionics standards (DO-178C, ARP-4754A) require the equivalent of an ADR per transition. Software systems shipping into safety-critical contexts borrow from this whether they realise or not.

**E-commerce — order-fulfilment hold queues.** Large platforms maintain a "fraud-review queue" that is structurally a hard gate on the fulfilment transition. Each queue item is a paused workflow waiting for human resume. The ADR specifies threshold (risk score above N) and primitive (checkpoint pause on the risk-score edge).

## 6. Best Practices

- Write the ADR before the implementation, not after — the discipline is to defend the decision in prose first.
- Use one ADR per gate, not one ADR per topology — small, atomic ADRs age better and are easier to supersede.
- Cite the specific clause or the specific externalisation; vague justifications produce ADRs no-one will trust in six months.
- Default to *fewer* hard gates than your intuition suggests; the irreversibility argument is sharper than the risk argument.
- Pair every hard gate with the three artefacts (audit row, notification, externalisation marker) — the gate without artefacts is theatre.
- Use static `interrupt_before` for clarity at named boundaries; reach for dynamic `interrupt()` when the pause condition is itself dynamic.
- Re-audit the gates every cycle (the §0 retro from Topic 5) — gates accumulate and many become theatre over time.

## 7. Hands-on Exercise

**ADR-writing prompt (15 minutes).** You are designing an AI-assisted hiring platform that triages incoming applications. The recruiter-team lead: *"The system can rank candidates, draft outreach, and schedule first-round screens. But no rejection email goes out without my explicit approval, and no offer is extended without two recruiters signing off."*

For each transition below, write an abbreviated ADR (title, decision, justification, primitive):

1. `rank_candidates` — orders applicants by predicted fit.
2. `draft_outreach_email` — generates a personalised email for a top-ranked candidate.
3. `send_outreach_email` — externalises via SendGrid.
4. `schedule_screen` — books a recruiter calendar slot via Google Calendar API.
5. `send_rejection_email` — externalises via SendGrid.
6. `extend_offer` — generates and emails a formal offer letter.

Expected components:

- Two no-gates (rank_candidates, draft_outreach_email — internal-only, reversible).
- One soft-gate (send_outreach_email — externalises but the team lead delegates it).
- One soft or no gate (schedule_screen — externalises to recruiter's calendar; reversibility via cancellation).
- One hard-gate (send_rejection_email — explicit team-lead constraint).
- One hard-gate, *two-actor* (extend_offer — needs two recruiters to sign off, not just one).

**What good looks like.** Each ADR has a one-sentence justification you can defend in interview. You did not gate the internal-only steps — those are reversible and the constraint did not require it. You used the team lead's actual words as the citation for the rejection-email gate. The two-actor offer-extension gate is implemented as two soft gates that both must pass, not one hard gate with a special two-actor flag. Your total hard-gate count is two, your soft-gate count is one or two, and your no-gate count is two or three — within the heuristic from §3.5.

## 8. Key Takeaways

- Can you classify a given transition as no/soft/hard gate and write the one-sentence justification before reading on?
- Can you produce a complete HITL-gate ADR (title, context, decision, justification, primitive, consequences, alternatives) in under 15 minutes?
- Can you cite the justification as either a regulation clause or a reversibility argument, refusing the temptation to say "because it felt risky"?
- Can you defend the "one hard gate per pair" coaching heuristic and recognise when a pair is over-gating or under-gating?
- Can you name the three artefacts (audit row, notification, externalisation marker) every hard gate's implementation must produce?

## Sources

1. [LangGraph Interrupts — Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/interrupts) — retrieved 2026-05-26
2. [LangGraph Persistence — Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/persistence) — retrieved 2026-05-26
3. [Architectural Decision Records (ADRs)](https://adr.github.io/) — retrieved 2026-05-26
4. [Joel Parker Henderson — Architecture Decision Record examples](https://github.com/joelparkerhenderson/architecture-decision-record) — retrieved 2026-05-26
5. [Building Effective AI Agents — Anthropic](https://www.anthropic.com/research/building-effective-agents) — retrieved 2026-05-26

Last verified: 2026-05-26
