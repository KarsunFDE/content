---
week: W02
day: Tue
topic_slug: reranking-fundamentals
topic_title: "Re-ranking fundamentals"
parent_overview: W02/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 8
sources:
  - url: https://docs.cohere.com/docs/rerank-overview
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://localaimaster.com/blog/reranking-cross-encoders-guide
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://teachmeidea.com/reranking-in-rag-cohere-cross-encoders/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://docs.bswen.com/blog/2026-02-25-best-reranker-models/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://bigdataboutique.com/blog/rag-reranking-improving-retrieval-quality-with-cross-encoders
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
last_verified: 2026-05-26
---

# Re-ranking fundamentals

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish a bi-encoder retriever from a cross-encoder reranker and explain the cost–accuracy trade-off between them.
- Explain why the "retrieve 50, rerank 5" cascade is the standard production pattern, and what changes when you exit those numbers.
- Compare managed-API rerankers (e.g., Cohere Rerank-3) with self-hosted open-weight rerankers (e.g., BGE-reranker-v2-m3) on the dimensions that actually matter at procurement time.
- Recognise when reranking is worth its latency cost and when it isn't.

## 2. Introduction

Retrieval gives you candidates; reranking gives you **confidence** in the small subset of candidates that reaches the language model. The point of a reranker is not to find new candidates — it is to take an over-broad candidate list (say, top-50 from hybrid retrieval) and produce a tighter, better-ordered top-k (say, top-5) for the LLM to ground on.

