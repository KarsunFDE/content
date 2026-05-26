---
week: W01
day: Fri
topic_slug: rag-six-dimensions
topic_title: "Six dimensions to plan a RAG system against"
parent_overview: W01/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://www.pinecone.io/learn/chunking-strategies/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.anthropic.com/news/contextual-retrieval
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://aws.amazon.com/what-is/retrieval-augmented-generation/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-overview/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Six dimensions to plan a RAG system against

> The overview frames this against the W2 Mon ADR scaffold — the six rows the pair must defend on Monday's plan-spec. This reading is the generic case: the six choices every RAG team makes, what each one trades off against, and how to argue for one option without dismissing the others as obviously wrong.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the six architectural dimensions of any RAG system and the question each one answers.
- For each dimension, identify the two or three reasonable default choices and the workloads each fits best.
- Argue for a specific RAG configuration in three sentences without resorting to "this is what the tutorials use."
- Recognize when a RAG system's failure mode points to a specific dimension as the root cause.

## 2. Introduction

Every RAG system makes the same six choices. The choices interact — picking a particular embedding model changes which chunk size is sensible, picking hybrid retrieval changes what your evaluation metrics measure, picking a re-ranker changes your latency budget. A good RAG plan walks through all six explicitly and names the trade-offs; a poor RAG plan picks defaults at each layer without arguing for them and ends up with a system whose failure modes nobody can debug because nobody can articulate what they wanted.

This reading is the generic-industry version of the W2 Mon ADR scaffold. The dimensions are independent of the federal-acquisitions framing — they apply to a fintech compliance bot, a healthcare clinical assistant, an e-commerce product Q&A system, or a customer support tool. The shape of the argument is the same in every case: name the dimension, name the default, name the alternative, name the workload signal that would shift the choice.

## 3. Core Concepts

### 3.1 Dimension 1 — Corpus shape (what gets indexed)

The question: at what granularity does a "chunk" live in your index — full documents, sections, paragraphs, sentences, or some semantic unit?

Pinecone's chunking-strategies guide names four common approaches: fixed-token, sentence-window, semantic (split on topic shifts), and structural (split on document hierarchy like headings or clauses) [Pinecone, 2026]. The trade-off is precision vs. context. Small chunks (one or two sentences) give precise retrieval but the model often lacks surrounding context to interpret the chunk. Large chunks (whole sections) preserve context but degrade retrieval — the embedding has to summarize too much, and the top-k will return too much irrelevant text.

The dominant default for general-purpose corpora is 200–500 token chunks with 50–100 token overlap. The dominant default for structured corpora (legal, regulatory, technical documentation) is *structural* chunking — split on natural document boundaries (clauses, headings, sub-sections) — because the document's authors already chose the right boundaries.

### 3.2 Dimension 2 — Embedding model

The question: which model converts text-to-vectors, and how large are the vectors?

The dimensions tradeoffs run in two directions. Larger vectors (3072 dim) capture more nuance but cost more to store and to compare. Smaller vectors (768 dim) are cheaper but trade some retrieval quality. Domain-specific models (legal-BERT, biomedical-BERT, code-specific embeddings) outperform general-purpose models on their domain at the cost of being worse everywhere else.

The decision framework: start with a strong general-purpose embedding (OpenAI text-embedding-3-large, Voyage AI v3, Cohere Embed v3, or AWS Bedrock Titan v2) and benchmark on your held-out retrieval set. Only switch to a domain-specific model if the general model leaves real recall on the table — and remember that switching embeddings means re-embedding the entire corpus (an operational cost the W2 Mon plan should name) [AWS, 2026].

### 3.3 Dimension 3 — Vector store

The question: where do the embeddings live, and what does the database give you for free?

The collocated-vs-dedicated framing is covered in this day's "Vector store collocation" reading; the short version is: collocated (Atlas Vector Search, pgvector, OpenSearch) when residency, multi-tenancy, or transactional consistency matter, and dedicated (Pinecone, Weaviate, Qdrant, Milvus) when raw scale or specialized features dominate. Beyond that headline, the store choice drives:

- **Filter expressiveness** — what kinds of pre-filter clauses are supported (equality, range, IN, full-text)? Atlas's MQL is one of the more expressive; Pinecone's metadata filters are simpler but well-tuned [MongoDB, 2026].
- **Hybrid retrieval support** — does the store natively support BM25 / sparse retrieval alongside dense? Atlas (via `$rankFusion`), Weaviate, OpenSearch, and Elasticsearch all do; pure-dense stores require external orchestration.
- **Operational story** — backup, restore, sharding, IAM. Collocated stores inherit the operational story of the parent database; dedicated stores have their own.

### 3.4 Dimension 4 — Retrieval mode (dense, sparse, hybrid)

