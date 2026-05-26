---
week: W04
day: 3-Wednesday
topic_slug: security-validation-as-automated-tests
topic_title: "Security Validation as Automated Tests + Audit-Trail Completeness Check"
parent_overview: W04/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://docs.pydantic.dev/latest/concepts/validators/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://johal.in/pydantic-validation-layers-secure-python-ml-input-sanitization-2025/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.getmaxim.ai/articles/ai-agent-audit-logs-full-visibility-over-tool-usage/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://oneuptime.com/blog/post/2026-02-06-immutable-audit-log-pipeline-otel/view
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://dev.to/isabelle_dubuis_d858453d7/audit-trails-for-llm-apps-what-regulators-really-demand-2lf9
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# Security Validation as Automated Tests + Audit-Trail Completeness Check

## 1. Learning Objectives

By the end of this reading, the learner can:

- Apply the discipline: *"a security control that isn't tested is decorative"* — every input validator, output sanitiser, and gate ships with the tests that prove it fires on adversarial input AND lets legitimate input through.
- Use Pydantic v2 `field_validator` / `model_validator` patterns for both input shape validation and content sanitisation.
- Distinguish input validation (catch malformed/adversarial requests) from output sanitisation (strip leaked content from LLM responses before they reach the caller).
- Enumerate the four columns of an audit-trail completeness check (event emitted? caller identity bound? tenant bound? prompt/response shape preserved?) and apply it to a system under test.
- Recognise that audit-log gaps are themselves an OWASP LLM02 + LLM06 finding because you cannot gate authority you cannot observe.

## 2. Introduction

A defense that has not been tested is not really a defense. The reviewer who eyeballs the validator code, sees it strips dangerous patterns, and approves the PR is one refactor away from a regression. The discipline practised in security engineering is the same as in everything else: **the PR that adds the validator MUST also add the tests that prove the validator works** — both negative cases (adversarial input → `ValidationError`) and positive cases (legitimate input → clean round-trip). Without both shapes, you have a control you cannot observe.

