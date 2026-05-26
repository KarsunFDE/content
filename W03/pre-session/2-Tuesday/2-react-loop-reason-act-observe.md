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
last_verified: 2026-05-26
---

# ReAct loop = Reason + Act + Observe

## 1. Learning Objectives

- By the end of this reading, the learner can describe the three phases of a ReAct loop (Reason, Act, Observe) and the termination condition for a tool-using LLM agent.
- By the end of this reading, the learner can articulate the distinction between a **workflow** (predefined code paths) and an **agent** (LLM-directed control flow) using Anthropic's framing.
- By the end of this reading, the learner can map a concrete business problem onto a ReAct loop and identify which step belongs in which phase.
- By the end of this reading, the learner can explain why interleaving reasoning traces with actions reduces hallucination compared to plain chain-of-thought.
- By the end of this reading, the learner can identify three common failure modes in a ReAct loop and the defensive design choices that mitigate them.

## 2. Introduction

ReAct — short for **Reason + Act** — is the loop behind most production tool-using LLM agents shipped between 2023 and 2026. It was introduced by Yao et al. in a 2022 paper ("ReAct: Synergizing Reasoning and Acting in Language Models," arXiv 2210.03629) that showed an LLM which **interleaves** a written reasoning trace with structured tool calls outperforms a model doing pure chain-of-thought on knowledge-intensive QA, and outperforms imitation-learned agents on interactive decision-making benchmarks like ALFWorld and WebShop by 30+ absolute percentage points.

The intuition is simple. A pure chain-of-thought model reasons in a closed loop and tends to drift — it has no way to check its work against the world. A pure scripted agent has no way to recover when the script's assumptions are wrong. ReAct splits the difference: at each step, the model writes a short reason ("I should look up the customer's order history first"), takes one structured action (a tool call), and is then shown the action's observation (the tool's return value) before deciding what to do next. The reasoning gives the model a place to plan; the action gives it a way to ground its plan in reality; the observation closes the feedback loop.

In 2026 the pattern shows up in nearly every agent framework — LangGraph's `create_agent`, the LangChain v1.0 agent runtime, OpenAI's Assistants API, AWS Bedrock Agents, Microsoft Semantic Kernel — because it is the simplest useful shape that gets you a recoverable, debuggable tool-using loop. It is the baseline you depart from when you need something more elaborate (planner-executor, multi-agent supervisor, evaluator-optimizer), and the shape Anthropic's "Building effective agents" guide recommends you reach for first, before adding any orchestration.

## 3. Core Concepts

### 3.1 The three phases

A ReAct loop has three phases per turn:

- **Reason.** The model emits a short natural-language thought about what to do next. In Claude on Bedrock this lands in the assistant message's `text` content block; in some frameworks it is parsed from a `Thought:` prefix.
- **Act.** The model emits a structured action — in modern tool-use APIs, a `tool_use` content block carrying `{name, input}`. The host runtime executes the named tool with the given input.
- **Observe.** The host runtime returns the tool's result to the model on the next turn, as a `tool_result` content block on a `user`-role message. The model now sees what happened and can reason again.

The loop terminates when the model decides it has enough information to answer. For Claude on Bedrock the termination signal is `stop_reason: "end_turn"` with no further `tool_use` content blocks. For most other providers the signal is similar — the assistant emits a final message with no tool calls.

### 3.2 Workflow vs agent — the line that matters

Anthropic's "Building effective agents" guide draws a sharp line. A **workflow** is a system where the LLM and tools are orchestrated through predefined code paths — the developer decided "first do A, then if X do B else do C." An **agent** is a system where the LLM dynamically directs its own control flow and tool usage. ReAct is the canonical agent shape; prompt-chaining, routing, parallelization, and orchestrator-workers are workflow shapes.

The choice is not aesthetic. Workflows are predictable, cheap, and easy to evaluate. Agents are flexible and necessary when you cannot enumerate the paths up front — but they are also more expensive, harder to test, and harder to bound. The guide's repeated advice is to find the simplest solution that works and only escalate to an agent when a workflow cannot handle the variance in inputs.

### 3.3 Reason traces reduce hallucination

The Yao et al. paper's headline result is that interleaving reasoning and acting beats both pure reasoning (CoT) and pure acting (Act-only) on HotpotQA and Fever fact-verification tasks. The mechanism: in CoT the model can confidently invent a fact and then build on it; in ReAct the model has to ground each reasoning step in an action's observation, and the observation can contradict the invention. Modern agent frameworks preserve this property — even when the "reason" step is implicit (Claude tool-use, OpenAI tools), the model is still being shown each tool's actual return value before its next decision.

### 3.4 Termination, max-steps, and the budget question

