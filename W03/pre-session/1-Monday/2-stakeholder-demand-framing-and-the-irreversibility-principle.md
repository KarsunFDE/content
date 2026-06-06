---
week: W03
day: Mon
topic_slug: stakeholder-demand-framing-and-the-irreversibility-principle
topic_title: "Stakeholder demand framing — the CO and the irreversibility principle"
parent_overview: W03/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://aws.amazon.com/executive-insights/content/how-amazon-defines-and-operationalizes-a-day-1-culture/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.anthropic.com/research/building-effective-agents
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.aws.amazon.com/wellarchitected/latest/operational-readiness-reviews/wa-operational-readiness-reviews.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/articles/cant-buy-integration.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://adr.github.io/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-06-06
---

# Stakeholder demand framing — the CO and the irreversibility principle

> [!NOTE]
> **From earlier:** W2 Thu's HITL #2 envelope (RAG fallback when retrieval underperforms) was the second time this cohort saw HITL as a concept. Today it becomes a named ADR with a FAR citation.

## 1. Learning Objectives

- Distinguish a *risky* action from an *irreversible* action — the two are not the same axis.
- Translate a stakeholder's prose constraint into a falsifiable rule a platform can enforce at runtime.
- Apply Bezos's one-way door / two-way door model to classify agent transitions.
- Name the three artefacts every irreversible decision must leave (audit row, notification, externalisation marker).

## 2. Introduction

The CO's constraint this week is a single sentence: *"I — and only I — fire the irreversible actions."* That sentence is load-bearing. It does not say "risky actions." It says *irreversible* — and the distinction is the whole game.

Stakeholder demand framing gets the stakeholder to name the irreversible boundary in language a system can encode, then encodes it at the right layer. Standard failure mode: the engineer guesses, gates everything that feels risky, and alert fatigue erodes the gate's value when an actual irreversible decision arrives. Gating reversible actions is a tax on velocity; missing an irreversible one is a compliance finding.

## 3. Core Concepts

### 3.1 Irreversibility is a property of the outside world, not the codepath

A database write is reversible until something downstream consumes it. An email is irreversible the moment it leaves the SMTP relay. A public-registry publish is irreversible at HTTP 200. Bezos's one-way door framing: a one-way door has "significant and often irrevocable consequences" — capital, regulatory, or reputational commitments the organisation cannot walk back ([AWS Executive Insights](https://aws.amazon.com/executive-insights/content/how-amazon-defines-and-operationalizes-a-day-1-culture/), retrieved 2026-05-26).

In the CO's world: routing to Evaluator A is reversible (re-route). Escalating to the CO's queue is irreversible (audit row exists, queue position visible to vendors). Publishing an amendment under FAR 15.206 is irreversible (vendor notifications fire). Awarding under FAR 5.705 is irreversible (public announcement).

> [!TIP]
> **Operational implication:** When classifying a transition in the war-room exercise, ask "what external system has already observed this?" before assigning a gate type. The externalisation point — not the code path — is where the irreversibility argument lives.

### 3.2 Risk and irreversibility are independent axes

| | Reversible | Irreversible |
|---|---|---|
| **Low risk** | Auto-execute, log | Single confirmation; no escalation |
| **High risk** | Soft gate (suggest, allow override) | Hard gate (block until authorised human acts) |

Gating reversible-but-risky actions with the same friction as irreversible ones causes alert fatigue. Classify first, gate second.

### 3.3 The velocity argument — act at 70% data on two-way doors

Amazon's framing: most decisions are two-way doors — act at ~70% data, observe, and reverse if evidence turns. Every false gate taxes that threshold, cumulatively pushing toward 90% data — the slow-and-cautious failure mode the principle prevents.

> [!IMPORTANT]
> **Three artefacts every irreversible decision must leave:** (1) **Audit row** — actor, timestamp, intent, outcome; written *before* the external call. (2) **Notification** — confirmation back to the authoriser. (3) **Externalisation marker** — outside-world proof (HTTP 200, receipt ID). A hard gate without these three artefacts is theatre.

## 4. Generic Implementation

```python
def release_to_dispense(order_id: str, actor_id: str) -> ReleaseResult:
    """Hard gate. Fresh pharmacist approval token required at call site."""
    token = approvals.get_token(order_id, actor_id)
    if not token or token.expired() or token.role != "pharmacist_in_charge":
        raise GateBlocked("release_to_dispense requires fresh pharmacist approval")

    # Audit row BEFORE the external call.
    audit.write(
        action="release_to_dispense",
        actor=actor_id,
        order_id=order_id,
        approval_token_id=token.id,
        intent="dispense",
    )

    # Externalising call.
    receipt = pump_client.send(order.to_pump_payload())

    # Update audit row with externalisation marker.
    audit.complete(receipt_id=receipt.id, status=receipt.status)
    notifications.send(actor_id, f"Released order {order_id}; receipt {receipt.id}")
    return ReleaseResult(order_id=order_id, receipt=receipt)
```

Gate sits at the single irreversible boundary (`release_to_dispense`). Upstream steps are reversible — no gate needed. Audit-row-before-call means a crashed external call still leaves forensic evidence.

## 5. Real-world Patterns

