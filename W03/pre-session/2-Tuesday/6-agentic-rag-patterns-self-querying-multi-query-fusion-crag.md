---
week: W03
day: Tue
topic_slug: agentic-rag-patterns-self-querying-multi-query-fusion-crag
topic_title: "Agentic-RAG patterns — self-querying, multi-query, RAG Fusion, CRAG"
parent_overview: W03/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://js.langchain.com/docs/how_to/self_query/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://github.com/Raudaschl/rag-fusion
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://arxiv.org/abs/2402.03367
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://arxiv.org/abs/2401.15884
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://arxiv.org/html/2501.09136v4
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://glaforge.dev/posts/2026/02/10/advanced-rag-understanding-reciprocal-rank-fusion-in-hybrid-search/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-06-06
---

# Agentic-RAG patterns — self-querying, multi-query, RAG Fusion, CRAG

> [!NOTE]
> **From earlier:** W2 built the RAG layer (`POST /rag/clause-search`). Today that layer becomes a tool the ReAct agent calls — the first time the W2 layer is consumed by an agent rather than a human.

## 1. Learning Objectives

- Distinguish four agentic-retrieval patterns — self-querying, multi-query, RAG Fusion, CRAG — and articulate the failure mode each addresses
- Describe how self-querying retrieval decomposes a natural-language query into a semantic component and a metadata-filter component
- Explain the Reciprocal Rank Fusion (RRF) algorithm and why it operates on ranks rather than scores
- Summarise CRAG's three confidence-level branches and the corrective action each triggers
- Sequence the four patterns by cost and complexity and articulate when escalation is worth the latency budget

## 2. Introduction

The 2023 baseline RAG pipeline — embed the query, single vector search, stuff top-K into the prompt — fails predictably: ambiguous queries, implicit metadata filters, multi-faceted questions, wrong top-K. By 2025–2026 the field converged on **agentic-RAG patterns** in which an LLM decides what to retrieve, evaluates the result, and re-retrieves if necessary.

The arXiv survey "Agentic Retrieval-Augmented Generation" (2501.09136) catalogues four patterns that appear in nearly every production stack: self-querying, multi-query, RAG Fusion, and CRAG. They form a **cost ladder** — choose the lowest-cost pattern that handles your actual failure modes. Stacking all four on every query is pure cost without proportional uplift.

For intake-triage today: the W2 `POST /rag/clause-search` endpoint is callable as an agent tool. The agent picks the pattern — self-querying for filtered clause retrieval, multi-query for ambiguous phrasing, CRAG when retrieval quality is unknown.

## 3. Core Concepts

### 3.1 Self-querying retrieval — extract filters from natural language

A self-querying retriever uses an LLM to parse a natural-language query into a **semantic query** for vector similarity search and a **structured filter** applied as metadata constraints.

```json
{
  "query": "small-business set-aside clauses",
  "filter": {
    "AND": [
      {"field": "NAICS", "op": "=", "value": "541512"},
      {"field": "set_aside_code", "op": "=", "value": "SBA"}
    ]
  }
}
```

Requires a vector store with metadata filtering (MongoDB Atlas, Pinecone, Qdrant). LangChain `SelfQueryRetriever` is the reference implementation. **Failure mode fixed:** plain semantic search ignores metadata — "clauses for small-business set-asides" retrieves clauses from every NAICS code without the filter.

### 3.2 Multi-query retrieval — N variants defend against phrasing brittleness

A multi-query retriever rewrites the query as N alternative phrasings (typically 3–5), runs parallel vector searches, and fuses the result sets. **Failure mode fixed:** vocabulary mismatch — "compliance obligations" vs. "regulatory requirements" — variant phrasings cover both.

### 3.3 RAG Fusion — multi-query + Reciprocal Rank Fusion

RAG Fusion (Raudaschl 2024, arXiv 2402.03367) adds **Reciprocal Rank Fusion (RRF)** as the merge step:

```
RRF_score(doc) = Σ_variant   1 / (k + rank_in_variant(doc))
```

with `k` typically 60. Documents ranking high in multiple variants float to the top. **Why ranks not scores:** BM25, cosine, and dot-product scores are not comparable across indexes. RRF ignores score magnitudes — ranks compose across heterogeneous retrievers.

### 3.4 CRAG — Corrective RAG with a quality evaluator

Corrective RAG (Yan et al. 2024, arXiv 2401.15884) adds one LLM-judge step after retrieval. The evaluator emits: **Correct** (use documents as-is), **Ambiguous** (augment with web search or KG), **Wrong** (fall back entirely to the alternative). Cost: one extra LLM call per retrieval. **Failure mode fixed:** retrieval-quality blindness — baseline RAG cannot detect whether its top-K is relevant.

