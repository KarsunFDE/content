---
week: W02
day: Mon
topic_slug: langchain-v1-posture-and-adr-discipline
topic_title: "LangChain v1.0 posture + ADR discipline — the two non-negotiable Mon ADRs"
parent_overview: W02/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://docs.langchain.com/oss/python/migrate/langchain-v1
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://blog.langchain.com/langchain-langgraph-1dot0/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://github.com/joelparkerhenderson/architecture-decision-record/blob/main/locales/en/templates/decision-record-template-by-michael-nygard/index.md
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://adr.github.io/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.infoq.com/articles/enterprise-spec-driven-development/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# LangChain v1.0 posture + ADR discipline — the two non-negotiable Mon ADRs

## 1. Learning Objectives

By the end of this reading, the learner can:

- Describe the **three architectural choices LangChain v1.0** made that break code written against pre-v1.0 tutorials (Chain class removal, LCEL de-emphasis, the `create_agent` entry point).
- Compose a small RAG pipeline in v1.0-compatible style **without** using the deprecated `Chain` class or LCEL pipe syntax.
- Write a Michael-Nygard-style **Architecture Decision Record** with the five canonical sections (Title, Status, Context, Decision, Consequences) and a sixth "Alternatives Considered" section.
- Explain why ADRs survive across iterations — what makes a decision **irreversible-ish enough** to deserve one, and how the spec-driven-development practice keeps ADRs honest over time.

## 2. Introduction

Two ADRs every RAG team commits on the Monday of week 2: **how we compose** (orchestration posture) and **how we capture decisions** (the ADR discipline itself). Both are load-bearing for the rest of the programme.

LangChain v1.0 (GA October 2025) made three deliberate architectural choices that bite anyone learning from older tutorials. If the tutorial you're reading shows `RetrievalQA.from_chain_type(...).run(...)` or positions the `|` pipe operator as the foundational composition pattern, it is teaching pre-v1.0 patterns. Adopting those in a v1.0+ project creates architectural drift flagged in code review at the latest, in production at the worst.

Architecture Decision Records (ADRs) are the discipline that keeps these choices honest. Michael Nygard's 2011 template (retrieved 2026-05-26) remains the canonical format. Spec-driven development — the 2026 wave of formalising the plan-spec-implement loop as a continuous discipline (InfoQ enterprise SDD, retrieved 2026-05-26) — depends on ADRs as the bridge between the high-level spec and the low-level code.

## 3. Core Concepts

### 3.1 What changed in LangChain v1.0

The LangChain v1.0 migration guide (retrieved 2026-05-26) and LangChain's GA announcement (blog.langchain.com, retrieved 2026-05-26) both call out three deliberate changes:

**The `Chain` class is removed from the core package.** Pre-v1.0, the canonical pattern was to subclass `Chain` (e.g., `LLMChain`, `RetrievalQA`, `SequentialChain`) and call `.run()` on it. In v1.0, this hierarchy is moved to a separate `langchain-classic` compatibility package and the main `langchain` package no longer ships it. Any tutorial that builds a `Chain` subclass or calls `.run()` is teaching pre-v1.0 patterns.

> [!instructor-review]
> The blocklist entry `langchain-chain-class` (in `~/fde-10-week/skills/tech-research/references/known-bad-patterns.yml`, last reviewed 2026-05-12) flags any source advocating the `Chain` class as an abstraction. If you encounter one in your own research this week, do not silently adopt; reference v1.0 docs for current composition patterns and flag the source for instructor review.

**LCEL pipe syntax is de-emphasised as "the foundation."** The `|` pipe operator still exists, but the v1.0 docs no longer centre it as the primary composition pattern. The recommended pattern in v1.0 is **plain Python** — assign-to-variables or nested function calls — using LangChain primitives at each step. Sources written in 2024 or early 2025 that present LCEL as "the standard way to compose chains" are stale relative to current v1.0 guidance.

> [!instructor-review]
> The blocklist entries `langchain-lcel-fundamental`, `langchain-lcel-pipe`, and `langchain-pre-v1-advocacy` (last reviewed 2026-05-12) flag any source positioning LCEL or the `|` operator as fundamental, foundational, or the primary composition pattern. The web search results returned during research for this reading included multiple sources still advocating LCEL pipe syntax for v1.x RAG — those sources were not adopted. Plain Python composition is used in §4 instead.