The question: are you ranking purely by semantic similarity, purely by lexical match, or some combination?

Dense (semantic) retrieval is good at "find me documents about the same topic phrased differently." It famously fails at exact-string matches — a query like "error code TS-999" may not find the document that contains exactly that string because the embedding doesn't weight the unique identifier highly [Anthropic, 2024].

Sparse (BM25 / lexical) retrieval is excellent at exact-string and rare-term matching but fails at synonyms and paraphrasing. A query for "how do I cancel my subscription" may miss a document titled "Terminating Your Membership" even though they mean the same thing.

Hybrid retrieval runs both in parallel and fuses the rankings (typically via Reciprocal Rank Fusion or weighted score combination). Anthropic's contextual-retrieval research shows that adding BM25 to dense retrieval reduces failed retrievals by 49%, and adding a re-ranker on top of hybrid reduces failures by 67% [Anthropic, 2024]. For most production systems the answer is "hybrid by default."

### 3.5 Dimension 5 — Re-ranking and citation grounding

The question: do you re-rank the top-k results with a heavier model before passing them to the LLM, and how do you make the LLM cite the chunks it uses?

Re-ranking takes the top 20–50 results from the retrieval step and re-scores them with a more expensive cross-encoder model (Cohere Rerank, Voyage Rerank, BGE-Reranker). The re-ranker sees the query and each candidate together, which lets it catch nuances the bi-encoder retrieval missed. The cost is latency and dollars per query — re-ranking adds ~100-300ms and per-document inference cost. For high-stakes domains (medical, legal, financial) the trade is almost always worth it; for chat-style consumer applications it often is not.

Citation grounding is a prompt-engineering pattern, not a retrieval step. The prompt instructs the model to cite the chunk-ID for each claim, and the application layer renders those IDs as clickable links back to the source. The discipline: every claim must trace back to a chunk; if the model can't find support, the prompt should require an explicit "the corpus does not contain this information" rather than freelancing.

### 3.6 Dimension 6 — Evaluation pattern

The question: how do you know the RAG system is working, and how do you detect drift?

The standard framework is the RAGAS family of metrics (or its descendants): faithfulness (do the answers stay grounded in the retrieved chunks?), context recall (did retrieval surface the right chunks?), context precision (were the retrieved chunks free of irrelevant ones?), and answer relevance (does the answer address what the user asked?). All four matter — faithfulness alone is a known anti-pattern, because a system can be perfectly faithful to irrelevant context and still useless [known-bad-patterns.yml entry `ragas-faithfulness-only`].

The deeper discipline is treating eval-as-test-fixture: a held-out QA set lives in the repo, runs in CI on every retrieval-pipeline change, and produces a delta report. Regression in any of the four metrics is a build failure, not a coaching note. Without this, the RAG system drifts silently — a chunking change or embedding swap that improves one metric and degrades another may go unnoticed for weeks.

## 4. Generic Implementation

A worked decision matrix outside federal acquisitions — a fintech compliance assistant that answers analyst questions from internal policy documents.

| Dimension | Default chosen | Why this default | Alternative considered |
|-----------|---------------|-------------------|------------------------|
| **Corpus shape** | Structural chunking on policy-document headings; ~400 token chunks with 80 token overlap | Policy docs are well-structured; structural splits preserve the authors' intended boundaries | Fixed-token (simpler but breaks mid-section more often) |
| **Embedding model** | Voyage v3 — general-purpose, 1024-dim | Strong baseline on legal/policy benchmarks; not domain-overfit | Cohere Embed v3 (similar quality, slightly higher cost) |
| **Vector store** | Postgres with pgvector | Already in the stack for transactional data; collocation preserves residency | Pinecone (would add a residency surface for no scale benefit at our document count) |
| **Retrieval mode** | Hybrid (dense + BM25 with RRF fusion) | Policy IDs and clause references are exact-match-critical; pure dense would miss them | Pure dense (rejected — BM25 + dense is well-documented to outperform either alone) |
| **Re-ranking** | Cohere Rerank-3 on top-30 → top-8 | Compliance is high-stakes; latency budget tolerates +200ms | No re-ranking (rejected — re-ranking is the single highest-leverage RAG improvement per Anthropic 2024) |
| **Eval** | RAGAS four-metric held-out set in CI; weekly drift dashboard | Need all four metrics — faithfulness alone is an anti-pattern | LangSmith managed evals (deferred — vendor-lock concern, may revisit in v2) |

The matrix is the ADR scaffold. Each row is a defendable decision with named alternatives. Nothing in it is "because that's what the tutorial used."

## 5. Real-world Patterns

