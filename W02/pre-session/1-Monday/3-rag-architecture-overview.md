---
week: W02
day: Mon
topic_slug: rag-architecture-overview
topic_title: "RAG architecture overview — the six coupled design dimensions"
parent_overview: W02/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://www.ibm.com/think/topics/retrieval-augmented-generation
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://orq.ai/blog/rag-architecture
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://blog.premai.io/building-production-rag-architecture-chunking-evaluation-monitoring-2026-guide/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.techment.com/blogs/rag-in-2026/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://arxiv.org/html/2506.00054v1
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# RAG architecture overview — the six coupled design dimensions

## 1. Learning Objectives

By the end of this reading, the learner can:

- Describe the two-phase **retrieve → generate** architecture of a RAG system and name the components on each side of the split.
- Enumerate the **six coupled design dimensions** of a RAG system (corpus & chunking, embedding model, vector store, retrieval mode, citation surface, evaluation) and explain why changing any one of them constrains the others.
- Distinguish a **naive RAG** topology from a **production RAG** topology (re-ranking, hybrid search, pre-filtering, observability) and name the failure mode each addition addresses.
- Identify which design dimensions are **commit-this-Monday** decisions (irreversible-ish) versus **iterable** decisions (cheap to change later).

## 2. Introduction

Retrieval-augmented generation (RAG) is the dominant production pattern for connecting large language models to private, regulated, or rapidly-changing knowledge. The 2026 industry survey from arXiv 2506.00054 frames it as the architecture that "lets language models reason over private corpora without retraining" — and that one-sentence summary hides about a dozen tightly-coupled design choices that determine whether a RAG system actually works.

The reason RAG is often presented as "simple" — embed your docs, store the vectors, retrieve the nearest ones, stuff them into the prompt — is that the demo path *is* simple. The production path is not. Industry write-ups in 2026 consistently report that **80% of RAG failures trace to the ingestion and chunking layer**, not to the language model itself (premai.io 2026 guide, retrieved 2026-05-26). That number alone tells you where the design effort needs to go: the model is the cheap part; the data pipeline is where the engineering lives.

This reading lays out the six coupled design dimensions of a RAG system. The point is not to make you an expert on any of them — Tue, Wed, and Thu of this week go deep on each. The point is to give you the **shape of the decision space** before you walk into Monday's planning conversation, so when someone says "let's just use cosine similarity" you can answer "against what embedding model and what chunking strategy?"

## 3. Core Concepts

### 3.1 The two-phase pipeline

A RAG system splits the language model's job into two phases:

```
┌─────────── INDEX-TIME (offline, batch) ───────────┐
│                                                    │
│  source docs → parse → chunk → embed → vector DB  │
│                                                    │
└────────────────────────────────────────────────────┘

┌─────────── QUERY-TIME (online, per-request) ──────┐
│                                                    │
│  user query → embed → retrieve top-k chunks       │
│            → (re-rank)                             │
│            → prompt(query, chunks) → LLM          │
│            → response + citations                  │
│                                                    │
└────────────────────────────────────────────────────┘
```

Index-time runs once per corpus version. Query-time runs on every user request. Most of the latency and cost budget at runtime is in the LLM call; most of the *quality* budget is determined offline at index time. This asymmetry is the central reason chunking dominates RAG failure modes — by the time you discover a chunking mistake, you've already paid for the index.

### 3.2 The six coupled design dimensions

A production RAG system is the joint outcome of six design decisions. They are not independent — changing one constrains the others.

**Dimension 1 — Corpus shape + chunking strategy.** What is the source corpus, and what is a "chunk"? Fixed-size character windows are the demo default; structural splits on document boundaries (sections, clauses, paragraphs) are the production default for regulated or semi-structured corpora; parent-child chunking is the precision-vs-context resolver. (Detail in W2 Tue.)