**`create_agent` is the canonical agent entry point.** For agent workloads (W3 territory), the v1.0 idiom is `from langchain.agents import create_agent` — a function that takes a model, tools, and optional middleware and returns a compiled LangGraph-backed agent. This collapses what used to be multiple `AgentExecutor` / `Chain` configurations into a single high-level abstraction. For pure RAG (this week), you don't need it — pure RAG is a retrieval call + a model call, not an agent.

The deeper architectural shift: **LangGraph is now the runtime substrate** under LangChain v1.0. The `langchain` package focuses on agents, models, messages, and tools; LangGraph provides the durable, stateful, orchestrated execution underneath. You don't have to touch LangGraph for week 2's pure-RAG work, but the framing matters when you do reach week 3.

### 3.2 v1.0-compatible composition without `Chain` or LCEL pipes

What v1.0 RAG composition actually looks like, in shape:

```python
# v1.0-style RAG composition — plain Python, no Chain class, no LCEL pipes.

def answer_question(question, retriever, model, system_prompt):
    """Compose retrieve → prompt → generate as plain Python function calls."""
    # 1. Retrieve.
    retrieved_chunks = retriever.invoke(question)

    # 2. Assemble the prompt as a list of messages — explicit, inspectable.
    context = format_chunks_for_prompt(retrieved_chunks)
    messages = [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": f"Context:\n{context}\n\nQuestion: {question}"},
    ]

    # 3. Generate.
    response = model.invoke(messages)

    # 4. Return the response alongside the chunks that grounded it — citation surface.
    return {
        "answer": response.content,
        "citations": [c.metadata for c in retrieved_chunks],
    }
```

Four properties v1.0 docs favour: every step is a plain Python function call (debuggable, stack traces point at your own code); inputs and outputs are explicit (no hidden state); composition is just function composition (add a re-ranker = add a function call); refactoring is easy (wrap or decorate the function, not the framework).

Trade-off: you give up the small ergonomic benefit of pipe syntax in exchange for transparency, debuggability, and forward-compatibility with v1.0+ docs.

### 3.3 The Michael Nygard ADR template

An ADR is a short, durable record of a design decision made at a specific moment, for specific reasons, with specific known trade-offs. Five canonical sections:

1. **Title** — short, imperative, like a git commit subject ("Use MongoDB Atlas Vector Search as the RAG store"). Under 50 characters when possible.
2. **Status** — Proposed / Accepted / Rejected / Deprecated / Superseded. A live status field keeps the ADR an active artefact.
3. **Context** — what is motivating this decision? Two to four short paragraphs: constraints, prior state, forces in tension.
4. **Decision** — what we will do, in active voice. Specific enough that code review can check compliance.
5. **Consequences** — positive (capabilities unlocked) and negative (debt incurred). Honest. Future-you reads these to understand why current-you made a choice that looks weird in hindsight.

A widely-adopted sixth section in modern practice: **Alternatives Considered** — what was on the table, why rejected. Makes ADRs truly durable through team turnover.

### 3.4 What makes a decision deserve an ADR

Bar: **could a future engineer plausibly want to know WHY, and would it cost them effort to reconstruct?**

ADR-worthy: choice of vector store, embedding model, orchestration framework, multi-tenancy boundary, refusal-on-low-confidence vs always-answer, cloud provider for managed AI services.

Not ADR-worthy (belongs in code comments or PR descriptions): variable naming, single-run hyperparameter choices, library version pins within a major version.

Heuristic: if reversing would require bringing three people into a meeting, write an ADR.

### 3.5 Spec-driven dev — the discipline ADRs serve

ADRs bridge a high-level specification (what the system does and why) and the implementation. The 2026 wave of **spec-driven development** (InfoQ SDD, retrieved 2026-05-26) formalises this loop into four phases practised iteratively: **specify → plan → task → implement**, with a retrospective closing each iteration. The discipline is mature, not novel — it has roots in formal methods, BDD, and design-first API workflows.

In this programme: every Monday is a Plan Day with a written plan-spec; every subsequent Plan Day opens with a §0 retrospective on the previous week's plan-spec (per D-036). ADRs sit inside plan-specs as the durable record of irreversible-ish decisions. By W6 you have six plan-specs and a cumulative ADR set a stakeholder can read end-to-end.

