---
week: W01
day: Thu
topic_slug: structured-json-output-pydantic
topic_title: "Structured JSON output — Pydantic and beyond"
parent_overview: W01/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://pydantic.dev/articles/llm-intro
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.pydantic.dev/latest/concepts/models/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.pydantic.dev/latest/concepts/validators/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://platform.openai.com/docs/guides/structured-outputs
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# Structured JSON output — Pydantic and beyond

## 1. Learning Objectives

By the end of this reading, the learner can:

- Explain why typed model outputs replace raw-string LLM responses in production code.
- Declare a Pydantic v2 model with strict-mode configuration and field constraints.
- Use field validators and `ValidationInfo.context` to enforce semantic invariants beyond static types.
- Identify when to use provider-native structured-output APIs versus a parsing library like Instructor.
- Recognise and refuse the "swallow-the-error" anti-pattern that hides regression from eval harnesses.

## 2. Introduction

For most of LLM application history, the contract between a model and its caller was a freeform string. The caller hoped the string contained JSON, hoped the JSON matched the agreed schema, and wrapped the whole thing in defensive try/except blocks. That worked for demos. It did not survive contact with production, where every silent parse failure compounds into invisible drift.

The 2025-2026 shift, documented across provider docs and ecosystem libraries, is to make structured output a *first-class* return type. The model still produces tokens, but the application sees a typed object validated against a declared schema before it ever leaves the inference boundary. Pydantic — the de facto Python data-validation library, now on its v2 line with a Rust-core rewrite — is the substrate that makes this contract enforceable in Python codebases.

The shift matters because the alternative — manual JSON parsing with defensive error handling — has two failure modes that compound. First, it hides regressions: a schema drift in the model's output looks identical to a transient network blip when you catch everything as a generic exception. Second, it pushes type-discipline to the boundaries of the codebase, so callers downstream have no compile-time guarantee about what they receive. Structured output collapses both problems into a single validation gate at the source.

This reading is the foundation Friday's PR work builds on. It covers the schema-declaration patterns, the validation lifecycle, and the discipline of failing loud so the eval harness (built in W2) actually sees the failures it needs to catch.

## 3. Core Concepts

### 3a. Pydantic v2 model declaration

A Pydantic model is a Python class that declares typed fields with optional constraints. The v2 idiom uses `BaseModel` as the parent and `ConfigDict` for model-level configuration — the v1 inner `class Config:` pattern is removed.

```python
from pydantic import BaseModel, ConfigDict, Field

class InvoiceLineItem(BaseModel):
    model_config = ConfigDict(extra="forbid", frozen=True)

    sku: str = Field(..., min_length=3, max_length=64)
    quantity: int = Field(..., gt=0, le=10_000)
    unit_price_usd: float = Field(..., ge=0)
```

Three configuration knobs do the heavy lifting:

- `extra="forbid"` rejects unexpected fields. Without it, a model that returns extra keys passes validation silently — and you discover the drift weeks later when something downstream depends on a field that no longer exists.
- `frozen=True` makes instances immutable after construction. Downstream code cannot mutate validated data; if the LLM contract changes, you create a new instance, not mutate the old one.
- `Field(...)` constraints (length, range, regex, default-factory) encode the semantic shape — what counts as a valid value, not just a valid type.

### 3b. Validators for semantic invariants

Type constraints catch the "is this an integer?" question. They do not catch "is this integer self-consistent with the rest of the object?" That's the validator's job. The v2 idiom is `@field_validator` for single-field checks and `@model_validator` for cross-field invariants.

```python
from pydantic import BaseModel, ValidationInfo, field_validator, model_validator

class Reservation(BaseModel):
    check_in: str
    check_out: str
    guest_count: int

    @field_validator("guest_count")
    @classmethod
    def positive_guests(cls, v: int) -> int:
        if v < 1:
            raise ValueError("at least one guest required")
        return v

    @model_validator(mode="after")
    def check_dates_ordered(self) -> "Reservation":
        if self.check_out <= self.check_in:
            raise ValueError("check_out must be after check_in")
        return self
```

The `mode` argument controls when the validator runs — `before` runs on raw input, `after` runs on the already-parsed value (the common case).

### 3c. Context-aware validation

Sometimes a validator needs information that isn't part of the model itself — the caller's identity, an allow-list, a tenant id. Pydantic v2 supports this via `ValidationInfo.context`, an opaque dict passed at validation time.