**Dimension 2 — Embedding model.** Which model turns chunks into vectors? Choice of model determines vector dimensionality, language coverage, domain bias, and cost-per-token at index time. **This decision is hard to reverse** — changing the embedding model means re-indexing the entire corpus, because vectors from model A and vectors from model B don't live in the same semantic space.

**Dimension 3 — Vector store.** Where do the vectors live, and how are they queried? The store determines whether you have **pre-filtering** (efficient metadata-aware retrieval), **hybrid search** (dense vector + sparse keyword), and **multi-tenant isolation** options. Storage tier and query latency are downstream consequences.

**Dimension 4 — Retrieval mode.** Dense-only (vector similarity), sparse-only (BM25 / keyword), or hybrid (both, with score fusion)? Hybrid search consistently outperforms either pure mode on production benchmarks — a widely-cited 2026 industry write-up reports **+17% recall** for hybrid over dense-only on enterprise document collections (axiscoretech.com, retrieved 2026-05-26). The trade-off is operational complexity.

**Dimension 5 — Citation / grounding surface.** What does the response envelope look like? Is each generated claim traceable to a specific source chunk with a stable identifier? This dimension is where compliance, audit, and trust live. It is also the dimension most often underspecified in early designs and most often rewritten under deadline pressure.

**Dimension 6 — Evaluation pattern.** How do you know the system is working, and how do you know a change made it better or worse? Without an evaluation harness — a held-out QA set, a metric suite (faithfulness, context recall, context precision, answer relevance), and a regression workflow — every change to the other five dimensions is a guess.

### 3.3 Why the dimensions are coupled

A toy example. Suppose your team chooses **structural chunking** at the sub-clause level for a regulatory corpus (dimension 1). That choice means each chunk is short — maybe 50–200 tokens. Short chunks have less context per vector, which means the embedding model (dimension 2) has less signal to work with, which means semantic recall drops. To recover, you turn on hybrid search (dimension 4) so keyword match can compensate for weak semantic signal, which means your vector store (dimension 3) must support BM25 alongside vectors. And because chunks are short, your citation surface (dimension 5) now needs to return the **parent** clause for context, not just the matched sub-clause, which means your retrieval pattern is now parent-child, which means your evaluation harness (dimension 6) must score recall at the parent grain, not the chunk grain.

Change the chunking strategy, change everything else. This is why a Monday planning conversation must treat the six dimensions as a system, not as a checklist.

### 3.4 Naive RAG vs production RAG topology

**Naive RAG** — the demo:

```
query → embed → top-k vector search → prompt(query, top-k) → LLM → answer
```

**Production RAG** — what actually ships:

```
query → query rewriting / expansion
      → pre-filter (tenant, ACL, date range, document type)
      → hybrid retrieval (dense + sparse, with score fusion)
      → re-rank (cross-encoder or LLM-as-reranker)
      → context assembly (parent-child expansion, diversity sampling)
      → prompt assembly with citation scaffolding
      → LLM generation with citation-required system prompt
      → post-validation (citation check, refusal-on-empty-evidence)
      → response envelope with structured citations + confidence
      → log to evaluation harness
```

Each addition addresses a specific failure mode: query rewriting closes the user-vs-corpus phrasing gap; pre-filtering enforces multi-tenancy and ACLs; hybrid retrieval breaks the dense-only recall ceiling on enterprise corpora; re-ranking restores precision at top-k for large candidate pools; parent-child context assembly fixes small-chunk context loss; citation scaffolding + post-validation addresses hallucination and audit defensibility; refusal-on-empty-evidence prevents the "model makes something up when retrieval returns nothing" failure.

### 3.5 Which decisions are commit-this-Monday vs iterable

A simple framing for the Monday planning conversation:

| Dimension | Commit-this-Monday? | Why |
|---|---|---|
| Chunking strategy | Yes | Drives every other dimension. Hard to retrofit. |
| Embedding model | Yes | Re-indexing is expensive; vector spaces don't mix. |
| Vector store | Yes | Migration costs scale with corpus size. |
| Composition / orchestration framework | Yes | Touches every retrieval and generation call site. |
| Retrieval mode (dense / hybrid) | No — iterate Tue | Can be toggled per-query once the store supports it. |
| Citation surface schema | No — iterate Thu | The schema can evolve; what matters is committing that there IS one. |
| Evaluation pattern | No — iterate Fri | Land a harness Fri; iterate metrics across W3+. |

The four "commit-this-Monday" decisions are exactly what your Plan-Day ADRs are for. The three iterables get rough placeholders in the plan-spec and are sharpened day by day.

## 4. Generic Implementation

A worked example outside federal acquisitions. Imagine a **fintech compliance team** building a RAG system over the firm's internal AML (anti-money-laundering) policies, customer-due-diligence checklists, and recent regulator advisories. The team's day-one scope envelope across the six dimensions might look like:

- **Corpus** — ~1,800 policy documents, mixed PDF + Confluence + internal wiki. **Chunking** — structural splits on policy section headers; parent-child retrieval with 256-token child chunks and full-section parent chunks.
- **Embedding model** — choose a general-English embedding model (e.g., a managed embedding service with 1024-dim vectors and known cost-per-token). Note that financial jargon may need domain-tuning at iteration 2.
- **Vector store** — managed vector DB collocated with the document store, with metadata fields for `policy_id`, `effective_date`, `jurisdiction`, `confidentiality_tier`.
- **Retrieval mode** — hybrid (BM25 + dense) from day 1 because policy queries often contain specific clause numbers (where BM25 wins) AND conceptual questions (where dense wins).
- **Citation surface** — every generated sentence cites `policy_id` + `section` + `effective_date`. If a query has no retrieval hit above a confidence floor, the system MUST refuse to answer.
- **Evaluation** — 200-question held-out set covering policy lookups, conceptual questions, and adversarial "make something up" probes. Faithfulness + context recall + answer relevance scored weekly.

Note what's missing from this scope: the LLM itself. The choice of generation model is downstream and substantially easier to change than the six core dimensions — it's a config flip, not a re-indexing job.

## 5. Real-world Patterns

**Healthcare — clinical guideline lookup.** A 2024 RAG system at a US hospital network indexed CDC and specialty-society treatment guidelines for ER physicians at triage. Structural chunking on guideline section headers beat fixed-size because clinicians' questions mapped to guideline sections ("what's the sepsis bundle?"). Hybrid retrieval was essential because drug names and dosages are exact-match queries where BM25 dominates. Every answer cited guideline-ID + publication date so the physician could see whether they were reading a 2018 or 2024 protocol.

**E-commerce — product Q&A.** A large online retailer's Q&A bot, per Atlan's 2026 RAG guide (retrieved 2026-05-26), used dense-only retrieval over ~50M product descriptions. Conversion improved measurably but the team hit a precision ceiling and added cross-encoder re-ranking. Chunking was per-product-attribute rather than per-document — counterintuitive but right because customer questions target single attributes ("is this dishwasher-safe?").

**Logistics — internal operations runbook.** A global shipper indexed warehouse and dispatch SOPs. First failure mode: agents asked colloquial questions ("what if the truck doesn't show?") and dense-only retrieval missed because the corpus used formal language ("carrier no-show procedure"). Query rewriting (a small LLM step that reformulated queries into corpus-style language) was the fix — now their highest-leverage RAG component despite being smallest and cheapest.

**Gaming — community wiki.** A multiplayer game indexed its wiki + patch notes for an in-game help assistant. The corpus changed weekly, so the re-indexing pipeline was a first-class concern. Evaluation included a "stale citation" detector flagging answers citing patch notes older than the current game version. The underappreciated sixth-dimension lesson: in a fast-moving corpus, evaluation scores not just correctness but **currency**.

## 6. Best Practices