The week-2 Mon ADR is Day 1 of practising the discipline. The §0 retrospective is deferred this Monday (it formally starts W3 Mon, because W1 Fri was not a multi-day plan-spec worth retro-ing).

## 4. Generic Implementation

A v1.0-compatible RAG composition for a generic technical-docs Q&A (outside federal acquisitions), plus the ADR that grounds it.

```python
# v1.0-style — no Chain class, no LCEL pipes, no .run().

SIMILARITY_FLOOR = 0.62
SYSTEM_PROMPT = """Answer ONLY using the provided context.
If context is insufficient, respond: "I don't have enough evidence to answer."
For every factual claim, cite the source document id at end of sentence."""

def retrieve(question, store, k=5, score_floor=SIMILARITY_FLOOR):
    hits = store.similarity_search_with_score(question, k=k * 4)
    return [hit for hit, score in hits if score >= score_floor][:k]

def build_messages(question, chunks):
    context = "\n\n---\n\n".join(
        f"[doc_id={c.metadata['doc_id']}] {c.page_content}" for c in chunks
    )
    return [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": f"Context:\n{context}\n\nQuestion: {question}"},
    ]

def answer_question(question, store, model):
    chunks = retrieve(question, store)
    if not chunks:
        return {"answer": "I don't have enough evidence to answer.", "citations": []}
    response = model.invoke(build_messages(question, chunks))
    return {"answer": response.content, "citations": [c.metadata for c in chunks]}
```

And a Nygard-style ADR grounding it:

```markdown
# ADR-003: Compose RAG pipeline as plain Python functions
## Status: Accepted — 2026-05-26
## Context
Adopting LangChain v1.0 as orchestration layer. v1.0 removed `Chain` class
and de-emphasised LCEL pipe syntax. Composition style picked today determines
how every subsequent component (re-rankers, post-validators, agent layers) is
written and reviewed.
## Decision
Compose RAG as plain Python function calls using v1.0 primitives. No `Chain`
subclass, no `|` pipe, no `.run()`. Each step is an ordinary function with
explicit inputs and outputs.
## Consequences
Positive: debuggable, inspectable, forward-compatible with v1.0+ docs, no
framework DSL for new engineers to learn. Negative: ~10% more verbose than
LCEL; some 2024 tutorials require translation.
## Alternatives Considered
- LCEL pipes — rejected: v1.0 docs no longer position as foundational; blocklist flags.
- `langchain-classic` package — rejected: ongoing migration debt.
- Custom DSL — rejected: complexity premium unwarranted.
```

Three properties this ADR has: the Decision is specific and falsifiable (code review can check compliance); the Consequences are honest about negatives; the Alternatives section names what was rejected and why.

## 5. Real-world Patterns

**Healthcare — clinical-trial matching.** A 2024 patient-matching system at a biotech adopted plain-Python composition after a code review revealed LCEL pipe chains had become unmaintainable: every chain was subtly different, no two engineers wrote them the same way, and debugging required understanding LangChain internals. The team's retrospective ADR explicitly rejected the pipe pattern and standardised on explicit function-call composition. Code-review cycle time dropped meaningfully.

**Fintech — fraud-narrative drafting.** A bank's fraud-investigation assistant migrated from pre-v1.0 `RetrievalQA` to v1.0 plain-Python composition in Q1 2026. The ADR captured the rationale honestly: positive (regulator-friendly traceability, since every retrieval and generation call is logged at the application layer rather than buried in framework internals); negative (a two-week migration project they hadn't budgeted for). Six months later the ADR remains the canonical reference for why the codebase doesn't look like the LangChain tutorials new engineers trained on.

**E-commerce — product-question answering.** A consumer-electronics retailer kept an ADR catalogue of all retrieval and generation decisions across two years. When a new VP proposed switching orchestration frameworks entirely, the ADR catalogue was the artefact the team used to demonstrate the existing choices were grounded — and to identify which specific ADRs were stale enough to warrant supersession. The ADRs didn't prevent the change; they made it accountable.

