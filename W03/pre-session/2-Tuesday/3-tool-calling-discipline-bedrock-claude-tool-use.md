---
week: W03
day: Tue
topic_slug: tool-calling-discipline-bedrock-claude-tool-use
topic_title: "Tool-calling discipline — schemas and the Bedrock tool-use API"
parent_overview: W03/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters-anthropic-claude-messages-tool-use.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.aws.amazon.com/bedrock/latest/userguide/tool-use.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://platform.claude.com/cookbook/tool-use-tool-use-with-pydantic
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.claude.com/en/docs/build-with-claude/structured-outputs
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://pydantic.dev/docs/validation/latest/concepts/json_schema/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Tool-calling discipline — schemas and the Bedrock tool-use API

## 1. Learning Objectives

- By the end of this reading, the learner can describe the three-part request/response/thread-back shape of Claude's tool-use API on Bedrock InvokeModel.
- By the end of this reading, the learner can author a tool definition with a Pydantic-generated `input_schema` and name three classes of validation error caught at the boundary.
- By the end of this reading, the learner can distinguish a `tool_use` content block from a `tool_result` content block and explain how `tool_use_id` pairs them.
- By the end of this reading, the learner can list four production-grade hygiene rules for the tool surface (read/write separation, schema strictness, error handling, multi-tenant filters).
- By the end of this reading, the learner can recognise stale Bedrock model identifiers and stale tool-use shapes (OpenAI `functions`/`function_call`) and substitute the current equivalents.

## 2. Introduction

Tool calling — the API surface that lets an LLM ask the host application to execute a named function with structured arguments — is the load-bearing primitive under every agent framework. The model does not actually invoke your function; it emits a structured **request** for the function, and your host runtime decides whether and how to run it. That separation is what makes tool calls auditable, idempotent, and human-overridable.

On AWS Bedrock with Claude (Sonnet 4.5, Haiku 4.5, Opus 4.1 in 2026), tool calling is part of the Messages API surface — both the lower-level `InvokeModel` route and the higher-level `Converse` route support it. The shape is consistent across both: you send a list of tool definitions in the request, the model returns one or more `tool_use` content blocks in the assistant's response, you execute the named tools, and you thread the results back as `tool_result` content blocks on the next user turn. Anthropic also publishes structured-outputs and strict-tool-use modes that constrain the model's output to validate against your schema.

The reason to read this carefully before writing your first agent: tool-calling bugs are sneaky. The model will produce something that *looks* like a valid tool call but is missing a required field, or has the wrong type, or has been built from a stale tutorial that uses the deprecated OpenAI `functions` parameter or the long-removed `anthropic.claude-v2` model ID. Pydantic-validated schemas at the boundary catch these before they reach your database; without that discipline you ship a working demo that silently corrupts data in production.

## 3. Core Concepts

### 3.1 The three-part shape — request, response, thread-back

A Claude tool-use exchange on Bedrock has three messages:

**Request (you → Bedrock).** A `Messages` payload with a `tools` array. Each tool is `{name, description, input_schema}`, where `input_schema` is a JSON Schema object describing the inputs.

```json
{
  "anthropic_version": "bedrock-2023-05-31",
  "max_tokens": 4096,
  "tools": [
    {
      "name": "get_customer_orders",
      "description": "Retrieve the list of orders for a customer.",
      "input_schema": {
        "type": "object",
        "properties": {
          "customer_id": {"type": "string", "format": "uuid"},
          "limit": {"type": "integer", "minimum": 1, "maximum": 100, "default": 20}
        },
        "required": ["customer_id"]
      }
    }
  ],
  "messages": [{"role": "user", "content": "List the recent orders for Acme."}]
}
```

**Response (Bedrock → you).** A `content` array that may contain one or more `tool_use` blocks alongside any reasoning text:

```json
{
  "content": [
    {"type": "text", "text": "I'll look up Acme's recent orders."},
    {
      "type": "tool_use",
      "id": "toolu_01ABC123",
      "name": "get_customer_orders",
      "input": {"customer_id": "9f0d-...", "limit": 10}
    }
  ],
  "stop_reason": "tool_use"
}
```

**Thread-back (you → Bedrock, next turn).** The prior assistant turn is preserved **verbatim**, then a `user` turn carries a `tool_result` content block whose `tool_use_id` matches the `id` from the response:

