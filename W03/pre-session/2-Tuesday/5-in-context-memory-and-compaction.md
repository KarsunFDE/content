---
week: W03
day: Tue
topic_slug: in-context-memory-and-compaction
topic_title: "In-context memory and compaction for long-running agent runs"
parent_overview: W03/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 12
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
last_verified: 2026-06-06
---

# In-context memory and compaction for long-running agent runs

> [!NOTE]
> **From earlier:** Topics 2–4 wired the ReAct loop, the tool schemas, and the idempotency contracts. Federal solicitation responses are long — a 50-page Section L plus a 30-page Section M puts the agent near context limits fast. This topic handles that.

## 1. Learning Objectives

- Describe the two failure modes of long-running agent context: token-budget exhaustion and context drift
- Distinguish three compaction strategies — summarize-on-overflow, selective retention, and hierarchical summarization — and choose between them based on the trade-off between fidelity and budget
- Articulate why anchored iterative summarization outperforms full-reconstruction in published evaluations
- Implement a generic threshold-triggered summarization middleware that preserves recent turns and folds older turns into a structured summary
- Recognize provider-native compaction APIs and decide when to lean on them vs roll your own

## 2. Introduction

Every long-running agent hits a wall in two shapes. The **hard shape** is the token budget — even a 200K-token window can be exhausted by long tool-call histories, and per-call latency and cost grow well before exhaustion. The **soft shape** is context drift — as the conversation grows the model loses track of earlier decisions, repeats itself, or contradicts its own reasoning. Zylos's 2026 survey attributes nearly 65% of enterprise AI failures to context drift, not raw exhaustion.

Compaction selectively sheds, summarises, or rewrites older history so the agent continues with bounded prompt size. The field converged on three approaches: provider-native APIs (Anthropic), summarization middleware (LangChain), and the anchored-iterative-summary pattern Factory's evaluations identified as the strongest baseline.

## 3. Core Concepts

### 3.1 What gets compacted — retention is not uniform

An agent's running history is heterogeneous. Each turn is one of:

- **User input** — small, almost always worth keeping
- **Assistant reasoning text** — verbose but rarely durable; once the implied action is taken, the reflection is scratch work
- **Tool use blocks** — small JSON structures; cheap to keep
- **Tool result blocks** — can be very large (PDF text, query results, scrapes); hold the agent's durable observations

Right policy: **keep `tool_result` blocks** (durable facts), **summarise `assistant` reasoning beyond N recent turns** (scratch work), **age out oldest tool results** into a summary once a budget threshold is crossed.

### 3.2 Three compaction strategies

**Strategy A — Summarize-on-overflow.** When history crosses 70% of window, one LLM call summarises everything older than the last K turns into a structured preamble. High compression; a bad summary can lose facts the agent needs later.

**Strategy B — Selective retention.** Different policies per content type: keep recent `tool_result` blocks (durable facts), summarise older `assistant` reasoning (scratch work). No extra LLM call; a finely-tuned policy can be brittle to schema changes.

**Strategy C — Hierarchical / iterative summarization.** Summarise run chunks independently, then summarise the summaries. Scales to very long runs; most complex to debug.

> [!TIP]
> **Anchored iterative summarization beats full-reconstruction.** Factory's 2026 evaluation (36,000 real engineering session messages) found the anchored shape — maintain a persistent summary state, merge new chunks into it — consistently outperformed full-reconstruction on accuracy, completeness, and continuity. The persistent anchor preserves canonical facts that re-generation might rephrase away.

### 3.3 Two things compaction must never lose

- **Tool_use / tool_result pairing.** Drop both or neither — a `tool_use_id` with no matching `tool_result` causes an API rejection.
- **The first user message.** It is the agent's task; summarising it degrades goal alignment without meaningful token savings.

### 3.4 Provider-native compaction APIs