- **Treat the six dimensions as a system, not a checklist.** Changing any one constrains the others. Walk the dependency graph before committing.
- **Make chunking the first ADR.** It's the most coupled decision and the hardest to reverse. Everything else flows from it.
- **Default to hybrid retrieval on enterprise corpora.** Dense-only is a demo posture. The +17% recall is real, and the operational complexity is manageable with a vector store that supports it natively.
- **Specify the citation surface before you have anything to put in it.** The schema constrains what your prompt can ask for and what your post-validator can check. Write the JSON envelope on Monday; fill it Tue–Fri.
- **Build the evaluation harness before you tune anything.** Without held-out eval, every change is a guess. Faithfulness alone is insufficient — you need at least three of the four RAGAS dimensions (faithfulness, context recall, context precision, answer relevance) to detect regressions.
- **Plan for re-indexing from day 1.** Corpora change. Embedding models get superseded. The first re-index is always more painful than expected; the runbook for it should exist before you need it.
- **Refuse on empty evidence.** A system prompt that instructs the model to say "I don't have enough evidence" when retrieval is weak is the single highest-ROI hallucination defence. Build it in, don't bolt it on.

## 7. Hands-on Exercise

**Time: 10–15 minutes — whiteboarding prompt.** Pick a domain you know well (other than federal acquisitions). Sketch the six-dimension scope envelope for a RAG system in that domain. For each dimension, write one sentence committing to a specific choice, AND one sentence naming what you're explicitly NOT doing this iteration (the Won't).

Then draw the data flow as an ASCII or whiteboard diagram showing:

- One index-time path with at least four boxes (parse → chunk → embed → store).
- One query-time path with at least six boxes (the production topology, not naive).
- The point in the query path where multi-tenant pre-filtering happens.
- The point in the query path where citation post-validation happens.

**What good looks like:** Your diagram should make it visually obvious that chunking happens once (index-time) but determines what's recoverable at every query-time step. Your scope envelope should commit to specific choices on the four irreversible-ish dimensions (chunking, embedding model, vector store, composition framework) and leave the three iterables (retrieval mode, citation schema, evaluation harness) as placeholders you'll sharpen by Friday. If your diagram has fewer than six boxes on the query path, you're still in naive-RAG territory — add at least one production-topology element.

## 8. Key Takeaways

- Can you draw the two-phase **index-time + query-time** RAG topology and name what runs where?
- Can you list the six coupled design dimensions and explain why changing one constrains the others?
- Can you distinguish a naive RAG path (4 boxes) from a production RAG path (8+ boxes), and name the failure mode each production-only addition addresses?
- Can you identify which RAG design decisions are commit-this-Monday (hard to reverse) vs iterable (cheap to change later)?
- Can you explain why **80% of RAG failures trace to ingestion and chunking**, not to the language model, and what that implies for where engineering effort should go?

## Sources

1. [What is Retrieval-Augmented Generation (RAG)? — IBM](https://www.ibm.com/think/topics/retrieval-augmented-generation) — retrieved 2026-05-26
2. [RAG Architecture Explained: A Comprehensive Guide [2026] — Orq.ai](https://orq.ai/blog/rag-architecture) — retrieved 2026-05-26
3. [Building Production RAG: Architecture, Chunking, Evaluation & Monitoring (2026 Guide) — PremAI Blog](https://blog.premai.io/building-production-rag-architecture-chunking-evaluation-monitoring-2026-guide/) — retrieved 2026-05-26
4. [RAG in 2026: How Retrieval-Augmented Generation Works for Enterprise AI — Techment](https://www.techment.com/blogs/rag-in-2026/) — retrieved 2026-05-26
5. [Retrieval-Augmented Generation: A Comprehensive Survey of Architectures, Enhancements, and Robustness Frontiers — arXiv 2506.00054](https://arxiv.org/html/2506.00054v1) — retrieved 2026-05-26

Last verified: 2026-05-26