The 2025 Gartner AI Security Report frames the cost: input vulnerabilities account for 62% of production ML failures, and Pydantic-layer validation has been shown to reduce ML input exploits by 94% when applied with both shape and content validators ([Johal, 2025](https://johal.in/pydantic-validation-layers-secure-python-ml-input-sanitization-2025/)). Pydantic v2 is now adopted by 73% of FastAPI ML services in production. It is the canonical Python validation layer for LLM-backed APIs.

The audit-trail piece is the *companion* discipline. SOC 2, HIPAA (45 CFR § 164.312(b)), and PCI-DSS 4.0 Requirement 10 all require audit trails with tamper-evident storage ([OneUptime, 2026](https://oneuptime.com/blog/post/2026-02-06-immutable-audit-log-pipeline-otel/view)). For LLM applications specifically, the audit trail captures the prompt-response pair, the caller, the tenant, the tool calls, and the gate decisions — because every one of those is something a regulator may later ask you to prove. The discipline this reading frames: a gate is only as good as the audit entry that records it fired.

## 3. Core Concepts

### 3.1 Two-shape testing — negative AND positive

A security test suite needs both:

- **Negative cases.** Known-bad input must raise (`ValidationError`, `AuthorityViolation`, `TenantBoundaryError`) — *not* silently pass and *not* return a default. A silent pass is the failure mode that lets controls erode over time.
- **Positive cases.** Known-good input must round-trip cleanly. A validator that catches everything by also catching legitimate traffic is a denial-of-service control, not a security control.

A PR that adds only negative tests is incomplete. The positive test prevents over-aggressive sanitisation; the negative test prevents under-aggressive sanitisation. Both shapes guard against opposite regressions.

### 3.2 Pydantic v2 — input validation patterns

The two canonical decorators ([Pydantic Docs](https://docs.pydantic.dev/latest/concepts/validators/)):

| Decorator | Scope | When to use |
|---|---|---|
| `@field_validator('field_name')` | Single field | Per-field shape, length, character whitelist, format check |
| `@model_validator(mode='after')` | Whole model | Cross-field invariants (if A is set, B must also be set), composite constraints |

Mode `before` validators run on raw input before coercion; `after` validators run on the parsed value. For security work, `before` validators are most useful because they catch bytes the attacker controls before any coercion smooths them out.

### 3.3 Input validation vs output sanitisation

Two different defenses, often confused:

- **Input validation.** Runs at the API edge. Rejects malformed or adversarial requests before they touch the LLM. Catches: oversized payloads, character-set violations, encoding attacks, known-bad instruction shapes.
- **Output sanitisation.** Runs on the LLM's response *before* it returns to the caller. Strips: echoed-back system prompts (LLM07 partial defense), echoed user PII (LLM02 partial defense), tool-call mimicry (markup that would trigger downstream parsers).

Both are required because they catch different failure modes. Input validation catches attacks at delivery; output sanitisation catches successful injections at exfiltration.

### 3.4 The audit-trail completeness check

For every LLM-mediated endpoint, ask four questions ([Maxim, 2025](https://www.getmaxim.ai/articles/ai-agent-audit-logs-full-visibility-over-tool-usage/); [Dubuis, 2025](https://dev.to/isabelle_dubuis_d858453d7/audit-trails-for-llm-apps-what-regulators-really-demand-2lf9)):

1. **Event emitted?** Does the endpoint write an audit-log event on every call (success AND failure)?
2. **Caller identity bound?** Does the event include `sub` / `email` / `user_id` and the auth method?
3. **Tenant bound?** Does the event include `tenant_id` and the role at time of call?
4. **Prompt/response shape preserved?** Does the event capture enough of the input + output that an investigator could reconstruct what the LLM did?

A "no" in any column is itself a finding. You cannot prove a gate fired if no audit entry recorded that it did.

### 3.5 Immutable, append-only, cryptographically-verified

The 2025-2026 consensus for audit-trail storage:

- **Append-only.** No UPDATE, no DELETE. Once written, the row cannot change.
- **Cryptographic hashing.** Each entry contains a hash of the previous entry (chain of trust). Tampering is detectable because the chain breaks.
- **Immutable backend.** S3 with Object Lock, Azure Immutable Blob, GCS retention policies — backends that enforce immutability at the infrastructure layer.
- **Centralised collection.** OpenTelemetry SDK + Collector is the emerging standard for emitting and forwarding audit events without coupling application code to a specific storage backend ([OneUptime, 2026](https://oneuptime.com/blog/post/2026-02-06-immutable-audit-log-pipeline-otel/view)).

The infrastructure layer matters because application code alone is not tamper-evidence. An attacker with database access can rewrite history if the database allows UPDATE; only an append-only backend with cryptographic chaining survives that attack.

## 4. Generic Implementation

A generic Pydantic v2 + audit-emitting endpoint, illustrating both shapes:

```python
# app/api/safe_endpoint.py
# Generic example: input validation + output sanitisation + audit emission.

import re
from pydantic import BaseModel, Field, field_validator

# 1. Input model — both shape and content validation.
class SummariseRequest(BaseModel):
    text: str = Field(..., min_length=1, max_length=10_000)
    locale: str = Field(default="en")

    @field_validator("text", mode="before")
    @classmethod
    def strip_known_bad_patterns(cls, v: str) -> str:
        # Strip null bytes, zero-width chars (sophistication-tier-4 mitigation).
        v = v.replace("\x00", "").replace("​", "").replace("‌", "")
        # Reject the common "ignore previous" instruction shape outright.
        if re.search(r"(ignore|disregard)\s+(previous|prior|all)\s+instructions",
                     v, re.IGNORECASE):
            raise ValueError("input contains known prompt-injection pattern")
        return v

    @field_validator("locale")
    @classmethod
    def whitelist_locale(cls, v: str) -> str:
        if v not in {"en", "es", "fr", "de", "pt"}:
            raise ValueError(f"locale '{v}' not in allowlist")
        return v


# 2. Output sanitisation — strip system-prompt fragments before returning.
SYSTEM_PROMPT_MARKERS = [
    "you are a helpful assistant",
    "<system>",
    "the following is the system prompt",
]

def sanitise_output(generated_text: str, original_input: str) -> str:
    # Strip leaked system-prompt markers (LLM07 partial defense).
    out = generated_text
    for marker in SYSTEM_PROMPT_MARKERS:
        if marker.lower() in out.lower():
            audit_log.security_event("system_prompt_leak_detected",
                                     marker=marker)
            # Replace the leaked content with a generic refusal marker.
            out = re.sub(re.escape(marker), "[REDACTED]", out, flags=re.IGNORECASE)
    # Strip echoed user input that exceeds a safe quote length (LLM02 partial).
    if len(original_input) > 50 and original_input in out:
        out = out.replace(original_input, "[user input redacted]")
    return out


# 3. Endpoint — wires the two together with audit emission.
@app.post("/summarise")
async def summarise(req: SummariseRequest, caller: User = Depends(current_user)):
    audit_log.event(
        kind="llm_call_start",
        caller_id=caller.id, tenant_id=caller.tenant_id,
        endpoint="/summarise", input_length=len(req.text),
    )
    raw = await llm.summarise(req.text, locale=req.locale)
    safe = sanitise_output(raw, req.text)
    audit_log.event(
        kind="llm_call_end",
        caller_id=caller.id, tenant_id=caller.tenant_id,
        endpoint="/summarise", output_length=len(safe),
        input_hash=hashlib.sha256(req.text.encode()).hexdigest(),
        output_hash=hashlib.sha256(safe.encode()).hexdigest(),
    )
    return {"summary": safe}
```

What each section does:

- **`field_validator(mode="before")`** runs on the raw string before any coercion — catches zero-width characters and known-bad patterns at delivery.
- **Locale allowlist** — a positive whitelist beats a denylist; the validator rejects everything not explicitly allowed.
- **`sanitise_output`** runs on the LLM's reply. Both system-prompt-leak markers (LLM07) and echoed-input fragments (LLM02) get stripped. Detection itself emits a security event.
- **Audit emission bracketing the LLM call** — input length + output length + hashes give an investigator enough to reconstruct without storing the full prompt verbatim (which could itself be a PII problem).
- **Caller + tenant + endpoint** on every audit event — answering the four completeness-check questions inline.

> [!instructor-review]
> The example deliberately doesn't use a LangChain `Chain` or LCEL pipe to compose the validation + LLM + sanitisation steps — that pattern is flagged in `known-bad-patterns.yml` as pre-v1.0 advocacy. The composition here is plain Python function calls, which is the LangChain v1.0 idiom.

## 5. Real-world Patterns

**Healthcare — Pydantic validators on FHIR-format clinical inputs.** A 2025 hospital pilot deployed an LLM-mediated clinical-notes assistant. The input layer is a Pydantic model with field validators that enforce FHIR resource schemas AND scan for known phishing-style instruction patterns in free-text note fields. The PR that added the validators also added 47 test cases — 22 negative (each adversarial pattern raises) and 25 positive (each normal-but-edge-case note round-trips). The 94%-exploit-reduction figure comes from this pattern applied at scale across the deployment ([Johal, 2025](https://johal.in/pydantic-validation-layers-secure-python-ml-input-sanitization-2025/)).

**Fintech — audit-trail completeness as an audit deliverable.** A 2025 trading-firm SaaS was preparing for a SOC 2 audit. The auditor's request was simple: "Show me the audit trail for any LLM-mediated decision in the trading workflow." The firm could not produce evidence for 30% of LLM-mediated calls because audit-log entries were missing the tenant binding (column 3 of the completeness check). The audit failed. Re-mediation cost: 6 weeks of engineering time to retrofit tenant binding into every audit-log call site. The lesson: completeness-check 4 columns is a release-gate, not a nice-to-have ([Dubuis, 2025](https://dev.to/isabelle_dubuis_d858453d7/audit-trails-for-llm-apps-what-regulators-really-demand-2lf9)).

**E-commerce — output sanitisation against PII echo.** A 2025 e-commerce platform's LLM-mediated customer-service agent began echoing back order details (full credit-card numbers in some cases) when customers asked "what's my order status?" The root cause: the LLM was retrieving order records that included raw PAN in a debug field. The fix combined three layers — strip the debug field at retrieval, redact PAN-shaped substrings in output sanitisation, and add a regression test that asserts no 16-digit-number sequence appears in any LLM response. Three layers because no single layer would have caught all the failure modes ([Maxim, 2025](https://www.getmaxim.ai/articles/ai-agent-audit-logs-full-visibility-over-tool-usage/)).

**Logistics — OpenTelemetry-emitted audit pipeline.** A 2026 logistics platform's audit-trail pipeline uses OpenTelemetry instrumentation in application code, with the OTel Collector configured to forward audit-tagged spans to S3 Object Lock storage. The pipeline meets SOC 2 + HIPAA + PCI-DSS 4.0 Requirement 10 simultaneously because the immutability is enforced at the storage layer, not in application code. The engineering blog notes that the abstraction also let them swap S3 for Azure Immutable Blob during a multi-cloud migration without changing application code ([OneUptime, 2026](https://oneuptime.com/blog/post/2026-02-06-immutable-audit-log-pipeline-otel/view)).

## 6. Best Practices

- **Every security PR includes both negative and positive test shapes.** A PR with only negative tests is incomplete.
- **Use `mode='before'` field validators for content sanitisation.** Catches attacker-controlled bytes before any coercion smooths them out.
- **Layer input validation and output sanitisation.** Each catches a different failure mode.
- **Audit-log every LLM-mediated call with the four-column completeness shape.** Event + caller + tenant + prompt/response shape.
- **Push immutability to the storage layer.** S3 Object Lock / Azure Immutable Blob / GCS retention. Application code alone is not tamper-evidence.
- **Hash the prompt and response, store the hash + length, not the full text by default.** Lets you prove what happened without making the audit log itself a PII liability.
- **Re-run the suite on every model swap and every Pydantic-version bump.** Validators and sanitisers can regress silently across upgrades.

## 7. Hands-on Exercise

**Code task (10–15 min):** You are given a Pydantic v2 input model for an LLM-mediated `/translate` endpoint:

```python
class TranslateRequest(BaseModel):
    text: str
    target_language: str
```

Harden it. Add:
- A length cap on `text` (1–5000 chars).
- A `before` field validator that strips zero-width characters and rejects requests containing the literal phrase "Ignore previous instructions".
- A `target_language` allowlist (en / es / fr / de / ja / ko / zh).
- A model validator that rejects requests where the source text appears to contain the target language string already (defense against language-confusion injection).

Then write three negative test cases AND three positive test cases. Each test asserts the exact expected outcome (`ValidationError` raised for negative; clean parse for positive).

**What good looks like:** the validators use `@field_validator` with `mode="before"` for content checks. The negative tests use `pytest.raises(ValidationError)` and check the error message matches the rule that fired. The positive tests round-trip normal translation requests and assert the parsed model equals the input. The candidate also adds a comment noting that the validators do NOT replace output sanitisation — the validator catches input attacks; the response from the LLM still needs output filtering before returning to the caller.

## 8. Key Takeaways

- A security control that isn't tested is decorative — every PR adding a validator ships with negative AND positive tests, can I cite both shapes?
- Pydantic v2 `field_validator(mode="before")` runs on raw input — am I using it for content sanitisation, not just shape validation?
- Input validation and output sanitisation are different layers catching different failure modes — does my endpoint do both?
- The audit-trail completeness check has four columns (event + caller + tenant + prompt/response shape) — does my audit log answer "yes" to all four for every LLM-mediated call?
- Immutability lives in the storage layer (S3 Object Lock, Azure Immutable Blob), not in application code — is my tamper-evidence enforced at infrastructure?

## Sources

1. [Validators (Pydantic Docs)](https://docs.pydantic.dev/latest/concepts/validators/) — retrieved 2026-05-26
2. [Pydantic Validation Layers: Secure Python ML Input Sanitization 2025 (Johal)](https://johal.in/pydantic-validation-layers-secure-python-ml-input-sanitization-2025/) — retrieved 2026-05-26
3. [AI Agent Audit Logs: Full Visibility Over Tool Usage (Maxim, 2025)](https://www.getmaxim.ai/articles/ai-agent-audit-logs-full-visibility-over-tool-usage/) — retrieved 2026-05-26
4. [How to Build an Immutable Audit Log Pipeline Using OpenTelemetry (OneUptime, 2026)](https://oneuptime.com/blog/post/2026-02-06-immutable-audit-log-pipeline-otel/view) — retrieved 2026-05-26
5. [Audit Trails for LLM Apps: What Regulators Really Demand (Dubuis, 2025)](https://dev.to/isabelle_dubuis_d858453d7/audit-trails-for-llm-apps-what-regulators-really-demand-2lf9) — retrieved 2026-05-26

Last verified: 2026-05-26
