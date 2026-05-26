---
week: W03
day: Tue
topic_slug: in-context-memory-and-compaction
topic_title: "In-context memory and compaction for long-running agent runs"
parent_overview: W03/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 8
sources:
  - url: https://platform.claude.com/docs/en/build-with-claude/context-editing
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://platform.claude.com/cookbook/tool-use-context-engineering-context-engineering-tools
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://deepwiki.com/langchain-ai/deepagentsjs/5.4-summarization-middleware
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://factory.ai/news/evaluating-compression
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://zylos.ai/research/2026-02-28-ai-agent-context-compression-strategies
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# In-context memory and compaction for long-running agent runs

## 1. Learning Objectives

- By the end of this reading, the learner can describe the two failure modes of long-running agent context: token-budget exhaustion and "context drift."
- By the end of this reading, the learner can distinguish three compaction strategies — summarize-on-overflow, selective retention, and tool-result trimming — and choose between them based on the trade-off between fidelity and budget.
- By the end of this reading, the learner can articulate why **anchored iterative summarization** outperforms full-reconstruction in published evaluations.
- By the end of this reading, the learner can implement a generic threshold-triggered summarization middleware that preserves recent turns and folds older turns into a structured summary.
- By the end of this reading, the learner can recognise provider-native compaction APIs (Claude context editing, Microsoft Agent Framework compaction, LangChain summarization middleware) and decide when to lean on them vs roll your own.

## 2. Introduction

Every long-running agent eventually runs into a wall. The wall has two shapes. The hard shape is the **token budget** — even Claude Sonnet 4.5's 200K-token context window can be exhausted by a few thousand turns of tool calls and observations, and well before exhaustion the per-call latency and cost both grow with prompt length. The soft shape is **context drift** — the model's attention is finite, and as the conversation grows the model increasingly loses track of earlier decisions, repeats itself, or contradicts its own prior reasoning. Zylos's 2026 industry survey attributes nearly 65% of enterprise AI failures to context drift, not raw context exhaustion.

Compaction is the practice of selectively shedding, summarising, or rewriting older portions of an agent's running history so the agent can continue working with bounded prompt size and minimal performance degradation. The 2025–2026 field has converged on a small set of recipes — provider-native compaction APIs from Anthropic, summarization middleware from LangChain and Microsoft Agent Framework, and the anchored-iterative-summary pattern Factory's evaluations identified as the strongest baseline.

This reading walks the strategies, the trade-offs, and the practical middleware shape you would build today.

## 3. Core Concepts

### 3.1 What gets compacted, and why retention is not uniform

An agent's running history is heterogeneous. Each turn is one of:

- **User input** — usually small, almost always worth keeping.
- **Assistant reasoning text** — verbose but rarely durable. The model's reflection on what to do next is scratch work; once the action it implied is taken, the reflection has produced its output.
- **Tool use blocks** — small JSON structures that name the tool and arguments. Cheap to keep; useful for replay.
- **Tool result blocks** — can be very large (PDF text, query results, page scrapes). These hold the durable observations the agent has accumulated.

