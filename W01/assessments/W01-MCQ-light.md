---
template: scenario-mcq
week: W01
tier: light
released_at: Fri W1 16:30
duration_minutes: 20
fde_situations_assessed: [1, 2, 6, 10]
last_verified: 2026-05-18
---

# W1 Light MCQ — LLM Engineering Essentials

> **Light tier** for W1 — calibration is lower because the cohort is still bedding into the programme. W2 ramps to the standard senior + entry two-tier format. 10 questions, 20 minutes. No partial credit on multi-select.

## Section A — Bedrock + LLM fundamentals

### 1. Cohort just shipped a `/draft-solicitation` endpoint that calls Bedrock. The endpoint sometimes returns a 500 to the Spring caller when Bedrock returns a `ThrottlingException`. Which is the MOST appropriate first fix?

- A. Wrap the Bedrock call in `try/except: pass` to silently fall back to a static response.
- B. Add a structured `RetryStrategy` with exponential backoff + jitter, returning a 503 with `Retry-After` if all retries are exhausted.
- C. Catch the exception and return the raw exception message as the response body.
- D. Switch to a different LLM provider on first failure.

**Correct: B.** A masks failures; C leaks internals + violates the structured-output contract; D doesn't address the underlying transient nature of throttling.

### 2. The CO complains that the platform cited "48 CFR 47.305-2" which doesn't exist. Which is the BEST near-term mitigation given W1 has no RAG yet?

- A. Add a regex check that rejects any response containing a CFR citation.
- B. Add a HITL surface — flag any response containing a citation pattern below a confidence threshold for CO review before publish.
- C. Hard-code the FAR clause text into the system prompt.
- D. Tell the CO to verify all citations manually after each draft.

**Correct: B.** A is fragile (drops valid citations too); C doesn't scale + still won't ground; D shifts blame to the human without designing a surface. (B is the W1 Fri HITL touchpoint design.)

### 3. Which of these is NOT a hallucination failure mode an LLM endpoint should expect?

- A. Confabulation (inventing citations that don't exist)
- B. Stale knowledge (training cutoff predates a recent amendment)
- C. Schema drift (returning prose instead of the requested JSON)
- D. **Idempotency loss** (the model returns different output on identical input)

**Correct: D.** D is a property of the model (non-determinism), not a *hallucination* mode. A, B, C are hallucination/output-quality failures.

### 4. The pair pinned `langchain==0.1.16` in `requirements.txt`. By W2 Mon the cohort must migrate to LangChain v1.0. Which is the SIGNIFICANT thing v1.0 removes?

- A. The `Chain` class — sequential composition is now plain Python function calls.
- B. Bedrock integration.
- C. PromptTemplate.
- D. Output parsers.

**Correct: A.** The `Chain` class is removed; LCEL `|` pipes are no longer central; sequential composition is plain Python (per D-033, `skills/tech-research/references/known-bad-patterns.yml`).

## Section B — Federal context + security

### 5. The Agency CIO asks: "Why are we on Bedrock and not OpenAI direct?" Which is the BEST first response?

- A. "Bedrock is cheaper."
- B. "OpenAI direct doesn't have FedRAMP authorization as of Q1 2026; Bedrock has FedRAMP High in GovCloud regions, which matters for federal-data workloads."
- C. "Bedrock has better models."
- D. "Karsun's stack is on Bedrock."

**Correct: B.** A is sometimes true but isn't the deciding factor; C is debatable; D is loyalty to the default, not honest comparison.

### 6. The training-project's `api-gateway` skips JWT signature verification on `/evaluations/*` endpoints. Which OWASP LLM category does this MOST closely map to?

- A. LLM01 Prompt Injection
- B. LLM02 Sensitive Information Disclosure
- C. LLM07 Insecure Plugin Design / Insecure Output Handling
- D. **LLM08 Excessive Agency**

**Correct: D.** LLM08 (or in the 2025 list, "Excessive Agency / Unauthorized Actions") is the closest — the agent (via tools called through this gateway) has access it shouldn't because auth is bypassable. C is also defensible; instructor accepts either C or D with rationale.

## Section C — FDE situations + cohort patterns

### 7. The cohort's W1 Tue brownfield-debt inventory captured 12 items. The item "audit-log write happens AFTER response (race condition: process crash between response and audit row = lost row)" surfaces in which later week?

- A. W2 RAG eval harness.
- B. **W3 multi-agent HITL audit-trail work + W5 Wed AIOps governance.**
- C. W4 Wed AI Security Engineering only.
- D. W6 Client Deliverability only.

**Correct: B.** The audit-log race condition is named in the v2 grid for W3 Thu HITL audit-trail work and re-surfaced in W5 Wed AIOps governance.

### 8. The W2 Mon Plan Day Scenario Design Planning brief asks the pair to produce six artifacts before any code lands. Which is NOT one of them?

- A. Requirements synthesis
- B. 6R / system map + diagram
- C. **Failing tests for every endpoint**
- D. Risk register

**Correct: C.** The six are: requirements synthesis, 6R/system map, ADRs, estimate, risk register, open questions. Failing tests don't appear until implementation Tue.

### 9. Cohort first sees Codex Adversarial Review on PRs in W1. Per D-034's calibration ramp, what does "Light" strictness mean for W1?

- A. Codex runs but doesn't post findings.
- B. Codex flags P0 findings as blocking, P1 findings as coaching framing.
- C. Codex flags everything as blocking from Day 1.
- D. Codex doesn't run until W2.

**Correct: B.** Light strictness per D-034 — P0 floor blocks, P1 architectural-drift framed as coaching. Ramps to Full by W4.

### 10. The Pair Project starts on W1 Thu and ends with the Final Defense on W6 Fri. Phase 1 of the Pair Project concludes when?

- A. W2 Fri (RAG eval Friday)
- B. **W3 Fri (Phase 1 Presentation & Defense + Mid-Program Retro)**
- C. W4 Mon (Phase 2 plan day)
- D. W5 Fri (Final Adversarial Review PR)

**Correct: B.** Phase 1 = W1 Thu – W3 Fri per D-038. The Phase 1 gate at W3 Fri concludes it.

---

## Scoring

10 questions × 1 point each. Light tier passing threshold: **6/10**.

The MCQ is light because W1 is the cohort's foundation week. The mid-tier MCQ from W2 onward adds reasoning depth — multi-step questions, more anti-patterns to spot, more "defend your answer" framing.

## Instructor notes

- Items 2, 6, 7, 9 are the "FDE-situation density" items — the cohort missing these signals deeper trouble.
- Item 4 (LangChain v1.0) is a *known-bad-pattern* awareness test — see `skills/tech-research/references/known-bad-patterns.yml`.
- Item 10 is the cohort-orientation check — pairs who can't yet name the gate structure won't pass W3.
- Use scores to inform the W2 Mon Plan Day instructor coaching — pairs scoring 6 get extra Mon morning attention.
