---
template: research-brief
tech: LangChain v1.0
version_pinned: 1.3.0
last_verified: 2026-05-22
last_verified_via: WebSearch+WebFetch (fallback; web-research unavailable)
recency_window: hot-tech 3mo
sources_count: 4
target_weeks: [W01-Thu-Fri, W02]
candidates_deferred: []
known_bad_patterns_flagged:
  - id: langchain-lcel-fundamental
    note: "Many third-party tutorials still advocate LCEL/`|` pipe as 'the foundational way' — these are pre-v1.0 stale. v1.0 docs centre on `create_agent`, not LCEL. Brief explicitly does not promote LCEL-as-foundation."
  - id: langchain-chain-class
    note: "Pre-v1.0 `Chain` class and `chain.run()` are removed/moved to `langchain-classic`. Brief frames migration as 'replace chain.run() with create_agent + middleware or plain Python composition'."
known_bad_patterns_checked: true
author: research-subagent
---

# Tech research brief — LangChain v1.0

Last verified: 2026-05-22 · Recency window applied: hot-tech 3mo · Pinned version: 1.3.0

## 1. What it is (3–5 sentences, no jargon)

LangChain is a Python (and TypeScript) framework for building LLM-powered applications, with a v1.0 stable release that retargeted the library around agent construction rather than the v0.x Chain/LCEL composition primitives. v1.0 ships `create_agent` as the canonical entry point — a single function that builds an agent on top of the LangGraph runtime, with a middleware system for human-in-the-loop, summarization, and PII redaction. Legacy chain abstractions (`Chain` class, `chain.run()`, much of LCEL-as-foundation framing) moved to the separate `langchain-classic` package for backwards compatibility. The framework now commits to no breaking changes until 2.0. We touch LangChain in W1 (introduction + first agent) and W2 (RAG agent composition).

## 2. Current stable state (as of `last_verified`)

- Latest stable release: **langchain v1.3.0 (2026-05-12)**, alongside langgraph v1.2.0 and deepagents v0.6.0. ([changelog](https://changelog.langchain.com/))
- Active major version line: v1.x (v1.0 GA on 2025-10-22; "no breaking changes until 2.0" commitment).
- Breaking changes in the last 3 months: minor-version cadence has been additive (v1.1 → v1.3.0 over Feb–May 2026). The bigger break — `Chain` class removal, `langchain-classic` move — landed at 1.0 GA in Oct 2025 and remains the migration target for any v0.x code.
- Known-bad-pattern check: **flagged** — third-party tutorials still teach LCEL `|` pipe as foundational. v1.0 docs do not. See frontmatter for which KBP ids triggered.

## 3. What we teach (and what we deliberately don't)

- **In scope (W1–W2):** `create_agent` as the canonical agent entry point; middleware (HITL, summarization, PII); LangGraph runtime as the substrate; plain-Python composition for sequential steps (`result = parser(model(prompt(x)))` or assign-to-variables) rather than `|` pipes.
- **Out of scope:** `langchain.chains.*`, `Chain` subclassing, `chain.run()`, LCEL `|` operator as the "foundation". These appear only in a single migration callout slide ("if you see this in old code, here's the v1.0 equivalent").
- **Misconceptions to pre-empt:** (a) "LCEL and Runnables are the core of LangChain." — false for v1.0; they're available but not centred. (b) "chain.run() is the standard call pattern." — removed; use `create_agent(...).invoke(...)`. (c) "LangChain v1.0 is just a rename of v0.3." — false; it's a deliberate architectural pivot to agent-first.

## 4. Recommended primary sources

- [LangChain v1 overview](https://docs.langchain.com/oss/python/releases/langchain-v1) — accessed 2026-05-22. Canonical v1.0 framing: `create_agent`, content blocks, namespace cleanup.
- [LangChain 1.0 now generally available (changelog announcement)](https://changelog.langchain.com/announcements/langchain-1-0-now-generally-available) — published 2025-10-22, accessed 2026-05-22. Authoritative GA date and feature set.
- [LangChain changelog](https://changelog.langchain.com/) — accessed 2026-05-22. Source of v1.3.0 / v1.2.0 / deepagents 0.6.0 dates (2026-05-12).
- [LangChain + LangGraph 1.0 milestones blog](https://blog.langchain.com/langchain-langgraph-1dot0/) — accessed 2026-05-22. Companion explainer for the agent-first pivot and middleware design.

## 5. Code reference snippets (idiomatic, current API)

```python
# v1.0 idiomatic — create_agent + middleware
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware

agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[search_tool, retrieve_tool],
    middleware=[HumanInTheLoopMiddleware(tools=["search_tool"])],
)

result = agent.invoke({"messages": [{"role": "user", "content": "..."}]})
```

```python
# v1.0 idiomatic sequential composition — plain Python, NOT `|` pipes
prompt_value = prompt.invoke({"question": q})
model_message = model.invoke(prompt_value)
parsed = parser.invoke(model_message)
# (or just compose as nested calls; this is normal Python — no framework abstraction needed)
```

Anti-pattern (do NOT teach as current):

```python
# v0.x — NOT current. Shown only in the migration slide.
chain = prompt | model | parser     # LCEL pipe — not centred in v1.0
result = chain.run(question=q)      # `Chain.run` removed; lives in langchain-classic
```

## 6. Risks and watch-items

- v1.x is on a fast minor cadence (v1.0 Oct → v1.3 May = ~6 months for 3 minor bumps). W2 RAG content may need a Sunday-before re-verify if v1.4 ships between W1 and W2.
- `langchain-classic` is a back-compat parachute, not a long-term target. Don't teach it as a path forward.
- LangGraph 1.x is the underlying runtime. Pin both `langchain` and `langgraph` minor versions in cohort requirements.txt.

## 7. Alternatives the cohort should be aware of

- LlamaIndex (RAG-focused alternative) — surfaced as comparison in W2 scenario-alternatives.
- Direct Bedrock / Anthropic SDK calls without an agent framework — useful baseline for "do we need LangChain at all?" discussion.

## 8. Brief sign-off

- Drafted by: research-subagent (2026-05-22)
- Reviewed against `known-bad-patterns.yml`: **flagged** — see frontmatter; brief explicitly counter-teaches the flagged patterns.
- 1-month-release check: triggered (v1.3.0 landed 2026-05-12, 10 days ago). Resolved by anchoring brief to v1.3.0 and noting that the v1.0 GA architecture (Oct 2025) is the load-bearing fact, not the .x bump.
- Approved for downstream artifact authoring: 2026-05-22

## Sources

- LangChain v1 docs overview, docs.langchain.com, retrieved 2026-05-22 via WebFetch. <https://docs.langchain.com/oss/python/releases/langchain-v1>
- LangChain 1.0 GA announcement, changelog.langchain.com, retrieved 2026-05-22 via WebFetch. <https://changelog.langchain.com/announcements/langchain-1-0-now-generally-available>
- LangChain changelog, changelog.langchain.com, retrieved 2026-05-22 via WebFetch. <https://changelog.langchain.com/>
- LangChain + LangGraph 1.0 milestones, blog.langchain.com, retrieved 2026-05-22 via WebSearch. <https://blog.langchain.com/langchain-langgraph-1dot0/>
