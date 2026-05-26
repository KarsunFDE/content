---
week: W06
day: Tue
topic_slug: tradeoff-honesty
topic_title: "Tradeoff honesty"
parent_overview: W06/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 9
sources:
  - url: https://martinfowler.com/bliki/ArchitectureDecisionRecord.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.pdh-pro.com/pe-resources/engineering-ethics-the-importance-of-transparency/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.richardchambers.com/bridging-the-trust-gap-between-internal-audit-and-management/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://adr.github.io/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://suozziforny.com/scope-limitation/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Tradeoff honesty

## 1. Learning Objectives

By the end of this reading, the learner can:

- Recognise the three-part shape of a *defensible* tradeoff statement: two named values in tension + the rule that adjudicates + explicit acceptance.
- Distinguish "broken-but-named-broken" disclosure from "hidden" gaps and explain why the first builds trust while the second destroys it.
- Identify the three most common dishonest-tradeoff phrases ("we didn't have time", "it works fine in practice", "edge case we'll handle later") and rewrite each into a defensible form.
- Apply the "named gap, named owner, named ETA" remediation pattern when declaring an open issue.
- Pre-empt the most likely auditor objection by surfacing it in your own narrative first.

## 2. Introduction

Tradeoff honesty is the discipline of naming, in your own words, the limitation in your system before the audience finds it. It is *not* humility theatre and it is *not* a hedge against criticism. It is the operational technique that turns adversarial reviews (the auditor catches the gap and demands an explanation) into collaborative ones (the team named the gap and is leading the remediation conversation).

