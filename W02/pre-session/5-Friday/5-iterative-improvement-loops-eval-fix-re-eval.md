---
week: W02
day: Fri
topic_slug: iterative-improvement-loops-eval-fix-re-eval
topic_title: "Iterative improvement loops — eval → fix → re-eval"
parent_overview: W02/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 8
sources:
  - url: https://www.lancedb.com/blog/rag-isnt-one-size-fits-all
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://datavlab.ai/post/rag-evaluation-methods-metrics-2026-guide
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.firecrawl.dev/blog/best-chunking-strategies-rag
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://neo4j.com/blog/genai/advanced-rag-techniques/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://ragflow.io/docs/configure_child_chunking_strategy
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# Iterative improvement loops — eval → fix → re-eval

## 1. Learning Objectives

- By the end of this reading, the learner can describe the canonical eval → fix → re-eval loop and explain why "change one variable at a time" is non-negotiable.
- By the end of this reading, the learner can map a regressed metric to the system layer most likely responsible (prompt, retrieval, reranker, chunking) and pick a candidate fix.
- By the end of this reading, the learner can articulate the cross-dimension drift problem — fixing one dimension often regresses another — and explain why reporting all dimensions even when only some block is the mitigation.
- By the end of this reading, the learner can describe a "layered fix order" that prioritises the highest-leverage layer first rather than tweaking the easiest layer first.

## 2. Introduction

Once a regression appears in a PR-gated eval, the real work begins. The PR shows that something dropped; the engineer has to identify which layer of the system is responsible, propose a fix, and re-run the eval to confirm. Done casually, this becomes a debugging slog with many round-trips. Done with discipline, it is one or two cycles to land the fix.

The discipline is older than RAG itself — it is the same single-variable-at-a-time discipline that experimental science requires. Change one thing. Measure. Decide. Change the next thing. The temptation in a hot-tech stack is to bundle three changes (new embedding model, new chunking strategy, new prompt) and see if the aggregate score went up. If it did, you have learned nothing about which change actually helped, and the system is now harder to maintain than before.

This reading walks the loop, the metric-to-layer mapping, and the cross-dimension drift problem that makes naïve fix-and-ship dangerous.

## 3. Core Concepts

### 3.1 The loop

The pattern that has settled into 2026 practice:

1. **Eval runs on the PR.** Per-dimension scoreboard lands in the PR comment.
2. **Identify the regressed dimension.** Not "the eval failed" — *which* dimension regressed and by how much.
3. **Map the regression to a system layer** — different layers fix different regressions (Section 3.2).
4. **Make one targeted change** in the layer most likely responsible.
5. **Re-run the eval.** Did the fix land cleanly? Did it regress another dimension?
6. **If clean, merge. If not, revert and try the next candidate fix.**

