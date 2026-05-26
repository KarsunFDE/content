---
week: W03
day: Tue
topic_slug: agentic-rag-patterns-self-querying-multi-query-fusion-crag
topic_title: "Agentic-RAG patterns — self-querying, multi-query, RAG Fusion, CRAG"
parent_overview: W03/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 25
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
last_verified: 2026-05-26
---

# Agentic-RAG patterns — self-querying, multi-query, RAG Fusion, CRAG

## 1. Learning Objectives

- By the end of this reading, the learner can distinguish four agentic-retrieval patterns — self-querying, multi-query, RAG Fusion, and CRAG — and articulate the failure mode each is designed to address.
- By the end of this reading, the learner can describe how self-querying retrieval decomposes a natural-language query into a semantic component and a metadata-filter component.
- By the end of this reading, the learner can explain the Reciprocal Rank Fusion (RRF) algorithm and why it operates on ranks rather than on scores.
- By the end of this reading, the learner can summarise CRAG's three confidence-level branches (correct / ambiguous / wrong) and the corrective action each triggers.
- By the end of this reading, the learner can sequence the four patterns by cost and complexity, and articulate when escalation from the cheap pattern to the expensive one is worth the latency budget.

## 2. Introduction

> **Reading-time note.** This is the densest pre-session topic in W03 — 25 min reading time. It covers four agentic-RAG patterns (self-querying, multi-query, RAG Fusion, CRAG) as a single family because the patterns form a *cost ladder* and choosing between them is the whole pedagogical point. Splitting them across separate files would lose the comparative framing. Allocate the time deliberately — skimming this one is the wrong move.

The 2023 baseline RAG pipeline — embed the user's query, do a single vector search, stuff the top-K results into the prompt — was a useful first iteration. It also fails on a predictable list of inputs: ambiguous queries, multi-faceted questions, queries that need metadata filters the user did not articulate, queries where the top-K retrieval is just wrong.

By 2025–2026 the field had converged on a family of **agentic-RAG patterns** — retrieval pipelines in which an LLM is in the loop, deciding what to retrieve, evaluating the result, and re-retrieving if necessary. The arXiv "Agentic Retrieval-Augmented Generation: A Survey" (2501.09136) catalogues the family; four patterns appear in nearly every production stack: self-querying, multi-query, RAG Fusion (multi-query + Reciprocal Rank Fusion), and CRAG (Corrective RAG with an evaluator). They form a ladder of cost and capability, and the discipline is choosing the lowest-cost pattern that handles your actual failure modes.

This reading walks the four patterns as a family — what each fixes, how each is implemented, and how to escalate from cheap to expensive without paying for capability you do not need.

## 3. Core Concepts

### 3.1 Self-querying retrieval — extract filters from natural language

A self-querying retriever uses an LLM to parse the user's natural-language query into two parts:

- A **semantic query**, used for the vector similarity search.
- A **structured filter**, applied as metadata constraints on the vector store.

Example. The query *"papers on transformer architectures published after 2022 with at least 100 citations"* becomes:

```json
{
  "query": "transformer architectures",
  "filter": {
    "AND": [
      {"field": "year", "op": ">", "value": 2022},
      {"field": "citations", "op": ">=", "value": 100}
    ]
  }
}
```

The vector store retrieves on the semantic part, filters on the metadata part, returns the intersection. The pattern requires a vector store that supports metadata filtering (MongoDB Atlas Vector Search, Pinecone, Qdrant, Weaviate, Elasticsearch — most modern ones do). The LangChain `SelfQueryRetriever` is the most-cited reference implementation; equivalent shapes exist in LlamaIndex and Haystack.

**The failure mode it fixes.** Plain semantic search treats every query as semantic. *"Papers after 2022"* will retrieve papers from 2018 and 2024 with equal weight if the year is only in metadata. Self-querying explicitly routes the date constraint through the metadata filter.

### 3.2 Multi-query retrieval — N variants to defend against single-phrasing brittleness

A multi-query retriever asks the LLM to **rewrite** the user query as N alternative phrasings (typically 3–5), runs vector search for each variant in parallel, and fuses the result sets. The defence is against single-phrasing brittleness — the user's exact wording may not match the corpus's exact wording.

```
user query: "how do I make my SaaS app GDPR compliant?"

variants:
  1. "GDPR compliance requirements for SaaS applications"
  2. "data privacy regulation implementation for software services"
  3. "European data protection law compliance steps"
```

Each variant retrieves its own top-K; the union is deduplicated and passed forward. Cost is N × the baseline vector-search cost plus one LLM call to generate the variants.