**Fintech — Stripe test vs live mode.** Payment platforms separate reversible from irreversible with a parallel universe (test mode). Promoting to live mode is the irreversible step, visible in the API key namespace (`sk_test_…` vs `sk_live_…`). When the externalising boundary is large, make it visible in the type system, not buried in a config flag ([AWS Executive Insights](https://aws.amazon.com/executive-insights/content/how-amazon-defines-and-operationalizes-a-day-1-culture/), retrieved 2026-05-26).

**Healthcare — FDA-cleared infusion pump firmware.** Pumps require a second-check workflow before initiating delivery, encoded in firmware rather than the order-entry UI. Every upstream gate has been bypassed by exception flows over years; the only surviving gate is the one at the externalisation point. Mirrors AWS ORR discipline: review gates live where the consequence lands ([AWS ORR](https://docs.aws.amazon.com/wellarchitected/latest/operational-readiness-reviews/wa-operational-readiness-reviews.html), retrieved 2026-05-26).

**E-commerce — cancellation window as manufactured reversibility.** Most platforms expose a 30–60-minute cancellation window before dispatch. The platform manufactures reversibility by adding a delay — often cheaper than a hard gate. Ask whether a small delay converts any hard-gate candidate into a soft one.

> [!NOTE]
> **Cross-domain lesson:** Every domain has a manufactured reversibility pattern — a time window, staging step, or soft-commit — that converts a hard gate into a delay. For the CO's intake-triage flow, ask whether a 15-minute routing hold converts any hard-gate candidate into a cheaper soft gate.

## 6. Best Practices

- State the irreversibility argument in one sentence: "This is irreversible because *X has already observed it*." Can't complete it? Action is probably reversible.
- Place the gate at the outermost reversible boundary — the last call before the external world observes the change.
- Write the audit row *before* the externalising call; update with the externalisation marker after.
- Capture the stakeholder's exact words in the ADR — paraphrasing loses the constraint ([ADR spec](https://adr.github.io/), retrieved 2026-05-26).
- Prefer manufactured reversibility (delay + soft gate) over a hard gate where a minutes-scale lag is tolerable.

> [!WARNING]
> **Anti-pattern: `vibes-based-thresholds`.** Gating on "this feels risky" rather than a named irreversibility produces alert-fatigued operators who click through every confirmation dialog — including the ones that matter. The irreversibility argument is the gate's justification; without it, the gate is vibes dressed as architecture. Cite `skills/tech-research/references/known-bad-patterns.yml` slug `ragas-faithfulness-only` for the analogous eval failure mode.

## 7. Hands-on Exercise

You are designing checkout for a ticketing platform. The venue manager says: *"Reserves and abandons are fine. Once a ticket is confirmed paid, it MUST appear on the door-scan list within 60 seconds, and we MUST NOT exceed room capacity."*

Draw the state machine (start: browsing; end states: ticket-issued, abandoned, refunded). Label each transition: reversible or irreversible, gate type, implementation primitive.

> [!NOTE]
> **Self-check** (30s — answer mentally before expanding)
>
> 1. A row inserted into your own database — is it reversible or irreversible, and what determines the answer?
> 2. You design a hard gate on `enter_order` (the first step). What's the specific failure mode this creates?

<details>
<summary>Show answers</summary>

1. Reversible — until something downstream consumes it. The database write becomes irreversible at the moment the external world observes it (e.g., the Stripe receipt fires, the door-scan list is updated).
2. Alert fatigue on the low-friction steps. The gate at `enter_order` will fire constantly on normal reversible actions; operators learn to click through it, eroding its value when a genuinely irreversible action eventually hits.

</details>

## 8. Key Takeaways

- Irreversibility is what the *outside world* has observed — not the codepath.
- Risk and irreversibility are independent axes; the matrix drives the gate choice.
- Hard gates belong at the outermost reversible boundary — not scattered upstream.
- Every irreversible action leaves three artefacts: audit row (before the call), notification, externalisation marker.
- The CO's exact words are the citation; paraphrasing loses the constraint.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://aws.amazon.com/executive-insights/content/how-amazon-defines-and-operationalizes-a-day-1-culture/ — retrieved 2026-05-26 — foundation-stable
- https://www.anthropic.com/research/building-effective-agents — retrieved 2026-05-26 — foundation-stable
- https://docs.aws.amazon.com/wellarchitected/latest/operational-readiness-reviews/wa-operational-readiness-reviews.html — retrieved 2026-05-26 — foundation-stable
- https://martinfowler.com/articles/cant-buy-integration.html — retrieved 2026-05-26 — foundation-stable
- https://adr.github.io/ — retrieved 2026-05-26 — foundation-stable

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

Amazon's "two-way door at 70% data" principle has an important extension for multi-stakeholder systems: when *different* stakeholders have different irreversibility thresholds, encode the boundary per-role, not per-system. In the federal context, the CO, the Contracting Specialist, and the SSA each have distinct authority ceilings (FAR 1.602-1, FAR 15.308). A senior FDE should be able to map each stakeholder's ceiling to a specific gate type and ask whether the system correctly enforces the ceiling — not just whether a gate exists.

For healthcare parallels: FDA 21 CFR Part 820 (Software as a Medical Device) requires risk classification documentation analogous to the gate ADR. The regulatory analog is instructive: the FDA's risk classification is not about likelihood of harm, it is about *severity of harm when harm occurs* — which maps directly to irreversibility, not to risk.

</details>

Last verified: 2026-06-06