> [!IMPORTANT]
> **The ladder.** Baseline RAG (1 LLM call, 1 search) → Self-querying (2 calls, 1 search — implicit metadata filters) → Multi-query (2 calls, N searches — vocab mismatch) → RAG Fusion (2 calls, N searches — ranking robustness) → CRAG (3+ calls — retrieval-quality blindness). Instrument the baseline, identify the dominant failure mode, escalate to the pattern that addresses it.

## 4. Generic Implementation

A generic self-querying retriever shape — product search, illustrating the parse-then-filter pattern:

```python
from pydantic import BaseModel, Field
from typing import Optional

class ProductSearchQuery(BaseModel):
    semantic_query: str
    price_max: Optional[float] = Field(default=None, ge=0)
    price_min: Optional[float] = Field(default=None, ge=0)
    category: Optional[str] = None
    in_stock_only: bool = False

def parse_query(natural_query: str, llm) -> ProductSearchQuery:
    """LLM call: intent extraction only — no semantic understanding of the corpus."""
    system = (
        "Parse the user's product search into a ProductSearchQuery. "
        "Extract price, category, stock constraints explicitly. "
        "Remaining intent goes in semantic_query."
    )
    return llm.structured_output(ProductSearchQuery, system=system, user=natural_query)

def search(natural_query: str, llm, vector_store) -> list:
    parsed = parse_query(natural_query, llm)
    metadata_filter = {}
    if parsed.price_max is not None: metadata_filter["price"] = {"$lte": parsed.price_max}
    if parsed.price_min is not None: metadata_filter.setdefault("price", {})["$gte"] = parsed.price_min
    if parsed.category: metadata_filter["category"] = parsed.category
    if parsed.in_stock_only: metadata_filter["in_stock"] = True
    return vector_store.similarity_search(
        query=parsed.semantic_query, filter=metadata_filter, k=20
    )

def reciprocal_rank_fusion(result_lists: list[list[str]], k: int = 60) -> list[tuple[str, float]]:
    """result_lists: ranked doc-id lists, one per query variant. Returns docs by RRF score."""
    fused: dict[str, float] = {}
    for ranking in result_lists:
        for rank, doc_id in enumerate(ranking):
            fused[doc_id] = fused.get(doc_id, 0.0) + 1.0 / (k + rank + 1)
    return sorted(fused.items(), key=lambda kv: kv[1], reverse=True)
```

The parse is a separate LLM call from generation — cache, test, and observe each independently. Metadata-filter shape is vector-store-specific; wrap with an adapter. RRF treats every variant equally — weighted RRF is a refinement; start unweighted.

## 5. Real-world Patterns

**Legal research (Casetext CoCounsel, Harvey).** "Negligent misrepresentation by an architect" vs. "professional negligence by a design professional" — multi-query with synonym variants is the default. RAG Fusion merges results across case-law and secondary-source indexes.

**E-commerce (Etsy, Wayfair, Shopify).** "Blue dress under $50 for a summer wedding" — baseline vector search ignores colour and price constraints. Self-querying extracts `colour=blue`, `price<=50`, `occasion=wedding` as metadata filters.

**Customer support (Zendesk Resolve, Intercom Fin).** CRAG added on top of baseline RAG in 2025–2026. The evaluator's "wrong" branch prevents confident wrong answers.

> [!NOTE]
> **Acquire-gov framing.** The W2 RAG layer is the unstructured side (clause text, FAR/DFARS prose). Self-querying is the daily pattern — extract NAICS code and set-aside type from context, use as metadata filters on `clause-search`. Multi-query + CRAG are escalation tools when the simple retrieval fails.

## 6. Best Practices

- Instrument the baseline before escalating — metrics tell you which pattern to add
- Cache the parse step in self-querying — same query, same structured output; caching is a free latency win
- Use RRF over score-based fusion for heterogeneous retrievers — ranks compose; scores do not
- Calibrate CRAG thresholds against ground-truth — "correct" too eagerly lets bad retrievals through; "wrong" too eagerly burns budget
- Make retrieval choices observable — the trace should record why the agent picked multi-query over baseline

> [!WARNING]
> **Anti-pattern: `naive-rag-as-default`.** Defaulting to single-query baseline RAG for every agentic retrieval — regardless of query type or measured failure mode — is the most common deployment mistake. For the acquisition domain specifically, a query like "clauses requiring small-business participation post-amendment" will silently retrieve irrelevant clauses if the NAICS and set-aside metadata are not filtered. Baseline RAG is the *starting point* to instrument and iterate from, not the permanent solution.

