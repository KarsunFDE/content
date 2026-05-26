---
week: W02
day: Wed
topic_slug: reranking-cascade-designs
topic_title: "Reranking strategy depth — cascade designs and the cost-of-precision curve"
parent_overview: W02/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://aws.amazon.com/blogs/machine-learning/cohere-rerank-3-5-is-now-available-in-amazon-bedrock-through-rerank-api/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.bswen.com/blog/2026-02-25-best-reranker-models/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://futureagi.com/blog/evaluating-cohere-rerank-rag-2026/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://developer.nvidia.com/blog/how-using-a-reranking-microservice-can-improve-accuracy-and-costs-of-information-retrieval/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://medium.com/@vaibhav-p-dixit/reranking-in-rag-cross-encoders-cohere-rerank-flashrank-c7d40c685f6a
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# Reranking strategy depth — cascade designs and the cost-of-precision curve

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish a bi-encoder retriever from a cross-encoder reranker on the dimensions of latency, cost, and quality.
- Sketch a three-stage cascade and identify which stage each component (vector store, cross-encoder, LLM) plays.
- Choose between a managed reranker, an open-weight self-hosted reranker, and an in-prompt LLM rerank for a given budget.
- Read the cost-of-precision curve and identify the inflection point past which more candidates buy nothing.
- Decide when adding a reranker would *hurt* — the recall-loss failure mode that no model rescues.

## 2. Introduction

Reranking is the quiet workhorse of production RAG. A bi-encoder embedding model retrieves quickly but approximately — it compresses a query and a document into independent vectors and compares them with cosine similarity, which is fast but cannot model the *interaction* between query and document. A cross-encoder reranker scores each query-document pair jointly: query and document share attention through every transformer layer, so the model can ask "does this specific document answer this specific query" rather than "are these two vectors near each other in some abstract space."

The benchmarks have been consistent for two years. Cross-encoder reranking on top of a bi-encoder retriever typically delivers 5–15 NDCG@10 points of quality lift over retrieval-only on standard benchmarks like MTEB and BEIR. On lexically hard datasets (rare terms, jargon-heavy domains, multilingual corpora) the lift can exceed 20 points. That gap is often the difference between a RAG system that demos well and one that holds up under adversarial production traffic.

But reranking is not free. Each cross-encoder call costs latency (50–300ms for a managed API, 20–100ms for a self-hosted GPU model on the rerank set, longer on CPU). Each call costs money or compute. And — counter-intuitively — adding a reranker can *hurt* end-to-end answer quality in a specific failure mode: when retrieval recall is already low, the reranker has no chance of pulling the right answer to the top because the right answer was never in the candidate pool. No reranker rescues a chunk that wasn't retrieved.

This reading covers the cascade design, the three reranker options on the spectrum, the cost-of-precision curve, and the eval-triangle for choosing where on the curve to land.

## 3. Core Concepts

### The three-stage cascade

The canonical production shape is *retrieve N → rerank M → top-k to LLM*, where typically N is 50–200, M is 10–50, and k is 3–10. Each stage trades latency for precision.

**Stage 1: bi-encoder retrieval.** The query embeds once. The corpus embedded at index time. Approximate nearest-neighbor search (HNSW, IVF, ScaNN) returns the top N candidate documents in tens of milliseconds for million-document corpora. This stage cares about *recall* — it is the wide net.

**Stage 2: cross-encoder rerank.** The query and each of the N candidates re-enter a smaller transformer model that scores the query-document pair jointly. The model returns a relevance score per pair; the system keeps the top M. This stage cares about *precision* — it sorts the wide net's catch.

**Stage 3: LLM generation.** The LLM is, in effect, a third reranker — it implicitly weights the M candidates by which ones it cites and how. The k-to-LLM count is set by context-window economics and the LLM's lost-in-the-middle behavior (precision tends to fall sharply past k=10 on most models).

The three stages do different work. Adding more candidates at Stage 1 buys recall. Adding precision at Stage 2 buys ranking quality. Adding context at Stage 3 buys… less than you'd think, past a point.

### Three reranker options on the spectrum

