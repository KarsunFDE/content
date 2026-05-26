---
week: W01
day: Fri
title: "Pre-session — RAG Architecture preview + W2 Mon Plan Day prep"
audience: All cohort members
time_on_task_minutes: 57
last_verified: 2026-05-26
fde_situations: [1, 6, 7, 10, 12]
tech: [rag, langchain-v1, mongodb-atlas-vector-search, bedrock, codex-adversarial-review]
---

# W1 Fri Pre-Session — RAG Architecture preview + W2 Mon Plan Day prep

> Read after Fri's first PR Adversarial Review + HITL commit, before W2 Mon's Plan Day. ~57 min. W2 Mon is the first formal Plan Day (per D-029) — the pair authors a `scenario-design-planning.md` artifact before any code lands. **Hard rule:** do not push code to your pair-project over the weekend. Bring questions, not commits.

## 1. Why RAG, not fine-tuning, for federal acquisitions (10 min)

The FAR/DFARS corpus changes regularly (amendments, agency-specific supplements, OIG advisories). Fine-tuning a model means re-training every time a clause changes — impractical and expensive. RAG lets the model cite the *current* corpus at inference time.

**Callback — today's war-room.** This morning's Act 4 closed with the "48 CFR 47.305-2" hallucination: yesterday's solicitation draft cited a paragraph that does not exist in actual FAR, and the CO caught it. Friday's answer was HITL (manage the failure). Monday's answer is RAG (eliminate the failure mode at its source). The W2 Mon Plan Day frames RAG as the answer to *that specific incident*, not as an abstract topic — bring the hallucination case to Monday's whiteboard.

