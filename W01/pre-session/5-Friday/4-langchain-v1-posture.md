---
week: W01
day: Fri
topic_slug: langchain-v1-posture
topic_title: "LangChain v1.0 posture — composition without the Chain class or LCEL pipes"
parent_overview: W01/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 13
sources:
  - url: https://docs.langchain.com/oss/python/releases/langchain-v1
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://changelog.langchain.com/announcements/langchain-1-0-now-generally-available
    retrieved_on: 2026-05-22
    recency_category: hot-tech
  - url: https://changelog.langchain.com/
    retrieved_on: 2026-05-22
    recency_category: hot-tech
  - url: https://blog.langchain.com/langchain-langgraph-1dot0/
    retrieved_on: 2026-05-22
    recency_category: hot-tech
last_verified: 2026-05-26
---

# LangChain v1.0 posture — composition without the Chain class or LCEL pipes

> The overview frames this against the training-project's deliberately-pinned `langchain==0.1.x` brownfield-debt item that the cohort migrates in W2. This reading is the generic case: what v1.0 actually changed, what stale patterns still circulate on the internet, and how to compose LLM calls in v1.0 without falling back to v0.x abstractions.

> [!instructor-review]
> This topic must stay aligned with the `known-bad-patterns.yml` blocklist (entries `langchain-chain-class`, `langchain-chaining-verb`, `langchain-lcel-pipe`, `langchain-lcel-fundamental`, `langchain-runnable-protocol-core`, `langchain-pre-v1-advocacy`). All code snippets and prose in this reading are written against those constraints — no `Chain` class advocacy, no `|` pipe foundation framing, no `chain.run()`. The W2 migration target pin is `langchain v1.3.x`; confirm with cohort lead before sprint kickoff that 1.3 is still the current stable.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the three structural changes from LangChain v0.x to v1.0 and where in the code they show up.
- Compose a sequential LLM-tool workflow in v1.0 using plain Python function calls — without `Chain.run()` and without the LCEL `|` pipe operator as the foundation.
- Identify the canonical v1.0 agent-construction entry point and the middleware hooks it exposes.
- Spot stale tutorials (pre-2025-Q3) by their advocacy of patterns that v1.0 has deliberately moved away from.

## 2. Introduction

LangChain v1.0 went generally available on 2025-10-22 [LangChain changelog, 2025]. The release was not a routine version bump — it was a deliberate architectural pivot. The framework that had grown around the `Chain` class, LCEL (LangChain Expression Language) `|` pipe operators, and the Runnable protocol re-centred itself on a single entry point — `create_agent` — running on the LangGraph runtime, with a middleware system for cross-cutting concerns like HITL, summarization, and PII redaction.

The pivot matters because the *internet's* corpus of LangChain content lags reality by 12–18 months. The majority of third-party tutorials, blog posts, and Stack Overflow answers still teach v0.x patterns — `chain.run()`, `prompt | model | parser`, `RetrievalQA.from_chain_type`. Most of those patterns either no longer work in v1.0 or work only through the `langchain-classic` backwards-compatibility package. A learner who Googles "how to build a RAG chain in LangChain" without filtering for recency will land on advice that is wrong-by-construction for v1.0.

This reading is the inoculation. It names the three patterns that have changed, shows the v1.0 equivalents, and gives heuristics for filtering stale sources. The latest stable as of authoring is **langchain v1.3.0** (released 2026-05-12 [LangChain changelog]), and the framework has committed to no breaking changes until 2.0.

## 3. Core Concepts

### 3.1 Three structural changes

**(a) `Chain` class removed from the main package.** Anywhere a v0.x tutorial says "subclass `Chain`," "build a Chain abstraction," or "use the Chain class" — that pattern lives in the separate `langchain-classic` package now [LangChain v1 release notes, 2025]. The split is deliberate: v1.0 wants composition to be plain Python, not framework-magic. Subclassing a base class to express "first do A, then do B" was always more ceremony than the problem demanded.