Three options as of 2026: **Claude context editing** (server-side directives offload older context to persistent storage; provider lock-in trade-off), **LangChain summarization middleware** (drop-in for `create_agent`; threshold + summarise + replace), **Microsoft Agent Framework compaction** (same pattern, .NET/Python). Default to the framework middleware — the authors have already debugged corner cases (preserving tool_use/tool_result pairing across the boundary, not summarising the system prompt). Roll your own only when a measured limitation forces it.

## 4. Generic Implementation

A threshold-triggered summarisation middleware for any chat-completions agent:

```python
from dataclasses import dataclass
from typing import Callable

@dataclass
class Message:
    role: str       # "user" | "assistant" | "system"
    content: list   # list of {type: "text"|"tool_use"|"tool_result", ...}
    tokens: int     # pre-computed token count

SUMMARY_TRIGGER_PCT = 0.70    # compact when running total crosses 70% of window
KEEP_RECENT_TURNS   = 6       # always keep the last 6 turns verbatim
MAX_CONTEXT_TOKENS  = 200_000

def compact_if_needed(
    messages: list[Message],
    *,
    summarise: Callable[[list[Message]], str],
) -> list[Message]:
    total = sum(m.tokens for m in messages)
    if total < SUMMARY_TRIGGER_PCT * MAX_CONTEXT_TOKENS:
        return messages  # nothing to do

    if len(messages) <= KEEP_RECENT_TURNS + 1:
        return messages  # cannot compact further without losing recent context

    # Preserve the original first user turn — that is the task
    head = [messages[0]]
    tail = messages[-KEEP_RECENT_TURNS:]
    middle = messages[1:-KEEP_RECENT_TURNS]

    # Drop tool_use/tool_result pairs together — never split a pair
    middle = strip_orphan_tool_pairs(middle, tail)

    # Anchored summarisation: merge middle into structured preamble
    new_summary = summarise(middle)
    summary_msg = Message(
        role="system",
        content=[{"type": "text", "text": f"Earlier in this run: {new_summary}"}],
        tokens=estimate_tokens(new_summary),
    )

    return head + [summary_msg] + tail
```

The first user turn is preserved literally. Tool pairs are kept atomic. The summary lives in a `system` role — clearly framed as background context, not dialogue.

## 5. Real-world Patterns

**Coding assistants (Claude Code, Codex CLI).** Sessions span hundreds of turns. Claude Code uses three tiers: tool-result trimming, prompt-cache-friendly compaction (stable prefixes keep cached tokens valid), and a structured LLM summary at the high-water mark.

**Customer-support copilots (Intercom Fin, Zendesk Resolve).** Each ticket outcome folds into a persistent per-customer summary retrieved at session start. Within-session compaction is short-lived; the cross-session anchor is durable.

**Research assistants (Notion AI, Cursor agents).** 100K+ tokens of intermediate notes use hierarchical compaction: per-source notes summarised independently, merged at section level, then document level.

> [!NOTE]
> **Acquire-gov framing.** The 50-page Section L + 30-page Section M scenario is the research-assistant shape — hierarchical compaction applies directly.

> [!CAUTION]
> **Anti-pattern: `compaction-loses-decisions`.** The summariser may drop intermediate decisions ("amendment confirmed stale, escalating to CO") because they look like reasoning scratch-work. On resume, the agent re-reasons from facts and may reach a different decision. Pin interim decisions explicitly in the summary anchor.

## 6. Best Practices

- Trigger on percentage of budget, not absolute count
- Always keep the original first user turn verbatim
- Never split a `tool_use` / `tool_result` pair across the compaction boundary
- Prefer anchored summarisation over full-reconstruction
- Lean on framework middleware first; roll your own only when a measured limitation forces it
- Log every compaction event — budget at trigger, turns compacted, summary produced

> [!WARNING]
> **Anti-pattern: `context-stuffing-no-compaction`.** Stuffing every tool result verbatim into the running context until the window overflows is the naive approach — it degrades reasoning quality long before it hits the hard limit, as the model's attention dilutes across an ever-larger prompt. Set a compaction threshold at 70% of the context window and treat compaction as a first-class feature, not a fix added after production failures.

## 7. Hands-on Exercise

