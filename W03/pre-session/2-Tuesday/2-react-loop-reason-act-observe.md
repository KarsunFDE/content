---
week: W03
day: Tue
topic_slug: react-loop-reason-act-observe
topic_title: "ReAct loop = Reason + Act + Observe"
parent_overview: W03/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://arxiv.org/abs/2210.03629
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.anthropic.com/research/building-effective-agents
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.langchain.com/oss/python/langchain/agents
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://blog.langchain.com/langchain-langgraph-1dot0/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.ibm.com/think/topics/react-agent
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-06-06
---

# ReAct loop = Reason + Act + Observe

> [!NOTE]
> **From earlier:** Mon's ADR committed the agent topology and named the HITL #3 boundary. Today you implement the loop that topology describes.

## 1. Learning Objectives

- Describe the three phases of a ReAct loop (Reason, Act, Observe) and the termination condition for a tool-using LLM agent
- Articulate the distinction between a **workflow** (predefined code paths) and an **agent** (LLM-directed control flow) using Anthropic's framing
- Map a concrete business problem onto a ReAct loop and identify which step belongs in which phase
- Explain why interleaving reasoning traces with actions reduces hallucination compared to plain chain-of-thought

## 2. Introduction

ReAct — short for **Reason + Act** — is the loop behind most production tool-using LLM agents. Yao et al. (arXiv 2210.03629) showed that an LLM which **interleaves** a written reasoning trace with structured tool calls outperforms pure chain-of-thought on knowledge-intensive tasks. A chain-of-thought model can invent a fact and build on it; ReAct forces each step to be grounded in an actual observation, so the loop self-corrects.

In 2026 the pattern ships in LangGraph's `create_react_agent`, LangChain v1.0's `create_agent`, Bedrock Agents, and OpenAI Assistants — the simplest recoverable, debuggable tool-using loop. Anthropic's "Building effective agents" guide recommends it first, before adding orchestration.

For `acquire-gov` intake-triage: receive the proposal payload → reason about what to check → call a read tool → observe the result → loop until triage is complete or an anomaly triggers escalation.

## 3. Core Concepts

### 3.1 The three phases

A ReAct loop has three phases per turn:

- **Reason.** The model emits a short natural-language thought about what to do next. In Claude on Bedrock this lands in the assistant message's `text` content block.
- **Act.** The model emits a `tool_use` content block carrying `{name, input}`. The host runtime executes the named tool.
- **Observe.** The host returns the tool's result as a `tool_result` content block on a `user`-role message. The model sees what happened and can reason again.

The loop terminates when the model decides it has enough information. For Claude on Bedrock the termination signal is `stop_reason: "end_turn"` with no further `tool_use` content blocks.

> [!TIP]
> **Workflow vs agent — the line that matters.** Anthropic draws a sharp distinction: a **workflow** orchestrates LLMs through predefined code paths the developer enumerated. An **agent** lets the LLM dynamically direct its own control flow. Intake-triage is on the agent side — the CO did not pre-specify "always check amendments first"; the model decides based on the payload.

### 3.2 Three budgets every production loop must set

A ReAct loop without a step cap can loop forever. Set all three budgets explicitly:

| Budget | What it bounds | Typical default |
|--------|----------------|-----------------|
| `max_iterations` / `recursion_limit` | Reason-Act-Observe cycles | 10–25 |
| Token budget | Total prompt + completion tokens across the run | 100K–200K |
| Wall-clock timeout | Time before the runtime kills the run | 60–300 seconds |

A loop that has not terminated by any of these is a prompt or tool-surface bug — not a sign you should raise the cap.

### 3.3 Reason traces reduce hallucination

The Yao et al. headline result: interleaving reasoning and acting beats both pure CoT and pure Act-only on HotpotQA and FEVER fact-verification. In pure CoT the model invents facts and builds on them unchecked. In ReAct every step is grounded in an actual tool observation — the observation can contradict the invented fact before the next decision is made.

> [!IMPORTANT]
> **Structured tool errors, not silent nulls.** Return `{"error": "not_found", "id": "..."}` when a tool call fails — the model can reason about the failure and try a different path. Silent `None` returns let the model hallucinate around the gap.

## 4. Generic Implementation

A minimal ReAct loop in plain Python, illustrating termination, structured error handling, and the observe phase:

```python
def run_react_loop(initial_message: str, tools: list, max_iterations: int = 10) -> str:
    """Generic ReAct loop. Terminates when no tool calls or max_iterations hit."""
    messages = [{"role": "user", "content": initial_message}]

    for step in range(max_iterations):
        # Reason + Act: model produces text thought + zero-or-more tool_use blocks
        response = llm_call(messages=messages, tools=tools)

        # Termination: no tool calls means the model is done
        if not response.tool_calls:
            return response.content

        # Observe: echo assistant turn, thread each tool result back
        messages.append({"role": "assistant", "content": response.content})

        tool_results = []
        for call in response.tool_calls:
            try:
                result = execute_tool(call.name, call.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": call.id,
                    "content": result,
                })
            except Exception as e:
                # Structured error — model reasons about the failure, not around it
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": call.id,
                    "content": f"ERROR: {type(e).__name__}: {e}",
                    "is_error": True,
                })

        messages.append({"role": "user", "content": tool_results})

    # Hit the cap — return structured incomplete, not an exception
    return {"status": "incomplete", "reason": "max_iterations_reached"}
```

Termination check fires before any tool-result threading. Errors are structured and observable. The cap exits with structured incompleteness — callers can inspect `messages` to see what the agent tried.

