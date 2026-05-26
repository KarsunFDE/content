---
week: W01
day: Fri
topic_slug: why-rag-not-fine-tuning
topic_title: "Why RAG, not fine-tuning, for federal acquisitions"
parent_overview: W01/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://www.ibm.com/think/topics/rag-vs-fine-tuning
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://aws.amazon.com/what-is/retrieval-augmented-generation/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.anthropic.com/news/contextual-retrieval
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.aws.amazon.com/sagemaker/latest/dg/jumpstart-foundation-models-customize-rag.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Why RAG, not fine-tuning, for federal acquisitions

> **Callback — today's war-room (W1 Fri Act 4).** This morning the cohort's solicitation-drafting bot hallucinated a citation to "48 CFR 47.305-2" — a paragraph that does not exist in the actual FAR. The contracting officer caught it. Friday's answer was HITL (manage the failure). Monday's answer is RAG (eliminate the failure mode at its source by grounding the model in the *actual* current corpus at inference time). The overview frames the Karsun-applied version; this reading is the generic case for picking RAG over fine-tuning whenever your corpus changes faster than you can re-train.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Explain the structural difference between RAG and fine-tuning in terms of *where new knowledge enters the system* (inference-time context vs. retraining weights).
- Identify the three properties of a knowledge corpus that make RAG the correct choice (change frequency, citation requirements, multi-tenancy or access control).
- Describe RAG's four-stage runtime flow (query → retrieve → integrate → respond) and where in that flow the hallucination-reduction effect actually lives.
- Name two failure modes RAG does **not** solve and require complementary techniques.

## 2. Introduction

Retrieval-Augmented Generation (RAG) and fine-tuning are both ways to give a general-purpose large language model (LLM) knowledge it did not have at pre-training time. They look superficially similar — "make the model smarter about my domain" — but they sit at opposite ends of an architectural spectrum. Fine-tuning changes the model's *weights* by continued training on domain data; RAG leaves the model's weights alone and injects relevant facts at *inference time* through a retrieval pipeline that runs alongside the model. IBM's framing is the cleanest: RAG "augments" the prompt, fine-tuning "retrains" the model.

