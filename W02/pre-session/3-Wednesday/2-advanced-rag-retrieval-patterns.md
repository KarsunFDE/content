---
week: W02
day: Wed
topic_slug: advanced-rag-retrieval-patterns
topic_title: "Parent-child chunking + multi-query + contextual compression — three patterns for when top-k fails"
parent_overview: W02/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 15
sources:
  - url: https://medium.com/data-science/advanced-rag-01-small-to-big-retrieval-172181b396d4
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://graphrag.com/reference/graphrag/parent-child-retriever/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://medium.com/@ThinkingLoop/rag-chunking-9-strategies-that-stop-lost-context-b4777df4c908
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://dev.to/sreeni5018/multi-query-retriever-rag-how-to-dramatically-improve-your-ais-document-retrieval-accuracy-5892
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.langchain.com/blog/improving-document-retrieval-with-contextual-compression
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Parent-child chunking + multi-query + contextual compression — three patterns for when top-k fails

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the three retrieval-failure modes (context loss, vocabulary mismatch, token bloat) and match each to the pattern that addresses it.
- Sketch the data model that a parent-child retriever requires (child documents indexed for similarity, parent payload returned to the LLM).
- Decide between `EmbeddingsFilter` and an LLM-based extractor for a given quality/cost budget without invoking the v0.x `Chain` class.
- Identify when multi-query retrieval helps and when it hurts (the precision-recall trade-off that requires pairing with a reranker).
- Compose the three patterns as plain Python steps rather than as framework-mediated pipelines.

## 2. Introduction

Naive top-k retrieval — embed the query, kNN-search the corpus, return the top five chunks — is a strong baseline. It is also, on production corpora, a strong source of failure. The three patterns in this reading are the industry's most-cited responses to the three most-cited failure modes.

The first failure mode is **context loss**. A retrieval system that splits a document into 200-token chunks for precision retrieves a tightly-scoped chunk that, in isolation, reads "(2) Notwithstanding paragraph (1), the Contractor shall…" The chunk is the right neighborhood of the document but the wrong unit to hand to the LLM. Parent-child indexing — sometimes called "small-to-big retrieval" — separates the unit of similarity match from the unit of generation payload.

The second failure mode is **vocabulary mismatch**. A user asks a colloquial question; the corpus uses a different register, a different abbreviation, a different framing. The dense embedding's cosine similarity collapses because the lexical overlap is near-zero. Multi-query retrieval generates paraphrases of the user query and unions their retrieval sets, trading precision for recall and leaning on a downstream reranker to put the precision back.

The third failure mode is **token bloat**. The retrieved chunks contain mostly irrelevant material — boilerplate cross-references, table-of-contents headers, footnotes that drag in 600 tokens for a one-sentence claim. Contextual compression runs a filtering pass after retrieval and before generation, dropping sentences that don't carry signal for this particular query. The filter can be embedding-based (cheap) or LLM-based (better quality, expensive).

The three patterns compose. A production RAG system can run all three on the same query, in sequence, as plain Python steps. The composition shape — small embedding match → parent payload → multi-query expansion → contextual compression — is the substrate the rest of the week's eval and observability work measures against.

## 3. Core Concepts

### Parent-child indexing (small-to-big)

The data model has two stores. A **child store** holds small chunks (100–500 tokens) embedded for similarity match. A **parent store** holds larger chunks (500–2000 tokens) — the units actually sent to the LLM. Each child carries a `parent_ref` foreign key.

The retrieval loop is two-step. Vector search runs against the child collection and returns the top-k child IDs. A second lookup (in MongoDB Atlas, a `$lookup` stage chained onto the `$vectorSearch` aggregation; in a relational store, a JOIN; in a key-value store, a batched MGET) hydrates the matched children into their parents, deduplicating so the same parent doesn't appear twice.

Sizing is corpus-dependent. A common starting point: child chunks at 256 tokens with 32-token overlap; parents at 1024 tokens. The cost is storage (the parent text duplicates content that already lives in the child) and a small additional read latency from the lookup step. The benefit is generation quality on documents where the relevant span is small but its meaning depends on the surrounding paragraph.

### Multi-query retrieval

A user query goes to a cheap LLM (a fast model like Claude Haiku, Llama 3 8B, or any inexpensive instruction-following model) with a prompt of the form: *"Generate 3–5 alternative phrasings of this query that preserve its intent. Output JSON."* Each variant query embeds independently and runs through the vector store. The retrieval sets union, then deduplicate by document ID.

The trade-off is sharp. Recall improves — queries that miss with the original phrasing often hit on a paraphrase. Precision drops — the union pulls in more candidates, including more wrong candidates. Multi-query retrieval should never be deployed without a reranker downstream (the topic of the next reading). The combined pattern is: multi-query → union retrieval set → cross-encoder rerank → top-k to LLM.

