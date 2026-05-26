---
week: W02
day: Wed
topic_slug: deterministic-fallback-patterns
topic_title: "Deterministic fallback patterns — what runs when retrieval returns nothing"
parent_overview: W02/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://ashutoshkumars1ngh.medium.com/hybrid-search-done-right-fixing-rag-retrieval-failures-using-bm25-hnsw-reciprocal-rank-fusion-a73596652d22
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.blockchain-council.org/ai/reducing-ai-hallucination-in-production-rag-guardrails-evaluation-hitl/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://blog.premai.io/rag-evaluation-metrics-frameworks-testing-2026/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://futureagi.com/glossary/llm-grounding/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.lakera.ai/blog/guide-to-hallucinations-in-large-language-models
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# Deterministic fallback patterns — what runs when retrieval returns nothing

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the three fallback paths (empty-result, low-confidence, corpus-gap) and the cost-ordered escalation they imply.
- Explain why BM25 sparse search is the standard empty-result fallback and what signals it catches that dense vectors miss.
- Set a RAGAS-faithfulness threshold against an eval set rather than picking one from a blog post.
- Distinguish "the model abstained" from "the model didn't know" in production logs and route each correctly.
- Treat human review as a designed-for fallback path, not as exception handling.

## 2. Introduction

In a naive RAG system, an empty retrieval result is treated as an error — the pipeline falls back to whatever the base LLM produces from its parametric knowledge, which is exactly where hallucinations enter production. Even when retrieval returns *something*, "the retrieved chunks didn't actually answer the question" is a routine outcome that an undefended pipeline ships as a confidently-wrong response.

The 2026 industry consensus is that fallbacks are not exception handlers — they are first-class operational paths, designed alongside the happy path and measured with the same rigor. A production RAG system has at minimum three fallback layers, and the boundary between "answered" and "abstained" is one of the most important metrics in the eval set.

The cost ordering matters. The cheapest fallback (a parallel BM25 search) costs a few milliseconds and recovers many of the empty-result cases. The next-cheapest (a confidence threshold gate) costs nothing once the eval threshold is set. The most expensive (human escalation) costs minutes of an expert's time and is reserved for the cases the cheaper fallbacks don't address. Designing the system means picking the right path at each level so the expensive ones don't fire on cases the cheap ones could have handled — and so the cheap ones don't silently swallow cases that should escalate.

## 3. Core Concepts

### Empty-result fallback: BM25 sparse search

When dense `$vectorSearch` returns zero hits, the conventional fallback is BM25 (Best Matching 25) keyword/sparse search on the same corpus. BM25 catches the failure mode where dense embeddings underweight literal token matches — short identifiers (error codes, version strings, statute numbers, product SKUs), rare proper nouns, and queries dominated by exact-match tokens.

The empirical observation from production RAG systems is that dense and sparse retrievers fail on *different queries*. Dense vectors recover meaning when the user paraphrases or uses different terminology; BM25 catches literal matches the embedding model can't get a strong signal on because the discriminating tokens are too rare to dominate the dense vector. The combination of dense + sparse with Reciprocal Rank Fusion (RRF) is the hybrid retrieval pattern from Tuesday's reading; the *fallback* pattern uses BM25 as a recovery path when dense returns nothing.

The signal: log the count of vector hits per query. When it's zero, route through BM25 before deciding the query is unanswerable. Latency cost: a few extra milliseconds on the BM25 path, which often closes within the same p95 budget because the cases where BM25 fires are also the cases where the user wasn't going to get an answer anyway.

### Low-confidence fallback: faithfulness gate

Retrieval can return results without those results actually answering the query. The standard gate is RAGAS-style faithfulness scoring: after generation, decompose the LLM's answer into atomic claims, ask "does each claim appear in the retrieved chunks," and compute the fraction of grounded claims. Production targets cited in 2026 sources land around 0.8 for general workloads and 0.9+ for regulated domains (finance, healthcare, legal).

The gate runs as: generate answer → score faithfulness → if below threshold, suppress the answer and route to the next fallback. The cost is one extra evaluation pass (a small LLM call or a deterministic scorer); the benefit is that you stop shipping plausible-sounding answers that aren't grounded in the corpus.

The threshold is a calibration problem. The right number depends on (a) what the cost of a false-positive answer is in your domain, (b) what your retrieval recall ceiling looks like, and (c) what your tolerance for false-negative refusals is. Set it against an eval set, not from a blog post. Production teams typically iterate the threshold across multiple deploys based on observed escalation volume and user-feedback signals.

A common multi-dimensional refinement: RAGAS combines four metrics — faithfulness, context precision, context recall, answer relevance — and the gate fires when any one falls below its calibrated threshold.