**Stripe internal documentation search.** Stripe runs a hybrid (dense + BM25) RAG system over their internal engineering docs and runbooks. Hybrid was the chosen mode because engineers search for both conceptual ("how does the chargeback dispute flow work") and exact ("error code 4855") queries — pure dense missed too many of the exact-match queries to be viable [Anthropic, 2024 — analogous customer-support pattern].

**Klarna's customer-service AI.** Klarna's widely-reported customer-service AI uses contextual retrieval with re-ranking, hitting reported failure-rate reductions in line with Anthropic's research benchmarks. Their corpus is high-churn (policies update with each market they expand into), so re-indexing cadence is a first-class operational concern [Anthropic, 2024].

**Notion AI Q&A over user pages.** Notion's per-workspace Q&A uses collocated embeddings (in their primary DB), structural chunking on page hierarchy, dense-first retrieval with permission pre-filtering, and re-ranking on the top results. The structural-chunk choice was driven by the observation that Notion pages already have clear semantic boundaries (headings, blocks) that authors maintain.

**Game-studio dialog and lore systems.** A specific pattern in narrative-heavy game development: indexing game lore (character backstories, world events, dialog history) for RAG-powered NPCs. Chunking is structural on story beats; embedding is general-purpose; retrieval is hybrid because exact character names matter; re-ranking is usually skipped because the latency budget is single-digit milliseconds [Pinecone, 2026 — analogous patterns].

## 6. Best Practices

- **Default to hybrid retrieval** unless you have a strong reason otherwise — the failure-rate reduction is too large to skip.
- **Pick chunking to match your corpus's existing structure** — if the authors gave you boundaries, use them; don't impose a fixed-token grid on a structured corpus.
- **Always include all four RAGAS metrics in eval** — faithfulness alone is a known anti-pattern.
- **Plan re-indexing operations on day one** — embedding model changes, chunk-size changes, and metadata-schema changes all require re-embedding. Know the cost and the downtime story before you ship v1.
- **Treat the prompt template as a first-class artifact** — "answer only from the provided chunks, cite chunk-IDs for every claim, say 'the corpus does not contain this information' if you cannot find support" is load-bearing prompt content, not boilerplate.
- **Argue for each dimension explicitly in an ADR** — when the system fails six months from now, the next engineer needs to know which choices to revisit.

## 7. Hands-on Exercise

**Whiteboarding exercise (15 minutes).** Pick a non-federal-acquisitions corpus (a hospital's clinical guidelines, an e-commerce product catalog, a game's lore, a SaaS company's help center) and produce the same six-row decision matrix as the Generic Implementation example above. For each row:

- Name the default you'd pick.
- Name one realistic alternative.
- Write one sentence justifying the default against the alternative — citing a property of the corpus, not "what tutorials use."

**What good looks like.** The matrix has six populated rows. The defaults align with the corpus's properties (small corpus + permissioned → collocated; high exact-match content → hybrid; high-stakes domain → re-ranking on; structured corpus → structural chunking). At least one row picks a non-default — for instance, "no re-ranking" for a latency-bound consumer chat — and defends it. The argument never reduces to "the tutorial used this" or "this is the most popular."

## 8. Key Takeaways

- Can I name all six dimensions of a RAG system and the question each one answers?
- Do I know the default-vs-alternative for each dimension and what would push me to the alternative?
- Can I write an ADR-row for any dimension without resorting to "what the tutorials use"?
- Do I understand why hybrid retrieval is the default — not just a feature?
- Can I name the four RAGAS metrics and explain why faithfulness alone is an anti-pattern?

## Sources

1. [Chunking Strategies for LLM Applications — Pinecone](https://www.pinecone.io/learn/chunking-strategies/) — retrieved 2026-05-26 via /web-research. Authoritative survey of fixed-token, sentence-window, semantic, and structural chunking trade-offs.
2. [Introducing Contextual Retrieval — Anthropic Engineering](https://www.anthropic.com/news/contextual-retrieval) — retrieved 2026-05-26 via /web-research. Source for the dense-vs-BM25 failure-mode framing, the 49% / 67% improvement numbers, and the hybrid-by-default conclusion.
3. [What is RAG? — AWS](https://aws.amazon.com/what-is/retrieval-augmented-generation/) — retrieved 2026-05-26 via /web-research. Vendor-neutral framing of the RAG flow and the cost/operational considerations across embedding-model choice.
4. [Vector Search Overview — MongoDB Atlas](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-overview/) — retrieved 2026-05-26 via /web-research. Reference for filter-expressiveness and ANN/HNSW algorithm choice in the vector-store dimension.
5. Known-bad-pattern reference (instructor): `~/fde-10-week/skills/tech-research/references/known-bad-patterns.yml` entry `ragas-faithfulness-only`.

Last verified: 2026-05-26