```python
from pydantic import BaseModel, ValidationInfo, field_validator

class PolicyQuery(BaseModel):
    tenant_id: str
    query: str

    @field_validator("tenant_id")
    @classmethod
    def tenant_allowed(cls, v: str, info: ValidationInfo) -> str:
        allowed = (info.context or {}).get("allowed_tenants", set())
        if allowed and v not in allowed:
            raise ValueError(f"tenant {v} not authorised")
        return v

PolicyQuery.model_validate(
    {"tenant_id": "acme", "query": "..."},
    context={"allowed_tenants": {"acme", "globex"}},
)
```

The gotcha worth memorising: context is only available via `model_validate(data, context=...)`. The direct constructor `PolicyQuery(...)` does not accept context. Get this wrong and validators silently see an empty context dict — they pass when they should reject.

### 3d. Three levels of structure enforcement

Industry guides for 2026 separate structured-output strategies into three tiers, escalating in reliability:

1. **Prompt-only:** ask the model nicely for JSON in the system prompt. Cheapest, least reliable; the model sometimes prepends prose or formats the JSON differently. Use only when the contract is loose.
2. **Parser-library:** call the model normally, parse the response with a library like Instructor or LangChain's structured-output helpers. Better; retries with validation feedback are usually automatic. Some latency overhead from compile + retry.
3. **Provider-native:** use the provider's structured-output API (OpenAI, Anthropic, Google all support this as of early 2026). The model's token generation is *constrained* at inference time so the output cannot violate the schema. First call compiles the grammar (~10s); subsequent calls are fast.

Friday's PR work targets tier 2 (parse-and-validate) as the baseline; tier 3 (Bedrock structured output) lands in W2 when the cohort starts pinning provider-specific behaviour.

### 3e. Fail loud, not silent

The most common production bug in LLM code is the "swallow-the-error" pattern:

```python
# DO NOT WRITE THIS CODE
try:
    parsed = MyModel.model_validate_json(raw_response)
except ValidationError:
    parsed = MyModel(default_fallback_values)   # silent recovery
```

The intent is reliability — the caller never sees an exception. The effect is invisible regression: every drift in the model's output, every prompt change that breaks the schema, every model-version rollover that subtly shifts behaviour, all of it disappears into the fallback path. Six weeks later, eval scores look fine and the system is quietly broken.

The correct shape is to *propagate* the failure as a structured error, log enough context for the eval harness to reproduce it, and either retry with corrected prompting or escalate to a human gate. The eval harness is the early-warning system; silent fallback disarms it.

## 4. Generic Implementation

Below is a concrete worked example outside the federal-acquisitions domain — a fintech model validating a payment instruction extracted from a customer chat:

```python
from pydantic import BaseModel, ConfigDict, Field, field_validator
import json

class PaymentInstruction(BaseModel):
    model_config = ConfigDict(extra="forbid", frozen=True)

    payer_account: str = Field(..., pattern=r"^[A-Z]{2}\d{14}$")
    payee_account: str = Field(..., pattern=r"^[A-Z]{2}\d{14}$")
    amount_minor_units: int = Field(..., gt=0, le=10_000_000)
    currency: str = Field(..., min_length=3, max_length=3)
    memo: str = Field("", max_length=140)

    @field_validator("payer_account", "payee_account")
    @classmethod
    def iban_uppercase(cls, v: str) -> str:
        if v != v.upper():
            raise ValueError("account number must be uppercase")
        return v


def parse_instruction(llm_response: str) -> PaymentInstruction:
    """Strict validation. Errors propagate; caller decides retry vs HITL escalation."""
    return PaymentInstruction.model_validate_json(llm_response)


# Caller observes failures explicitly
try:
    instruction = parse_instruction(llm_output)
except Exception as exc:
    # Log structured error for eval harness, then retry or escalate.
    # DO NOT silently substitute defaults.
    log_validation_failure(raw=llm_output, error=str(exc))
    raise
```

The annotations carry intent. `extra="forbid"` says "if the model invents a field, that's a contract violation." Field patterns say "an account number has a known shape." The validator says "uppercase is a domain rule, not a syntactic one." The `parse_instruction` boundary is the only place these rules live — downstream callers see typed objects, not raw strings.

## 5. Real-world Patterns

**Fintech — payment routing extraction:** A consumer-banking startup extracts payment instructions from chat messages. The team initially used regex to pull amounts and account numbers. Edge cases (transposed digits, decimal-vs-comma separators, currency-symbol ambiguity) caused mis-routed payments. Migrating to a Pydantic-validated extraction model with field-level constraints (`pattern=`, `ge`, `le`) cut mis-routes by an order of magnitude — not because the model got smarter, but because the validation gate refused malformed extractions and surfaced them as retries rather than silently routing them.

