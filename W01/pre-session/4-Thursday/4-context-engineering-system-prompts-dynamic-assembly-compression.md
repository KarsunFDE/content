---
week: W01
day: Thu
topic_slug: context-engineering-system-prompts-dynamic-assembly-compression
topic_title: "Context engineering — system prompts, dynamic assembly, compression"
parent_overview: W01/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 15
sources:
  - url: https://www.deepset.ai/blog/context-engineering-the-next-frontier-beyond-prompt-engineering
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://blog.bytebytego.com/p/a-guide-to-context-engineering-for
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://logic.inc/resources/context-engineering-for-production-llm-applications
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.firecrawl.dev/blog/context-engineering
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://zylos.ai/research/2026-03-17-dynamic-context-assembly-projection-llm-agent-runtimes
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# Context engineering — system prompts, dynamic assembly, compression

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish prompt engineering (string-level craft) from context engineering (system-level discipline).
- Decompose a system prompt into named, individually-owned sections with separate test strategies.
- Describe a context-assembly pipeline that combines static templates, retrieved documents, tool outputs, and conversation memory.
- Choose between summary-buffer, sliding-window, and hierarchical-compression strategies for context-window pressure.
- Set up a token-budgeting plan that fails closed instead of silently truncating.

## 2. Introduction

A prompt used to be a string. You wrote it, you handed it to the model, you read the result. The 2025–2026 evolution — documented across deepset, ByteByteGo, Firecrawl, and Logic.inc — reframes the prompt as a *view* assembled at request time from many sources: a static persona, retrieved documents, recent tool outputs, user history, formatting hints. The discipline of assembling that view well, repeatably, and within token budgets is context engineering.

The shift matters because production systems aren't deterministic. The same user query against the same model yields different outputs depending on what got assembled into the context: which documents the retriever surfaced, how the tool calls landed, how recent the conversation history is, whether the prompt template version drifted. Treating the prompt as a static string hides all of that variability inside an opaque blob. Treating it as a *materialised view* — assembled on demand from typed sources — makes the variability inspectable, versionable, and testable.

This reading has three beats matching the topic title. First, the *shape* of a system prompt as architecture. Second, the *assembly* pipeline that builds the runtime context. Third, the *compression* patterns that keep the assembled view inside the model's context window. Friday's PR work touches all three lightly; W2 returns here in depth when RAG retrieval starts producing chunk sets that exceed a single-pass window.

## 3. Core Concepts

### 3a. System prompt as architecture

A production system prompt is not free-form prose. It is a structured document with named sections, each owned by a specific concern. The shape that has converged in 2026 looks like this:

```
<persona>...who the model is...</persona>
<constraints>...what the model must / must not do...</constraints>
<output_format>...JSON schema, formatting rules...</output_format>
<grounding>...retrieved documents, tool outputs (assembled at runtime)...</grounding>
<task>...the user's specific request...</task>
```

Each section has three properties worth tracking:

- **Owner.** A specific team or person responsible for changes. Drift between section owners is where prompts rot — a constraint added by the safety team conflicts with a format rule added by the product team and nobody owns the integration.
- **Change history.** Sections evolve at different cadences. Persona changes rarely; constraints change with policy; output_format changes with downstream contract; grounding changes every request. They need separate version-tracking discipline.
- **Eval set.** Different sections need different tests. Persona has snapshot tests. Constraints have policy-violation tests. Output_format has schema-validation tests. Grounding has retrieval-quality evals (built in W2 Fri).

The discipline is: never write a "make it work" patch to a prompt. Identify which section owns the concern, change *that* section, run *that* section's evals.

### 3b. Dynamic context assembly

The runtime view is assembled by a small piece of software — variously called a "context engine," "context-assembly pipeline," or just "the prompt builder." Its job is to take a user request and produce the final prompt, deterministically, from typed sources.

A canonical assembly pipeline:

```
user query
  → (1) intent classifier — what kind of request is this?
  → (2) retriever(s) — fetch relevant documents from vector store, knowledge graph, structured DB
  → (3) tool-output collector — gather any prior tool responses in this turn
  → (4) memory loader — fetch conversation history, user profile
  → (5) template renderer — fill the system-prompt template with sections (3a)
  → (6) token-budget enforcer — ensure the assembled view fits the model window
  → final prompt → model
```

The Firecrawl and ByteByteGo guides describe this as moving "from static templates to systematic, dynamic construction" — the work shifts from prompt wording to assembly infrastructure: token budgeting, retrieval evaluation, version control for prompt artifacts, testing harnesses that handle non-determinism.

The reason it matters: with a static prompt template, you change the prompt and redeploy. With dynamic assembly, you change one *source* — a retriever's index, a tool's output format, the conversation-memory policy — and the effect on the final prompt is *implicit*. Without the pipeline as a first-class artifact (logged, traced, versioned), debugging an output regression is guesswork.

### 3c. Context compression patterns

Even with disciplined assembly, the model's context window is finite. Frontier models in 2026 have large windows (hundreds of thousands of tokens), but using all of them is expensive and slow, and retrieval-quality degrades on long contexts ("lost-in-the-middle" effect). Three patterns dominate:

**Summary buffer.** Keep the most recent N turns verbatim. Replace earlier turns with a running summary maintained by a smaller, cheaper model. The summary is updated each turn — recent context stays detailed, older context compresses gracefully. Best for chat-style applications where recency carries meaning.

**Sliding window.** Drop oldest turns when context approaches the limit. Simplest to implement; loses information silently. Acceptable when older turns are genuinely irrelevant — e.g., long-running agent tasks where each step only needs the last few observations.

**Hierarchical compression.** Summarise summaries. Tier 1 keeps recent verbatim, tier 2 keeps medium-age summaries, tier 3 keeps long-ago meta-summaries. More implementation overhead; better for systems where older context still carries value but at lower fidelity.

A fourth pattern worth naming: **retrieval-replaces-history.** For knowledge-heavy tasks, don't carry the document in the prompt at all — re-retrieve when needed. The trade is latency-per-turn vs. context-window cost.

### 3d. Token budgeting — fail closed

Every assembly pipeline must enforce a token budget *before* the call leaves the application. Two failure modes to avoid:

1. **Silent truncation.** The pipeline assembles a too-long prompt; the model SDK truncates. The model now answers from a partial view, with no signal to the caller. Quality drops invisibly.
2. **Naive ratio cuts.** The pipeline notices the prompt is too long and lops 30% off uniformly. Critical sections (output_format, constraints) lose tokens equally with low-priority sections (older history). Quality drops in load-bearing places.

The correct shape is **fail closed with priorities.** Assign each section a priority and a minimum token count. Enforce the budget by dropping low-priority sections entirely, not by uniform truncation. If even the minimums don't fit, fail the request and surface a clear error to the caller — never silently degrade.

### 3e. Logging the resolved prompt

A non-negotiable for any context-engineering system: log the *resolved* prompt at debug level. Not the template — the final assembled string the model actually saw. Without it, debugging a bad output is impossible: you don't know whether the issue was the template, the retriever, the conversation history, or the compression policy.

In production this is typically gated behind a sampling rate (1–5% of requests) plus a configurable per-tenant override for debugging specific incidents. The discipline is to keep the trace available, not to log every request fully.

## 4. Generic Implementation

A worked example outside federal acquisitions — an e-commerce customer-support chatbot:

```python
from dataclasses import dataclass
from typing import Protocol

@dataclass
class PromptSection:
    name: str
    content: str
    priority: int      # 0 = must-keep, 10 = drop-first
    min_tokens: int    # below this we drop the section entirely

class Retriever(Protocol):
    def fetch(self, query: str, k: int) -> list[str]: ...

class Tokenizer(Protocol):
    def count(self, text: str) -> int: ...

def assemble_prompt(
    user_query: str,
    persona: str,
    constraints: str,
    output_format: str,
    history_summary: str,
    recent_turns: list[str],
    retriever: Retriever,
    tokenizer: Tokenizer,
    budget: int,
) -> str:
    # 1. Retrieve grounding documents
    docs = retriever.fetch(user_query, k=5)
    grounding = "\n\n".join(docs)

    # 2. Build sections with priorities
    sections = [
        PromptSection("persona", persona, priority=0, min_tokens=50),
        PromptSection("constraints", constraints, priority=0, min_tokens=80),
        PromptSection("output_format", output_format, priority=0, min_tokens=60),
        PromptSection("grounding", grounding, priority=2, min_tokens=200),
        PromptSection("history_summary", history_summary, priority=4, min_tokens=100),
        PromptSection("recent_turns", "\n".join(recent_turns), priority=3, min_tokens=150),
        PromptSection("task", f"User: {user_query}", priority=0, min_tokens=20),
    ]

    # 3. Fit to budget: drop sections by priority until we fit
    sections.sort(key=lambda s: (s.priority, -s.min_tokens))
    kept: list[PromptSection] = []
    used = 0
    for s in sections:
        cost = max(tokenizer.count(s.content), s.min_tokens)
        if used + cost <= budget:
            kept.append(s)
            used += cost
        elif s.priority == 0:
            # must-keep section can't fit — fail closed
            raise RuntimeError(
                f"section '{s.name}' is required but would exceed budget by "
                f"{used + cost - budget} tokens"
            )

    # 4. Render in stable section order (not the priority sort)
    canonical_order = ["persona", "constraints", "output_format",
                       "grounding", "history_summary", "recent_turns", "task"]
    rendered = []
    by_name = {s.name: s for s in kept}
    for name in canonical_order:
        if name in by_name:
            rendered.append(f"<{name}>\n{by_name[name].content}\n</{name}>")
    return "\n\n".join(rendered)
```

The shape of the function carries the discipline: sections are *typed*, priorities are *explicit*, fit-to-budget is *deterministic*, and the must-keep sections fail-closed instead of degrading.

> [!instructor-review]
> The `chain.run()` / LCEL `|`-pipe phrasing that appears in many older context-engineering blog posts is pre-LangChain v1.0 (October 2025) and is on the known-bad-patterns list. This reading deliberately uses plain Python function composition for the assembly pipeline. **Specifically flagged stale patterns in v0.x advocacy tutorials:** `from langchain.chains import LLMChain` imports, `chain.run(input)` invocation, LCEL `prompt | model | parser` pipe composition, `RunnableLambda`/`RunnablePassthrough` plumbing presented as canonical, and any tutorial that opens with the legacy `Chain` class. The v1.0 idioms are `init_chat_model` + standard Python composition. If a third-party tutorial still teaches the `|`-pipe shape or the legacy `Chain` API, treat the entire tutorial as stale — not just the import line.

## 5. Real-world Patterns

**E-commerce — personalised product-search assistant:** A large retailer's search-assistant uses dynamic assembly to inject the user's recent browsing history, current cart contents, and the catalogue subset retrieved by the semantic search step. The team treats each context source as a separate eval surface — the retrieval quality is measured separately from the prompt-template quality. When conversion rates dipped one quarter, the post-mortem traced the regression to a retriever index update, not the prompt — exactly the kind of debugging only context engineering enables.

**Healthcare — clinical decision support:** A provider-facing summarisation tool assembles patient history, current vitals, recent lab results, and the on-call physician's persona-prompt. The constraints section enforces "never recommend medication directly — always frame as discussion items for the physician." Because that section is owned by the clinical-safety team and versioned separately, it can be audited independently — a load-bearing property for medical-device regulatory review.

**Gaming — live-service NPC dialogue:** A live game uses on-device LLM inference to drive NPC conversations. Context budgets are tight (smaller window, latency-sensitive). The team uses hierarchical compression — last 3 turns verbatim, last 20 turns summarised, full conversation distilled into a single "relationship summary" memory slot. The summarisation runs in the background between turns so the user never feels the cost.