**The failure mode it fixes.** Vocabulary mismatch — the user said "compliance" but the canonical doc says "data protection law."

### 3.3 RAG Fusion — multi-query + Reciprocal Rank Fusion

RAG Fusion (Raudaschl 2024, arXiv 2402.03367) extends multi-query retrieval with **Reciprocal Rank Fusion (RRF)** as the merge step. Instead of unioning the result sets, RRF computes a fused score from the ranks each document appears at across the N variant searches:

```
RRF_score(doc) = Σ_variant   1 / (k + rank_in_variant(doc))
```

with `k` typically 60. Documents that rank high in multiple variants float to the top; documents that rank high in only one variant are deprioritised. The signal is robust because it depends on the relative ordering across variants, not on score magnitudes — which is essential when you are fusing results from different retrievers or indexes that score on different scales.

**Why ranks not scores.** Vector-search scores from different indexes (BM25 vs cosine vs dot product vs hybrid) are not directly comparable. Two systems may both return "highly relevant" but one returns a score of 0.92 and the other a score of 4.7. RRF sidesteps this by ignoring the score magnitudes entirely.

The Raudaschl repository ships an evaluation harness against NFCorpus/BEIR; the reported uplift over a baseline is +22% NDCG@5 and +40% recall@10 when RRF is combined with hybrid (BM25 + vector) search.

### 3.4 CRAG — Corrective RAG with a quality evaluator

Corrective RAG (Yan et al. 2024, arXiv 2401.15884) sits one rung up the ladder. After retrieval, an LLM-judge evaluator scores the retrieved documents on whether they actually answer the question. The evaluator emits one of three labels:

- **Correct.** The retrieval is reliable. Use the documents directly (or refine via a decompose-then-recompose step).
- **Ambiguous.** Confidence is in the middle band. Augment with an alternative retrieval source — typically a web search — and combine.
- **Wrong.** The retrieval is unreliable. Fall back entirely to the alternative source (web, KG, or a rewritten query).

The original paper uses a lightweight T5-based evaluator; production implementations more commonly use a small LLM call with a structured-output prompt. The cost is one extra LLM call per retrieval. The benefit is that hallucinated answers — which usually trace back to the model trying to answer from irrelevant retrieved context — are substantially reduced.

**The failure mode it fixes.** Retrieval-quality blindness — the baseline RAG pipeline has no way to tell whether its top-K is actually relevant; CRAG adds a sanity check.

### 3.5 The ladder — cost, complexity, when to escalate

| Pattern | LLM calls per query | Vector searches per query | Use when |
|---------|---------------------|---------------------------|----------|
| Baseline RAG | 1 (generation) | 1 | Top-K consistently usable |
| Self-querying | 2 (parse + generation) | 1 | Queries carry implicit metadata filters |
| Multi-query | 2 + N×0 (just N searches) | N | Vocabulary mismatch is a failure mode |
| RAG Fusion | 2 (rewrite + generation) | N | You need both vocabulary defence and ranking robustness |
| CRAG | 3+ (retrieve + evaluate + maybe re-retrieve + generate) | 1 + (maybe 1 fallback) | Hallucination on irrelevant retrieval is a measured problem |

The discipline is empirical: instrument your baseline, identify the failure mode that hurts your users most, escalate to the pattern that addresses *that* failure mode. Stacking all four patterns on every query is pure cost without proportional uplift.

### 3.6 Where these patterns live in the agent loop

In an agentic-RAG implementation, the patterns above are **tools the agent can call** rather than fixed pipeline stages. The agent receives the user question, reasons about which retrieval shape fits, calls the appropriate tool, evaluates the result, and either answers or re-retrieves. LangGraph models this as a directed cyclic graph with conditional branching and persistent checkpoints; the agent traverses the graph at runtime. The advantage over a fixed pipeline: simple queries pay the baseline cost; only hard queries trigger the expensive patterns.

## 4. Generic Implementation

A generic self-querying retriever shape, expressed independent of LangChain's specific API. Imagine a product-search service for a generic e-commerce site.

