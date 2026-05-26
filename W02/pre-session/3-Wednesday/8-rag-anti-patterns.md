---
week: W02
day: Wed
topic_slug: rag-anti-patterns
topic_title: "RAG anti-patterns + framing for the afternoon /web-research slot"
parent_overview: W02/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 8
sources:
  - url: https://www.digitalapplied.com/blog/rag-anti-patterns-7-failure-modes-2026-engineering-guide
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://medium.com/@hadiyolworld007/nobody-warns-you-about-silent-truncation-8-rag-correctness-leaks-3e4912cd012c
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://tianpan.co/blog/2026-04-09-embedding-models-production-versioning-index-drift
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://decompressed.io/learn/rag-observability-postmortem
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://air-governance-framework.finos.org/mitigations/mi-13_providing-citations-and-source-traceability-for-ai-generated-information.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# RAG anti-patterns + framing for the afternoon /web-research slot

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the five most-cited RAG anti-patterns and explain which observability signal catches each one.
- Distinguish "the system crashed" failures (rare and easy) from "the system silently degrades" failures (common and dangerous).
- Build a chunk-citation discipline that supports audit-grade source traceability.
- Identify embedding-model-version drift before it shows up as a quality regression.
- Frame a `/web-research` exercise so candidate techs are evaluated on the dimensions on which they actually lose, not just on the dimensions on which the chosen tech wins.

## 2. Introduction

The defining trait of a failed production RAG system is that nothing crashes. The pipeline keeps serving answers. The dashboards report healthy latency. The error rate stays flat. Underneath, the system is degrading on every axis that matters: retrieved chunks grow noisier, the model paraphrases confidently from low-relevance context, citations point to documents the answer didn't actually use, and the corpus quietly diverges from the embedding model that indexed it.