The cost is one extra small-LLM call per user query plus N extra vector searches (where N is the paraphrase count). On modern serverless vector stores this is a sub-second addition. On a self-hosted setup with cold connection pools, measure before assuming the latency is free.

### Contextual compression

Contextual compression sits between the retriever and the generator. After top-k retrieval, run a filter pass that — for each retrieved chunk — decides which sentences or sub-paragraphs to keep and which to drop.

Two filter implementations dominate. An **embeddings filter** embeds each candidate sentence and the query, computes cosine similarity, and drops sentences below a threshold. This is cheap (one embedding model call, batchable) and works well when the retrieved chunks contain a handful of obviously-irrelevant sentences. An **LLM-based extractor** sends each chunk plus the query to a small LLM with a prompt of the form *"Return only the sentences from this chunk that are directly relevant to the query"*. This is more expensive but handles harder cases — semantic relevance that doesn't show up in embedding distance.

> [!instructor-review]
> Many tutorials for these compression filters were written in the LangChain v0.x era and instantiate them as part of an `LLMChain` or use the `RetrievalQA.from_chain_type(...).run(query)` entry point. Both patterns are flagged in `known-bad-patterns.yml` (`langchain-chain-class`, `langchain-chaining-verb`). The v1.0 substitute is to call each component's `.invoke()` method directly from plain Python — no `Chain` class, no pipe operator as foundation. If a learner finds a v0.x example in the afternoon `/web-research` slot, the correct response is a plain-Python translation, not a copy-paste.

### How the three compose

A production composition for a hard query looks like this, in plain Python pseudocode:

```
variants = expand_query(user_query, n=4)
candidate_ids = []
for v in variants:
    candidate_ids.extend(child_vector_search(v, k=20))
candidate_ids = deduplicate(candidate_ids)
parents = hydrate_to_parents(candidate_ids)
filtered = compress_against_query(parents, user_query)
top_k = rerank(filtered, user_query, k=5)
answer = generate(top_k, user_query)
```

Each step is a normal function. There is no framework abstraction binding them. The retriever returns documents; the compressor takes documents and a query; the reranker takes documents and a query; the generator takes documents and a query. Each one is independently testable, independently swappable, and independently observable.

## 4. Generic Implementation

A minimal parent-child retriever, framework-agnostic and outside the federal-acquisitions domain. Imagine indexing product reviews for an e-commerce search system — each review is a parent document, and each sentence inside the review is a child chunk.

```python
# Two collections: 'review_children' (embedded), 'review_parents' (payload)
# Each child carries: { _id, parent_id, sentence_text, embedding }
# Each parent carries: { _id, full_review_text, product_id, rating }

def parent_child_retrieve(query: str, k: int = 5) -> list[dict]:
    # Step 1: embed query
    query_vec = embedding_model.embed(query)

    # Step 2: vector search children (precise match on sentences)
    child_hits = vector_db.search(
        collection="review_children",
        query_vector=query_vec,
        top_k=k * 4,           # over-fetch to allow parent dedup
    )

    # Step 3: collect unique parent IDs in match order
    parent_ids = []
    seen = set()
    for hit in child_hits:
        pid = hit["parent_id"]
        if pid not in seen:
            parent_ids.append(pid)
            seen.add(pid)
        if len(parent_ids) >= k:
            break

    # Step 4: hydrate parents (full reviews, not the matched sentence)
    parents = parent_db.find_many(
        collection="review_parents",
        ids=parent_ids,
    )

    # Step 5: preserve match order
    by_id = {p["_id"]: p for p in parents}
    return [by_id[pid] for pid in parent_ids if pid in by_id]
```

The shape is database-agnostic. Swap in pgvector, Atlas, Weaviate, or Qdrant for the children and any KV store for the parents. The composition is plain Python.

## 5. Real-world Patterns

**E-commerce product search (Wayfair, Shopify-style stores).** A 2024 talk from an e-commerce search team described their move from naive chunking on product descriptions to a parent-child pattern: sentence-level children indexed for similarity ("waterproof, machine washable, fits a queen bed"), product-level parents returned to the ranking model. Conversion-rate experiments showed measurable lift over the baseline; the failure mode they were solving was the same context-loss pattern — embedding hit on the right product, generation lacked context to compose a relevant answer.

**Healthcare clinical decision support.** A health-tech company building a clinician-facing literature search tool published an internal write-up describing a multi-query layer. Clinicians ask in vernacular ("what's the dose for a kid"); the literature uses precise terminology ("pediatric dosing for patients aged 2–12"). Their multi-query expander generated three medical-terminology variants of each query and unioned the retrieval set; a domain-tuned cross-encoder reranker put precision back. Recall@10 improved meaningfully on their internal eval set without unacceptable latency cost.

**Fintech earnings-call transcript search.** A quantitative-trading firm published a blog post describing contextual compression on earnings-call transcripts. Raw retrieved chunks were 2k tokens with most of the budget consumed by analyst pleasantries and disclaimer boilerplate. An `EmbeddingsFilter` against the analyst's actual question dropped 60–70% of the tokens before they reached the summarization LLM, cutting generation cost by roughly the same fraction and improving signal-to-noise in the summary.