```json
{
  "messages": [
    {"role": "user", "content": "List the recent orders for Acme."},
    {"role": "assistant", "content": [...prior assistant turn verbatim...]},
    {
      "role": "user",
      "content": [
        {
          "type": "tool_result",
          "tool_use_id": "toolu_01ABC123",
          "content": "[{\"order_id\": \"...\", \"total\": 1240.00}, ...]"
        }
      ]
    }
  ]
}
```

The `tool_use_id` is the linking primitive. If you reorder, drop, or rename it, the model loses the thread.

### 3.2 `input_schema` from Pydantic

JSON Schema is verbose. Pydantic v2 makes it ergonomic — define a `BaseModel`, call `.model_json_schema()`, hand the result to Bedrock. The Anthropic cookbook publishes this pattern as the recommended shape:

```python
from pydantic import BaseModel, Field
from uuid import UUID

class GetCustomerOrdersInput(BaseModel):
    customer_id: UUID
    limit: int = Field(default=20, ge=1, le=100)

tool_def = {
    "name": "get_customer_orders",
    "description": "Retrieve the list of orders for a customer.",
    "input_schema": GetCustomerOrdersInput.model_json_schema(),
}
```

Pydantic v2 is Rust-backed and validates 10–50× faster than v1, so the validation cost at the boundary is negligible even on hot paths.

### 3.3 Strict tool use (2025-11-13 beta and later)

Anthropic's strict-tool-use mode adds `"strict": True` to a tool definition and a beta header to the request. With strict mode on, Claude's emitted `input` is **guaranteed** to validate against the schema — no missing required fields, no type mismatches, no extra properties. Without strict mode you still need Pydantic validation at the boundary; with strict mode you keep the validation as a defence-in-depth check.

> [!instructor-review]
> Strict tool use is in public beta as of late 2025 on Sonnet 4.5 and Opus 4.1. Confirm GA status and Bedrock parity in the Sunday-before /web-research refresh before relying on it.

### 3.4 Reads vs writes — separating the surface

The hardest discipline in tool design is keeping reads cheap and writes deliberate. Reads can be plentiful — the model benefits from being able to interrogate state freely. Writes should be small in number, named for their effect (not their CRUD verb), and explicitly tagged in the tool surface. A useful pattern:

- **Read tools** are pure functions of the underlying state. Name them after the entity (`get_*`, `list_*`, `search_*`).
- **Write tools** mutate state and are named after the **business effect** (`route_to_evaluators` rather than `insert_routing_row`; `schedule_appointment` rather than `create_appointment_record`).
- **Write tools always carry an `idempotency_key` argument** (see topic 4) and are documented as such in their description so the model includes one.

### 3.5 Multi-tenant filters at the boundary

When the agent operates in a multi-tenant context (an enterprise SaaS, a federal-acquisitions platform with multiple agencies, a hospital network with per-clinic data), **the tenant filter does not belong in the prompt** — it belongs in the host runtime that executes the tool. The model receives a tool definition that accepts an `agency_id` or `tenant_id`; the host runtime overrides whatever the model passed with the authenticated request context. Treat any model-provided tenant identifier as untrusted input.

### 3.6 Stale shapes to avoid

Three patterns from older tutorials that no longer apply:

- **OpenAI `functions` / `function_call` parameter.** Deprecated in favour of `tools` / `tool_calls`. Any source using the old names is pre-2023-Q4.
- **`anthropic.claude-v2` / `anthropic.claude-instant` model IDs.** Outdated; current Claude on Bedrock uses Sonnet 4.5, Haiku 4.5, Opus 4.1 with regional inference profile IDs.
- **LangChain v0.x `Chain`-class tool wrappers.** Removed in v1.0; the current shape is `create_agent` or hand-rolled tool execution.

> [!instructor-review]
> Bedrock model IDs and regional inference-profile mappings change roughly quarterly. Re-verify the exact IDs used in cohort exercises in the Sunday-before /web-research sweep.

## 4. Generic Implementation

A generic read-and-write tool pair for a logistics dispatch system, illustrating the read/write naming discipline and the multi-tenant filter pattern. Notice: no Claude/Bedrock specifics — just the boundary discipline.