The asymmetry means compaction is not uniform. The right policy is usually: **keep all `tool_result` blocks for a while** (they are the agent's facts), **summarise `assistant` reasoning beyond N recent turns** (it is scratch work), and **age out the oldest tool results into a summary** once a budget threshold is crossed.

### 3.2 Three compaction strategies

**Strategy A — Summarize-on-overflow.** When the running history crosses a soft threshold (e.g., 70% of the max context window), invoke a separate LLM call that summarises everything older than the last K turns into a single structured "earlier-in-this-run" preamble. Replace the older turns in the prompt with the summary. Repeat as needed.

Strengths: simple to implement; high compression ratio. Weaknesses: a bad summary can lose facts the agent later needs; the summarization LLM call is itself a cost.

**Strategy B — Selective retention.** Apply different policies to different content types. Keep the most recent N `assistant` turns; keep all `tool_result` blocks from the last M turns; drop or summarise older `assistant` reasoning entirely. Tool-result trimming — Claude Code's first compaction tier — is a special case: truncate very large tool results to their first/last K bytes plus a "...truncated..." marker.

Strengths: preserves the durable facts; cheap because no extra LLM call per turn. Weaknesses: a finely tuned policy is brittle to schema changes.

**Strategy C — Hierarchical / iterative summarization.** Break the run into logical chunks (per-tool-call, per-subgoal, or per-N-turns), summarise each chunk independently, then optionally summarise the summaries. Used for very long runs (>50 turns) where a single flat summary would itself be too large.

Strengths: scales to arbitrary length. Weaknesses: most complex to debug; summaries-of-summaries lose fidelity quickly.

### 3.3 Anchored iterative summarization — the published-best baseline

Factory's 2026 evaluation across 36,000 real engineering session messages compared "full-reconstruction summarisation" (regenerate the entire history summary every time) against "anchored iterative summarisation" (maintain a persistent summary state and merge each new chunk into it). The anchored shape consistently outperformed full-reconstruction on accuracy, completeness, and continuity scores. The mechanism: the persistent anchor preserves canonical facts that a re-generation might rephrase away or drop.

In code, anchored summarization looks like a running buffer of structured key-value claims ("user_id resolved to 9f0d-...", "amendment 3 confirmed posted at 16:30", "score for proposal X was 0.82") that the summariser only adds to, never rewrites.

### 3.4 Provider-native compaction APIs

Three production-grade options as of 2026:

- **Claude context editing.** Anthropic's API supports server-side compaction directives that offload older context to persistent storage and let Claude look up cleared information from memory files. Reduces the host's responsibility for compaction logic; the trade-off is provider lock-in.
- **LangChain summarization middleware.** A drop-in middleware for `create_agent` and Deep Agents. Monitors token usage, summarises older messages when a threshold is hit, replaces them with a condensed summary while keeping recent turns intact.
- **Microsoft Agent Framework compaction.** Equivalent feature in the .NET / Python Agent Framework. Same pattern: threshold + summary + replace.

For most teams, lean on the framework's middleware until you have a reason to roll your own. The middleware authors have already debugged the corner cases (preserving tool_use/tool_result pairing across the summarisation boundary; not summarising the system prompt; handling concurrent summarisation calls).

### 3.5 Two non-obvious things to preserve

When you compact, two things are easy to lose and painful to recover:

- **Tool_use / tool_result pairing.** If you summarise a turn that contained a `tool_use` and drop the corresponding `tool_result`, the model gets a `tool_use_id` reference in its history with no matching `tool_result` — the API rejects the request. The summariser must drop both or neither.
- **The first user message.** The initial user prompt is the agent's task. Summarising it is rarely worth the savings and almost always degrades alignment to the goal. Keep the original first user message verbatim.

## 4. Generic Implementation

A generic threshold-triggered summarisation middleware for any chat-completions agent. Independent of Claude/OpenAI/LangChain specifics.

```python
from dataclasses import dataclass
from typing import Callable

@dataclass
class Message:
    role: str               # "user" | "assistant" | "system"
    content: list           # list of {type: "text"|"tool_use"|"tool_result", ...}
    tokens: int             # pre-computed token count

SUMMARY_TRIGGER_PCT = 0.70    # compact when running total crosses 70% of context window
KEEP_RECENT_TURNS = 6         # always keep the last 6 turns verbatim
MAX_CONTEXT_TOKENS = 200_000

def compact_if_needed(
    messages: list[Message],
    *,
    summarise: Callable[[list[Message]], str],
) -> list[Message]:
    total = sum(m.tokens for m in messages)
    if total < SUMMARY_TRIGGER_PCT * MAX_CONTEXT_TOKENS:
        return messages   # nothing to do

    if len(messages) <= KEEP_RECENT_TURNS + 1:
        return messages   # cannot compact further without dropping recent context

    # Preserve the original first user turn verbatim — that is the task.
    head = [messages[0]]
    tail = messages[-KEEP_RECENT_TURNS:]
    middle = messages[1:-KEEP_RECENT_TURNS]

    # Drop tool_use/tool_result pairs together — never split a pair across the boundary.
    middle = strip_orphan_tool_pairs(middle, tail)

    # Anchored summarisation: merge the middle into a structured "earlier-in-this-run"
    # preamble, appended to the running summary anchor.
    new_summary = summarise(middle)
    summary_msg = Message(
        role="system",
        content=[{"type": "text", "text": f"Earlier in this run: {new_summary}"}],
        tokens=estimate_tokens(new_summary),
    )

    return head + [summary_msg] + tail
```

Three things worth noticing:

1. **The first user turn is preserved literally.** It is the agent's task; summarisation would degrade goal-alignment.
2. **Tool pairs are kept atomic.** `strip_orphan_tool_pairs` drops any `tool_use` whose matching `tool_result` is in the tail (and vice versa) — never splits the pair.
3. **The summary lives in a `system` role.** This keeps it clearly framed as background context, not part of the user-assistant dialogue.

## 5. Real-world Patterns

### 5.1 Coding assistants — Claude Code, Codex CLI, OpenCode

The most heavily evaluated context-management implementations live in coding assistants, where sessions routinely span hundreds of turns. Claude Code uses a three-tier strategy: tool-result trimming (truncate large file reads), prompt-cache-friendly compaction (keep stable prefixes intact so cached tokens stay valid), and a 9-section structured LLM summary at the high-water mark. Codex CLI uses a single-layer "handoff summary." OpenCode hides messages by timestamp without destruction. Justin3go's comparative writeup is a useful reference for how these strategies trade off recency, fidelity, and cost.

### 5.2 Customer-support copilots — Intercom Fin, Zendesk Resolve

Customer-support agents handle multi-thread conversations that can span days. They almost universally use anchored summarisation against a per-customer profile: each new ticket's outcome is folded into a persistent customer summary that gets retrieved at the start of the next session. The pattern decouples within-session compaction from cross-session memory; the within-session compaction is short-lived, the cross-session anchor is durable.

### 5.3 Research and writing assistants — Notion AI, Mem, Cursor agents

Long-form writing and research assistants face the most extreme compression demands — a research session can produce 100K+ tokens of intermediate notes. The pattern here is hierarchical: per-source notes are summarised independently, then summaries are merged at the section level, then sections at the document level. Each layer is a smaller LLM call than a flat summary would be.

### 5.4 Game AI and simulation — long-running NPCs

Open-world games with persistent NPCs (Inworld, Replica) treat in-context memory as a working set and rely on a vector-store-backed long-term memory for anything older than the last N interactions. The agent's context is short-lived; the long-term memory is what gives the NPC continuity. The pattern travels: when an agent's lifetime exceeds what compaction can sustainably hold, externalise to a vector store and retrieve on demand.

## 6. Best Practices

- **Trigger on percentage of budget, not absolute count.** Different models have different windows; thresholds should be relative.
- **Always keep the original first user turn.** It is the task; summarising it degrades alignment.
- **Never split a `tool_use` / `tool_result` pair across the compaction boundary.** The API will reject the resulting prompt.
- **Prefer anchored summarisation over full-reconstruction.** Published evaluations favour anchored on accuracy, completeness, and continuity.
- **Lean on provider-native and framework middleware first.** Roll your own only when a measured limitation forces it.
- **Log every compaction event.** What was the budget when you triggered? How many turns were compacted? What was the summary? Without this, debugging "the agent forgot something" is guesswork.
- **Test that compaction is idempotent.** Compacting twice on the same input should produce the same output; if it doesn't, your summariser is non-deterministic and your traces are non-replayable.

## 7. Hands-on Exercise

**Whiteboard exercise (15 min).** You are designing the compaction policy for a long-running research assistant that helps users write literature reviews. A typical session:

- Starts with a user research question.
- Iterates through 30–60 web searches, fetching and reading sources.
- Periodically produces section drafts that the user edits.
- Eventually outputs a final review.

Design the compaction policy. Specifically:

1. What is your trigger threshold (% of context window)?
2. Which content types do you preserve verbatim, which do you summarise, which do you drop?
3. How does the user's first message survive across compactions?
4. How do you make sure section drafts the user has edited are not summarised away?
5. Where would you reach for hierarchical summarisation vs flat?

**What good looks like.** A solution sets the trigger at 60–75% of context, preserves the first user turn and all user-edited drafts verbatim, summarises older assistant reasoning, keeps tool_use/tool_result pairs atomic, and uses hierarchical summarisation per-source (each source's read is summarised independently before the section-level summary). Bonus: the writer identifies that user-edited drafts should be pinned with an explicit marker so the summariser knows not to touch them.

## 8. Key Takeaways

- **What are the two failure modes of long-running agent context?** (Token-budget exhaustion — hard limit; context drift — soft degradation as attention fragments.)
- **Why is retention non-uniform across content types?** (Tool results are durable facts; assistant reasoning is scratch work that has already produced its output.)
- **Why does anchored iterative summarisation beat full-reconstruction?** (The persistent anchor preserves canonical facts that a re-generation might rephrase or drop.)
- **What two things must compaction never lose?** (The original first user turn; the atomic pairing between `tool_use` and `tool_result`.)
- **When should you build your own compaction vs use provider-native or framework middleware?** (Default to the framework; build your own only when a measured limitation forces it.)

## Sources

1. [Context editing — Claude API Docs](https://platform.claude.com/docs/en/build-with-claude/context-editing) — retrieved 2026-05-26
2. [Context engineering: memory, compaction, and tool clearing — Claude Cookbook](https://platform.claude.com/cookbook/tool-use-context-engineering-context-engineering-tools) — retrieved 2026-05-26
3. [Summarization Middleware — LangChain deepagentsjs](https://deepwiki.com/langchain-ai/deepagentsjs/5.4-summarization-middleware) — retrieved 2026-05-26
4. [Evaluating Context Compression for AI Agents (Factory.ai)](https://factory.ai/news/evaluating-compression) — retrieved 2026-05-26
5. [AI Agent Context Compression: Strategies for Long-Running Sessions (Zylos Research)](https://zylos.ai/research/2026-02-28-ai-agent-context-compression-strategies) — retrieved 2026-05-26

Last verified: 2026-05-26