Industry analysis published through 2026 consistently lands on the same headline number: when production RAG systems fail, the failure point is retrieval roughly 73% of the time — not generation. Generation looks like the visible failure (a confident wrong answer); retrieval is the actual root cause (the wrong chunks reached the model, or the right chunks didn't). The anti-patterns in this reading map to the silent-failure modes that produce that 73%.

This reading also frames the afternoon `/web-research` slot — the protected three-prompt block where pairs evaluate vector store, hybrid-retrieval, and embedding-model alternatives against the constraints surfaced through the morning's topics. The framing matters: the discipline is *not* to pick the alternative that wins the most dimensions. The discipline is to *name the dimension on which the chosen tech loses* — because if you can't, you haven't actually researched the alternatives.

## 3. Core Concepts

### Anti-pattern 1: silent context truncation

The retrieval step returns 12 chunks. The prompt assembly concatenates them until the token budget fills, then drops the tail without logging. The LLM never saw chunks 9–12. The system never says so.

The diagnostic is simple: log `chunks_retrieved` and `chunks_sent_to_llm` per call as separate counters. Alert when they diverge. The fix is either (a) raise the context budget, (b) compress the retrieved chunks before assembly, or (c) reduce `k` in retrieval. Choosing without observing is guessing.

### Anti-pattern 2: confidence-as-string

The system logs "high confidence" / "low confidence" / "medium confidence" instead of the underlying numeric score (cosine similarity, faithfulness, rerank score). Regression analysis becomes impossible — you can't aggregate or threshold a string. You can't compare "is yesterday's score distribution shifted from today's" without numbers.

The fix is to always log the float. The categorical label can sit beside the score for human-readable dashboards, but never *replace* it. Buckets are derivable from raw values; raw values are not derivable from buckets.

### Anti-pattern 3: citation without chunk ID

The LLM produces *"according to the policy on data retention…"* but the cited "policy" is a paraphrase the model wrote rather than a deterministic pointer to a specific corpus document. There is no `chunk_id` an audit log can resolve back to a source location.

The discipline: every retrieved chunk carries a deterministic ID (a hash of the source URL + offset, or a database-generated UUID). The LLM prompt includes the chunk IDs and the prompt template instructs the model to emit them in citations. The audit log writes `query_id → answer → cited_chunk_ids[]` so any answer can be traced to the exact chunks that grounded it.

This is the audit-grade citation pattern. Regulated workloads (financial services, healthcare, federal contractors, EU AI Act high-risk) require it explicitly. Even unregulated workloads benefit — when a user reports a bad answer, "which chunks did the model see" is the first question; without a `chunk_id` in the log, the answer is "we don't know."

### Anti-pattern 4: schema drift between embedding model and stored vectors

You upgrade the embedding model from a 1024-dim Titan v2 to a 3072-dim alternative. New documents embed with the new model. Old documents stay in the index with their old embeddings. The vector store rejects mixed-dimension queries (best case — a hard error) or silently accepts them and returns nonsense (worse case — same dimension, different model, garbage similarity scores).

This is the embedding-model-version drift problem. The fix is preventive: store the embedding model name and version alongside every embedding. Re-index when the model changes. Build the new index in parallel, validate quality on a held-out eval set, and atomically swap the alias only after validation passes.

The migration playbook: (1) provision new index; (2) re-embed corpus with new model into the new index; (3) run the eval set against both old and new; (4) verify no regression on the metric you care about; (5) atomic alias swap; (6) keep old index for rollback for one release cycle; (7) delete old index after confidence.

### Anti-pattern 5: async ingestion without dead-letter

The ingestion pipeline reads documents, chunks them, embeds them, writes to the vector store. A subset of documents fail (malformed input, transient network errors, schema validation failures). The pipeline swallows the failures, logs nothing, and moves on. The corpus has holes the dashboards don't show.

The diagnostic is corpus completeness assertion. *"The source has N parts. The index contains M chunks. M < N → ingestion incomplete. Investigate before serving queries against this corpus."* The pipeline must have a dead-letter queue that catches failed documents and a completeness check that fires before the index is marked production-ready.

Many real production incidents trace to this anti-pattern: a silent drop of a corpus segment that nobody notices until a user asks a question the dropped segment would have answered.

### Other recurring anti-patterns worth knowing

- **Fixed-size chunking on heterogeneous corpora.** Naïve 512-token splits cut sentences in half. Use semantic-boundary-aware chunking (paragraph, sentence, or structural boundaries) or accept measurable retrieval degradation.
- **Stale-corpus serving.** A document updated last month, an index that hasn't been re-embedded for two months. The system confidently returns last month's answer. The fix: index timestamps + re-embedding triggers tied to source-system update events.
- **Eval-set leakage into training.** Using the same documents in the eval set that were used to validate the corpus during ingestion. The numbers look great; production performance is worse. The eval set must be drawn from queries that did not influence corpus tuning.
- **Single-metric optimization.** Optimizing for NDCG and watching answer quality regress. The eval triangle (NDCG + recall preservation + latency) and the four RAGAS dimensions (faithfulness, context precision, context recall, answer relevance) are the multi-metric counter to this.

### Framing the afternoon `/web-research` slot

The W2 Wed afternoon block (per D-040) is the *dedicated `/web-research` time* for the week. Three scenario-alternatives prompts are loaded for pairs to work, with 2–3 candidate techs each, ADR drafts due Thu EOD. The three prompts orbit the day's morning topics:

- **Vector store choice (`W02-SA-1`)**. Atlas Vector Search, pgvector on Postgres, Pinecone, OpenSearch, Weaviate, Qdrant — pick 2–3 candidates. Evaluate against multi-tenant isolation (today's topic 3), recall@k, filter cost, ingestion throughput, federal-context constraints (cloud authorization, data-residency), and operational maturity.
- **Hybrid-retrieval strategy (`W02-SA-2`)**. Dense-only with reranker, full hybrid (dense + BM25 + RRF + rerank), contextual-embedding-only, query-expansion + rerank — pick 2–3. Evaluate against latency budget, recall preservation, the cost-of-precision curve (today's topic 2), and the eval harness arriving Friday.
- **Embedding-model selection (`W02-SA-3`)**. Bedrock Titan v2, OpenAI text-embedding-3, domain-specialized embeddings, BGE-M3, query/document-routed embeddings (different models for different chunk types) — pick 2–3. Evaluate against per-call cost, dimension trade-off (today's topic 4 mentions HNSW memory scaling), context-window for longer chunks, and recency category (some "hot" embeddings are <90-day-old).

**The discipline.** Each candidate tech gets a `/web-research` citation with retrieval date and recency category. The ADR draft names: (1) the chosen tech and its strengths; (2) the rejected alternatives and the strengths the chosen tech *lacks* relative to them; (3) the dimension on which the chosen tech *loses* — and how the chosen tech mitigates or accepts that loss.

If a pair can name only the strengths of the chosen tech and not the dimension on which it loses, the research is incomplete. The Friday Live Defense rubric weights "honest acknowledgment of trade-offs" as a separate dimension from "correct choice" — a correct choice presented as if it had no trade-offs is not a passing defense.

## 4. Generic Implementation

A corpus-completeness assertion in plain Python, outside the federal-acquisitions domain. Imagine ingesting product manuals for an e-commerce knowledge base where each manufacturer publishes a `manifest.json` declaring the expected document set.

```python
def assert_corpus_complete(manufacturer_id: str) -> None:
    manifest = load_manifest(manufacturer_id)        # source of truth
    indexed = index_count_by_manufacturer(manufacturer_id)
    expected = len(manifest["documents"])

    if indexed < expected:
        missing = expected - indexed
        # Detailed dead-letter dump for triage
        log.error({
            "event": "corpus.incomplete",
            "manufacturer_id": manufacturer_id,
            "expected": expected,
            "indexed": indexed,
            "missing": missing,
            "manifest_ids": [d["id"] for d in manifest["documents"]],
            "indexed_ids": list_indexed_ids(manufacturer_id),
        })
        raise CorpusIncompleteError(
            f"{manufacturer_id}: indexed {indexed} of {expected}"
        )

    # Embedding-version sanity check
    model_versions = embedding_model_versions(manufacturer_id)
    if len(model_versions) > 1:
        log.error({
            "event": "corpus.embedding_drift",
            "manufacturer_id": manufacturer_id,
            "versions_present": list(model_versions),
        })
        raise EmbeddingVersionMismatchError(
            f"{manufacturer_id}: multiple embedding model versions in index"
        )
```

The function is deliberately blocking. A corpus that fails completeness or embedding-version checks should not serve queries. The cost of a missing-document silent leak is much higher than the cost of a brief outage while ingestion completes.

## 5. Real-world Patterns

**E-commerce product knowledge base.** A consumer-electronics retailer published a 2026 post-mortem on a silent-truncation incident: a prompt assembly path dropped chunks 6–10 of a 10-chunk retrieval set when the token budget was hit. The model answered from chunks 1–5; the answer was confidently wrong on details that lived in chunks 7–9. Detection took six weeks of customer complaints. The fix was a single-line log emission (`chunks_retrieved`, `chunks_sent_to_llm`) plus a divergence alert — they added "always log the gap" to their RAG-pipeline review checklist permanently.

**Fintech research-summary platform.** A fintech-research team described an embedding-model upgrade that broke their RAG quality silently. They upgraded the embedding model on new documents only, planning to re-index older documents "later." The "later" was three weeks. During that window, queries that hit a mix of old- and new-embedded documents returned poor recall on the old ones. Their post-mortem named the missing discipline: "atomic re-index, parallel index, alias swap" — never the rolling upgrade they had attempted.

**Healthcare clinical search.** A clinical-search startup described the audit-grade citation requirement that drove their architecture: every answer the system produced had to be traceable to specific chunks via deterministic chunk IDs. The compliance team rejected an architecture where the LLM "paraphrased the source" without preserving the chunk reference. The fix was a prompt template change that instructed the model to emit `[chunk:abc123]` tokens inline; the rendering layer turned those into citations and the audit layer logged them as references.

**Gaming player-support ingestion.** A multiplayer-game studio described a dead-letter-queue failure that lost a patch's worth of patch-notes from their support corpus. The patch released, the players' questions about the patch hit the search, and the search returned the pre-patch documents because the post-patch documents had failed ingestion silently. The fix was a completeness-assertion gate before any patch's documents went live; if the indexed count didn't match the source count, the patch was held until the gap closed.

## 6. Best Practices

- Log every count divergence — chunks retrieved vs chunks sent to LLM, expected corpus size vs indexed size, retrieval candidate pool vs post-filter survivors.
- Log scores as floats. The categorical label can sit beside the float; it cannot replace it.
- Make chunk IDs deterministic (hash of source + offset, or stable database ID) and include them in citations.
- Treat embedding-model upgrades as schema migrations. Parallel index, validate, atomic alias swap. No rolling embedding upgrades.
- Build dead-letter queues into ingestion. Failed documents go to DLQ with their failure reason; an alert fires; queries against an incomplete corpus are blocked until ingestion completes or the gap is explicitly accepted.
- Run the corpus-completeness assertion as a pre-deploy gate. A new corpus version is not production-ready until the assertion passes.
- For the afternoon `/web-research` block: name the dimension on which the chosen tech *loses*. If you can't, the research is incomplete.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** Pick one of the three afternoon `/web-research` scenario prompts (vector store, hybrid retrieval, or embedding model). Without doing the research yet, sketch the decision matrix you intend to populate.

The matrix has rows (the 2–3 candidate techs you'll evaluate) and columns (the evaluation dimensions: multi-tenant boundary support, recall, latency, cost, operational maturity, federal-context constraints, etc.). Fill in *which dimensions matter* for this specific prompt — different prompts emphasize different dimensions. Mark which dimension you expect to be the *losing dimension* for the tech you're currently leaning toward.

**What good looks like.** Five to eight dimensions named, with a brief justification for each — why this dimension matters for this specific prompt. The expected losing dimension is named explicitly. The matrix is the planning artifact you'll fill in during the afternoon research block; the structure exists before the research begins so the research is guided rather than meandering.

## 8. Key Takeaways

- *What is the dominant RAG failure mode?* — Silent retrieval degradation. Industry data lands on roughly 73% of production failures tracing to retrieval, not generation.
- *Why is silent context truncation hard to detect?* — Because nothing crashes. Detection requires explicit logging of `chunks_retrieved` vs `chunks_sent_to_llm` and a divergence alert.
- *What is the audit-grade citation pattern?* — Deterministic chunk IDs preserved through retrieval, included in the LLM prompt, emitted in the answer, and logged in the audit trail.
- *How do you handle an embedding-model upgrade?* — Atomic re-index: parallel index, validate on eval set, alias swap. Never rolling.
- *What is the discipline for the afternoon `/web-research` slot?* — Name the dimension on which the chosen tech loses. If you can't, the research is incomplete.

## Sources

1. [RAG Anti-Patterns: 7 Failure Modes Engineering Guide 2026 (Digital Applied)](https://www.digitalapplied.com/blog/rag-anti-patterns-7-failure-modes-2026-engineering-guide) — retrieved 2026-05-26
2. [Nobody warns you about silent truncation: 8 RAG correctness leaks (Mar 2026)](https://medium.com/@hadiyolworld007/nobody-warns-you-about-silent-truncation-8-rag-correctness-leaks-3e4912cd012c) — retrieved 2026-05-26
3. [Embedding Models in Production: Selection, Versioning, and the Index Drift Problem (TianPan, Apr 2026)](https://tianpan.co/blog/2026-04-09-embedding-models-production-versioning-index-drift) — retrieved 2026-05-26
4. [I Updated My Embedding Model and My RAG Broke: A Post-Mortem (Decompressed Learn)](https://decompressed.io/learn/rag-observability-postmortem) — retrieved 2026-05-26
5. [Providing Citations and Source Traceability for AI-Generated Information (FINOS AI Governance Framework)](https://air-governance-framework.finos.org/mitigations/mi-13_providing-citations-and-source-traceability-for-ai-generated-information.html) — retrieved 2026-05-26

Last verified: 2026-05-26