```python
from pydantic import BaseModel, Field
from typing import Literal, Optional

class ProductSearchQuery(BaseModel):
    semantic_query: str
    price_max: Optional[float] = Field(default=None, ge=0)
    price_min: Optional[float] = Field(default=None, ge=0)
    category: Optional[str] = None
    in_stock_only: bool = False
    rating_min: Optional[float] = Field(default=None, ge=0, le=5)

def parse_query_to_structured(natural_query: str, llm) -> ProductSearchQuery:
    """
    LLM call that decomposes the user's natural-language query into a structured
    search object. The LLM's job is *just* the parse — no semantic understanding
    of the corpus, just intent extraction.
    """
    system_prompt = (
        "Parse the user's product search into a ProductSearchQuery. "
        "Extract any price, category, stock, or rating constraints explicitly. "
        "Put the remaining intent in semantic_query."
    )
    return llm.structured_output(ProductSearchQuery, system=system_prompt, user=natural_query)

def search(natural_query: str, llm, vector_store) -> list:
    parsed = parse_query_to_structured(natural_query, llm)

    metadata_filter = {}
    if parsed.price_max is not None: metadata_filter["price"] = {"$lte": parsed.price_max}
    if parsed.price_min is not None: metadata_filter.setdefault("price", {})["$gte"] = parsed.price_min
    if parsed.category: metadata_filter["category"] = parsed.category
    if parsed.in_stock_only: metadata_filter["in_stock"] = True
    if parsed.rating_min is not None: metadata_filter["rating"] = {"$gte": parsed.rating_min}

    return vector_store.similarity_search(
        query=parsed.semantic_query,
        filter=metadata_filter,
        k=20,
    )
```

A generic RRF implementation against the result sets from a multi-query search:

```python
def reciprocal_rank_fusion(result_lists: list[list[str]], k: int = 60) -> list[tuple[str, float]]:
    """
    result_lists: a list of ranked document-id lists, one per query variant.
    Returns documents sorted by RRF score.
    """
    fused: dict[str, float] = {}
    for ranking in result_lists:
        for rank, doc_id in enumerate(ranking):
            fused[doc_id] = fused.get(doc_id, 0.0) + 1.0 / (k + rank + 1)
    return sorted(fused.items(), key=lambda kv: kv[1], reverse=True)
```

Three things worth noticing in these snippets:

1. **The parse is a separate LLM call from the generation.** Self-querying does not embed the structured-output parse inside the answer-generation prompt; keeping them separate lets you cache, test, and observe each independently.
2. **The metadata-filter shape is vector-store-specific.** Mongo, Pinecone, Qdrant, and Weaviate each have their own filter dialect. Production implementations either standardise on one or wrap with an adapter.
3. **RRF treats every variant equally.** Confidence-weighted variants (the Weighted RRF mechanism in recent 2026 papers) are a refinement; the unweighted shape is the right starting point.

## 5. Real-world Patterns

### 5.1 E-commerce — Etsy, Wayfair, Shopify product search

E-commerce product search is the canonical self-querying use case. Shoppers type *"blue dress under $50 for a summer wedding"*, and a baseline vector search returns dresses of every colour and price. Etsy, Wayfair, and Shopify's storefront search products all use LLM-driven parsing to extract `colour=blue`, `price<=50`, `occasion=wedding`, `season=summer` as metadata filters before the semantic search. The lift is enormous on long-tail queries that no static facet UI could reasonably anticipate.

### 5.2 Legal research — Casetext CoCounsel, Harvey, Lexis+ AI

Legal-research assistants face a brutal vocabulary-mismatch problem — the lawyer asks about "negligent misrepresentation by an architect" and the relevant case law uses "professional negligence by a design professional." Multi-query retrieval with LLM-generated synonym variants is the default pattern. RAG Fusion with RRF then merges results from variant searches across both the case-law index and the secondary-source index.

### 5.3 Customer support — Zendesk Resolve, Intercom Fin

Support-bot vendors layered CRAG on top of baseline RAG in 2025–2026 to control the hallucination rate. The evaluator step looks at the top retrieved KB articles and decides "yes this answers the user's question" / "partially, but I need more" / "no, escalate to a human." The "no" branch is critical — without it, the bot produces a confident wrong answer that costs more than the deflection saved.

### 5.4 Academic research — Elicit, Consensus, Scite