The engineering-ethics literature is direct about this: transparency is "the practice of openly and honestly communicating the facts, assumptions, risks, and uncertainties that influence engineering decisions" — including the disclosure of all viable options when differences could affect cost or safety ([Engineering Ethics: The Importance of Transparency, PDH-Pro, retrieved 2026-05-26](https://www.pdh-pro.com/pe-resources/engineering-ethics-the-importance-of-transparency/)). The internal-audit literature mirrors it: auditors earn credibility "not by being perfect, but by being predictable and principled" — the trust foundation is *what the audience can expect to hear*, not *what the team has solved* ([Audit Beacon, Richard Chambers, retrieved 2026-05-26](https://www.richardchambers.com/bridging-the-trust-gap-between-internal-audit-and-management/)).

The corollary holds in reverse. A *named* gap is half-closed; a *hidden* gap is whole-open and growing. Audiences make two judgements simultaneously: a judgement about your system, and a judgement about whether they can trust your description of your system. The second judgement decides how generously the first is interpreted. Tradeoff honesty is what makes the second judgement land in your favour.

This reading covers the discipline generically. The day's overview applies it to the W6 Tue stakeholder rotation; this reading is the underlying technique.

## 3. Core Concepts

### 3.1 The shape of a defensible tradeoff statement

A defensible tradeoff statement has three parts in order:

1. **Two named values in tension.** Not one disguised value. Saying "we optimised for performance" is not a tradeoff — there is no second value. Saying "we picked latency over completeness" is — both values are named and the audience can verify the choice.
2. **The rule that adjudicates.** A clause, policy, constraint, or business requirement that makes the chosen ordering of the two values *acceptable*. Without this, the tradeoff reads as preference; with it, the tradeoff reads as a reasoned decision.
3. **Explicit acceptance.** The word "acceptable" or its equivalent, stated clearly. This is the structural element that distinguishes a *tradeoff* (you chose A over B and you can defend it) from a *gap* (you would prefer both A and B but only have A).

A canonical example sentence:

> "We picked **latency** over **completeness** on the retrieval fallback path because the **24-hour acknowledgement window** allows missing the last paragraph of a late-arriving update — so missing the final paragraph is **acceptable**; missing the 24-hour SLA is **not**."

Parsed: two values named (latency, completeness); rule that adjudicates (the 24-hour window); explicit acceptance ("acceptable") plus the inverse boundary ("not"). The auditor cannot dispute this structurally — they can only dispute whether the rule cited is the right one or whether the values are correctly ordered. Both are productive disagreements, not destructive ones.

### 3.2 The "broken-but-named-broken" rule

Software systems ship with known limitations. The question is never "are there gaps?" but "do you know about them and have you said so?" The principle from the audit-trust literature applies: *predictable* communication earns trust faster than perfect communication ([Audit Beacon, retrieved 2026-05-26](https://www.richardchambers.com/bridging-the-trust-gap-between-internal-audit-and-management/)).

A broken-but-named-broken disclosure has four elements:

- **What is broken** — one sentence, in user-visible behaviour terms, not internal-implementation terms.
- **Why it is broken** — the proximate cause, named in language an auditor can verify (a missed test class, a deferred refactor, an upstream-dependency limitation).
- **Why it is acceptable for now** — the rule or business reality that makes the gap tolerable until remediation.
- **What closes it and when** — the remediation owner, the work item, and a date.

A *hidden* gap has none of these. When the auditor finds it, the conversation has to invent all four under pressure — usually badly.

### 3.3 Three common dishonest-tradeoff phrases and their fixes

**"We didn't have time."**

Non-falsifiable, reads as excuse. The audience cannot verify "didn't have time" — they can only verify what was prioritised. Replace with a *prioritisation* statement: "We deferred X to Q3 because Y had higher coupling to the upcoming Z migration; the X work would have churned twice if done now." Now the audience can verify the coupling claim and the migration timing.

**"It works fine in practice."**

Reads as "we don't have evidence." Replace with the evidence you do have, and name what the evidence *doesn't* cover: "We have not seen this fail in the four canary segments we measured over the last 60 days; we have not yet measured the fifth segment because we don't have representative traffic there." Now "fine in practice" becomes a bounded evidentiary claim — strong where it holds, honest about where it doesn't.

**"Edge case we'll handle later."**

Vague, ownerless, dateless. Replace with the four-element disclosure: name the case, name the proximate cause, name why it's acceptable for now, name the remediation Finding ID + owner + ETA. "Reviewer-edit-during-AI-draft race condition; root cause is unsynchronised state in the draft store; acceptable because draft store is one-author-at-a-time today; tracked as Finding F-2026-W6-031, owner backend lead, ETA before next FedRAMP attestation cycle."

The ADR community has a related discipline: every Architecture Decision Record explicitly captures *consequences*, not just the decision. "ADRs admit trade-offs, and if trade-offs are hidden, you lose learning" ([adr.github.io, retrieved 2026-05-26](https://adr.github.io/)). The Consequences section of an ADR is where defensible tradeoff statements live.

### 3.4 Pre-empting the auditor's first question

The most effective tradeoff-honesty technique is to imagine the auditor's first question and surface the answer before the question is asked. The internal-audit literature describes this as *predictable* communication — the management team that says, before the auditor finishes their opening sentence, "here are the three things we'd want you to look at first if you were us" wins the trust dynamic for the entire engagement ([Bridging the Trust Gap, Audit Beacon, retrieved 2026-05-26](https://www.richardchambers.com/bridging-the-trust-gap-between-internal-audit-and-management/)).

This is also the rule that makes the "scope limitation" finding manageable in formal audit contexts. When a scope is bounded — by available data, by time, by access — disclosing the boundary up front lets the audit reach a meaningful opinion within the boundary. When the boundary is hidden, the entire opinion is at risk of being downgraded to a disclaimer ([Scope Limitation in Auditing, retrieved 2026-05-26](https://suozziforny.com/scope-limitation/)).

### 3.5 Tradeoff-honesty vs over-disclosure

There is a failure mode in the opposite direction: disclosing so many caveats that the headline drowns. The discipline is *named-and-prioritised* disclosure, not exhaustive disclosure. Three to five named gaps in a headline-bearing system is typically the right range. Twenty named gaps signals either a system with deeper problems than the team has surfaced, or a team using over-disclosure as a defensive shield. Either way, the audience loses signal.

The corrective: rank gaps by impact. The headline carries the one or two gaps that materially affect the audience's decision. The appendix carries the long tail. The audit-trail walkthrough (see the corresponding reading) is where the long tail lives.

## 4. Generic Implementation

A worked example outside federal acquisitions: a consumer-banking team is launching a new ATM-fraud detection model and the internal model-risk team is reviewing.

**Dishonest version:**

> "The new model performs well across our customer base. There are some edge cases around international travel that we'll address in a future release. Overall, accuracy is good and we recommend launching."

What's wrong: "performs well" — undefined; "some edge cases" — uncounted, unowned; "future release" — no date; "accuracy is good" — no metric, no threshold, no holdout-vs-production comparison; "we recommend launching" — no acceptance criterion. The model-risk team has nothing to verify.

**Honest version:**

> "Recall on the 2024 confirmed-fraud holdout is **0.91**, up from **0.84** for the legacy rules engine. Precision is **0.78**, down from **0.86** — we picked **recall** over **precision** because the **deposit-account agreement §4.2** holds the bank to a 24-hour reimbursement SLA on confirmed fraud; missing fraud is materially more costly than the increased manual-review queue from false positives. The increased false-positive volume is acceptable under our current review-team capacity model.
>
> Two named limitations: (a) international-travel fraud patterns are under-represented in the training set — the holdout precision for transactions outside the cardholder's home country is **0.61**, not **0.78**. Closing via the 2026-Q4 international-fraud labelling project, owner: fraud-strategy lead; tracked as Risk-Finding RF-2026-Q2-014. (b) the model has not been re-tuned for the 2026 ACH same-day window change — re-tune scheduled for the next quarterly governance cycle.
>
> We recommend launch with the international-travel limitation disclosed in the customer-comms package and the review-queue capacity adjusted by **+15%** to absorb the precision shift."

What's right: precision and recall both named (the tension); deposit-account agreement §4.2 cited (the rule that adjudicates); "acceptable under our current review-team capacity model" (explicit acceptance); two limitations named, each with a metric, a remediation owner, a Finding ID, and a date; the recommendation has a concrete operational consequence (+15% review-queue capacity).

The model-risk team can now act. They can verify §4.2. They can dispute whether RF-2026-Q2-014 is sequenced correctly. They can question the +15% capacity number. All of those are productive disagreements. None of them are "what didn't you tell us?"

## 5. Real-world Patterns

**Aviation — pre-flight Minimum Equipment List (MEL).** Commercial aircraft dispatch with a named list of deferred equipment items under FAA-approved MELs. Every deferred item has a category (A/B/C/D), a maximum deferral period, and a cockpit placard. This is institutional broken-but-named-broken discipline — gaps are named, bounded, dated, and accepted by rule. Hiding a deferred item is a violation.

**Pharmaceuticals — drug-label "limitations of use".** FDA-approved labels include explicit "Limitations of Use" sections naming populations the drug is not approved for, conditions where efficacy was not demonstrated, and known interactions. The Limitations section is part of the approved safety case; marketing copy that contradicts it is a labelling violation.

**Civil engineering — geotechnical reports.** Structural engineers receive reports that explicitly name soil-sampling locations, seasonal conditions, and bore-hole depth limits. Design is built atop named assumptions; if as-built conditions differ, the geotechnical engineer's liability is bounded because the boundary was declared up front.

**Software — Architecture Decision Records (Nygard, Cognitect).** Nygard's ADR format makes the Consequences section co-equal with the Decision section ([Cognitect, retrieved 2026-05-26](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions); [Martin Fowler, retrieved 2026-05-26](https://martinfowler.com/bliki/ArchitectureDecisionRecord.html)). Teams report that Consequences is the most valuable section because it forces honest enumeration of downstream pain.

## 6. Best Practices

- **Name two values in tension, not one disguised value.** If you cannot name the second value, you have a feature description, not a tradeoff.
- **Always cite the rule that adjudicates.** A clause, policy, SLA, or business constraint. Without it, your tradeoff reads as preference.
- **Use the word "acceptable" explicitly.** Implied acceptance is no acceptance — audiences need to hear you accept the consequence in your own voice.
- **Replace excuse phrases with prioritisation statements.** "Didn't have time" → "deferred to Q3 because Y had higher coupling to Z."
- **Every named gap has an owner, a Finding ID, and a date.** Nameless, ownerless, dateless gaps read as theatre. Owned, IDed, dated gaps read as managed.
- **Pre-empt the auditor's first question in your opening narrative.** Imagine the question they would ask; surface the answer before they ask it.
- **Rank disclosures by impact.** Three to five named gaps in the headline; the long tail in the appendix. Over-disclosure drowns signal.

## 7. Hands-on Exercise

**Time:** 15 minutes. **Format:** rewriting + self-review.

You have three dishonest-tradeoff sentences below. Rewrite each into a defensible form using the three-part shape (two named values + adjudicating rule + explicit acceptance) and, where relevant, the four-element broken-but-named-broken disclosure.

**Sentence 1:**
> "We didn't have time to add caching to the search endpoint."

**Sentence 2:**
> "The API handles most traffic patterns fine. We'll revisit edge cases later."

**Sentence 3:**
> "We chose Postgres because it was the right tool for the job."

**Constraints:**

- You may invent plausible rules, policies, or business constraints to cite — just keep them realistic.
- Each rewrite should be 2–4 sentences.
- Each rewrite should be verifiable: an auditor reading it could write down "I need to check X" and X is concrete.

**What good looks like:** Sentence 1 cites a sequencing constraint (e.g., "the upcoming search-rerank work would invalidate the current caching strategy; caching deferred until rerank lands in Q3"). Sentence 2 names a specific traffic pattern that *isn't* handled, with a Finding ID and ETA. Sentence 3 names what Postgres was chosen *over* (e.g., "Postgres over DynamoDB"), the rule that adjudicates (e.g., "ACID requirement for our settlement table"), and accepts the consequence (e.g., "we accept the cross-region failover complexity in exchange for transactional guarantees"). If your rewrite still contains "fine", "later", or "the right tool for the job", it is still dishonest.

## 8. Key Takeaways

- Can I state a tradeoff with two named values, an adjudicating rule, and the word "acceptable" — in one sentence?
- For every known gap in my system, can I name an owner, a Finding ID, and a date?
- Have I removed every instance of "didn't have time", "works fine in practice", and "edge case we'll handle later" from my narrative?
- Have I imagined the auditor's first question and surfaced the answer in my opening?
- Are my disclosures ranked by impact — top three to five in the headline, long tail in the appendix?

## Sources

1. [Architecture Decision Record (Martin Fowler)](https://martinfowler.com/bliki/ArchitectureDecisionRecord.html) — retrieved 2026-05-26
2. [Engineering Ethics: The Importance of Transparency (PDH-Pro)](https://www.pdh-pro.com/pe-resources/engineering-ethics-the-importance-of-transparency/) — retrieved 2026-05-26
3. [Bridging the Trust Gap Between Internal Audit and Management (Audit Beacon)](https://www.richardchambers.com/bridging-the-trust-gap-between-internal-audit-and-management/) — retrieved 2026-05-26
4. [Architectural Decision Records (adr.github.io)](https://adr.github.io/) — retrieved 2026-05-26
5. [Scope Limitation in Auditing (Suozzi Forny)](https://suozziforny.com/scope-limitation/) — retrieved 2026-05-26

Last verified: 2026-05-26
