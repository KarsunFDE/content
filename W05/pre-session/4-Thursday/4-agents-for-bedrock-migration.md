---
week: W05
day: Thu
topic_slug: agents-for-bedrock-migration
topic_title: "AWS Managed-Service Migration 2 — Agents-for-Bedrock on a custom multi-agent flow"
parent_overview: W05/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 13
sources:
  - url: https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://aws.amazon.com/bedrock/agents/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://docs.aws.amazon.com/bedrock/latest/userguide/agents-action-create.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://aws.amazon.com/blogs/machine-learning/agents-for-amazon-bedrock-introducing-a-simplified-creation-and-configuration-experience/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Migrating a Custom Multi-Agent Flow to Amazon Bedrock Agents

## 1. Learning Objectives

By the end of this reading, the learner can:

- Describe what an Agents-for-Bedrock action group is and how it differs from an arbitrary tool-call in a custom agent framework.
- Compare a custom LangGraph (or equivalent state-machine framework) implementation against an Agents-for-Bedrock implementation on at least six dimensions.
- Identify where Agents-for-Bedrock's built-in user-confirmation surface aligns and conflicts with a team's existing human-in-the-loop policy.
- Write a per-endpoint ADR for an agent flow that names the trade-off the chosen direction accepts.
- Recognise the vendor-lock-in shape of Agents-for-Bedrock and the conditions under which it is acceptable.

## 2. Introduction

A multi-agent flow — an LLM that calls tools, hands off to other agents, asks the user for confirmation, and persists state across turns — is one of the harder things to build from scratch and one of the easier things to host badly. The state machine has to be observable, recoverable, and auditable. The tool-call surface has to authenticate and authorise. The human-in-the-loop interrupt has to be discoverable to the front-end. Teams that build the substrate themselves end up owning a small framework.

Amazon Bedrock Agents is AWS's managed answer to that substrate. The team writes the *tools* (as action groups defined by OpenAPI schemas or Lambda functions) and the *instructions* (the agent prompt); Bedrock owns the orchestration loop, the session state, the tool invocation, and the user-confirmation prompt [Bedrock Agents overview, retrieved 2026-05-26].

The migration decision parallels the Knowledge Bases decision but with a higher stakes profile: the agent loop is where the business logic lives, and migration changes how that logic is observed and debugged. A team that has tuned a LangGraph state machine for two months is not abandoning that work casually; a team that is six weeks into a "we'll build our own agent framework" is often relieved to hand the substrate over.

This reading covers the dimensions an honest agent-migration evaluation walks. Karsun-specific framing of which endpoint, which HITL touchpoint, and which feature-inventory items are affected lives in the overview. This reading stays generic.

## 3. Core Concepts

### 3.1 What Bedrock Agents owns

A Bedrock Agent is an orchestrator that runs the following loop on the team's behalf [Bedrock Agents documentation, retrieved 2026-05-26]:

1. Receive a user message.
2. Decide whether to call a tool (an action group) or invoke a Knowledge Base or answer directly.
3. If calling a tool, choose the operation, fill its parameters, and (optionally) ask the user to confirm before executing.
4. Invoke the tool — either an OpenAPI-described HTTP endpoint or a Lambda function.
5. Feed the tool result back into the model.
6. Repeat until the model produces a final answer.

The orchestration code, the session state (per `sessionId`), and the user-confirmation prompt UX are all Bedrock's responsibility. The team owns the tool implementations and the agent's natural-language instructions.

### 3.2 Action groups — the tool surface

An action group is a named collection of operations the agent can call [Action groups documentation, retrieved 2026-05-26]. Each operation is described one of two ways:

- **OpenAPI schema** — the team supplies an OpenAPI 3 specification of the HTTP endpoints; Bedrock parses it and the agent calls those endpoints. Auth is via IAM, custom auth, or Bedrock-managed credentials.
- **Lambda function** — the team supplies a Lambda that handles the action; Bedrock calls the Lambda with a structured event.

The OpenAPI-schema path is the cleaner migration target if the existing custom agent calls REST APIs on internal services: turn the service into an OpenAPI-described action group, and Bedrock can hit it directly.

### 3.3 User-confirmation as a first-class surface

Bedrock Agents supports a built-in **user-confirmation** mode for any action: when configured, before the agent executes the action it returns a confirmation request to the client. The client renders the confirmation UI, captures the user's yes/no, and resumes the session [Bedrock Agents overview, retrieved 2026-05-26].

This is operationally the same idea as a LangGraph `interrupt()` node, but the surface is different:

- In LangGraph, the team owns the interrupt-resume protocol — what payload to send to the front-end, how the front-end resumes, where the paused state lives.
- In Bedrock Agents, the confirmation protocol is fixed — the API returns a specific event shape, the client must respond on the same `sessionId`, and the resumption is handled by Bedrock.