## 5. Real-world Patterns

**Fintech fraud investigation (Stripe Radar Assistants).** The agent reasons about a flag, acts by calling `get_card_history`, `get_ip_reputation`, `get_device_fingerprint`, and observes before deciding escalate or auto-clear. Writes (`block_card`, `notify_customer`) route through human-approval middleware — the ReAct shape traverses a multi-signal graph that a static rules engine cannot express.

**E-commerce root-cause (Shopify Sidekick).** Sidekick answers "why did my conversion drop?" by reasoning a hypothesis, acting on analytics tools, and observing until it attributes the drop to a concrete factor. The loop terminates when the agent produces a natural-language explanation citing specific tool observations.

**Healthcare clinical-decision support (Glass Health, Hippocratic AI).** The reason step articulates a differential hypothesis; the act step queries labs and imaging; the observe step grounds the next hypothesis in patient data. Writes are forbidden — every recommendation routes through a human approval queue.

> [!NOTE]
> **Cross-domain signal.** All three patterns share the acquire-gov constraint: writes are small in number, named for their effect, and deliberately slow-pathed through a gate. Reads are plentiful and cheap.

## 6. Best Practices

- Cap iterations, tokens, and wall-clock timeout explicitly — never trust the model to self-limit
- Make tool errors observable — return structured error strings, not nulls or silent exceptions
- Separate reads from writes — writes named for business effect, idempotent, slow-pathed through a gate
- Trace every step — without per-step trace you cannot debug tool selection decisions
- Prefer the framework's prebuilt loop until you have a measured reason to hand-roll it

> [!WARNING]
> **Anti-pattern: `react-loop-no-termination`.** A ReAct loop without an explicit `max_iterations` cap and token budget will run until the model's context overflows or the runtime OOMs. This is the single most common production bug in agentic systems. Every loop needs all three budgets set before it leaves `main` — iteration cap, token budget, wall-clock timeout.

## 7. Hands-on Exercise

You are designing a ReAct agent for an open-source security-review tool. Tools: `get_package_metadata(name, version)` (read), `get_known_vulnerabilities(name, version)` (read), `get_dependency_tree(name, version)` (read), `scan_dependency_tree_for_cves(tree)` (read), `submit_review(package_id, verdict, rationale)` (write). Draw the loop for input *"is `requests==2.31.0` safe?"* — first Reason, first Act, Observation, continue to termination. State your `max_iterations`, `tokens_max`, and `wall_clock_timeout` values.

> [!NOTE]
> **Self-check** (30 s — answer mentally before expanding)
>
> 1. What is the termination signal for Claude on Bedrock's tool-use loop?
> 2. Name one consequence of catching a tool exception silently and returning `None`.

<details>
<summary>Show answers</summary>

1. `stop_reason: "end_turn"` with no further `tool_use` content blocks in the assistant response.
2. The model has no observation to reason from, so it may hallucinate a plausible result — inventing that the tool succeeded when it did not. The structured-error pattern gives the model a real signal to work with.

</details>

## 8. Key Takeaways

- **Three phases:** Reason → Act → Observe; loop terminates on `end_turn` with no tool calls
- **Workflow vs agent:** workflows follow predefined paths; agents let the LLM direct its own control flow
- **Reason traces ground each step** in real tool output, breaking the hallucinate-and-build-on-it pattern
- **All three budgets** must be set before the loop leaves `main` — iterations, tokens, wall-clock
- **Structured errors, not nulls** — the model can only reason about a failure it can see

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://arxiv.org/abs/2210.03629 — ReAct: Synergizing Reasoning and Acting in Language Models (Yao et al.) — retrieved 2026-05-26 — foundation-stable
- https://www.anthropic.com/research/building-effective-agents — Building Effective Agents (Anthropic) — retrieved 2026-05-26 — hot-tech
- https://docs.langchain.com/oss/python/langchain/agents — LangChain v1.0 Agents documentation — retrieved 2026-05-26 — hot-tech
- https://blog.langchain.com/langchain-langgraph-1dot0/ — LangChain and LangGraph 1.0 (LangChain blog, Oct 2025) — retrieved 2026-05-26 — hot-tech
- https://www.ibm.com/think/topics/react-agent — What is a ReAct Agent? (IBM Think) — retrieved 2026-05-26 — foundation-stable

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

The Yao et al. paper's ablation tables are worth reading: Act-only (no reasoning trace) outperforms CoT-only on interactive decision-making (ALFWorld, WebShop) but loses on knowledge-intensive QA (HotpotQA, FEVER). ReAct is the only shape that wins on both benchmark families. The mechanism: interactive tasks require adapting to observations; knowledge tasks require checking invented facts. ReAct provides both affordances simultaneously.

For the LangGraph-specific implementation: `create_react_agent` in LangGraph v1.0 wraps `StateGraph` + `ToolNode` + `tools_condition`. The condition routes: if the model returned `tool_use` blocks → `ToolNode`; if it returned `end_turn` → the response node. Senior FDEs should read the LangGraph source for `tools_condition` to understand what "termination" looks like at the graph-conditional level — it is a Python `Literal["tools", "__end__"]` type annotation, not a sentinel value.

LangGraph's `recursion_limit` (default 25 supersteps) is the `max_iterations` equivalent. Raising it above 50 is a yellow flag; above 100 is a red flag. If you find yourself needing more than 25 iterations, the likely root cause is a tool that returns insufficient information, forcing re-retries, or a prompt that does not give the model a clear termination criterion.

</details>

Last verified: 2026-06-06