Research-assistant tools rely on self-querying for the metadata side (publication date, journal, citation count) and multi-query/fusion for the semantic side (the same research question phrased five different ways to match different communities' jargon). The combination is documented in product blogs as a meaningful uplift on the "I asked about X but the canonical paper uses term Y" failure mode.

### 5.5 Internal enterprise knowledge — Glean, Microsoft Copilot

Enterprise search products use the full ladder. Self-querying handles "documents from the marketing team last quarter." Multi-query handles "policy on remote work" → "WFH policy" → "telecommuting guidelines." RRF handles fusion across SharePoint, Confluence, Slack, and email indexes. CRAG handles the high-stakes legal/compliance queries where a hallucinated answer is unacceptable.

## 6. Best Practices

- **Instrument the baseline before escalating.** Without metrics on which queries fail and why, you cannot tell which pattern to add.
- **Cache the parse step in self-querying.** The same natural-language query produces the same structured query; deterministic caching is a free latency win.
- **Use RRF over score-based fusion when combining results from heterogeneous retrievers.** Ranks compose; scores do not.
- **Calibrate the CRAG evaluator thresholds against ground-truth data.** A miscalibrated evaluator either lets bad retrievals through ("correct" too eagerly) or burns budget on unnecessary fallbacks ("wrong" too eagerly).
- **Reach for the cheapest pattern that addresses the failure mode.** Adding all four to every query is pure cost.
- **Make the agent's retrieval choice observable.** When the agent picks "multi-query with fusion," the trace should record *why* — what feature of the query triggered the choice.
- **Treat the retrieval evaluator's labels as data.** Periodically review the "wrong" cases to find systematic gaps in your corpus.

## 7. Hands-on Exercise

**Whiteboard exercise (15 min).** You are designing the retrieval layer for an internal knowledge base at a global logistics company. Users include drivers, dispatchers, and corporate operations. Common queries:

- *"What's the protocol for a refrigeration failure on a perishable shipment?"* (procedure lookup)
- *"Show me all incident reports from EMEA depots in 2025 involving forklift damage."* (filtered list)
- *"Why are our Helsinki on-time delivery rates dropping?"* (diagnosis — multi-source)
- *"Is there a recent policy update on dangerous-goods labeling?"* (recency-sensitive)

For each query, decide:

1. Which retrieval pattern best fits — baseline, self-querying, multi-query, RAG Fusion, or CRAG?
2. What metadata fields are needed on the underlying corpus to support the pattern?
3. What is the failure mode the pattern is defending against?

**What good looks like.** A solution maps query 1 to baseline (single procedure, clear semantics), query 2 to self-querying (`region=EMEA`, `year=2025`, `incident_type=forklift_damage`), query 3 to multi-query + RAG Fusion across multiple corpus partitions (operations logs + incident reports + customer feedback), and query 4 to self-querying with a date filter plus possible CRAG to verify the retrieved policy is actually the latest one. Bonus: the writer identifies that query 4 needs corpus-level versioning so "latest" is unambiguous.

## 8. Key Takeaways

- **What are the four agentic-RAG patterns, and what failure mode does each address?** (Self-querying — metadata implicit in natural language; multi-query — vocabulary mismatch; RAG Fusion — fusing heterogeneous retrievers; CRAG — retrieval-quality blindness.)
- **Why does Reciprocal Rank Fusion use ranks instead of scores?** (Scores from different retrievers are not on the same scale; ranks are.)
- **What are CRAG's three confidence branches?** (Correct — use as-is; ambiguous — augment with fallback; wrong — replace with fallback.)
- **How do you decide which pattern to escalate to?** (Instrument the baseline, identify the dominant failure mode, escalate to the pattern that addresses it.)
- **Why is agentic-RAG a graph rather than a fixed pipeline?** (Simple queries pay baseline cost; only the queries that need the expensive patterns trigger them.)

## Sources

1. [Self-querying retrieval — LangChain](https://js.langchain.com/docs/how_to/self_query/) — retrieved 2026-05-26
2. [RAG-Fusion: multi-query + Reciprocal Rank Fusion — Raudaschl GitHub](https://github.com/Raudaschl/rag-fusion) — retrieved 2026-05-26
3. [RAG-Fusion: A New Take on Retrieval-Augmented Generation (Raudaschl 2024, arXiv 2402.03367)](https://arxiv.org/abs/2402.03367) — retrieved 2026-05-26
4. [Corrective Retrieval Augmented Generation (Yan et al. 2024, arXiv 2401.15884)](https://arxiv.org/abs/2401.15884) — retrieved 2026-05-26
5. [Agentic Retrieval-Augmented Generation: A Survey (arXiv 2501.09136)](https://arxiv.org/html/2501.09136v4) — retrieved 2026-05-26
6. [Advanced RAG — Understanding Reciprocal Rank Fusion in Hybrid Search (Guillaume Laforge)](https://glaforge.dev/posts/2026/02/10/advanced-rag-understanding-reciprocal-rank-fusion-in-hybrid-search/) — retrieved 2026-05-26

Last verified: 2026-05-26