```python
from pydantic import BaseModel, Field
from typing import Literal
from uuid import UUID

# --- Read tool: pure function of state, plentiful, parameterised on the entity ---
class GetVehicleStatusInput(BaseModel):
    vehicle_id: UUID
    include_recent_events: bool = Field(default=False)

def get_vehicle_status(input: GetVehicleStatusInput, *, ctx: RequestContext):
    # Tenant override: ignore any tenant_id the model may have passed;
    # use the authenticated request context.
    return vehicles.find_one(
        {"vehicle_id": input.vehicle_id, "fleet_id": ctx.fleet_id},
        projection={"events": input.include_recent_events},
    )

# --- Write tool: named for business effect, carries idempotency_key, single-purpose ---
class DispatchVehicleInput(BaseModel):
    vehicle_id: UUID
    job_id: UUID
    eta_minutes: int = Field(ge=0, le=720)
    idempotency_key: str = Field(min_length=8, max_length=128)

def dispatch_vehicle(input: DispatchVehicleInput, *, ctx: RequestContext):
    # Idempotency check at the database layer — second call with the same
    # key returns the original result without side-effects.
    existing = dispatches.find_one({"idempotency_key": input.idempotency_key})
    if existing:
        return existing

    return dispatches.insert_one({
        "vehicle_id": input.vehicle_id,
        "job_id": input.job_id,
        "fleet_id": ctx.fleet_id,           # from context, not from the model
        "eta_minutes": input.eta_minutes,
        "idempotency_key": input.idempotency_key,
        "dispatched_at": utcnow(),
    })

# --- Tool registry handed to the model ---
TOOLS = [
    {
        "name": "get_vehicle_status",
        "description": "Read a vehicle's current status. Safe to call repeatedly.",
        "input_schema": GetVehicleStatusInput.model_json_schema(),
    },
    {
        "name": "dispatch_vehicle",
        "description": (
            "Dispatch a vehicle to a job. State-mutating. "
            "Caller MUST provide an idempotency_key derived from "
            "(vehicle_id, job_id) — retries with the same key are no-ops."
        ),
        "input_schema": DispatchVehicleInput.model_json_schema(),
    },
]
```

Three things to notice:

1. **The read tool's description does not warn about side-effects** because there are none.
2. **The write tool's description explicitly tells the model** about the idempotency contract — the model can comply because it knows the rule.
3. **`fleet_id` is taken from `ctx`, not from `input`.** The model cannot accidentally (or maliciously) dispatch a vehicle from another fleet.

## 5. Real-world Patterns

### 5.1 E-commerce — Shopify Sidekick

Shopify's merchant-assistant Sidekick exposes dozens of analytics read tools to the model (sessions, conversion, ad-spend, inventory, returns) and a small set of write tools (apply-discount, adjust-inventory). Sidekick's write tools all require explicit merchant confirmation — the agent emits a `tool_use` for a write, the host pauses the loop, the merchant sees a confirmation card, and only on approval does the host actually execute. The read/write asymmetry in the tool surface mirrors the trust asymmetry.

### 5.2 Healthcare — Epic's "Art" and clinical-decision-support copilots