The choice is not really "which is better in the abstract" — it is "which matches how your knowledge actually behaves." Two questions determine the answer for most enterprise corpora: how fast does the truth change, and does every answer need to cite the source it came from? When the corpus is stable for years and citation is not required (consumer chatbot tone, code-style learned from a company's internal repos), fine-tuning is reasonable. When the corpus changes weekly and every answer must be defended against a source paragraph, RAG is the only architecture that can keep up.

Regulatory bodies, legal corpora, product catalogs that change with releases, and medical guidelines all share the same shape: high churn, hard citation requirement, low tolerance for stale or fabricated content. That is the shape this reading is written against — and it is also the shape of the federal-acquisitions corpus the overview applied this to.

## 3. Core Concepts

### 3.1 What changes when you fine-tune vs. retrieve

Fine-tuning takes a pretrained model and continues training it on a smaller, domain-focused dataset. The model's parameters — the billions of floating-point weights that govern how it generates tokens — shift to fit the new data. Once training finishes, the new knowledge lives inside the weights themselves. There is no "look up the answer" step at inference time; the model just generates from its updated parameters [IBM, 2026].

RAG leaves the base model's weights frozen. Instead, it builds a separate retrieval system — typically a vector database holding embedded chunks of the corpus — and injects retrieved chunks into the model's prompt at inference time. The model sees the chunks as part of the user's question and is asked to answer *with reference to* those chunks. The knowledge lives outside the model, in the index [AWS, 2026; IBM, 2026].

The structural consequence is enormous: updating a RAG corpus means re-indexing some documents (minutes to hours). Updating a fine-tuned model means a new training run (hours to days, with validation and rollback overhead).

### 3.2 The four-stage RAG runtime flow

IBM and AWS converge on the same four-stage description:

1. **Query** — user submits a question, system normalizes / embeds it.
2. **Retrieval** — the embedded query is used to find the top-k most semantically similar chunks in the vector index. Optionally a lexical retriever (BM25) runs in parallel and the results are fused.
3. **Integration (augmentation)** — the retrieved chunks are formatted into the prompt template alongside the user's original question. Up to this point the LLM has not been called.
4. **Generation** — the LLM is invoked with the augmented prompt and asked to produce an answer grounded in the retrieved chunks, often with explicit "cite the chunk you used" instructions [IBM, 2026; AWS, 2026].

The hallucination-reduction property comes from step 3, not step 4. The model still generates from its weights — but the prompt now contains the actual source text, so the cheapest-and-most-rewarded completion (a faithful summary of what's in the context) is the *correct* one. Without retrieval, the model has to guess from training-time memory, and guesses cost the model nothing.

### 3.3 The five properties that make RAG the right choice

- **Change frequency.** If the corpus changes faster than you can retrain (weekly, daily, per-release), RAG wins by default — re-indexing is orders of magnitude cheaper than retraining.
- **Citation requirement.** If every answer must trace to a source paragraph (regulation, contract, clinical guideline, policy memo), only RAG gives you the chunk-ID-to-citation link at inference time. Fine-tuned models cannot tell you *which training example* shaped a given answer.
- **Multi-tenancy or row-level access.** If different users must see different subsets of the corpus, RAG can pre-filter the retrieval query at runtime. Fine-tuning bakes everyone's data into one set of weights and there is no clean unbake operation.
- **Auditability for compliance.** RAG produces a structured trace (query, retrieved chunk IDs, generated answer) that auditors can replay. Fine-tuning produces a single model artifact that auditors must trust as a black box.
- **Cost / latency profile.** RAG adds a retrieval step (typically 50–200 ms) but avoids the multi-thousand-dollar training run. For most enterprise workloads the inference-time cost is dominated by the LLM call, not the retrieval [AWS, 2026].

### 3.4 What RAG does NOT solve

RAG is not a hallucination eliminator. It is a hallucination *reducer* — and only when the retrieval step actually returned relevant chunks. Two failure modes survive:

- **Retrieval failure → confident fabrication.** If the retriever returns irrelevant or empty chunks and the prompt template does not explicitly say "answer only from the provided context, otherwise say you don't know," the model will fall back to its weights and may hallucinate. Anthropic's contextual-retrieval research notes that traditional RAG fails 49% less when contextual embeddings + BM25 are added, but the residual failure rate is still non-zero [Anthropic, 2024].
- **Tone, style, and structured output.** RAG injects knowledge into the prompt; it does not change *how* the model writes. If you need the model to produce a specific JSON shape, a specific writing voice, or a specific reasoning pattern, fine-tuning (or careful prompting with examples) is still the right tool. The two techniques compose: RAG for knowledge, fine-tuning for behavior.

## 4. Generic Implementation

A worked example outside the federal-acquisitions domain — picture a hospital's clinical decision-support system. The cardiology guidelines change about every six months when the American Heart Association publishes updates, and every recommendation a physician sees must cite the exact guideline paragraph for malpractice and audit reasons.

```python
# Generic RAG flow — frozen base model + indexed guideline corpus
# (Pseudocode; pattern, not framework-specific)

def answer_clinical_question(question: str, physician_id: str) -> Answer:
    # Step 1: query — embed the question
    query_embedding = embedder.embed(question)

    # Step 2: retrieval — pre-filter on physician's specialty and active guidelines
    chunks = vector_index.search(
        query=query_embedding,
        k=5,
        filter={
            "guideline_status": "active",
            "specialty_tags": {"$in": physician_specialties(physician_id)},
        },
    )

    # Step 3: integration — build a grounded prompt
    prompt = render_prompt(
        question=question,
        chunks=chunks,
        instruction="Answer ONLY from the provided guideline excerpts. "
                    "Cite the chunk_id for every claim. If the excerpts "
                    "do not contain the answer, say so explicitly.",
    )

    # Step 4: generation — base model, weights unchanged
    response = base_model.invoke(prompt)

    return Answer(
        text=response.text,
        citations=[c.chunk_id for c in chunks],
        retrieved_at=now(),
    )
```

The pattern is unchanged across domains. The base model is generic; the knowledge — and the audit trail — lives in the index and in the structured response object. To roll out the new guidelines next quarter, the hospital re-indexes; the model itself is untouched.

## 5. Real-world Patterns

**E-commerce — product catalog Q&A (Shopify-style stores).** Online retailers index their full SKU catalog (descriptions, specs, availability, return policy) and use RAG to power customer-facing chatbots. The catalog changes every time a SKU is added, marked out-of-stock, or repriced — too fast for fine-tuning. Citations matter less here than freshness; the retrieval layer is the source of truth for "is this in stock right now?" — and many platforms route the same query to the live inventory API in parallel for the volatile fields [AWS, 2026 — "domain-specific knowledge" use case].

**Customer support knowledge bases (Zendesk, Intercom, Anthropic's own examples).** Support teams maintain large bodies of articles, troubleshooting flows, and macros. RAG-over-help-center is now the default; Anthropic's contextual retrieval research uses customer-support corpora as one of the canonical benchmarks because the corpus updates daily as new failure modes are discovered [Anthropic, 2024].

**Internal HR and policy chatbots.** Companies with global workforces have policy corpora that vary by country, role, and tenure — fine-tuning would require one model per tenant per role. RAG with pre-filtered retrieval on the employee's metadata reduces this to one model + one index with metadata-aware queries [IBM, 2026].

**Legal research platforms (Lex Machina, Casetext-era CoCounsel).** Case law accretes daily. New rulings change the answer to old questions. Citation to the specific paragraph is the entire product. Legal-tech vendors built RAG-first systems precisely because no fine-tuning cadence could keep up with the rate of new opinions.

## 6. Best Practices

- **Default to RAG for any corpus that changes more than quarterly or that requires citation** — the implementation cost is lower than fine-tuning's training-and-evaluation cycle even at small scale.
- **Compose, don't choose.** Use RAG for knowledge currency and fine-tuning (or careful prompt engineering) for output format and tone — they solve different problems and stack cleanly.
- **Write the prompt template with "answer only from the provided context" guidance** — RAG's hallucination-reduction effect collapses if the model is allowed to fall back to its weights silently.
- **Treat the retrieval failure mode as a first-class observable.** Log retrieval scores, no-result rates, and "model said 'I don't know'" rates separately — these are your early warning that the index is drifting from the truth.
- **Re-index on a schedule that matches the corpus's change rate** — daily for support content, on-publish for regulatory feeds, real-time for inventory. The index is a cache; treat its freshness as an SLO.
- **Keep the base model swappable.** RAG's biggest operational win is that you can switch from one base model to another without re-indexing — the retrieval layer is model-agnostic.

## 7. Hands-on Exercise

**Whiteboarding exercise (15 minutes).** Pick a corpus from your past project history — could be product docs, a knowledge base, a regulation, an internal wiki, a set of contracts. Draw a 2x2 grid on the axes "change frequency" (low/high) and "citation requirement" (no/yes). Place your corpus on the grid. For your placement, write three sentences:

1. Where does new knowledge enter — through retraining or through indexing? Why?
2. If the corpus changed tomorrow at 9 AM, how long until your system reflects the change? What is the bottleneck?
3. If a downstream consumer challenges an answer ("where did this come from?"), what is the trace you can produce?

**What good looks like.** The corpus lands in one of the four quadrants and the placement is *defended* — not "I think RAG because everyone uses RAG" but "this corpus changes every release and the compliance team requires paragraph-level citation, so retrieval-at-inference is the only architecture that satisfies both constraints." A good answer also names the *one* place RAG does not help (typically tone or structured-output format) and what they would compose alongside it.

## 8. Key Takeaways

- Can I name the structural difference between RAG and fine-tuning in one sentence? (Where new knowledge enters: prompt context vs. model weights.)
- Given a corpus, can I decide RAG-vs-fine-tuning by checking change frequency and citation requirements, not by brand loyalty to one technique?
- Do I know the four-stage RAG flow and which stage actually reduces hallucination? (Integration, by putting the source text in the prompt.)
- Can I name two failure modes RAG does not fix and the technique that does?
- Do I understand that RAG and fine-tuning compose — knowledge vs. behavior — and that "either/or" is usually a false frame?

## Sources

1. [RAG vs. fine-tuning (IBM Think)](https://www.ibm.com/think/topics/rag-vs-fine-tuning) — retrieved 2026-05-26 via /web-research. Authoritative side-by-side on the architectural difference, the four-stage RAG flow, and the "augment vs. retrain" framing.
2. [What is RAG? — AWS](https://aws.amazon.com/what-is/retrieval-augmented-generation/) — retrieved 2026-05-26 via /web-research. Vendor-neutral overview of RAG's value proposition, the four known LLM challenges it addresses, and cost-vs-retrain framing.
3. [Introducing Contextual Retrieval (Anthropic Engineering)](https://www.anthropic.com/news/contextual-retrieval) — retrieved 2026-05-26 via /web-research. Source for the hallucination-reduction percentage (49% reduction with contextual embeddings + BM25, 67% with reranking) and the failure-mode framing.
4. [Retrieval Augmented Generation — Amazon SageMaker AI](https://docs.aws.amazon.com/sagemaker/latest/dg/jumpstart-foundation-models-customize-rag.html) — retrieved 2026-05-26 via /web-research. Reference for the indexing/retrieval/generation pipeline on a managed-platform substrate.

Last verified: 2026-05-26