The RAG decision space (each becomes an ADR commit in Monday's plan-spec):
- **Embedding model** — which one captures regulatory language well?
- **Chunking strategy** — clauses, sub-clauses, sentences, paragraphs?
- **Vector store** — Atlas Vector Search (our default), Pinecone, pgvector?
- **Retrieval mode** — dense, sparse (BM25), hybrid?
- **Re-ranking** — on or off? Which model?
- **Citation grounding** — how does the model trace claims to chunks?

## 2. The training-project's Atlas baseline — vector store collocation (8 min)

MongoDB Atlas is **already in the training repo** (you saw it Tue) as a document store for solicitations + evaluations + FAR-clause metadata. W2 *extends* the same Atlas deployment to serve as our vector store via the `$vectorSearch` aggregation stage + an Atlas Vector Search index.

**One database, two access patterns.** This is deliberate — collocation reduces the data-residency surface for the federal context (per D-031). Embeddings live next to tenant data, so multi-tenant pre-filtering happens *inside* the same vector-search query (the `filter` clause on `$vectorSearch`), not as a post-filter that leaks rows then drops them.

**The operational gotcha to know walking in:** `$vectorSearch` MUST be the **first stage** of any aggregation pipeline where it appears. Cannot be used inside `$lookup` sub-pipelines or `$facet` stages. Sources retrieved via /web-research 2026-05-22 (see Sources §). Pinecone + pgvector come up as alternatives in W2 Tue's scenario-alternatives prompt; defending the Atlas default is part of the Monday planning rationale.

## 3. LangChain v1.0 posture — critical (12 min)

W2 Mon's plan-spec **must include an ADR commitment** that the pair uses LangChain v1.0 — and what that means. v1.0 GA shipped 2025-10-22 with a deliberate architectural pivot to *agent-first*: `create_agent` is the canonical entry point on top of the LangGraph runtime, with a middleware system for HITL, summarization, and PII. Latest stable as of /web-research 2026-05-22: v1.3.0 (released 2026-05-12).

What that pivot means concretely — and what the cohort must **not** write:

- **The `Chain` class is removed** and moved to the separate `langchain-classic` package for backwards compatibility only. v0.x code that uses `chain.run()` does not work in v1.0.
- **LCEL `|` pipe syntax is no longer the central composition mechanism.** v0.x composition `prompt | model | parser` is deprecated as the *foundation*; it still works in some places but is no longer the framework's spine.
- **Runnable-as-foundation framing is deprecated.** v1.0 docs centre on `create_agent`, not Runnables.
- **Sequential composition in v1.0 is plain Python function calls** — assign to variables, no framework magic.

The pattern:
```python
# v1.0 (current — correct)
chunks = retriever.invoke(query)
compressed = compressor.invoke(chunks)
reranked = reranker.invoke(compressed)
context = format_for_prompt(reranked)
response = model.invoke(build_prompt(query, context))

# v0.x (deprecated — DO NOT WRITE)
# from langchain.chains import RetrievalQA
# qa = RetrievalQA.from_chain_type(...)
# qa.run(query)
```

When the cohort uses Claude or Google to find LangChain examples, **treat anything older than 2025-Q3 as suspect.** Most third-party tutorials still teach LCEL `|` pipe and `Chain`-class patterns; the v1.0 official docs do not. Verify against the v1.0 docs (linked in Sources). The training-project's `requirements.txt` is pinned to `langchain==0.1.x` *deliberately* — that's item 5 in the brownfield-debt inventory. The cohort migrates it Monday + Tuesday of W2.

> [!instructor-review]
> The pinned `langchain==0.1.x` deliberate-debt item triggers known-bad-pattern ids `langchain-chain-class`, `langchain-chaining-verb`, `langchain-lcel-pipe`, `langchain-pre-v1-advocacy` (per `skills/tech-research/references/known-bad-patterns.yml`). Confirm with cohort lead before W2 Mon that the migration target is `langchain v1.3.0` (current stable), not just "latest v1.x" — pinning to a known-good version reduces surprise during the W2 Mon ADR.

## 4. W2 Mon Plan Day shape + 6 required artifacts (12 min)

W2 Mon is the **first formal Plan Day** in the programme. Plan Days follow a specific shape (per D-029):

- **Morning war-room** — federal-acquisitions RAG problem framed (FAR/DFARS as corpus, contracting-officer drafting as use case, OIG reproducibility as hard NFR). The "48 CFR 47.305-2" hallucination from today is the live exhibit.
- **Practical block** — pair-led understanding + planning with Claude Code as comprehension accelerant. Pair authors a **Scenario Design Planning artifact** (graded; replaces the lighter `week-plan-spec.md` template for graded weeks).
- **Conceptual block** — briefs Tuesday's first implementation steps.
- **§0 Plan retrospective** — required from W3 Mon onward (per D-036). W2 Mon has nothing to retro yet (Fri W1 was Day 1, not a plan-spec). Surface this explicitly so the pair doesn't waste 30 min looking for the retro target.

The Scenario Design Planning artifact has **six required artifacts** (see `templates/scenario-design-planning.md`):

1. Requirements synthesis (1–2 pages)
2. 6R / system map + diagram (1 page)
3. Two-to-three ADRs (1 page each — **must include chunking strategy, embedding model, vector store choice + LangChain v1.0 posture**)
4. Estimate (effort + cost)
5. Risk register (the hallucination case from Friday belongs here)
6. Open questions

The pair has one working day — Monday — to produce all six artifacts. **No code lands Monday.**

## 5. RAG framing — six dimensions to plan against (10 min)

When the pair sits down Monday to author the plan-spec, these are the dimensions to argue through. The table below is the ADR scaffold — copy it into Monday's plan-spec as the first decision-matrix:

| Dimension | W2 Mon ADR question |
|-----------|---------------------|
| **Corpus shape** | What are we indexing? Full clauses, sub-clauses, sentences? Where does the cohort draw the chunk boundary? |
| **Embedding model** | Bedrock Titan, Cohere Embed v3 via Bedrock, Sentence Transformers self-hosted? |
| **Vector store** | Atlas Vector Search (default), Pinecone (alternative), pgvector (alternative)? |
| **Retrieval mode** | Dense, sparse, hybrid (Reciprocal Rank Fusion)? |
| **Citation grounding** | How does the pair trace every claim to a chunk? UI surface? Audit log? |
| **Evaluation pattern** | RAGAS metrics, held-out QA pairs, eval-as-test-fixture in CI? Where do reports live? |

Each row is an ADR-shaped argument: state the default, name the alternatives, defend the choice against the federal-acquisitions context (data residency, audit, hallucination tolerance). Monday's instructor integrity-check at 17:00 reads each ADR for "would this defend itself in front of a CO?"

## 6. Codex Adversarial Review — Ramping tier in W2 (5 min)

Per D-034 calibration ramp, W2 is the **Ramping** tier:

- W1 was **Light** (P0 floor only; P1 findings as coaching).
- W2 ramps **P1 findings back to blocking**.
- W3 is **Near-full**.
- W4 is **Full**.

You'll see findings get more pointed in W2 than W1. The ramp is deliberate — by W4, Codex is acting as a second reviewer at production strictness. Defending or addressing findings is part of W2's plan-spec PR cycle, not a side-task. Block out time Tuesday for a first Codex-review-fix loop on Monday's plan-spec PR.

## What you'll do W2 Mon

Plan Day — no code. Six planning artifacts authored. ADRs committed for chunking + embedding + vector store + LangChain v1.0 posture. Plan-spec integrity check by instructor at 17:00. Bring the "48 CFR 47.305-2" hallucination case + your Friday HITL decision rationale + your top-3 RAG questions.

Tuesday opens with implementing the plan, not with a blank page.

## Sources

- LangChain v1 overview — `https://docs.langchain.com/oss/python/releases/langchain-v1`, retrieved 2026-05-22 via /web-research. Canonical v1.0 framing (`create_agent`, middleware, namespace cleanup).
- LangChain 1.0 GA announcement — `https://changelog.langchain.com/announcements/langchain-1-0-now-generally-available`, retrieved 2026-05-22 via /web-research. Authoritative GA date (2025-10-22) and `Chain`-class removal.
- LangChain changelog (v1.3.0, langgraph v1.2.0) — `https://changelog.langchain.com/`, retrieved 2026-05-22 via /web-research.
- MongoDB Atlas `$vectorSearch` operator reference — retrieved 2026-05-22 via /web-research. First-stage-only rule, `filter` clause for multi-tenant pre-filter.
- Local research briefs (cached, generated via /web-research): `research/langchain-v1-20260522.md`, `research/mongodb-atlas-vector-search-20260522.md`.
- Known-bad-pattern reference: `skills/tech-research/references/known-bad-patterns.yml` (entries `langchain-chain-class`, `langchain-chaining-verb`, `langchain-lcel-pipe`, `langchain-pre-v1-advocacy`, `vector-cosine-default`).

Last verified: 2026-05-26