A ReAct loop without a step cap can loop forever. Production agents apply a hard `max_iterations` (or `recursion_limit` in LangGraph) — typical defaults are 10 to 25 iterations. If the cap fires before the model emits a final answer, the runtime returns a structured "incomplete" result. The cap is one of three budgets you must set explicitly:

| Budget | What it bounds | Typical default |
|--------|----------------|-----------------|
| `max_iterations` | Number of Reason-Act-Observe cycles | 10–25 |
| Token budget | Total prompt + completion tokens across the run | Model-dependent; often 100K–200K |
| Wall-clock timeout | Time before the runtime kills the run | 60–300 seconds |

A loop that has not terminated by any of these is a bug in your prompt or your tool surface — not a sign you should raise the cap.

### 3.5 Frameworks that ship ReAct as a primitive

- **LangChain v1.0** ships `create_agent` as the default agent factory. It builds a graph-based agent runtime on LangGraph and exposes middleware slots for HITL, summarization, and PII redaction.
- **LangGraph** ships both `create_react_agent` (a prebuilt) and the lower-level `StateGraph` + `ToolNode` + `tools_condition` primitives. Most production teams start with the prebuilt and migrate to hand-rolled when they need custom routing.
- **OpenAI Assistants API**, **Bedrock Agents**, and **Vertex AI Agent Builder** all wrap a ReAct loop under the hood; the developer hands them tools and a prompt and the loop is run server-side.

The point of using a framework is that the loop, tool-result threading, and termination logic are handled for you. The point of understanding the loop is that when something breaks, you can read the trace and tell which phase failed.

## 4. Generic Implementation

A minimal ReAct loop in plain Python (no framework) against a generic chat-completions tool-use API:

```python
def run_react_loop(initial_message: str, tools: list, max_iterations: int = 10) -> str:
    """
    Generic ReAct loop. Terminates when the model returns no tool calls
    or when max_iterations is exceeded.
    """
    messages = [{"role": "user", "content": initial_message}]

    for step in range(max_iterations):
        # --- Reason + Act phase ---
        # The model produces a "thought" (text) and zero-or-more tool calls.
        response = llm_call(messages=messages, tools=tools)

        # --- Termination check ---
        # No tool calls means the model is done reasoning.
        if not response.tool_calls:
            return response.content

        # --- Observe phase ---
        # Echo the assistant turn back, then thread each tool result onto the
        # next user turn. Order matters: tool_use_id pairs each result to its call.
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
                # Structured error — let the model reason about the failure
                # rather than silently dropping it.
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": call.id,
                    "content": f"ERROR: {type(e).__name__}: {e}",
                    "is_error": True,
                })

        messages.append({"role": "user", "content": tool_results})

    # Hit the iteration cap without terminating — return a structured incomplete.
    return {"status": "incomplete", "reason": "max_iterations_reached", "messages": messages}
```

Three things in this snippet earn their place:

1. **Termination check happens before any tool-result threading.** If the model emits no tool calls, the loop is done; do not push an empty user turn.
2. **Tool errors are structured and observable.** Catching exceptions silently and returning `None` lets the model hallucinate around the gap; returning a structured error message lets the model reason about the failure and try a different tool or give up cleanly.
3. **The cap exits with structured incompleteness**, not an exception. Callers can inspect `messages` to see what the agent tried before timing out.

## 5. Real-world Patterns

### 5.1 Fintech — fraud-investigation copilot (Stripe Radar Assistants, Anthropic case studies)

Fintech vendors use ReAct loops to investigate flagged transactions. The agent reasons about a flag ("this transaction is from a new IP with a 3am timestamp on a card that usually transacts in business hours"), acts by calling tools (`get_card_history`, `get_ip_reputation`, `get_device_fingerprint`), and observes the results before deciding whether to escalate or auto-clear. The investigation runs against frozen account state; writes (`block_card`, `notify_customer`) are routed through a human-approval middleware step. The ReAct shape lets the model traverse a multi-signal graph that a static rules engine would struggle to express cleanly.

### 5.2 E-commerce — Shopify Sidekick

Shopify's Sidekick assistant uses an agentic loop to answer merchant questions ("why did my conversion drop yesterday?"). Reason: form a hypothesis. Act: call analytics tools to pull session funnels, ad-spend data, inventory state. Observe: read the numbers. Loop until the model can attribute the drop to a concrete factor (a misconfigured discount, an out-of-stock SKU, a regional ad-spend cut). The loop terminates when the agent produces a natural-language explanation citing the specific tool observations — making the answer auditable.

### 5.3 Healthcare — clinical-decision-support copilots