You are designing the compaction policy for a long-running research assistant (30–60 web searches, periodic section drafts the user edits, final review output). Design the policy: (1) trigger threshold as % of window; (2) which content types to preserve verbatim, summarise, or drop; (3) how the first user message survives across compactions; (4) how user-edited drafts are pinned against summarisation; (5) where hierarchical vs flat summarisation applies.

> [!NOTE]
> **Self-check** (30 s — answer mentally before expanding)
>
> 1. Why must compaction never split a `tool_use` / `tool_result` pair?
> 2. Why does anchored iterative summarization outperform full-reconstruction?

<details>
<summary>Show answers</summary>

1. The API requires every `tool_use_id` in the assistant turn to have a matching `tool_result` in the subsequent user turn. If you summarise the `tool_use` block out of the history but leave the `tool_result`, or vice versa, the API rejects the conversation with a validation error. The summariser must treat the pair as an atomic unit.
2. Full-reconstruction regenerates the entire history summary from scratch each time — the summarisation LLM may rephrase or drop canonical facts that were correctly captured in the prior summary. The anchored approach maintains a persistent summary state and only merges new chunks into it, preserving previously established facts across compaction events.

</details>

## 8. Key Takeaways

- **Two failure modes:** token-budget exhaustion (hard) and context drift (soft — attention dilutes across a growing prompt)
- **Retention is non-uniform:** tool results are durable facts; assistant reasoning is scratch work already acted on
- **Anchored iterative summarization beats full-reconstruction** — the persistent anchor preserves canonical facts
- **Never lose:** the original first user turn, and `tool_use`/`tool_result` pairs (always drop both or neither)
- **Default to framework middleware;** roll your own only when a measured limitation forces it

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://platform.claude.com/docs/en/build-with-claude/context-editing — Context editing (Claude API Docs) — retrieved 2026-05-26 — hot-tech
- https://platform.claude.com/cookbook/tool-use-context-engineering-context-engineering-tools — Context engineering: memory, compaction, and tool clearing (Claude Cookbook) — retrieved 2026-05-26 — hot-tech
- https://deepwiki.com/langchain-ai/deepagentsjs/5.4-summarization-middleware — Summarization Middleware (LangChain deepagentsjs) — retrieved 2026-05-26 — hot-tech
- https://factory.ai/news/evaluating-compression — Evaluating Context Compression for AI Agents (Factory.ai) — retrieved 2026-05-26 — hot-tech
- https://zylos.ai/research/2026-02-28-ai-agent-context-compression-strategies — AI Agent Context Compression: Strategies for Long-Running Sessions (Zylos Research) — retrieved 2026-05-26 — hot-tech

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

Prompt-cache-friendly compaction (Claude Code's approach) preserves stable prefixes so that cached tokens remain valid across compaction events. The trade-off: a stable prefix means you cannot reorganise the early part of the conversation, so compaction must append-only to the summary, never rewrite it. This aligns exactly with the anchored-iterative pattern — the anchor is the stable prefix; new chunks are appended. Senior FDEs building the W4–W5 observability layer should design compaction events to be cache-preserving by default.

Factory's evaluation methodology: 36,000 session messages from real engineering sessions, split into full-reconstruction vs anchored-iterative conditions, evaluated by a separate judge model on three dimensions (accuracy = did the summary retain all facts; completeness = did it drop any important decision; continuity = did the agent's post-compaction behaviour remain consistent with pre-compaction context). Anchored-iterative won on all three. The margin was largest on completeness, which aligns with the "compaction-loses-decisions" anti-pattern described above.

For very long runs (>100 turns, e.g., a multi-day research session), the anchored summary itself can become large. Hierarchical compaction handles this by summarising the anchor every N compaction events. The risk is the same as single-level compaction — facts can be dropped — so the anchor for the anchor should be even more conservative about what it summarises away. In practice, for the acquire-gov intake-triage flow (typically 5–15 turns), none of this complexity is needed; the single-level middleware suffices.

</details>

Last verified: 2026-06-06