**Managed API reranker.** Cohere Rerank 3.5 is the current category leader, available on AWS Bedrock as a first-party model and directly via the Cohere API. Pricing in early 2026 is roughly $2.00 per 1,000 search units (a unit = a query plus up to 100 documents). Latency for a 20–50 document rerank is in the 100–300ms range, with reports that Cohere wins quality comparisons at roughly 10x lower latency than custom self-hosted equivalents. Other managed options include Jina Reranker and Voyage Rerank. The trade is operational simplicity for opacity (no model weights, no fine-tuning) and per-call cost.

> [!instructor-review]
> **D-060 scope on Cohere Rerank on Bedrock.** This deep-dive (and Tue's `4-reranking-fundamentals.md`) advocates Cohere Rerank 3.5 on Bedrock as the W2 default. D-060 explicitly permitted real Bedrock `InvokeModel` for Titan and Claude in W2 but deferred AWS managed RAG services (Bedrock Knowledge Bases, Agents-for-Bedrock, OpenSearch Managed) to W5. Confirm with cohort lead whether managed-reranker invocation falls under the W2-permitted Bedrock surface or the W5-deferred managed-AI surface. **Working default for now:** managed-reranker invocation (a single Bedrock Rerank API call, no agent loop, no managed retrieval store) is treated as **W2-permitted**; managed retrieval (Bedrock Knowledge Bases) stays **W5-deferred**. If cohort lead overrules, the Tue/Wed cascade falls back to self-hosted BGE-reranker-v2-m3 for W2 and re-introduces the managed reranker in W5.

**Open-weight self-hosted reranker.** BGE-Reranker-v2-m3 (from BAAI) is the 2026 default for teams that want self-hosting. It's a 278M-parameter MiniLM-derived cross-encoder, multilingual, with quality close to Cohere Rerank on English text and a permissive license. Self-hosting on a GPU node (a single A10G or L4 handles ~100 docs/second comfortably) cuts marginal cost per query but introduces deployment, scaling, and observability work. For higher quality at higher latency cost, BGE-Reranker-v2-gemma is a 9B-parameter variant that approaches the top of public leaderboards.

**In-prompt LLM rerank.** Pass the candidate set to a fast LLM with a prompt of the form *"rank these N documents 1..N by relevance to query X."* Quality is often the highest of the three (the LLM has the strongest reasoning), latency is the worst, and cost is unpredictable (depends on candidate length and LLM pricing). Useful as a third-tier cascade for high-value queries — e.g., the user-facing answer where the latency budget is generous — but rarely as the default reranker.

### The cost-of-precision curve

Two empirical observations dominate cascade tuning.

The first: **going wider at Stage 1 has sharply diminishing returns.** Once Recall@N is already above 95%, doubling N from 50 to 100 (or 100 to 200) buys very little additional recall — the missing 5% tends to be queries where the embedding model fundamentally doesn't model the query's intent. Throwing more candidates at the reranker won't fix an embedding-space problem.

The second: **going wider at Stage 2 reliably improves Precision@k_post_rerank** — up to a point. A cross-encoder reranking 50 candidates outperforms one reranking 20, because the cross-encoder is doing real work the bi-encoder couldn't do. The reranker pulls the right answer from candidate position 35 up to position 3. But at very high candidate counts, latency dominates and quality plateaus.

The tuning recipe: hold Recall@N fixed at ~95% (find the N where this holds for your corpus), then tune M (the rerank pool) by measuring NDCG@k_post_rerank and p95 latency at several settings. Pick the smallest M where NDCG flattens.

### The recall-loss failure mode

A reranker *only sorts* — it never adds. If a relevant document wasn't in the top-N from Stage 1, no Stage-2 reranker can put it in the top-k. Recall@N is the ceiling.

There is a subtler version of the same problem. A reranker can *lose recall* between Stage 1 and Stage 2: if Recall@N (pre-rerank) is 0.94 and Recall@k (post-rerank, k=5) is 0.78, the reranker dropped 16 points of recall to win precision on the top-5. For factual QA where any-correct-answer counts, that may be the right trade. For multi-hop questions where you need *both* supporting documents in the final context, it's not — the reranker just dropped the second-most-relevant document because it didn't look as good as the top-3 in isolation.

The diagnostic is the **eval triangle**: NDCG@k_post_rerank for ranking quality, Recall@k_pre_rerank versus Recall@k_post_rerank for recall preservation, and p95/p99 latency for user-perceived cost. All three must be measured. Tuning on NDCG alone hides the recall-loss failure.

## 4. Generic Implementation

A minimal cascade in plain Python, outside the federal-acquisitions domain. Imagine a job-search platform reranking matched listings against a candidate's free-text query.

```python
def retrieve_and_rerank(query: str, k: int = 5, n_pre_rerank: int = 50) -> list[dict]:
    # Stage 1: bi-encoder retrieval (wide net, cheap)
    query_vec = embedding_model.embed(query)
    candidates = vector_db.search(
        collection="job_listings",
        query_vector=query_vec,
        top_k=n_pre_rerank,
    )

    # Stage 2: cross-encoder rerank
    # Each candidate's text + query is scored as a pair
    pairs = [(query, c["job_description"]) for c in candidates]
    scores = reranker.rerank(pairs)  # returns scores aligned to pairs

    # Sort candidates by reranker score, keep top-k
    scored = sorted(
        zip(candidates, scores),
        key=lambda pair: pair[1],
        reverse=True,
    )
    top_k = [doc for doc, _score in scored[:k]]

    return top_k
```

The reranker object is interchangeable. A managed-API client (`cohere.rerank(...)`), a self-hosted BGE call (`bge_reranker.compute_scores(pairs)`), or an LLM-prompt rerank wrapped in a function — the cascade shape doesn't change. That interchangeability is the point: bench the alternatives against your eval set, swap the implementation, leave the rest of the pipeline alone.

## 5. Real-world Patterns

**E-commerce product search at scale (electronics retailer, 2025–2026).** A large electronics retailer published a case study moving from retrieval-only to a two-stage cascade. Stage 1 was their existing bi-encoder over 30M product listings. Stage 2 added a self-hosted BGE-Reranker-v2-m3 reranking the top-100 to top-10. Conversion rate on long-tail queries (where lexical match alone fell short) improved measurably; p95 latency rose by ~120ms, which their PM team judged acceptable given the conversion lift.

**Healthcare literature search (clinical decision support startup).** A health-tech team described running an in-prompt LLM rerank as a tier-3 stage for high-stakes queries. Standard queries hit the bi-encoder + cross-encoder cascade. Queries flagged as "clinically critical" (drug interactions, dose calculations) got an additional LLM-rerank pass that took 1.5–3 seconds. The cost was justified by the failure-mode asymmetry: a missed drug-interaction in retrieval is harmful in a way a missed product-listing in retail isn't.

**Customer support knowledge bases (B2B SaaS).** A common pattern in 2026 write-ups: deploy Cohere Rerank 3.5 via the Bedrock Rerank API rather than self-hosting. The reasoning is operational — most B2B SaaS teams don't have the GPU-fleet expertise to operate a cross-encoder at low p99 latency, and the managed API's per-call pricing is competitive at small-to-medium volume. The cross-over to self-hosting tends to happen north of ~10M reranks per month.

**Multilingual e-commerce (cross-border marketplace).** A cross-border-shopping platform described picking BGE-Reranker-v2-m3 specifically for its multilingual quality — their corpus is mixed English / Spanish / Portuguese / Chinese and the multilingual MiniLM base outperformed monolingual rerankers on their internal cross-lingual eval set. The English-only category leader didn't win when "the query is in one language and the relevant document is in another."

## 6. Best Practices

- Tune the cascade end-to-end on your own eval set; do not assume public benchmark numbers transfer to your corpus.
- Measure Recall@N before tuning the rerank pool size; if Recall@N is already low, the reranker cannot save you — go fix retrieval first.
- Track the eval triangle (NDCG@k_post_rerank, Recall@k_pre vs post, p95 latency) on every cascade change. Tuning on NDCG alone hides recall regressions.
- Start with a managed reranker (Cohere Rerank 3.5 on Bedrock) unless you have a specific reason to self-host. Operational simplicity outweighs marginal per-call cost for most teams below ~10M reranks/month.
- Cache reranker outputs aggressively. For a query-document pair you've scored once, the score is stable until the document changes; treat it like an embedding.
- Treat the LLM as your "tier 3" reranker only for high-value queries; running every query through an LLM rerank inflates cost without proportional quality gains.
- Never deploy a reranker without an eval set. The whole point of reranking is measurable quality lift — if you can't measure, you can't tune, and an untuned reranker can quietly degrade quality.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** You inherit a retrieval system at a media company that recommends articles for a personalized news feed. The current system is bi-encoder retrieval only: query (a user's interest profile, ~150 tokens) against a corpus of ~2M articles, returning top-10 to a ranking layer. Editorial complains that the top-10 frequently contains articles that are "topically close but the wrong angle" — a query about a sports team's coaching change returns articles about the team but unrelated topics.

Design a two-stage cascade for this system. On the whiteboard, name:

1. The N (pre-rerank candidate pool size) you'd start at and your rationale.
2. The reranker option (managed, self-hosted, in-prompt LLM) and why.
3. The eval-set construction (where do labeled relevant/not-relevant pairs come from?).
4. The single metric you'd track in production as a recall-loss canary.

**What good looks like.** N around 50–100 (recall headroom without crushing latency budget). A managed API for fast iteration unless data-residency or volume forces self-hosting. Eval-set drawn from editorial click-and-dwell logs combined with a small editorial-rated golden set. Recall-loss canary is Recall@10 (pre-rerank) versus Recall@10 (post-rerank) computed daily over the eval set; alert when the gap widens past a threshold.

## 8. Key Takeaways

- *What does each stage of the cascade do?* — Stage 1 (bi-encoder) maximizes recall, Stage 2 (cross-encoder) maximizes precision on the candidates Stage 1 produced, Stage 3 (LLM) implicitly weights via citation.
- *When does reranking fail to help?* — When Recall@N is already low. The reranker can only sort; it cannot recover documents that were never retrieved.
- *Managed reranker vs self-hosted vs in-prompt LLM — how do you choose?* — Managed is the default below ~10M reranks/month. Self-hosted wins on volume, custom-domain fine-tuning, or strict data-residency. In-prompt LLM is the tier-3 escape hatch for high-stakes queries.
- *Why measure the eval triangle, not just NDCG?* — Because NDCG@k_post_rerank can rise while Recall@k_post_rerank falls; the trade is fine for some workloads (single-answer factual QA) and disastrous for others (multi-hop questions).
- *What is the practical inflection point on the cost-of-precision curve?* — Around Recall@N = 0.95 for Stage 1, and around the smallest M where NDCG@k flattens for Stage 2.

## Sources

1. [Cohere Rerank 3.5 is now available in Amazon Bedrock through Rerank API (AWS Machine Learning Blog)](https://aws.amazon.com/blogs/machine-learning/cohere-rerank-3-5-is-now-available-in-amazon-bedrock-through-rerank-api/) — retrieved 2026-05-26
2. [Best Reranker Models for RAG: Open-Source vs API Comparison (BSWEN, Feb 2026)](https://docs.bswen.com/blog/2026-02-25-best-reranker-models/) — retrieved 2026-05-26
3. [Evaluating Cohere Rerank in RAG (FutureAGI, 2026)](https://futureagi.com/blog/evaluating-cohere-rerank-rag-2026/) — retrieved 2026-05-26
4. [How Using a Reranking Microservice Can Improve Accuracy and Costs of Information Retrieval (NVIDIA Technical Blog)](https://developer.nvidia.com/blog/how-using-a-reranking-microservice-can-improve-accuracy-and-costs-of-information-retrieval/) — retrieved 2026-05-26
5. [Reranking in RAG: Cross-Encoders, Cohere Rerank & FlashRank (Vaibhav Dixit, Mar 2026)](https://medium.com/@vaibhav-p-dixit/reranking-in-rag-cross-encoders-cohere-rerank-flashrank-c7d40c685f6a) — retrieved 2026-05-26

Last verified: 2026-05-26
