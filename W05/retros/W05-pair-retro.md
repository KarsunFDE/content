---
template: weekly-retro
week: W05
audience: pair
authored_by: pair (3 pair retros total)
released_at: Fri W5 14:30 (after cohort retro)
duration_minutes: 30
last_verified: 2026-05-23
---

# W5 Pair Retro Template

> One per pair, authored Fri afternoon after the cohort retro + Final Adversarial PR scoring.

## Frontmatter (per retro)

```yaml
---
week: W05
pair: pair-N
phase: 2 (operationalisation)
gate: Final Adversarial Review PR (cleared / conditional / below-threshold)
score: x/40
tier: Distinguished | Proficient | Developing | Below threshold
authored_at: 2026-06-26T14:45:00Z
---
```

## 1. What our pair shipped (3–5 bullets)

- ...
- ...

## 2. What our Fri PR taught us (5–8 bullets)

What landed cleanly. What broke. What codex found that surprised us. What we defended successfully and what we accepted.

- ...
- ...

## 3. AIOps + observability — Tue's OTel work

What worked when we instrumented the 4 services + the SPA. What didn't. Where the W3C `traceparent` propagation broke and how we fixed it. Whether the `gen_ai.*` semconv attributes feed Datadog dashboards meaningfully.

- ...

## 4. HITL #7 — our auto-remediation authority decision

Which option we committed (Full Auto / Propose-and-Await-Approval / Escalate-only). Why. What the instructor pushed back on during the Wed ADR session. Whether our Fri PR enforcement matches the ADR or drifted.

- ...

## 5. Datadog vs Dynatrace / New Relic / Coralogix — what we learned doing the compare-matrix

Where the Datadog hands-on choice wins for `acquire-gov`. Where it wouldn't. Whether we'd recommend a different platform for a different engagement shape.

- ...

## 6. AWS managed-service migration ADRs

What we migrated (Bedrock Knowledge Base / Agents-for-Bedrock / both / neither). Why. The measured latency / cost / debuggability deltas. Whether the worth-it ADR holds up after Fri's PR session.

- ...

## 7. OWASP LLM Top 10 mitigations — what we shipped

Which categories we addressed. Why we picked those 3+. Where we ran out of time. What we'd ship if we had another day.

- ...

## 8. Where Claude helped most this week

- ...

## 9. Where Claude actively hurt or misled

- ...

## 10. What our pair carried into Fri PR scoring well

Defenses that worked. Codex findings we caught early. Test coverage that surprised codex.

- ...

## 11. What we'd do differently for the next AIOps week

If we re-ran W5 Tue-Fri, what would we change?

- ...

## 12. W6 capstone — what we're carrying forward

Specific things from W5 that will shape our W6 runbook + ADR catalog + handoff package. The HITL #7 ADR enters W6 as a load-bearing artifact for the cohort-wide authority-boundary table.

- ...

## 13. Programme-level reflection

Phase 1 (W1–W3) was AI adoption into brownfield. Phase 2 (W4–W6) is modernization driven by Phase 1 discoveries. What did the W4→W5 handoff reveal? What's our pair's pace heading into W6?

- ...

## 14. Open questions we want surfaced to cohort retro

- ...

---

## Anti-patterns in retro authoring

- **Generic platitudes.** "We learned a lot" is not a retro. Be specific — name file:line, test name, ADR ID.
- **Avoiding the defense critique.** If codex caught something, name it. If we caught codex being wrong, name it. The retro is a record of engineering judgement, not a victory lap.
- **Skipping the Claude-hurt question.** Q9 must have at least one bullet. The cohort's onboarding-patterns playbook depends on honest assessment.