For teams whose HITL design matches Bedrock's confirmation shape, this is a substantial savings. For teams whose HITL design requires a different shape — multi-party approval, asynchronous approval (a Slack message that someone approves an hour later), conditional approval (approve only if drift threshold is below X) — Bedrock's surface is too narrow.

### 3.4 Six dimensions a migration evaluation pivots on

1. **HITL interrupt control** — full custom in a state-machine framework versus a fixed user-confirmation API in Bedrock. Decisive when HITL is multi-party or asynchronous.
2. **Tool authentication** — cohort-owned token propagation (JWT, mTLS) in custom versus IAM-grounded or Bedrock-managed credentials in Agents. The federal/regulated case usually favours IAM grounding.
3. **Debug surface** — state-machine inspection (LangGraph traces, custom logs) versus CloudWatch + Bedrock traces. Custom is higher-resolution; managed is lower-effort.
4. **Vendor lock-in** — LangGraph/LangChain is portable (other cloud, other models); Agents-for-Bedrock is AWS-only and Bedrock-model-only. Decisive for multi-cloud or model-portability requirements.
5. **Cost shape** — per-LLM-call only in custom versus per-LLM-call plus an agent-orchestration markup in Agents. The markup is small but real.
6. **Re-target-ability to non-AWS** — re-writing a LangGraph flow to a different framework is bounded work; rewriting a Bedrock Agent to a non-AWS substrate is rewriting the orchestration loop entirely.

### 3.5 The "fits a vendor's mental model" question

Every managed agent framework — Bedrock Agents, OpenAI Assistants API, Vertex AI Agents — has a mental model: action groups versus tools versus functions; sessions versus threads versus contexts; user-confirmation versus interrupt versus approval. The migration question is *not* "is the managed framework better?" — it is "does our flow shape fit the framework's mental model?"

A flow that fits the model loses very little to managed orchestration and gains a lot. A flow that doesn't fit the model bends itself into shapes that are harder to reason about than the original custom code — which is the worst outcome, because the team owns *more* complexity, not less, after migration.

## 4. Generic Implementation

A minimum-viable agent invocation against a Bedrock Agent with one action group, using the AWS SDK:

```python
import boto3

bedrock_agent_runtime = boto3.client("bedrock-agent-runtime", region_name="us-east-1")

def invoke_agent(
    agent_id: str,
    agent_alias_id: str,
    session_id: str,
    user_message: str,
) -> dict:
    """Invoke a Bedrock Agent and stream the response, including any confirmation prompts."""

    response = bedrock_agent_runtime.invoke_agent(
        agentId=agent_id,
        agentAliasId=agent_alias_id,
        sessionId=session_id,
        inputText=user_message,
        # enableTrace surfaces the orchestration steps for debugging.
        enableTrace=True,
    )

    output_chunks = []
    confirmation_needed = None
    trace_events = []

    # The response is a streamed event stream. Each event is either a text chunk,
    # a return-control event (user confirmation), or a trace event.
    for event in response["completion"]:
        if "chunk" in event:
            output_chunks.append(event["chunk"]["bytes"].decode("utf-8"))
        elif "returnControl" in event:
            # The agent is asking the client to confirm an action before executing it.
            confirmation_needed = event["returnControl"]
        elif "trace" in event:
            trace_events.append(event["trace"])

    return {
        "answer": "".join(output_chunks),
        "confirmation_needed": confirmation_needed,
        "trace": trace_events,
    }
```

> [!instructor-review]
> The agent's underlying foundation model is configured when the agent is created, not at invocation time. Verify the model ID against the current Bedrock catalog before agent creation; the known-bad-patterns blocklist explicitly flags `anthropic.claude-v2` and `anthropic.claude-instant` as stale identifiers, and the Claude family rotates faster than tutorial pages can be updated.

The snippet illustrates four things and deliberately omits a fifth:

- The `sessionId` is the load-bearing handle for multi-turn conversation. Bedrock owns session state behind it; the client only needs to keep the ID stable across turns.
- `returnControl` is the user-confirmation surface. The client renders a confirmation UI, then on the next call sends the user's decision back via the `sessionState.invocationResults` field (omitted here for brevity).
- `enableTrace=True` is the debug surface. The trace events show the agent's reasoning, tool invocations, and Knowledge Base retrievals.
- The action group itself is configured at agent-creation time, not at invocation time. The OpenAPI schema or Lambda is registered once; every invocation has access to it.
- The snippet does *not* show tool-call authentication. Action group operations called over HTTP carry whatever auth the OpenAPI security scheme defines; the team is still responsible for ensuring the tool's auth aligns with the agent's user context.

## 5. Real-world Patterns