**Logistics — routing-explanation assistant.** A regional carrier practised spec-driven dev with weekly Plan Days. Every Monday opened with a §0 retrospective on the previous week's plan-spec; ADRs from the prior week were either reaffirmed, superseded, or marked stale. After six iterations the team had a 14-ADR catalogue any new engineer could read in 90 minutes to understand the design space. New-hire ramp time dropped from ~6 weeks to ~3.

## 6. Best Practices

- **Adopt v1.0 idioms now**, even when pre-v1.0 tutorials are more abundant. Migration cost grows with codebase size.
- **Compose RAG as plain Python** — no `Chain`, no LCEL pipes, no `.run()`. Explicit functions with explicit inputs and outputs.
- **Use `create_agent` only when you actually need an agent.** Pure RAG is just retrieve + model.
- **Write ADRs for irreversible-ish decisions** — expensive to reverse, non-obvious to future engineers.
- **Always include Alternatives Considered.** Makes the decision durable through team turnover.
- **Keep ADR status live.** Mark superseded ADRs explicitly with a link to the replacement. Stale ADRs are worse than no ADRs.
- **Open every Plan Day with a §0 retrospective on the previous spec** — the spec-driven-dev loop.
- **Verify model identifiers via `/web-research` against the provider's catalogue** before committing. Bedrock and other model IDs change; outdated IDs cause silent production failures.

## 7. Hands-on Exercise

**Time: 12–15 minutes — combined whiteboarding + writing exercise.**

Part 1 — composition (5 min). Sketch the v1.0-style code for a small RAG pipeline that:

1. Retrieves the top-3 chunks from a vector store.
2. Filters out chunks below a similarity score of 0.6.
3. If no chunks survive the filter, returns a refusal message.
4. Otherwise, builds a system + user message pair and calls a generation model.

You may write Python or pseudocode. The rule: **no `Chain` class, no `|` pipe operator, no `.run()` calls**.

Part 2 — ADR (10 min). Write a Nygard-style ADR (Title, Status, Context, Decision, Consequences, Alternatives Considered) documenting your choice to refuse on low-confidence retrieval (step 3 above). The ADR should be specific enough that a code reviewer six months from now can tell whether the code complies.

**What good looks like:** Your composition is < 20 lines and reads as plain Python — a non-LangChain developer should be able to follow it. Your ADR's Decision section names a specific similarity-score floor (or names the parameter that holds it), not an aspiration. Your Consequences section honestly states the negative (e.g., "users will sometimes see refusals on questions the system could plausibly have answered"). Your Alternatives Considered section names at least one rejected option (e.g., "always answer with a low-confidence disclaimer") and why it was rejected.

## 8. Key Takeaways

- Can you name the three architectural choices LangChain v1.0 made (Chain removal, LCEL de-emphasis, `create_agent` as the agent entry point) and explain why each one breaks pre-v1.0 tutorials?
- Can you compose a small RAG pipeline in v1.0 style without `Chain` subclasses, LCEL pipes, or `.run()`?
- Can you write a Michael-Nygard-style ADR with the five canonical sections plus Alternatives Considered, and explain what makes a decision deserve an ADR at all?
- Can you explain why **spec-driven dev** is a living discipline that keeps ADRs honest over iteration, and how the weekly Plan Day §0 retrospective enforces that?
- Can you verify a model identifier (e.g., a Bedrock Claude model ID) against the current provider catalogue before committing it to a Mon ADR?

## Sources

1. [LangChain v1 migration guide — LangChain Docs](https://docs.langchain.com/oss/python/migrate/langchain-v1) — retrieved 2026-05-26
2. [LangChain and LangGraph Agent Frameworks Reach v1.0 Milestones — LangChain Blog](https://blog.langchain.com/langchain-langgraph-1dot0/) — retrieved 2026-05-26
3. [Architecture Decision Record template by Michael Nygard — joelparkerhenderson/architecture-decision-record](https://github.com/joelparkerhenderson/architecture-decision-record/blob/main/locales/en/templates/decision-record-template-by-michael-nygard/index.md) — retrieved 2026-05-26
4. [Architectural Decision Records (ADRs) — adr.github.io](https://adr.github.io/) — retrieved 2026-05-26
5. [Spec-Driven Development — Adoption at Enterprise Scale — InfoQ](https://www.infoq.com/articles/enterprise-spec-driven-development/) — retrieved 2026-05-26

Last verified: 2026-05-26