> [!instructor-review]
> Some RAG evaluation guides treat RAGAS faithfulness as sufficient on its own. This is the `ragas-faithfulness-only` known-bad-pattern: faithfulness alone misses two real failure modes — context-recall failures (the corpus didn't contain the right document at all; faithfulness can't see this) and answer-relevance failures (the answer is faithful to the retrieved chunks but doesn't actually answer the question asked). The W02 Fri eval-driven war-room uses all four RAGAS dimensions deliberately; don't let a reading or ADR draft pitch faithfulness as the complete metric.

### Corpus-gap fallback: explicit refusal with escalation

The third fallback fires when the system can detect that the query refers to something the corpus *should* contain but doesn't. This is the highest-value fallback because the failure mode it catches — a confidently wrong answer about a fact the corpus could have grounded but didn't — is the most damaging outcome in regulated workloads.

Detection patterns in production:

- **Explicit identifier mismatch.** The query references a document ID, statute reference, error code, or other structured identifier that the corpus is indexed on; the lookup returns nothing. Refuse with the gap named.
- **Citation incompleteness.** The retrieved chunks reference a sibling document (e.g., "see Section 4.3" or "as defined in §1234") that the corpus doesn't contain. The LLM should refuse to interpolate the missing reference.
- **Coverage-gap signal.** A separate monitoring metric tracks "queries the system answered" vs "queries the system refused" per topic cluster. A spike in refusals on a cluster signals a corpus gap to ingest.

The refusal must be explicit. *"I don't have enough grounded information to answer that — escalating to a human reviewer"* is the production-correct response. *"Based on my training, the answer is X"* is the failure mode the entire RAG architecture was built to prevent — and is unfortunately the default behavior of an LLM with no fallback discipline.

### Human review as a designed-for path

The four fallback paths above terminate in different outcomes. Empty-result + BM25 may recover an answer. Low-confidence + faithfulness gate suppresses the LLM answer. Corpus-gap + explicit refusal hands the query to a human.

The third outcome is not a failure — it is a first-class operational path. A production RAG system has a defined queue (a ticketing system, a Slack channel for a triage team, an explicit "queued for review" UI state) that handles escalated queries. The human-in-the-loop step is bounded — the human's response feeds back into the system as a labeled example, the corpus is updated if there's a gap, and the next instance of the same query is grounded.

The metric that signals the system is working: *refusal rate is non-zero and stable over time*. A RAG system with zero refusals is either lying or trivial. A RAG system with rising refusal rate on a specific topic cluster is telling you exactly where the corpus needs to grow.

### What "the model abstained" means in logs

In a well-designed system, three different log signals correspond to three different fallback paths:

| Path | Log signal |
|------|------------|
| Retrieval found nothing → BM25 fallback ran | `vector_hits=0, bm25_hits=N, source="bm25"` |
| Retrieval returned chunks but faithfulness < threshold | `vector_hits=N, faithfulness=0.62, refused=true` |
| Corpus gap detected (identifier mismatch) | `gap_signal="identifier_not_found", escalated=true` |

These are distinguishable in dashboards. Conflating them ("the model didn't answer") loses the diagnostic information needed to fix the underlying issue (corpus gap vs threshold tuning vs retrieval bug).

## 4. Generic Implementation

A cost-ordered fallback chain in plain Python, outside the federal-acquisitions domain. Imagine an IT-helpdesk search system: technicians query a knowledge base of internal runbooks and external vendor docs.

```python
def search_helpdesk(query: str, tenant_id: str) -> dict:
    # Path 1: dense retrieval
    vector_hits = vector_search(query, tenant_id=tenant_id, k=20)
    if vector_hits:
        candidates = vector_hits
        candidate_source = "vector"
    else:
        # Path 2: empty-result fallback — BM25 sparse
        bm25_hits = bm25_search(query, tenant_id=tenant_id, k=20)
        if not bm25_hits:
            # Path 4: corpus-gap fallback — explicit refusal
            return {
                "answer": None,
                "refused": True,
                "reason": "no_results_in_either_index",
                "escalate_to_queue": True,
            }
        candidates = bm25_hits
        candidate_source = "bm25"

    # Path 3: generation + faithfulness gate
    answer = generate(query, candidates)
    faithfulness = score_faithfulness(answer, candidates)
    if faithfulness < FAITHFULNESS_THRESHOLD:
        return {
            "answer": None,
            "refused": True,
            "reason": "low_faithfulness",
            "faithfulness_score": faithfulness,
            "candidate_source": candidate_source,
            "escalate_to_queue": True,
        }

    return {
        "answer": answer,
        "refused": False,
        "faithfulness_score": faithfulness,
        "candidate_source": candidate_source,
    }
```

Every return path logs the *reason* the system reached it. Refusal is structured, escalation is explicit, and the source of the candidates is preserved so dashboards can distinguish "the dense retriever failed but BM25 saved us" from "everything fired correctly."

## 5. Real-world Patterns

**Customer-support escalation queues.** A B2C ride-hailing platform described a fallback chain where queries the RAG system couldn't ground above 0.85 faithfulness were routed to a human-support queue with the original query, the retrieved chunks, and the rejected LLM answer attached. Support agents resolved the query and tagged whether the underlying issue was "corpus gap" (the help article didn't exist), "retrieval miss" (the article existed but wasn't retrieved), or "model error" (the article was retrieved but the LLM didn't extract correctly). The tags fed a weekly review cycle that targeted the largest contributor.