Healthcare LLM products (Hippocratic AI, Glass Health, Abridge) use ReAct over EHR-query tools. The reason step lets the model articulate a differential diagnosis hypothesis; the act step queries labs, imaging, prior notes; the observe step grounds the next hypothesis in actual patient data. The pattern is constrained — these systems typically forbid write actions entirely and require human sign-off on any recommendation — but the loop shape is the same.

### 5.4 Gaming — NPC behaviour and player-support agents

Game studios (Inworld AI, Replica, Riot's player-support assistants) use ReAct for two distinct cases: NPC behaviour where the agent reasons about player state and acts via game-engine APIs, and player-support where the agent investigates account-status reports across services. The latter is structurally identical to the fintech investigation pattern — different data, same loop.

## 6. Best Practices

- **Cap iterations and tokens explicitly; never trust the model to self-limit.** A runaway ReAct loop is the single most common production bug.
- **Make tool errors observable to the model.** Return structured error strings, not nulls. The model can then reason about the failure and either retry, switch tools, or give up.
- **Separate reads from writes in the tool surface.** Writes should be small in number, idempotent, and explicitly named; reads can be plentiful. The reason-act-observe pattern makes write-side mistakes more expensive because the model can chain them.
- **Trace every step.** Without a per-step trace (LangSmith, LangFuse, OpenTelemetry GenAI spans, or a homegrown log), you cannot debug why the model picked a tool or misinterpreted a result.
- **Prefer the framework's prebuilt loop until you have a reason to hand-roll it.** Hand-rolled loops are educational; production teams that hand-roll spend their time reimplementing termination, retries, and result-threading that the framework already gets right.
- **Test the termination condition explicitly.** Write a test that asserts the loop exits when the model returns no tool calls, and another that asserts the loop exits cleanly at `max_iterations`.

## 7. Hands-on Exercise

**Whiteboard exercise (15 min).** You are designing a ReAct agent that answers the question *"is this software package safe to use in our project?"* for an open-source security-review tool. The agent has the following tools:

- `get_package_metadata(name, version) -> PackageInfo` (read)
- `get_known_vulnerabilities(name, version) -> list[CVE]` (read)
- `get_dependency_tree(name, version) -> Tree` (read)
- `scan_dependency_tree_for_cves(tree) -> list[CVE]` (read)
- `submit_review(package_id, verdict, rationale) -> ReviewResult` (write)

Draw the ReAct loop on a whiteboard for the input *"is `requests==2.31.0` safe?"* Specifically:

1. Write a plausible **first Reason** step the model might emit.
2. Write the **first Act** call (tool name and arguments).
3. Write the **Observation** the host returns (you may invent realistic-looking data).
4. Write the **second Reason** step.
5. Continue until you reach termination.
6. State what your `max_iterations`, `tokens_max`, and `wall_clock_timeout` budgets should be, and why.

**What good looks like.** A solution shows three or four loop iterations, with the model walking from metadata → known CVEs → dependency tree → scanning dependencies → submitting a verdict. The final iteration calls `submit_review` and the loop terminates. Iteration cap is small (5–8), timeout is short (30–60s), and the writer can explain why the write tool only fires once. Bonus: the writer identifies that `submit_review` needs an idempotency key (covered in topic 4) so the model cannot accidentally submit twice.

## 8. Key Takeaways

- **What are the three phases of a ReAct loop, and what is the termination condition?** (Reason, Act, Observe; terminate when the model emits no tool calls.)
- **Why is a ReAct agent more robust than chain-of-thought on knowledge-intensive tasks?** (The Observe phase grounds each reasoning step in real tool output, breaking the hallucinate-and-build-on-it pattern.)
- **When should you reach for a workflow instead of an agent?** (When the control flow is enumerable up front — workflows are cheaper, more predictable, easier to evaluate.)
- **What budgets must every production ReAct loop set?** (Max iterations, token budget, wall-clock timeout — all three.)
- **What does the loop do when a tool errors?** (Return a structured error to the model so it can reason about the failure; never silently swallow.)

## Sources

1. [ReAct: Synergizing Reasoning and Acting in Language Models (Yao et al., arXiv 2210.03629)](https://arxiv.org/abs/2210.03629) — retrieved 2026-05-26
2. [Building Effective Agents (Anthropic)](https://www.anthropic.com/research/building-effective-agents) — retrieved 2026-05-26
3. [LangChain v1.0 Agents documentation](https://docs.langchain.com/oss/python/langchain/agents) — retrieved 2026-05-26
4. [LangChain and LangGraph Agent Frameworks Reach v1.0 Milestones (LangChain blog, Oct 2025)](https://blog.langchain.com/langchain-langgraph-1dot0/) — retrieved 2026-05-26
5. [What is a ReAct Agent? (IBM Think)](https://www.ibm.com/think/topics/react-agent) — retrieved 2026-05-26

Last verified: 2026-05-26