The "if not, revert" step is the one most teams skip. Once you have made a change and the score moved in the right direction, the temptation is to ship even if a different dimension regressed. Don't. Cross-dimension regression is a real signal that the fix has unintended consequences ([LanceDB — RAG Isn't One-Size-Fits-All](https://www.lancedb.com/blog/rag-isnt-one-size-fits-all), retrieved 2026-05-26).

### 3.2 Metric-to-layer mapping

Different dimensions fail for different reasons. The mapping below is approximate — every system has its own dynamics — but holds across many production RAG stacks:

- **Faithfulness regressed.** Most likely the prompt (the grounding instruction loosened, the system stopped quoting evidence) or the retrieval (the right chunks aren't being retrieved, so the model is filling in from priors). Candidate fixes: tighten the grounding instruction; rerank top-K to push the most-relevant chunks higher; check if the reranker cascade depth changed.
- **Context recall regressed.** Almost always retrieval coverage — the chunks that *should* have been retrieved aren't. Candidate fixes: chunking change (parent-child indexing — see Section 3.4), retrieval mode (add sparse / BM25 leg to a previously dense-only pipeline), or `top-K` increase.
- **Context precision regressed.** Retrieval noise — the system is pulling too many irrelevant chunks. Candidate fixes: reranker change (cross-encoder rerank top-50 → top-5), `top-K` decrease, or query rewriting (the query is too vague).
- **Answer relevance regressed.** Almost always the prompt — the model is answering a slightly different question than the user asked. Almost never retrieval. Candidate fix: clarify what the prompt asks for; add an explicit "if the question is ambiguous, ask a clarifying question" instruction.

The pattern: relevance issues live in the prompt; recall issues live in chunking and retrieval mode; precision issues live in the reranker; faithfulness sits between prompt and retrieval and needs both to be right ([DataVLab — RAG Evaluation 2026](https://datavlab.ai/post/rag-evaluation-methods-metrics-2026-guide), retrieved 2026-05-26).

### 3.3 Cross-dimension drift

The trap: fixing dimension X often regresses dimension Y. Concrete examples that have shown up in real systems:

- **Tightening the grounding prompt** for faithfulness → the model starts refusing questions that have valid but partial answers → answer relevance drops.
- **Switching to smaller chunks** for context precision → the retrieved chunks lose the surrounding context the model needs to answer correctly → faithfulness drops because the chunk truncates the evidence.
- **Adding a BM25 leg** for context recall → introduces noise from keyword matches → context precision drops.
- **Increasing top-K** for context recall → the model gets more context to draw from but also more distraction → answer relevance drops because the model wanders.

This is why the harness reports all four dimensions even when only two block merges. The non-blocking dimensions are the early-warning system for cross-dimension drift. A PR that improves faithfulness 8% and silently degrades answer relevance 4% is not a win — it is a shape-shift, and the eval scoreboard makes the shape visible.

### 3.4 Parent-child indexing

Worth a sub-section because it shows up so often as the fix for context-recall regressions caused by aggressive chunking.

The pattern: index small chunks (e.g., 256 tokens) for retrieval (semantic-similarity matches well on focused content) but pass larger parent chunks (e.g., 1024 tokens — the section the child lives in) to the LLM for generation. The smaller chunks find the right neighborhood; the larger parent gives the model enough context to actually answer. Modern vector stores (RAGFlow 0.23+, Atlas, several others) ship this as a first-class pattern ([RAGFlow — Configure child chunking strategy](https://ragflow.io/docs/configure_child_chunking_strategy), retrieved 2026-05-26; [Neo4j — Advanced RAG Techniques](https://neo4j.com/blog/genai/advanced-rag-techniques/), retrieved 2026-05-26).

This fix has been documented to move recall scores from ~0.53 to ~0.74 in financial RAG systems by avoiding the trap where a single regulatory provision spans two sub-paragraph chunks ([Best Chunking Strategies for RAG — Firecrawl](https://www.firecrawl.dev/blog/best-chunking-strategies-rag), retrieved 2026-05-26).

### 3.5 The layered fix order

When several layers could plausibly fix a regression, the principle is "highest-leverage first." The order that has held up:

1. **Document extraction** (is the source clean? OCR errors? table-extraction failures?).
2. **Chunking** (parent-child? overlap? section-aware boundaries?).
3. **Retrieval** (dense? sparse? hybrid? reranker?).
4. **Generation** (prompt? model? few-shot examples?).

Fixing the prompt first when the underlying chunks are garbled wastes effort — the prompt cannot rescue bad data. Conversely, fixing extraction is often the highest-leverage change in early-stage systems because every downstream layer compounds on top of clean data ([LanceDB — RAG Isn't One-Size-Fits-All](https://www.lancedb.com/blog/rag-isnt-one-size-fits-all), retrieved 2026-05-26).

## 4. Generic Implementation

The loop, expressed as a fixer's decision flowchart rather than code (since the work is investigation, not implementation):

```
1. Eval PR comment says faithfulness -6%, context recall -17%.
   |
   v
2. Diagnose: context-recall regression dominates. Look at the per-row diff.
   - Which rows regressed?
   - Do the regressed rows share a structural pattern (e.g., spanning sections,
     spanning documents, very long answers)?
   |
   v
3. Hypothesis: the new 512-token sliding window split section-spanning answers
   across chunks; the first chunk ranks high, the second (with the missing
   detail) does not.
   |
   v
4. Candidate fix: parent-child indexing. Index 256-token children for retrieval;
   pass 1024-token parents to the LLM.
   |
   v
5. Implement the change on the PR branch.
   |
   v
6. Re-run eval.
   - Did context recall recover? (Expected: yes.)
   - Did any other dimension regress?
     - Context precision? (Possible — parents may be noisier.)
     - Faithfulness? (Should improve — model has more context.)
     - Answer relevance? (Should be unaffected — prompt didn't change.)
   |
   v
7. If clean → merge. If a non-blocking dimension regressed slightly, document
   the trade-off in the PR description. If a blocking dimension regressed,
   revert and try the next candidate.
```

The investigation skill is reading the per-row diff to find the structural pattern. The implementation skill is making *one* change. The discipline skill is reverting when the change introduces a cross-dimension regression.

## 5. Real-world Patterns

**Healthcare — drug-interaction lookup.** A pharmacy-systems team running an interaction-warning agent observed that switching from sentence-level to paragraph-level chunking improved context precision but degraded context recall on multi-drug queries (the relevant interactions for a 4-drug combination spanned multiple paragraphs). They moved to parent-child indexing and recovered recall without losing the precision gain ([Neo4j — Advanced RAG Techniques](https://neo4j.com/blog/genai/advanced-rag-techniques/), retrieved 2026-05-26).

**E-commerce — product-spec Q&A.** A marketplace team tightening their grounding prompt to reduce hallucinations on technical-spec questions observed answer relevance dropping 5% — the model was refusing to answer questions that did not have a directly-cited spec. They walked back the prompt change and instead invested in better extraction of spec tables from PDFs, which improved faithfulness without the relevance regression ([Best Chunking Strategies for RAG — Firecrawl](https://www.firecrawl.dev/blog/best-chunking-strategies-rag), retrieved 2026-05-26).

**Logistics — customs-classification assistance.** A freight-forwarder running a tariff-classification agent observed that adding a BM25 leg to a previously dense-only retrieval improved recall on rare commodity codes but introduced precision noise. They added a reranker on top to cull the BM25 noise, and the final pipeline outperformed dense-only on both dimensions ([DataVLab — RAG Evaluation 2026](https://datavlab.ai/post/rag-evaluation-methods-metrics-2026-guide), retrieved 2026-05-26).

## 6. Best Practices

- Change one variable at a time; bundled changes teach you nothing about which one worked.
- Map the regressed dimension to the most likely system layer before making a change; don't tweak the easy layer if the hard layer is responsible.
- Report all dimensions even when only some block merges; non-blocking dimensions catch cross-dimension drift.
- Walk the layered fix order: extraction → chunking → retrieval → generation; don't fix the prompt over bad data.
- Read the per-row diff, not just the aggregate scores; the structural pattern is in the rows that regressed.
- Revert when a candidate fix introduces a cross-dimension regression; the next candidate is usually better.
- Document trade-offs in the PR description when accepting a small regression to land a larger improvement.

## 7. Hands-on Exercise

**Whiteboard exercise (10 min):** Your eval comment reports faithfulness +3%, context recall -8%, context precision +6%, answer relevance unchanged. The PR introduced a smaller chunk size (768 → 384 tokens).

Sketch:

1. The most likely root cause of the recall regression.
2. The candidate fix you would try first, and the prediction of how it would move each dimension.
3. The candidate fix you would try second if the first does not land.
4. The trade-off you would document if you decided to accept the recall regression rather than chase a fix.

**What good looks like:** You identify the section-spanning-answer problem as the likely root cause (chunks now too small to hold a full answer). Your first candidate is parent-child indexing with 384-token children and 1024-token parents; your prediction is recall recovers, precision may dip slightly, faithfulness stays improved, relevance unchanged. Your second candidate addresses a different layer (e.g., reranker change, embedding model swap) — not just a different chunk size. Your trade-off acceptance is justified by the magnitude (precision gain dominates recall loss) AND by the per-row analysis (the rows that regressed are low-stakes / low-frequency).

## 8. Key Takeaways

- What does the canonical eval → fix → re-eval loop look like, and why is the "revert if cross-dimension regression appears" step the one most teams skip?
- How do you map a regressed metric to a system layer, and why does layer order matter (extraction before prompt)?
- What is the cross-dimension drift problem, and how does reporting non-blocking dimensions catch it?
- What does parent-child indexing solve, and which dimensions does it typically improve and at what cost?

## Sources

1. [RAG Isn't One-Size-Fits-All: How to Tune It for Your Use Case — LanceDB](https://www.lancedb.com/blog/rag-isnt-one-size-fits-all) — retrieved 2026-05-26
2. [RAG Evaluation 2026: Methods, Metrics, Frameworks — DataVLab](https://datavlab.ai/post/rag-evaluation-methods-metrics-2026-guide) — retrieved 2026-05-26
3. [Best Chunking Strategies for RAG (and LLMs) in 2026 — Firecrawl](https://www.firecrawl.dev/blog/best-chunking-strategies-rag) — retrieved 2026-05-26
4. [Advanced RAG Techniques for High-Performance LLM Applications — Neo4j](https://neo4j.com/blog/genai/advanced-rag-techniques/) — retrieved 2026-05-26
5. [Configure child chunking strategy — RAGFlow](https://ragflow.io/docs/configure_child_chunking_strategy) — retrieved 2026-05-26

Last verified: 2026-05-26