**(b) LCEL `|` pipe operator deprecated as the *foundation*.** LCEL still exists; you can still see `prompt | model | parser` in some examples. But the framework no longer markets the `|` operator as the central composition mechanism. The v1 release notes barely mention it. The Runnable protocol that LCEL was built on has likewise been demoted from "core abstraction" to "implementation detail that some primitives expose." Sources that frame the `|` operator as "the way you compose in LangChain" are stale [LangChain v1 docs, 2026].

**(c) `create_agent` is the canonical entry point.** The v1.0 docs centre on a single function — `create_agent(model=..., tools=..., middleware=...)` — that returns an agent built on the LangGraph runtime. It replaces `langgraph.prebuilt.create_react_agent` and is the documented starting point for almost every example [LangChain v1 release notes, 2025].

### 3.2 How you compose in v1.0 (it is just Python)

Sequential composition in v1.0 is plain function calls. Assign to variables. No framework abstraction:

```python
# v1.0 sequential composition — plain Python
from langchain_anthropic import ChatAnthropic
from langchain.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a research assistant."),
    ("user", "{question}"),
])
model = ChatAnthropic(model="claude-sonnet-4-6")
parser = StrOutputParser()

# Compose with normal function calls, not a Chain class, not |
prompt_value = prompt.invoke({"question": "What is RAG?"})
model_message = model.invoke(prompt_value)
answer = parser.invoke(model_message)
```

This pattern is intentionally boring. There is no `Chain` to subclass, no `|` to read right-to-left, no `chain.run()` to call. Each step is a plain function call against a primitive. Anything more elaborate than this is either an *agent* (use `create_agent`) or a *graph* (use LangGraph directly).

### 3.3 `create_agent` and middleware

For anything more than a single sequential pass, v1.0 wants you to use `create_agent`:

```python
from langchain.agents import create_agent
from langchain.agents.middleware import (
    HumanInTheLoopMiddleware,
    PIIMiddleware,
    SummarizationMiddleware,
)

agent = create_agent(
    model="claude-sonnet-4-6",
    tools=[search_corpus, fetch_document],
    system_prompt="You answer questions from the provided corpus.",
    middleware=[
        PIIMiddleware("email", strategy="redact", apply_to_input=True),
        SummarizationMiddleware(
            model="claude-sonnet-4-6",
            trigger={"tokens": 500},
        ),
        HumanInTheLoopMiddleware(
            interrupt_on={
                "fetch_document": {"allowed_decisions": ["approve", "edit", "reject"]},
            },
        ),
    ],
)

result = agent.invoke({"messages": [{"role": "user", "content": "..."}]})
```

The middleware system is the load-bearing v1.0 feature. PII redaction, HITL approval, summarization for long conversations, and dynamic prompt construction all live as composable middleware on the agent — not as bespoke code inside whatever `chain.run()` used to wrap [LangChain v1 release notes, 2025].

### 3.4 What about RAG specifically?

A v1.0-idiomatic RAG flow looks like this — no `RetrievalQA`, no `|` pipes:

```python
# v1.0 RAG — sequential, just function calls
def answer(question: str) -> str:
    query_embedding = embedder.embed_query(question)
    chunks = vector_store.similarity_search_by_vector(
        embedding=query_embedding,
        k=8,
        filter={"tenant_id": current_tenant_id()},  # multi-tenant pre-filter
    )
    context = "\n\n".join(c.page_content for c in chunks)
    prompt = build_prompt(question=question, context=context)
    response = model.invoke(prompt)
    return response.content
```

That is the whole pattern. Five lines of orchestration, no framework abstractions in the orchestration layer itself. The primitives (`embedder`, `vector_store`, `model`) come from LangChain integrations; the *flow between them* is plain Python.

### 3.5 Spotting stale sources

Heuristics for filtering out v0.x advice when you search:

- **Date filter** — pre-2025-Q3 LangChain content is suspect by default.
- **Vocabulary tells** — words like "chaining," "the Runnable protocol," "build a Chain" almost always indicate v0.x framing [known-bad-patterns.yml entries `langchain-chaining-verb`, `langchain-runnable-protocol-core`, `langchain-chain-class`].
- **Import tells** — `from langchain.chains import ...` is the v0.x package layout; `from langchain.agents import create_agent` is v1.0.
- **Method tells** — `.run()` is v0.x; `.invoke()` is v1.0.
- **Operator tells** — `prompt | model | parser` as the *primary* example is v0.x; in v1.0 this pattern still works but the docs do not lead with it.

When in doubt: check the v1 docs at `docs.langchain.com/oss/python/releases/langchain-v1` and the release date of the source you are reading.

## 4. Generic Implementation

Worked example outside federal acquisitions — a fintech compliance assistant that answers analyst questions about internal trade-surveillance policies. The corpus is a few hundred policy documents; the assistant must always answer from the corpus, never freelance.

```python
# Fintech compliance assistant — v1.0 idiomatic
from langchain.agents import create_agent
from langchain.agents.middleware import (
    PIIMiddleware,
    HumanInTheLoopMiddleware,
)
from langchain_core.tools import tool


@tool
def search_policies(query: str) -> list[dict]:
    """Search internal policy corpus for relevant excerpts."""
    embedding = embedder.embed_query(query)
    chunks = vector_store.similarity_search_by_vector(embedding, k=6)
    return [{"id": c.id, "excerpt": c.page_content} for c in chunks]


@tool
def escalate_to_compliance_officer(question: str, answer: str) -> dict:
    """Hand off ambiguous compliance questions to a human reviewer."""
    return ticketing.create(
        category="compliance-review",
        question=question,
        draft_answer=answer,
    )


assistant = create_agent(
    model="claude-sonnet-4-6",
    tools=[search_policies, escalate_to_compliance_officer],
    system_prompt=(
        "You answer compliance questions ONLY from the search_policies tool. "
        "If the search returns nothing relevant, call escalate_to_compliance_officer. "
        "Never freelance regulatory advice."
    ),
    middleware=[
        PIIMiddleware("ssn", strategy="block"),  # hard-stop on SSN in input
        HumanInTheLoopMiddleware(
            interrupt_on={
                "escalate_to_compliance_officer": {
                    "allowed_decisions": ["approve", "reject"],
                },
            },
        ),
    ],
)

response = assistant.invoke({
    "messages": [{"role": "user", "content": "Can a trader use a personal account for hedging?"}]
})
```

Notice what is *not* in the code: no `Chain` subclass, no `prompt | model | parser`, no `chain.run()`. The orchestration is the agent loop (built into `create_agent`), the cross-cutting concerns are middleware, and the domain logic is in plain Python tool functions.

## 5. Real-world Patterns

**Notion AI assistant.** Notion's AI features are built on a pattern very close to the v1.0 idiom — an agent loop with custom middleware for citation insertion and permission-scoped retrieval. The middleware shape lets them add new cross-cutting features (e.g., a new audit-log middleware) without rewriting the orchestration.

**Replit Agent.** Replit's coding agent uses the agent-loop + middleware pattern for HITL on destructive operations (file deletes, package installs). The middleware approach means the HITL policy is one declarative block, not scattered branching code inside the main loop.

**LinkedIn — AI-powered hiring assistant (2025 launches).** LinkedIn's recruiter-facing AI features adopted the `create_agent` pattern for the same reason: composable middleware (rate-limiting, fairness audits, PII handling on candidate data) is easier to govern than custom orchestration code [LangChain blog, 2025].

**Customer support tools (Intercom, Zendesk in-product copilots).** The shift from v0.x `Chain`-based bots to `create_agent`-based agents has been a multi-quarter migration across the customer-support tooling space. The v1.0 middleware story makes things like "escalate to human after N failed retrievals" a 5-line policy instead of a 50-line orchestration patch.

## 6. Best Practices

