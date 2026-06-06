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
last_verified: 2026-06-06
---

# Tool-calling discipline — schemas and the Bedrock tool-use API

> [!NOTE]
> **From earlier:** The ReAct loop (topic 2) terminates when the model emits no tool calls. Every cycle before that is driven by the tool surface you define here.

## 1. Learning Objectives

- Describe the three-part request/response/thread-back shape of Claude's tool-use API on Bedrock InvokeModel
- Author a tool definition with a Pydantic-generated `input_schema` and name three classes of validation error caught at the boundary
- Distinguish a `tool_use` content block from a `tool_result` content block and explain how `tool_use_id` pairs them
- List four production-grade hygiene rules for the tool surface (read/write separation, schema strictness, error handling, multi-tenant filters)
- Recognise stale Bedrock model identifiers and stale tool-use shapes and substitute current equivalents

## 2. Introduction

Tool calling is the load-bearing primitive under every agent framework. The model does not actually invoke your function — it emits a structured **request** for the function, and your host runtime decides whether and how to run it. That separation is what makes tool calls auditable, idempotent, and human-overridable.

On AWS Bedrock with Claude (Sonnet 4.5, Haiku 4.5, Opus 4.1 in 2026), tool calling is part of the Messages API surface — both the lower-level `InvokeModel` route and the higher-level `Converse` route support it. The shape is consistent: you send tool definitions in the request, the model returns `tool_use` content blocks, you execute the named tools, and you thread results back as `tool_result` content blocks on the next user turn.

Tool-calling bugs are sneaky. The model produces something that *looks* like a valid tool call but is missing a required field, or has the wrong type, or was built from a stale tutorial using the deprecated OpenAI `functions` parameter or the long-removed `anthropic.claude-v2` model ID. Pydantic-validated schemas at the boundary catch these before they reach your database.

## 3. Core Concepts

### 3.1 The three-part shape — request, response, thread-back

**Request (you → Bedrock).** A `Messages` payload with a `tools` array. Each tool is `{name, description, input_schema}`, where `input_schema` is a JSON Schema object.

**Response (Bedrock → you).** A `content` array containing one or more `tool_use` blocks alongside any reasoning text, with `stop_reason: "tool_use"`.

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

**Thread-back (you → Bedrock, next turn).** The prior assistant turn preserved **verbatim**, then a `user` turn with a `tool_result` content block whose `tool_use_id` matches the `id` from the response. If you reorder, drop, or rename the id, the model loses the thread.

> [!IMPORTANT]
> **`tool_use_id` is the linking primitive.** Every `tool_result` must carry a `tool_use_id` that exactly matches its corresponding `tool_use` block. Drop or rename it and the API rejects the conversation.

### 3.2 `input_schema` from Pydantic

JSON Schema is verbose. Pydantic v2 makes it ergonomic — define a `BaseModel`, call `.model_json_schema()`, hand the result to Bedrock:

```python
from pydantic import BaseModel, Field
from uuid import UUID

class GetSolicitationInput(BaseModel):
    solicitation_id: UUID

class RouteToEvaluatorsInput(BaseModel):
    proposal_id: UUID
    evaluator_ids: list[UUID]
    idempotency_key: str = Field(min_length=8, max_length=128)

tool_defs = [
    {
        "name": "get_solicitation",
        "description": "Read solicitation record. Safe to call repeatedly.",
        "input_schema": GetSolicitationInput.model_json_schema(),
    },
    {
        "name": "route_to_evaluators",
        "description": (
            "Route proposal to evaluators. State-mutating. "
            "Provide idempotency_key derived from (proposal_id, evaluator_set_hash)."
        ),
        "input_schema": RouteToEvaluatorsInput.model_json_schema(),
    },
]
```

Pydantic v2 is Rust-backed and validates 10–50× faster than v1 — boundary validation cost is negligible on hot paths.

### 3.3 Reads vs writes — separating the surface

The hardest discipline is keeping reads cheap and writes deliberate:

- **Read tools** are pure functions of state. Name them after the entity: `get_*`, `list_*`, `search_*`.
- **Write tools** mutate state and are named for the **business effect**: `route_to_evaluators` rather than `insert_routing_row`.
- **Write tools always carry an `idempotency_key` argument** (see topic 4) and document the contract in the description so the model includes one.

### 3.4 Multi-tenant filters at the boundary

In a multi-tenant system — a federal-acquisitions platform with multiple agencies — **the tenant filter does not belong in the prompt**. It belongs in the host runtime that executes the tool. The model receives a tool definition that accepts an `agency_id`; the host runtime overrides whatever the model passed with the authenticated request context. Treat any model-provided tenant identifier as untrusted input.

## 4. Generic Implementation

A read-and-write tool pair for a logistics dispatch system, illustrating read/write naming discipline and the multi-tenant filter pattern:

```python
from pydantic import BaseModel, Field
from uuid import UUID

class GetVehicleStatusInput(BaseModel):
    vehicle_id: UUID
    include_recent_events: bool = Field(default=False)

def get_vehicle_status(inp: GetVehicleStatusInput, *, ctx: RequestContext):
    # Tenant override: ignore any fleet_id the model passed; use authenticated context
    return vehicles.find_one(
        {"vehicle_id": inp.vehicle_id, "fleet_id": ctx.fleet_id}
    )

class DispatchVehicleInput(BaseModel):
    vehicle_id: UUID
    job_id: UUID
    eta_minutes: int = Field(ge=0, le=720)
    idempotency_key: str = Field(min_length=8, max_length=128)

def dispatch_vehicle(inp: DispatchVehicleInput, *, ctx: RequestContext):
    existing = dispatches.find_one({"idempotency_key": inp.idempotency_key})
    if existing:
        return existing
    return dispatches.insert_one({
        "vehicle_id": inp.vehicle_id,
        "job_id": inp.job_id,
        "fleet_id": ctx.fleet_id,  # from context, NOT from the model
        "eta_minutes": inp.eta_minutes,
        "idempotency_key": inp.idempotency_key,
    })
```

`fleet_id` is taken from `ctx`, not from `input` — the model cannot accidentally dispatch a vehicle from another fleet.

## 5. Real-world Patterns

**Healthcare (Epic "Art," Hippocratic AI).** Tool surface is reads only — labs, imaging, prior notes — with writes through a human-approval queue. ICD-10 and RxNorm IDs are type-constrained; Pydantic catches model-generated codes that don't exist in canonical vocabularies.

**Fintech (Plaid + Stripe).** Write tools include a `reason` field the model fills; that reason is logged with the action for post-hoc audit. Capture reasoning in the input schema, not only in a system log.

**Gaming (Riot player-support).** `region` is taken from the authenticated session, not the prompt. Players have tried prompt-injection into other accounts; the boundary-layer override makes those attempts no-ops.

> [!TIP]
> **Cap tool count.** Beyond ~25 tools, model selection accuracy drops. Group tools behind a router or split into specialised sub-agents.

> [!NOTE]
> **Cross-domain pattern.** Healthcare, fintech, and gaming all enforce the same rule: boundary-layer identifiers (`tenant_id`, `region`, `agency_id`) are server-set, never prompt-sourced.

## 6. Best Practices

- Generate `input_schema` from a typed model (Pydantic in Python, Zod in TypeScript) — hand-written JSON Schema drifts from the executor's expectations within weeks
- Validate at the boundary regardless of `strict` mode — strict mode constrains model output; it does not protect against payloads from other sources
- Name write tools after their business effect — `escalate_to_co` is meaningful; `insert_escalation_row` leaks implementation detail
- Document the idempotency contract in the tool description — the model reads descriptions and tends to comply
- Override tenant identifiers in the host, not in the prompt
- Return structured errors, never silent nulls