**Gaming player-support knowledge base.** A large multiplayer-game studio described combining all three patterns: small chunk embeddings of FAQ entries, multi-query expansion to handle slang ("how do I rez my teammate"), parent-paragraph payloads with the full procedure, and contextual compression to drop the irrelevant platform-specific sub-bullets when a query came from a specific platform. The composition was several plain Python functions, called in sequence; no framework abstraction held them together.

## 6. Best Practices

- Index children at the unit of *meaning* (a sentence, a clause, a step) and return parents at the unit of *generation context* (a paragraph, a section, a procedure). Pick the boundaries by reading 20 examples of the corpus, not by tuning blindly.
- Always pair multi-query expansion with a downstream reranker. Without one, you'll see retrieval@10 improve and answer quality stay flat — the LLM can't distinguish the union's noise from its signal.
- Cap paraphrase count at 3–5. Beyond that you're paying for redundant work and the union becomes too noisy to rerank.
- Use the embeddings-based compression filter as the default; reach for an LLM-based extractor only when the embeddings filter measurably under-performs on your eval set. The cost difference is often an order of magnitude.
- Log every stage's input and output count (queries in, candidates out per variant, parents after dedup, sentences after compression, top-k after rerank). The observability surface is what tells you which stage is failing when answer quality regresses.
- Never frame any of these patterns as "a Chain you build". They are functions that take documents and a query and return documents and a query. The composition shape is your code, not a framework's.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** You are designing the retrieval layer for a customer-support knowledge base for a B2B SaaS company. The corpus is 4,000 internal help-center articles, each 500–3,000 words. Users send free-text questions through a chat widget; the system retrieves and the LLM answers. You have a budget of 1.5s p95 latency end-to-end and a per-query cost ceiling of $0.005.

On a whiteboard, draw the retrieval composition. Label each box with (1) the operation it performs, (2) the data store or model it calls, (3) the input it accepts, and (4) the output it produces. Mark at least one decision point where you would A/B-test a variant. Identify the stage you'd remove first if you needed to halve the per-query cost.

**Expected components.** Query embedding, child-chunk vector search, parent hydration, optional multi-query expansion, optional contextual compression, reranker, top-k to LLM. At least one cache layer with a clear cache key. An observability hook between each stage logging counts.

**What good looks like.** Each stage has a measurable input/output count. The drawing shows where you'd add multi-query (only with a reranker downstream) and where you'd add compression (after retrieval, before generation). The decision point names a hypothesis (e.g., "child chunk size 256 vs 512 — measure on recall@5 against the human-labeled eval set"). The "cut for cost" answer correctly names the LLM-based compression extractor or the multi-query expansion (the two most expensive layers).

## 8. Key Takeaways

- *What failure modes do parent-child, multi-query, and contextual compression each address?* — Context loss, vocabulary mismatch, and token bloat respectively. Match the pattern to the symptom; don't deploy all three reflexively.
- *Why does multi-query retrieval require a reranker?* — Because the union of retrieval sets trades precision for recall; the reranker is what restores precision before the LLM call.
- *What is the v1.0-correct way to compose these patterns?* — Plain Python function calls, with each component's `.invoke()` or equivalent method called directly. No `Chain` class, no LCEL pipe operator as foundation, no `.run()` entry points.
- *What is the data model for parent-child indexing?* — Two collections, with children embedded for similarity and parents hydrated as payload, linked by a `parent_ref` foreign key.
- *How do you decide between an embeddings filter and an LLM extractor for compression?* — Default to embeddings (cheap, batchable, good enough on most corpora). Reach for an LLM extractor only when the embeddings filter measurably underperforms on the eval set.

## Sources

1. [Advanced RAG 01: Small-to-Big Retrieval (Sophia Yang)](https://medium.com/data-science/advanced-rag-01-small-to-big-retrieval-172181b396d4) — retrieved 2026-05-26
2. [Parent-Child Retriever reference (GraphRAG docs)](https://graphrag.com/reference/graphrag/parent-child-retriever/) — retrieved 2026-05-26
3. [RAG Chunking: 9 Strategies That Stop "Lost Context" (Thinking Loop, Feb 2026)](https://medium.com/@ThinkingLoop/rag-chunking-9-strategies-that-stop-lost-context-b4777df4c908) — retrieved 2026-05-26
4. [Multi-Query Retriever RAG explainer (DEV Community)](https://dev.to/sreeni5018/multi-query-retriever-rag-how-to-dramatically-improve-your-ais-document-retrieval-accuracy-5892) — retrieved 2026-05-26
5. [Improving Document Retrieval with Contextual Compression (LangChain blog)](https://www.langchain.com/blog/improving-document-retrieval-with-contextual-compression) — retrieved 2026-05-26

Last verified: 2026-05-26