**Clinical decision support.** A health-tech startup running a clinical-literature search tool described setting faithfulness threshold at 0.95 — extremely conservative — because the cost of a wrong clinical answer was unacceptably high. Refusal rates were correspondingly higher than commercial workloads (around 18% of queries hit the threshold gate), and the human review path was staffed accordingly. The team explicitly framed refusal as "the system working correctly" rather than as a quality issue to drive down.

**E-commerce product Q&A.** A large electronics retailer documented their corpus-gap pattern around SKU references: when a user query contained a specific product SKU that didn't match any indexed product, the system refused immediately rather than retrieving "semantically similar" products and risking a confidently wrong answer about a different product. The refusal carried a clear UI affordance ("we don't have details on that product — try searching for…") which preserved user trust even on the failure path.

**Financial-services compliance Q&A.** A bank described an explicit policy that the RAG system would refuse any query whose retrieved chunks didn't include a citation to a specific compliance document, even if the LLM could plausibly answer from general knowledge. The refusal was hard-coded — not threshold-based — because regulatory expectations made "plausible answer" an unacceptable substitute for "cited answer."

## 6. Best Practices

- Treat empty-result as a designed-for state, not an exception. Wire the BM25 fallback before you deploy.
- Calibrate the faithfulness threshold against your eval set, segmented by query type. A single global threshold loses to per-segment thresholds.
- Use all four RAGAS dimensions (faithfulness, context precision, context recall, answer relevance), not faithfulness alone. The four dimensions catch different failure modes.
- Log the reason for every refusal — empty-vector, empty-everywhere, low-faithfulness, corpus-gap. Conflating them blinds your dashboards.
- Build the human-review queue as a first-class product surface, with UI affordances for "we queued this for a human" and turnaround-time visibility.
- Track refusal rate per topic cluster as a corpus-growth signal. A spike means the corpus needs the topic.
- Never let a low-confidence answer ship to the user. Suppress, refuse, or escalate — but do not hide the LLM's uncertainty behind confident phrasing.

## 7. Hands-on Exercise

**Code task (15 min).** Extend the cascade you wrote in the topic 1 exercise (or this reading's example) so that:

1. The retrieval step records `vector_hits` and `bm25_hits` separately and tries BM25 on dense-zero.
2. The generation step is followed by a faithfulness scoring step (you can mock the scorer as `score_faithfulness(answer, chunks) -> float`).
3. The function returns a structured dict that distinguishes `answered`, `refused_low_faithfulness`, `refused_corpus_gap`, and includes the source of candidates and the faithfulness score on every return.
4. There's a `should_escalate` boolean on the return that callers can check to route refused queries into a human-review queue.

**What good looks like.** Every fallback path is named in the return value (not a generic "no answer"). The empty-result path correctly distinguishes "dense failed, BM25 saved us" from "both failed." The faithfulness threshold is a named constant at the top of the file (not a magic number in the function body). The escalation boolean is set on the exact set of conditions you'd want a human to review.

## 8. Key Takeaways

- *What are the three fallback paths and their cost ordering?* — Empty-result + BM25 (cheap, single-digit ms), low-confidence + faithfulness gate (mid-cost, one extra scoring pass), corpus-gap + human escalation (expensive, minutes of expert time). Cost ordering drives the cascade.
- *Why is BM25 the empty-result fallback?* — Because dense and sparse retrievers fail on different queries; BM25 catches the literal-match queries embeddings underweight (rare identifiers, exact tokens, structured IDs).
- *Why is faithfulness alone insufficient as a confidence metric?* — Because it misses context-recall and answer-relevance failures. Use the four-metric RAGAS dimensions, not faithfulness alone.
- *What does a healthy refusal rate look like in production?* — Non-zero and stable. Zero means the system is lying; a rising rate on a topic cluster means the corpus needs to grow there.
- *Why is human review a first-class fallback path?* — Because the alternative — confidently wrong answers on queries the corpus can't ground — is the failure mode RAG was built to prevent. The human's response also closes the loop by feeding back into the corpus.

## Sources

1. [Hybrid Search Done Right: Fixing RAG Retrieval Failures using BM25 + HNSW + RRF (Feb 2026)](https://ashutoshkumars1ngh.medium.com/hybrid-search-done-right-fixing-rag-retrieval-failures-using-bm25-hnsw-reciprocal-rank-fusion-a73596652d22) — retrieved 2026-05-26
2. [Reducing AI Hallucination in Production (RAG Guardrails, Evaluation, HITL)](https://www.blockchain-council.org/ai/reducing-ai-hallucination-in-production-rag-guardrails-evaluation-hitl/) — retrieved 2026-05-26
3. [RAG Evaluation: Metrics, Frameworks & Testing (PremAI Blog, 2026)](https://blog.premai.io/rag-evaluation-metrics-frameworks-testing-2026/) — retrieved 2026-05-26
4. [What Is LLM Grounding? (FutureAGI, 2026)](https://futureagi.com/glossary/llm-grounding/) — retrieved 2026-05-26
5. [LLM Hallucinations in 2026: How to Understand and Tackle (Lakera)](https://www.lakera.ai/blog/guide-to-hallucinations-in-large-language-models) — retrieved 2026-05-26

Last verified: 2026-05-26