**Healthcare — appointment-scheduling assistant.** A regional health system built a custom LangGraph flow for an appointment-scheduling assistant: identify intent, look up patient records, propose times, ask for confirmation, book. They evaluated migrating to Bedrock Agents because the confirmation surface mapped exactly to their existing UX. The migration shipped; the team retired ~600 lines of custom orchestration code. They explicitly accepted vendor lock-in because the workload was AWS-resident already [Bedrock Agents documentation, retrieved 2026-05-26].

**Fintech — a customer-service refund assistant.** A consumer-finance company evaluated migrating a custom agent that processed refund requests. The HITL surface was multi-party: small refunds auto-approved, mid-tier refunds approved by an agent supervisor, large refunds approved by a fraud-team queue with a 24-hour SLA. Bedrock's user-confirmation surface assumed a synchronous yes/no from the calling user — it did not fit. The team kept the custom flow and documented the trade-off: "we accept ownership of orchestration substrate in exchange for HITL flexibility that the managed offering does not provide" [Agents creation blog, retrieved 2026-05-26].

**E-commerce — a customer-support triage agent.** A retailer migrated a customer-support triage agent to Bedrock Agents because their tool surface was clean: every operation was already an internal REST endpoint with an OpenAPI spec. Action-group creation took an afternoon. The lift on the LangGraph side was about a week of removing scaffolding. The decision was made easier by the team's commitment to AWS-only deployment and acceptance of the vendor-lock-in cost.

**Logistics — a shipment-rescheduling agent.** A parcel-delivery network kept a custom flow because their re-target-ability requirement was non-negotiable: the company's risk team had a written policy that no business-critical workload could be locked to a single cloud provider. Bedrock Agents was rejected on dimension 4 (vendor lock-in) before any other dimension was evaluated; the no-migration ADR cited the policy and named the lock-in cost they refused to accept.

## 6. Best Practices

- Choose action groups over Lambda when the operations are already REST endpoints with OpenAPI specs; choose Lambda when the operation has non-HTTP semantics (queue write, custom protocol).
- Pin the agent's foundation-model ID after verifying against the current Bedrock catalog; never paste an ID from a tutorial.
- Test the user-confirmation surface against your real HITL requirements before committing to migration; the surface is opinionated and the mismatch shape is sometimes only obvious in QA.
- Keep an integration test that asserts cross-tenant tool calls fail closed; tool auth is the team's responsibility even when orchestration is managed.
- Write down the vendor-lock-in cost as part of the ADR; "lock-in" is a position, not an absence of one.
- Use `enableTrace=True` aggressively during development; the trace event stream is the only debug surface and you need it familiar.
- For multi-party or asynchronous HITL, keep the custom orchestrator. Bedrock Agents' confirmation surface is synchronous and same-actor.

## 7. Hands-on Exercise

Pick a multi-step LLM-driven workflow you know (a chatbot you've built, a public agent demo, or a hypothetical one — e.g., "weather + flight-booking assistant"). In 15 minutes:

1. List the operations the agent can call. Mark each as an OpenAPI-action-group fit or a Lambda fit.
2. Identify the HITL touchpoints. For each, decide whether Bedrock's user-confirmation surface fits or whether you need a custom interrupt.
3. Write the trade-off paragraph: "We choose [custom / Bedrock Agents] because [dimension X dominates]; we accept [the cost on dimension Y]."

**What good looks like.** Every operation has a concrete fit-classification (action group via OpenAPI, action group via Lambda, or "does not fit"). The HITL analysis identifies a touchpoint that fits Bedrock's surface and at least one that may not. The trade-off paragraph names a real cost — not "we'll figure it out" — and aligns with whoever owns the orchestration after migration.

## 8. Key Takeaways

- What does Bedrock Agents own (orchestration loop, session state, confirmation surface) versus what stays with the team (tool implementations, instructions)?
- How do action groups (OpenAPI or Lambda) differ from arbitrary tool-calls in a custom agent framework?
- When does Bedrock's user-confirmation surface fit your HITL design, and when does it force the flow to bend to fit?
- Which dimensions decide a custom-versus-managed agent evaluation, and which one most often dominates?
- How does the vendor-lock-in cost of Agents-for-Bedrock get named honestly in the ADR rather than smuggled in as a tolerance?

## Sources

1. [Amazon Bedrock Agents — User Guide](https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html) — retrieved 2026-05-26
2. [Amazon Bedrock Agents — product page](https://aws.amazon.com/bedrock/agents/) — retrieved 2026-05-26
3. [Define actions for an agent using action groups](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-action-create.html) — retrieved 2026-05-26
4. [Agents for Amazon Bedrock — simplified creation and configuration](https://aws.amazon.com/blogs/machine-learning/agents-for-amazon-bedrock-introducing-a-simplified-creation-and-configuration-experience/) — retrieved 2026-05-26

Last verified: 2026-05-26