> [!WARNING]
> **Anti-patterns: `over-tooling-exhausts-context` + `tool-schema-as-docstring`.** Exposing every API as a tool destroys reasoning — each definition burns prompt tokens; beyond ~25 tools the model's selection accuracy degrades. The intake-triage agent has 5 tools (3 read, 2 write) because that is what the scenario requires. Separately: LLMs read the JSON `input_schema`, not Python or TypeScript docstrings. The schema IS the contract; the `description` field is the selection prompt. Start with minimum viable tool surface; generate schemas from typed models.

## 7. Hands-on Exercise

You are exposing tools to a Claude agent that runs an interactive customer-onboarding workflow. The agent needs: `get_signup_status(email)` (read), `send_welcome_email(email, template_id)` (write, idempotent), `provision_workspace(email, tier)` (write, idempotent). Write the three Pydantic models, three tool registry entries with descriptions, and a stub `execute_tool(name, input)` that calls `Model.model_validate(input_dict)` before invoking the underlying function.

> [!NOTE]
> **Self-check** (30 s — answer mentally before expanding)
>
> 1. Why does the write tool's description matter as much as its schema?
> 2. Where does the multi-tenant filter live — prompt, model input, or host runtime?

<details>
<summary>Show answers</summary>

1. The model reads descriptions and follows them. Telling the model "writes are idempotent; construct the key from (X, Y)" causes it to include the key correctly. The schema enforces the structure; the description governs the model's behaviour in constructing the inputs.
2. In the host runtime. The host overrides any tenant identifier the model provides with the authenticated request context. A model-provided tenant identifier is untrusted input.

</details>

## 8. Key Takeaways

- **Three-part shape:** `tools` array in request → `tool_use` blocks in response → `tool_result` blocks threaded back with matching `tool_use_id`
- **Pydantic `.model_json_schema()`** keeps schema and validator in lockstep — use it, don't hand-write JSON Schema
- **Write tools' descriptions matter** — the model reads them and constructs inputs accordingly
- **Tenant filters live in the host runtime**, not in the prompt or model-provided arguments
- **Stale shapes to replace:** OpenAI `functions`/`function_call`; `anthropic.claude-v2` model IDs; LangChain `Chain`-class tool wrappers

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters-anthropic-claude-messages-tool-use.html — Tool use, Amazon Bedrock Anthropic Claude Messages API — retrieved 2026-05-26 — hot-tech
- https://docs.aws.amazon.com/bedrock/latest/userguide/tool-use.html — Use a tool to complete an Amazon Bedrock model response — retrieved 2026-05-26 — hot-tech
- https://platform.claude.com/cookbook/tool-use-tool-use-with-pydantic — Note-saving tool with Pydantic and Anthropic tool use (Claude Cookbook) — retrieved 2026-05-26 — hot-tech
- https://docs.claude.com/en/docs/build-with-claude/structured-outputs — Structured outputs (Claude API Docs) — retrieved 2026-05-26 — hot-tech
- https://pydantic.dev/docs/validation/latest/concepts/json_schema/ — JSON Schema (Pydantic Docs) — retrieved 2026-05-26 — foundation-stable

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

Anthropic's strict-tool-use mode (beta as of late 2025, `"strict": True` in the tool definition + beta header) guarantees the model's emitted `input` validates against the schema — no missing required fields, no type mismatches, no extra properties. Without strict mode you still need Pydantic validation at the boundary (defence in depth); with strict mode the boundary validation becomes a secondary check.

Bedrock model IDs and regional inference-profile mappings change roughly quarterly. The current Claude on Bedrock model IDs (Sonnet 4.5, Haiku 4.5, Opus 4.1) use regional inference profile ARNs in the format `arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-4-5-20251101-v1:0`. Re-verify exact IDs in the Sunday-before `/web-research` sweep before each cohort week — any source using `anthropic.claude-v2` or `anthropic.claude-instant` is outdated and should be flagged.

The `known-bad-patterns.yml` entries `bedrock-old-model-ids` and `openai-functions-deprecated` are the canonical references for these stale shapes. Both entries have `last_reviewed: 2026-05-11` — they were live stale patterns on the open internet at that date.

</details>

Last verified: 2026-06-06