Healthcare LLM products (Epic's "Art," Hippocratic AI, Glass Health) almost universally forbid model-driven writes to the EHR. The model can call any read tool — labs, imaging, prior notes, drug interactions — but all writes are routed through a human-approval queue. The schemas for the read tools are particularly strict: dates, ICD-10 codes, RxNorm IDs are all type-constrained, and Pydantic validation catches model-generated codes that don't exist in the canonical vocabularies.

### 5.3 Fintech — Plaid + Stripe assistants

Plaid and Stripe both ship LLM assistants whose tool surfaces are designed for transparency. Every write tool's input schema includes a `reason` field that the model fills in with natural language; that reason is logged with the action so post-hoc audit can see *why* the model thought the action was correct. The pattern travels well — any time you need an after-the-fact audit trail for an LLM-initiated action, capture the model's articulated reasoning in the tool's input schema, not just in a system log.

### 5.4 Gaming — Riot's player-support agents

Riot's player-support assistant uses a multi-tenant tool surface where the `region` (NA / EUW / KR / …) is taken from the authenticated session, not from the prompt. Players have repeatedly tried to prompt-inject the agent into looking up other players' accounts in other regions; the boundary-layer override makes these attempts no-ops. The pattern: **tenant-scoped IDs in the tool surface are advisory at best; the host enforces.**

## 6. Best Practices

- **Generate `input_schema` from a typed model.** Pydantic in Python, Zod in TypeScript. Hand-written JSON Schema drifts from the executor's actual expectations within weeks.
- **Validate at the boundary regardless of `strict` mode.** Strict mode constrains the model's output; it does not protect against malicious payloads if your runtime ever accepts tool inputs from another source.
- **Name write tools after their business effect.** `escalate_to_supervisor` is meaningful; `insert_escalation_row` leaks implementation detail and lets the model misuse it.
- **Document the idempotency contract in the tool description.** The model reads descriptions and tends to follow them; if you tell it to construct an idempotency key from a specific tuple, it usually will.
- **Override tenant identifiers in the host, not in the prompt.** A multi-tenant agent's safety boundary lives in code, not in instructions.
- **Return structured errors, never silent nulls.** A failed `get_*` should thread back `{"error": "not_found", "id": "..."}` so the model can reason about the failure rather than hallucinate around an empty response.
- **Cap tool count.** Beyond ~25 tools, model selection accuracy drops. If you need more, group them behind a router tool or split into specialised sub-agents.

## 7. Hands-on Exercise

**Code exercise (15 min).** You are exposing tools to a Claude agent that runs an interactive customer-onboarding workflow for a SaaS product. The agent needs three tools:

- `get_signup_status(email)` — pure read; returns onboarding stage.
- `send_welcome_email(email, template_id)` — write; should be idempotent so the same email is never sent twice.
- `provision_workspace(email, tier)` — write; even more expensive — must not double-provision.

Write the three Pydantic models, the three tool registry entries (with descriptions), and a stub `execute_tool(name, input)` function that dispatches on the tool name and applies validation. The acceptance criteria:

1. Both writes require an `idempotency_key` argument with `min_length=8`.
2. The `tier` argument on `provision_workspace` is constrained to a `Literal["starter", "growth", "enterprise"]`.
3. `execute_tool` calls `Model.model_validate(input_dict)` before invoking the underlying function so invalid payloads raise `ValidationError` instead of corrupting state.
4. The descriptions tell the model explicitly that writes are idempotent and how to construct the key.

**What good looks like.** A solution defines three `BaseModel` classes with appropriately constrained fields, builds the registry with `model_json_schema()`, and the `execute_tool` dispatcher fails closed on validation errors. Bonus: the `email` field uses Pydantic's `EmailStr` type so malformed emails are rejected without writing a single line of regex.

## 8. Key Takeaways

- **What is the three-part shape of a Claude tool-use exchange on Bedrock?** (`tools` array in the request → `tool_use` content blocks in the response → `tool_result` content blocks threaded back with matching `tool_use_id` on the next user turn.)
- **Where does the JSON Schema for a tool come from, and why does that matter?** (From `Pydantic.model_json_schema()` — keeps the schema and the validator in lockstep.)
- **Why does the write tool's description matter as much as its schema?** (The model reads descriptions and follows them — telling the model "writes are idempotent; construct the key from (X, Y)" usually works.)
- **Where does the multi-tenant filter live?** (In the host runtime, not in the prompt or the model-provided arguments.)
- **What three stale shapes should you flag and replace if you see them in a tutorial?** (OpenAI `functions`/`function_call`; `anthropic.claude-v2` model IDs; LangChain `Chain`-class tool wrappers.)

## Sources

1. [Tool use — Amazon Bedrock Anthropic Claude Messages API](https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters-anthropic-claude-messages-tool-use.html) — retrieved 2026-05-26
2. [Use a tool to complete an Amazon Bedrock model response](https://docs.aws.amazon.com/bedrock/latest/userguide/tool-use.html) — retrieved 2026-05-26
3. [Note-saving tool with Pydantic and Anthropic tool use — Claude Cookbook](https://platform.claude.com/cookbook/tool-use-tool-use-with-pydantic) — retrieved 2026-05-26
4. [Structured outputs — Claude API Docs](https://docs.claude.com/en/docs/build-with-claude/structured-outputs) — retrieved 2026-05-26
5. [JSON Schema — Pydantic Docs](https://pydantic.dev/docs/validation/latest/concepts/json_schema/) — retrieved 2026-05-26

Last verified: 2026-05-26