**Healthcare — clinical-note structuring:** A digital-health vendor structures free-text clinical notes into discrete fields (diagnosis codes, medication names, dosages). Their first deployment used the prompt-only tier; ~8% of outputs were unparseable or had subtle schema errors that downstream EHR integrations silently accepted. Switching to provider-native structured output with a Pydantic-mirrored schema eliminated the unparseable cases and made the subtle-error cases surface as validation failures the clinical-review team triages.

**E-commerce — product-catalogue normalisation:** A marketplace ingests vendor product feeds in inconsistent formats. The team uses LLMs to map vendor-specific fields onto a canonical Pydantic schema. The load-bearing decision was `extra="forbid"` — without it, vendors could (and did) sneak unmapped fields into the canonical store, fragmenting downstream search. With strict mode, every unmapped field surfaces as a validation error that the data-quality team reviews.

**Logistics — shipping-label extraction:** A 3PL operator extracts addresses from scanned shipping documents. Pydantic models with `field_validator` enforce that postal codes match country-specific patterns. The team built a small retry loop: on validation failure, feed the validation error message back into the LLM as a corrective prompt; usually the second attempt fixes it. The eval harness tracks first-pass success rate as a leading indicator of model or prompt drift.

## 6. Best Practices

- Set `extra="forbid"` on every LLM-output model. Allowing extra fields is allowing silent drift.
- Make models `frozen=True` so downstream code cannot mutate validated data. Mutation hides bugs.
- Use `field_validator` for normalisation and `model_validator` for cross-field invariants — do not stuff cross-field logic into a single-field validator.
- Pass tenant/user context via `model_validate(data, context=...)`, never the direct constructor — the direct constructor silently drops context.
- Fail loud on validation errors. Log the raw response, the schema version, and the error. The eval harness needs this signal.
- Pin the Pydantic minor version in production (`>=2.10,<3.0`); v2 is API-stable but minor versions add knobs (alias config, protected_namespaces defaults).
- Prefer provider-native structured output when latency permits; fall back to parser libraries when you need cross-provider portability.

## 7. Hands-on Exercise

**Task (10–15 min):** Write a Pydantic v2 model `WeatherReport` with:

- `location` (string, required, length 2-100)
- `temperature_celsius` (float, required, range -90 to 60)
- `condition` (string, must be one of: `clear`, `cloudy`, `rain`, `snow`, `fog`)
- `observed_at_iso` (string, required, must parse as ISO-8601 timestamp)
- `extra="forbid"`, `frozen=True`

Write a `field_validator` on `observed_at_iso` that calls `datetime.fromisoformat` and raises `ValueError` on parse failure. Write a small test that feeds the model a string containing one extra field (`humidity`) and asserts the validation fails with an `extra_forbidden` error.

**What good looks like:** The model rejects the extra field by name (not as a generic error). The validator returns the original string (or the parsed datetime — either is defensible if documented). The test fails fast with a readable error message that names the offending field. There is no try/except around the validation call in the test — failures propagate.

## 8. Key Takeaways

- *What is the production contract for an LLM call?* A typed Pydantic model with strict-mode config, not a string.
- *Why does `extra="forbid"` matter?* It surfaces schema drift instead of swallowing it; without it, regressions go invisible.
- *When do I need `ValidationInfo.context`?* When a validator's correctness depends on caller identity or per-request allow-lists — and remember context only flows through `model_validate`, not the constructor.
- *What's the difference between the three structure-enforcement tiers?* Prompt-only is unreliable; parser-library retries with feedback; provider-native constrains decoding at inference. Choose based on latency and portability needs.
- *Why is silent fallback an anti-pattern?* It hides regressions from the eval harness, which is the only system that catches drift before users do.

## Sources

1. [How to Use Pydantic for LLMs: Schema, Validation & Prompts (Pydantic)](https://pydantic.dev/articles/llm-intro) — retrieved 2026-05-26
2. [Models (Pydantic docs)](https://docs.pydantic.dev/latest/concepts/models/) — retrieved 2026-05-26
3. [Validators (Pydantic docs)](https://docs.pydantic.dev/latest/concepts/validators/) — retrieved 2026-05-26
4. [Structured Outputs (OpenAI Platform docs)](https://platform.openai.com/docs/guides/structured-outputs) — retrieved 2026-05-26
5. [Tool use overview (Anthropic docs)](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview) — retrieved 2026-05-26

Last verified: 2026-05-26