Why bother? Because retrievers and rerankers solve different problems with different architectures. A retriever is a **bi-encoder**: the query and each document get embedded independently and compared by a cheap similarity metric. That independence is what makes ANN search fast — you precompute every document embedding once. But it also means the retriever never directly examines a `(query, document)` *pair*. A cross-encoder does exactly that: it concatenates the query and a candidate document and feeds the pair through a neural network, scoring how well they actually match. The cross-encoder is more accurate because it can model fine-grained relevance signals that bi-encoders cannot — but more expensive because it cannot be precomputed ([BigData Boutique — RAG Reranking with Cross-Encoders](https://bigdataboutique.com/blog/rag-reranking-improving-retrieval-quality-with-cross-encoders), retrieved 2026-05-26).

This reading is the architecture, the canonical cascade, the model landscape, and the latency budget. The day's overview names the specific candidates the day's pair-projects will choose between; this reading gives you the framework to defend whichever choice you commit to.

## 3. Core Concepts

### Bi-encoder vs cross-encoder

A **bi-encoder** is what your retriever uses. The query `q` and each document `d` are embedded separately:

```
e_q = encode(q)      e_d = encode(d)         score = sim(e_q, e_d)
```

The encoded document vectors are computed once at indexing time and reused for every query. This is what makes the lookup fast (sub-millisecond per query at the ANN layer). It is also why the retriever cannot model fine-grained relevance — the model never sees `q` and `d` together.

A **cross-encoder** is what your reranker uses. The query and the document are concatenated and fed through the model together:

```
score = cross_encoder.score(query=q, document=d)
```

The model now has direct access to interactions between query tokens and document tokens. Attention can model "does this specific token in the query line up with this specific token in the document?" That makes a cross-encoder dramatically more accurate at relevance scoring — and dramatically more expensive, because the model has to run a fresh forward pass for every `(q, d)` pair you want to score.

### The cascade — retrieve 50, rerank 5

The cost asymmetry above is why production RAG pipelines use a **cascade**:

```
hybrid retrieve  →  top-50 candidates  →  cross-encoder rerank  →  top-5  →  LLM
```

You let the cheap bi-encoder retriever cast a wide net over the corpus (`k = 50` is the field-standard default), and you let the expensive cross-encoder reranker score only those 50 — not the whole corpus. The reranker pays attention where it matters, and the retriever pays attention everywhere ([TeachMeIdea — Reranking in RAG: Cohere Rerank and Cross-Encoders](https://teachmeidea.com/reranking-in-rag-cohere-cross-encoders/), retrieved 2026-05-26).

Why 50? Below ~20 candidates the reranker has too narrow a pool — if the right document is not in the top-20 of the retriever, the reranker cannot pick it. Above ~100 candidates the reranker latency starts to dominate and the marginal recall gain flattens. 50 is the empirical sweet spot most reported benchmarks settle on.

Why 5 (or thereabouts) for the LLM? Because of the context-dilution failure mode covered in the "Why naive RAG breaks" reading. Past ~5–8 chunks the LLM's attention thins; you start trading away the quality you just paid for with the reranker.

### Reranker model landscape

There are three live patterns in 2026:

**Managed-API cross-encoder rerankers.** Cohere Rerank-3 / Rerank-3.5 is the dominant managed offering — pay per call, the vendor runs the model, English regulatory and enterprise text is in scope. Latency is in the 100–300ms range per request depending on candidate count and document length ([Cohere — Rerank Overview](https://docs.cohere.com/docs/rerank-overview), retrieved 2026-05-26). Jina and Voyage offer similar managed reranker APIs at comparable latency profiles.

> [!instructor-review]
> **D-060 scope on Cohere Rerank on Bedrock.** Cohere Rerank 3.5 is available as a managed inference endpoint on AWS Bedrock (single API call, no agent loop, no managed retrieval store). D-060 explicitly permitted real Bedrock `InvokeModel` for Titan and Claude in W2 but deferred AWS managed RAG services (Bedrock Knowledge Bases, Agents-for-Bedrock, OpenSearch Managed) to W5. Confirm with cohort lead whether managed-reranker invocation falls under the W2-permitted Bedrock surface or the W5-deferred managed-AI surface. **Working default for now:** managed-reranker invocation (a single Bedrock `InvokeModel`-equivalent call against the Rerank API, no agent loop, no managed retrieval) is treated as **W2-permitted**; managed retrieval (Bedrock Knowledge Bases) stays **W5-deferred**. If cohort lead overrules, swap to self-hosted BGE-reranker-v2-m3 for W2 and re-introduce the managed reranker in W5.

**Self-hosted open-weight cross-encoders.** BGE-reranker-v2-m3 from BAAI is the open-weight reference. It runs on a self-hosted GPU (Bedrock, Sagemaker, or local), typically ~50–100ms per 50-candidate batch on GPU and ~130ms per 16-pair batch on CPU ([Local AI Master — Reranking & Cross-Encoders for RAG 2026](https://localaimaster.com/blog/reranking-cross-encoders-guide), retrieved 2026-05-26). No per-call cost beyond infra, but you carry the GPU. On standard reranking benchmarks BGE-reranker-v2-m3 is competitive with Cohere Rerank-3 ([BSWEN — Best Reranker Models for RAG 2026](https://docs.bswen.com/blog/2026-02-25-best-reranker-models/), retrieved 2026-05-26).

**In-prompt LLM reranking.** Send the candidates to a general-purpose LLM and ask it to score them ("rate each of these passages 1–5 for relevance to the query"). Most flexible — you can encode arbitrary preferences in the prompt. Slowest — adds a full LLM round-trip to the retrieval path. Easiest to abuse — costs and latency creep silently if you do not budget per-query cost. Useful as a fallback or a custom-criteria layer; not a default.

### Latency budget — when reranking is worth it

A cross-encoder rerank over 50 candidates adds roughly 100–500ms to your retrieval call depending on the model and the hosting. For a user-facing chatbot with a 2-second total budget that is a real chunk. For an internal analyst workflow with a 5-second budget it is essentially free.

The question to ask: *what is the cost of a wrong answer vs the cost of an extra 300ms?* In high-stakes domains (medical, legal, regulatory, financial advice) the relevance lift from reranking is worth the latency every time. In low-stakes conversational interfaces the answer may be different.

Reranking is not free in dollar terms either. Managed APIs price per 1k searches; self-hosted models price per GPU-hour. Both are small compared to the LLM call that follows, but worth modeling in the cost-per-request budget.

### When to skip reranking

A cross-encoder reranker is not always the right call:

- If your retriever is already producing the right top-5 (measured on a held-out QA set), reranking adds latency without measurable lift.
- If your queries are dominated by short exact-token lookups (think: support article lookup by error code), BM25 is usually doing the work and reranking offers little.
- If you have strict latency constraints (sub-300ms total budget), the reranker may simply not fit — explore a stronger retriever instead.

The discipline is to measure, not to assume. Reranking is the right default for heterogeneous queries over a moderately-sized corpus with stakes attached; it is not the right default for everything.

## 4. Generic Implementation

A minimal reranker integration in plain Python — managed-API flavour (using a generic vendor-neutral interface):

```python
from dataclasses import dataclass

@dataclass
class Candidate:
    doc_id: str
    text: str
    retrieval_score: float

def rerank(query: str, candidates: list[Candidate], top_k: int = 5) -> list[Candidate]:
    """Cross-encoder rerank — single batched call to the reranker API."""
    # Step 1: extract just the texts to send to the reranker — keep doc_ids local
    texts = [c.text for c in candidates]

    # Step 2: single batched API call — vendor scores each (query, text) pair
    rerank_response = reranker_client.score(
        query=query,
        documents=texts,
        top_n=top_k,
    )

    # Step 3: map the reranker's indices back to the original Candidates
    reranked = []
    for hit in rerank_response.results:           # iterate in score-descending order
        c = candidates[hit.index]                 # vendor returns positional index
        c.rerank_score = hit.relevance_score      # keep both scores for diagnostics
        reranked.append(c)
    return reranked

# Caller integrates it into the cascade as plain function composition
def search_with_cascade(query: str) -> list[Candidate]:
    initial = hybrid_search(query, k_each=50, k_final=50)   # wide net
    final   = rerank(query, initial, top_k=5)               # tight, accurate top-5
    return final
```

Notes on what this is doing:

- The reranker receives the original query and the raw document texts. It does not see the retriever's score; it does not need to.
- One batched API call rather than 50 sequential calls — most reranker APIs accept a list of documents and return scores for all of them.
- The retrieval score is preserved on the `Candidate` for diagnostics — never use it directly in the final ranking; the rerank score is the authority.
- The cascade is plain function composition: `rerank(query, hybrid_search(query, ...))`. No framework abstractions earn their place.

## 5. Real-world Patterns

**Legal-research SaaS.** A legal-tech platform serving litigators reported that adding Cohere Rerank-3 in front of their LLM lifted top-3 relevance measurably on a complex-query subset where dense-only retrieval was returning semantically-adjacent but legally-irrelevant precedents. The reranker latency (~250ms) was acceptable for a research-tool UX where users expect 1–2 second waits ([TeachMeIdea — Reranking in RAG](https://teachmeidea.com/reranking-in-rag-cohere-cross-encoders/), retrieved 2026-05-26).

**Enterprise search at a software-services consultancy.** A large IT-services consultancy added BGE-reranker-v2-m3 (self-hosted on a single GPU) to their internal-document search to avoid per-query API costs and to keep client data inside their own VPC. They reported the self-hosted route was cost-favourable above ~10,000 queries/day; below that managed API was cheaper because of the constant-cost GPU ([BSWEN — Best Reranker Models for RAG 2026](https://docs.bswen.com/blog/2026-02-25-best-reranker-models/), retrieved 2026-05-26).

**Healthcare patient-portal Q&A.** A regional healthcare system deployed a patient-facing Q&A bot grounded in their public-facing patient education library. They initially shipped without reranking; user feedback showed the bot was confidently surfacing topically-related but condition-mismatched articles. Adding a reranker (managed API to avoid GPU operations) closed the gap, and the team reported that the reranker also helped them tighten `k` from 8 to 4 — a context-dilution improvement that came as a side-benefit of better top-k quality.

**E-commerce recommendation re-ranking.** A retail platform used in-prompt LLM reranking for a personalised-recommendation use case where the relevance criteria were genuinely soft and customer-specific ("the user clicks on items with X aesthetic, finds Y category interesting, dislikes Z brand"). They reported that the prompt-encoded criteria changed by season and by campaign, which made a managed cross-encoder reranker too rigid; the LLM-rerank flexibility was worth the latency cost for the small candidate pool (top-20 → top-5).

## 6. Best Practices

- Default to "retrieve 50, rerank 5" — it is the empirically-supported sweet spot for most production RAG.
- Budget reranker latency explicitly in your end-to-end SLO and prove with measurement that the relevance lift is worth it.
- Prefer managed-API rerankers (Cohere Rerank-3 family) until you have a measured reason to self-host (cost, data residency, customisation).
- Reach for self-hosted (BGE-reranker-v2-m3 family) when data residency or per-query cost dominates the calculation — and only once you have a GPU operations story.
- Use in-prompt LLM reranking only when the criteria are genuinely too soft for a learned cross-encoder, or as a fallback layer above a cross-encoder reranker.
- Instrument both the retriever's top-50 and the reranker's top-5 against your held-out QA set — you want to know which stage is doing the work.
- Match the reranker's training domain to your corpus — a reranker trained on English web text may not generalise to highly-specialised technical or regulatory corpora without fine-tuning.

## 7. Hands-on Exercise

**Reranker-vs-retriever attribution drill (10 min).** Take any small corpus you can stand up locally — your own notes folder, a Wikipedia category dump, a public dataset. Build a minimal three-step pipeline:

1. **Retriever:** dense top-50 using any embedding model and any vector store (or even an in-memory cosine search over precomputed vectors).
2. **Reranker:** any cross-encoder reranker you can call — a managed API call or a small open-weight model running locally.
3. **Comparison:** for ten queries you write by hand, log both the retriever's top-5 and the reranker's top-5.

Then, for each query, classify the difference:

- **Reranker rescued a buried result** — a relevant document was at rank 30+ in the retriever and rose into the top-5 after reranking.
- **Reranker reordered the top-5** — same documents, different order.
- **Reranker added no value** — top-5 is identical before and after.
- **Reranker hurt** — a relevant document was top-3 in the retriever and got demoted by the reranker.

**What good looks like.** Across ten queries you should see all four categories represented at least once (small corpora make "reranker hurt" rare but it happens). The lesson is the distribution — if the reranker is rescuing buried results often, your retriever's `k` was too small or its recall was poor; if it is reordering without value, your retriever was already producing the right top-5 and you can save the latency. The point of the drill is to put numbers on the latency-vs-lift trade-off before defending it under pressure.

## 8. Key Takeaways

- Can I distinguish a bi-encoder retriever from a cross-encoder reranker and explain why the cross-encoder is more accurate but more expensive?
- Do I know why "retrieve 50, rerank 5" is the standard cascade — and what changes when I move either number?
- Given a domain and a latency budget, can I argue for or against a managed reranker vs a self-hosted reranker vs no reranker at all?
- Can I tell from instrumentation whether my reranker is rescuing buried results, reordering without value, or hurting recall — and what each of those signals means?

## Sources

1. [Cohere — Rerank Overview](https://docs.cohere.com/docs/rerank-overview) — retrieved 2026-05-26
2. [Local AI Master — Reranking & Cross-Encoders for RAG: BGE, Cohere, Jina (2026)](https://localaimaster.com/blog/reranking-cross-encoders-guide) — retrieved 2026-05-26
3. [TeachMeIdea — Reranking in RAG: Cohere Rerank and Cross-Encoders Guide](https://teachmeidea.com/reranking-in-rag-cohere-cross-encoders/) — retrieved 2026-05-26
4. [BSWEN — Best Reranker Models for RAG: Open-Source vs API Comparison (2026)](https://docs.bswen.com/blog/2026-02-25-best-reranker-models/) — retrieved 2026-05-26
5. [BigData Boutique — RAG Reranking: Improving Retrieval Quality with Cross-Encoders](https://bigdataboutique.com/blog/rag-reranking-improving-retrieval-quality-with-cross-encoders) — retrieved 2026-05-26

Last verified: 2026-05-26