**Fintech — fraud-investigation copilot:** Investigators chat with a copilot about a flagged transaction. The assembly pipeline pulls the transaction record, the customer profile, recent transactions in the cluster, and prior alerts on the same account. Token budget is enforced by priority: transaction + customer profile are must-keep; the cluster history is best-effort and degrades to a count-only summary if budget is tight. Logging the resolved prompt at a per-investigator sample rate is the load-bearing audit-trail requirement.

## 6. Best Practices

- Decompose every system prompt into named sections with explicit owners, change-history, and per-section evals.
- Treat the context-assembly pipeline as a first-class artifact — version it, trace it, log the resolved prompt at a sampling rate.
- Set token budgets with priorities; fail closed on must-keep sections rather than silently truncate.
- Match compression strategy to traffic shape — summary-buffer for chat, sliding-window for short-horizon agents, hierarchical for long-running sessions.
- Test prompt sections in isolation before composing; an integration test on the whole prompt cannot tell you which section caused a regression.
- Separate static (persona, constraints) from dynamic (grounding, history) — snapshot-test the static side, retrieval-eval the dynamic side.
- Never log the full prompt in production unconditionally — sample with per-tenant override; PII concerns are real.

## 7. Hands-on Exercise

**Task (10–15 min, whiteboarding):** Draw a context-assembly pipeline for a customer-service chatbot for a delivery company. The user query is "where's my order?"

Sketch:

1. The sources that feed the context (user identity, order id resolver, order-tracking service, FAQ index, conversation history).
2. The section structure of the final system prompt (persona, constraints, output format, grounding, task).
3. The token budget for each section, with priorities.
4. The compression strategy for the history section.
5. What you log, and at what sample rate.

**What good looks like:** Each source is named with a typed return shape (not "the API"). Each section has a clear owner role (product, safety, etc.). Priorities are explicit and at least one must-keep section is identified. The compression strategy is justified (why sliding window vs. summary buffer here). The logging plan acknowledges PII and proposes a sampling rate. If your sketch has no token budget at all, you have not engineered the context — you have written a prompt template.

## 8. Key Takeaways

- *What is the difference between prompt engineering and context engineering?* Prompt engineering crafts a string; context engineering builds a system that assembles the right string from typed sources, repeatably and within budgets.
- *Why decompose a system prompt into named sections?* Different sections have different owners, change cadences, and eval strategies — flat strings hide all three.
- *When does silent truncation hurt me?* Always. Set priorities and fail closed on must-keep sections.
- *What compression strategy fits my application?* Summary-buffer for chat (recency matters), sliding-window for short-horizon agents (older turns irrelevant), hierarchical for long sessions (older still useful but at lower fidelity).
- *What's the single most useful production observability practice for context-engineered systems?* Logging the resolved prompt (the final assembled string) at a sample rate — debugging is impossible without it.

## Sources

1. [Context Engineering: The Next Frontier Beyond Prompt Engineering (deepset)](https://www.deepset.ai/blog/context-engineering-the-next-frontier-beyond-prompt-engineering) — retrieved 2026-05-26
2. [A Guide to Context Engineering for LLMs (ByteByteGo)](https://blog.bytebytego.com/p/a-guide-to-context-engineering-for) — retrieved 2026-05-26
3. [Context Engineering for Production LLM Applications (Logic.inc, 2026)](https://logic.inc/resources/context-engineering-for-production-llm-applications) — retrieved 2026-05-26
4. [Context Engineering vs Prompt Engineering for AI Agents (Firecrawl)](https://www.firecrawl.dev/blog/context-engineering) — retrieved 2026-05-26
5. [Dynamic Context Assembly and Projection Patterns for LLM Agent Runtimes (Zylos Research, 2026-03)](https://zylos.ai/research/2026-03-17-dynamic-context-assembly-projection-llm-agent-runtimes) — retrieved 2026-05-26

Last verified: 2026-05-26
