---
week: W04
day: Fri
topic_slug: amended-plan-spec-further-reading
topic_title: "Amended plan-spec — further reading (extended cases, multi-amendment ADRs)"
parent_overview: W04/pre-session/5-Friday/1-DailyTopicOverview.md
parent_topic: W04/pre-session/5-Friday/5-amended-plan-spec.md
estimated_minutes: 6
sources:
  - url: https://adr.github.io/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://github.com/joelparkerhenderson/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://github.com/github/spec-kit/blob/main/spec-driven.md
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Amended plan-spec — further reading

> **Companion to [5-amended-plan-spec.md](./5-amended-plan-spec.md).** Primary reading covers the four-slot amendment block, the amend-vs-supersede decision, and two real-world cases (fintech timeout, logistics route-optimisation). This appendix adds healthcare and e-commerce cases and covers multi-amendment ADR conventions and the supersession-promotion criteria.

## Extended Real-world Patterns

**Healthcare — clinical-rule engine ADR.** A hospital network's clinical-rule engine had an ADR documenting "fail-open" behaviour when the rule database was unreachable (return no warning rather than block the action). After a near-miss where a fail-open default produced an inappropriate clinical recommendation, the team amended: the amended decision was "fail-closed for warning-class rules above severity threshold; fail-open only for advisory rules." The amendment shipped same day (small code change) and the ADR's audit trail was complete. *The healthcare-network's compliance reviewer cited the amendment as evidence of structured change-control during their annual review — the structural integrity of the document was the deliverable, not just the new behaviour.*

The lesson generalises: in regulated industries, what the compliance reviewer scores is the *shape* of the change-control trail, not the engineering quality of the fix. An amended ADR with all four slots filled in passes the audit even when the original decision was suboptimal; a rewritten ADR fails the audit even when the new decision is excellent.

**E-commerce — search-relevance ADR.** A retailer's search-relevance team had an ADR specifying "BM25 ranking with manual boosts for promotional items." A drop in conversion after a deploy revealed the manual boosts were over-fitting. The team amended with a slot-1 reference to the specific manual-boost mechanism, a slot-2 description of the conversion-drop data point, a slot-3 amended decision moving to learned-boost weights with a rollback flag, and a slot-4 stage for the next sprint. *The ADR's git history showed both the original BM25-plus-manual-boosts decision and the amendment, and the team's product manager used the audit trail in the quarterly review to demonstrate the team had moved deliberately and with evidence — not chaotically.*

## Multi-amendment ADRs — when to amend again vs. promote to supersession

A real ADR can accumulate multiple amendments over time. The convention:

- **Append each amendment underneath the previous,** each with its own `## Amendment (YYYY-MM-DD)` header, status, and authors. Never inter-leave slots between amendments.
- **If a later amendment refines an earlier amendment** (not the original decision), the later amendment's slot 1 should reference the earlier amendment explicitly: "Amending Amendment 2 (2026-04-12)'s constraint on cache-bust debounce timing."
- **Re-review the parent ADR every 2–3 amendments and consider supersession.** Accumulated amendments often signal that the original decision needs a structural rethink rather than another refinement layer.

The promote-to-supersession criteria, per practitioner consensus ([Joel Parker Henderson](https://github.com/joelparkerhenderson/architecture-decision-record); [ADR canonical guidance](https://adr.github.io/)):

1. **The amendments collectively change the decision's *kind*, not just its parameters.** A series of amendments adjusting timeout values is still amendment territory. A series of amendments that have effectively replaced the chosen library is supersession territory — the new ADR should make that explicit.
2. **The original decision's context is no longer the operating context.** If the team has moved to a different scale, regulatory environment, or architectural baseline, supersession lets the new ADR re-frame the problem rather than try to bolt new constraints onto an old framing.
3. **Reviewers report that the original ADR is hard to read.** Three or four amendments deep, the document becomes a forensic narrative rather than a decision record. Supersession resets the readability.

## Extended Best Practices

- **Number amendments sequentially.** Amendment 1, Amendment 2, etc. — even if the dates are clear, the sequential number lets later amendments reference predecessors unambiguously.
- **Maintain a one-line table of contents at the top of the ADR** once it has 2+ amendments. Reader can scan "Decision (2025-11), Amendment 1 (2026-02), Amendment 2 (2026-04), Amendment 3 (2026-05)" before reading any of them.
- **When promoting to supersession, link both ways.** The new ADR should link back to the old; the old ADR's status block should link to the new.
- **Treat the original decision as immutable.** Once written and accepted, the original text never changes — only the status header at the top can be updated to reflect supersession or amendment-presence.
- **Audit the ADR pile quarterly.** ADRs that have accumulated amendments tend to need supersession at the quarter boundary, not at the moment of the third amendment.

## Extended Hands-on Exercise

After completing the primary reading's exercise (drafting the cache-TTL amendment for ADR-014), extend with:

1. **Multi-amendment thought experiment.** Assume your amendment ships and three months later a second incident reveals that even the 5-second-debounce cache-bust hook is too slow for flash-sale geometry. Draft Amendment 2 — and explicitly reference Amendment 1 in slot 1.
2. **Supersession criteria audit.** Look at your pair-project's existing ADRs (if any). Identify any that already have an informal "we changed our mind" comment in the code or commit log but no formal amendment. What's the smallest amendment block that would close the audit gap?
3. **Sequence of amendments to supersession.** Sketch a four-amendment trajectory for a hypothetical "circuit-breaker library choice" ADR that ends with promotion to supersession. What's the structural signal that triggers the supersession promotion?

## Extended Sources

1. [Architectural Decision Records — community canonical site](https://adr.github.io/) — retrieved 2026-05-26
2. [Joel Parker Henderson — architecture-decision-record GitHub](https://github.com/joelparkerhenderson/architecture-decision-record) — retrieved 2026-05-26
3. [AWS Prescriptive Guidance — ADR Process](https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/adr-process.html) — retrieved 2026-05-26
4. [GitHub spec-kit — Spec-Driven Development](https://github.com/github/spec-kit/blob/main/spec-driven.md) — retrieved 2026-05-26

Last verified: 2026-05-26