## 7. Hands-on Exercise

You are designing the retrieval layer for an internal knowledge base at a global logistics company. For each query below, identify the best pattern, the metadata fields needed, and the failure mode being defended against: (a) "What's the protocol for refrigeration failure on a perishable shipment?" (b) "Show me all incident reports from EMEA depots in 2025 involving forklift damage." (c) "Why are our Helsinki on-time delivery rates dropping?" (d) "Is there a recent policy update on dangerous-goods labeling?"

> [!NOTE]
> **Self-check** (30 s — answer mentally before expanding)
>
> 1. Why does Reciprocal Rank Fusion use ranks instead of scores?
> 2. What are CRAG's three confidence branches and what action does each trigger?

<details>
<summary>Show answers</summary>

1. Vector-search scores from different retrievers (BM25, cosine, dot product, hybrid) are not on the same scale. Two systems may both return "highly relevant" documents with scores of 0.92 and 4.7 respectively — these are not directly comparable. RRF operates on ranks, which are comparable across any retriever: rank 1 means "best result from this retriever" regardless of the underlying scoring function.
2. **Correct** — retrieval is reliable; use documents directly. **Ambiguous** — confidence is in the middle band; augment with an alternative source (web search or knowledge graph) and combine results. **Wrong** — retrieval is unreliable; fall back entirely to the alternative source. The labels are produced by an LLM-judge evaluator step that runs after the initial retrieval.

</details>

## 8. Key Takeaways

- **Four patterns, cost ladder:** self-querying → multi-query → RAG Fusion → CRAG
- **RRF uses ranks, not scores** — ranks are comparable across heterogeneous retrievers; raw scores are not
- **CRAG's three branches:** correct (use as-is), ambiguous (augment), wrong (replace with fallback)
- **Escalate empirically** — instrument baseline, identify dominant failure mode, pick the pattern that addresses it
- **Retrieval patterns are tools** — simple queries pay baseline cost; hard queries escalate

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://js.langchain.com/docs/how_to/self_query/ — Self-querying retrieval (LangChain) — retrieved 2026-05-26 — hot-tech
- https://github.com/Raudaschl/rag-fusion — RAG-Fusion: multi-query + Reciprocal Rank Fusion (Raudaschl GitHub) — retrieved 2026-05-26 — foundation-stable
- https://arxiv.org/abs/2402.03367 — RAG-Fusion: A New Take on Retrieval-Augmented Generation (Raudaschl 2024) — retrieved 2026-05-26 — foundation-stable
- https://arxiv.org/abs/2401.15884 — Corrective Retrieval Augmented Generation (Yan et al. 2024) — retrieved 2026-05-26 — foundation-stable
- https://arxiv.org/html/2501.09136v4 — Agentic Retrieval-Augmented Generation: A Survey — retrieved 2026-05-26 — hot-tech
- https://glaforge.dev/posts/2026/02/10/advanced-rag-understanding-reciprocal-rank-fusion-in-hybrid-search/ — Advanced RAG: Understanding Reciprocal Rank Fusion in Hybrid Search (Guillaume Laforge) — retrieved 2026-05-26 — hot-tech

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

The Raudaschl RAG-Fusion repository ships an evaluation harness against NFCorpus/BEIR. The reported uplift over a baseline is +22% NDCG@5 and +40% recall@10 when RRF is combined with hybrid (BM25 + vector) search. The +40% recall figure is what motivates the pattern for the acquisition-clause retrieval use case — missing a relevant clause is higher-stakes than including an irrelevant one (false-negative cost exceeds false-positive cost).

Weighted RRF (2026 refinement): rather than treating every query variant equally, assign weights based on the LLM's confidence in each variant. A variant phrased as a direct synonym of the original gets weight 1.0; a distant paraphrase gets weight 0.6. The weighted formula: `RRF_score(doc) = Σ_variant  weight_v / (k + rank_in_variant(doc))`. Published evaluations show modest gains over unweighted RRF on legal and medical corpora; the benefit on general corpora is smaller. Start with unweighted; add weighting only if you have ground-truth to calibrate against.

For the acquire-gov knowledge graph (Wed's KG/CG topic), agentic RAG becomes agentic graph traversal + RAG fusion: the agent queries the KG for structured entity relationships (which evaluators are assigned to which proposals, which amendments affect which solicitations), then fuses those structured results with unstructured clause text from the vector store. The fusion step uses RRF across the two result sets. Senior FDEs building the Wed multi-agent evaluator flow should design the tool surface so the agent can call both `graph_query` and `clause_search` and merge results deterministically.

</details>

Last verified: 2026-06-06