- **Use `create_agent` for anything more elaborate than a single LLM call** — it is the documented v1.0 entry point and the path that gets future improvements.
- **Compose sequential steps with plain Python**, not `|` pipes or a `Chain` subclass. Each step is a function call. Assign to variables. Be boring.
- **Put cross-cutting concerns in middleware** — HITL, PII, summarization, rate limiting, audit logging. The middleware system is designed for this; bespoke orchestration code is not.
- **Pin the framework version**. v1.x has a "no breaking changes until 2.0" commitment, but minor versions still add features. Pin to a known-good 1.x to avoid surprise during sprints.
- **Filter your reading by date**. Pre-2025-Q3 LangChain content is misleading more often than not. The v1 release notes and migration guide on `docs.langchain.com` are the source of truth.
- **When you must touch legacy code**, the migration path is "replace `chain.run()` with `agent.invoke()` and pull cross-cutting concerns into middleware" — not "rewrite from scratch." Most v0.x apps map cleanly into v1.0 idioms.

## 7. Hands-on Exercise

**Coding exercise (15 minutes).** You are handed a working but pre-v1.0 LangChain snippet:

```python
# Legacy (v0.x) code you've been asked to modernize
from langchain.chains import LLMChain
from langchain.prompts import PromptTemplate
from langchain.llms import OpenAI

template = "Summarize this customer ticket: {ticket}"
prompt = PromptTemplate(input_variables=["ticket"], template=template)
chain = LLMChain(llm=OpenAI(), prompt=prompt)
result = chain.run(ticket=open("ticket.txt").read())
```

Rewrite this into v1.0-idiomatic code. Constraints:

1. No `LLMChain`, no `chain.run()`, no `|` pipes.
2. Use `.invoke()` on the primitives.
3. Keep the same observable behavior.

Bonus: add a single middleware that redacts email addresses from the ticket before the model sees it.

**What good looks like.** The rewrite uses `model.invoke(prompt.invoke({"ticket": ...}))` (or a two-line variable-assignment equivalent), drops the `LLMChain` import entirely, and — if attempting the bonus — wraps the call in `create_agent` with `PIIMiddleware("email", strategy="redact", apply_to_input=True)`. The learner should be able to articulate *why* the rewrite is shorter — the framework no longer demands a `Chain` wrapper for a single-step call.

## 8. Key Takeaways

- Can I name the three structural changes from v0.x to v1.0?
- Do I know `create_agent` is the canonical entry point and what middleware hooks exist on it?
- Can I write a sequential LLM workflow in v1.0 without using `Chain`, `chain.run()`, or `|` pipes as the foundation?
- Can I spot a stale tutorial in under 30 seconds by its vocabulary and imports?
- Do I know which patterns moved to `langchain-classic` and that I should treat that package as backwards-compatibility only, not a current API surface?

## Sources

1. [What's new in LangChain v1 — LangChain docs](https://docs.langchain.com/oss/python/releases/langchain-v1) — retrieved 2026-05-26 via /web-research. Canonical v1.0 framing: `create_agent`, middleware hooks, `langchain-classic` split.
2. [LangChain 1.0 now generally available — LangChain changelog](https://changelog.langchain.com/announcements/langchain-1-0-now-generally-available) — retrieved 2026-05-22 via /web-research. Authoritative GA date (2025-10-22) and feature set.
3. [LangChain changelog](https://changelog.langchain.com/) — retrieved 2026-05-22 via /web-research. Source for the v1.3.0 release date (2026-05-12) and the no-breaking-changes-until-2.0 commitment.
4. [LangChain + LangGraph 1.0 milestones — LangChain blog](https://blog.langchain.com/langchain-langgraph-1dot0/) — retrieved 2026-05-22 via /web-research. Companion explainer for the agent-first pivot and middleware design.
5. Known-bad-patterns reference (instructor): `~/fde-10-week/skills/tech-research/references/known-bad-patterns.yml` (entries `langchain-chain-class`, `langchain-chaining-verb`, `langchain-lcel-pipe`, `langchain-lcel-fundamental`, `langchain-runnable-protocol-core`, `langchain-pre-v1-advocacy`).

Last verified: 2026-05-26
