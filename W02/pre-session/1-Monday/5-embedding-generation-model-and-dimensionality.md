---
week: W02
day: Mon
topic_slug: embedding-generation-model-and-dimensionality
topic_title: "Embedding generation — model and dimensionality choice"
parent_overview: W02/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://docs.aws.amazon.com/bedrock/latest/userguide/titan-embedding-models.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://aws.amazon.com/blogs/aws/amazon-titan-text-v2-now-available-in-amazon-bedrock-optimized-for-improving-rag/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://huggingface.co/blog/matryoshka
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.buildmvpfast.com/blog/best-embedding-model-comparison-voyage-openai-cohere-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://towardsdatascience.com/649627-2/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# Embedding generation — model and dimensionality choice

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define an **embedding model** and explain what a vector dimension represents (and what it does not).
- Distinguish three operational levers in the embedding decision — **model choice**, **dimensionality**, and **domain coverage** — and explain why model choice is the most expensive to reverse.
- Describe **Matryoshka Representation Learning (MRL)** and why it lets a single model produce multiple useful dimensionalities from one training run.
- Name the cost / recall / storage trade-offs between common embedding dimensionalities (256 / 512 / 1024 / 3072) using 2026 benchmark data.

## 2. Introduction

An embedding model is the bridge between human-readable text and the math the rest of a RAG system can operate on. It takes a string of tokens and returns a fixed-length vector of floating-point numbers. The vector's job is to **represent the meaning of the text** such that texts with similar meaning land near each other in vector space and texts with different meaning land far apart.

That's the theory. The engineering reality is that "meaning" is decided entirely by what data the embedding model was trained on, how it was trained, and what tasks it was optimised for. A model trained on English Wikipedia and Common Crawl will be excellent at general-English semantic similarity and mediocre at, say, German clinical notes, or at fine-grained legal-clause distinctions, or at code search. Choosing the embedding model is choosing the **definition of "meaning"** the whole RAG system will use.

Once chosen, the model is also the **most expensive piece of the pipeline to change**. Embeddings from model A and embeddings from model B do not live in the same vector space. Switching models means re-embedding the entire corpus from scratch. For a 3,500-page regulatory corpus this is mildly painful; for a 50M-document e-commerce catalogue it is a budget line item that requires executive approval.

This reading frames the embedding decision: what the levers are, what 2026 benchmarks say about the leaders, and which trade-offs you can defer.

## 3. Core Concepts

### 3.1 What an embedding is, and what a dimension is not

An embedding is a vector of N floating-point numbers. N is the model's **output dimensionality**. Common values in 2026 are 256, 512, 768, 1024, 1536, and 3072. Larger N gives the model more "room" to encode distinctions; smaller N is cheaper to store and faster to compute similarity over.

A critical clarification that trips up most newcomers: **individual dimensions in a learned embedding are not interpretable**. Dimension 17 does not "mean" anything in particular. The model decides what each dimension represents during training, and the result is a distributed representation where meaning lives in the relationships between dimensions, not in any single one. You cannot read an embedding by squinting at it.

What you CAN do is compare embeddings: cosine similarity (the angle between two vectors), dot product (their projection onto each other), or Euclidean distance (the straight-line distance between their endpoints). All three are different ways of asking "how similar are these two texts according to this model?"

### 3.2 Three operational levers

When committing to an embedding strategy on a Monday ADR, three levers determine the outcome:

**Model choice.** Which trained model do you use? In 2026 the leaderboard is more crowded than it used to be — managed services (Voyage AI, Cohere, OpenAI, Bedrock Titan) compete with open-weights families (BGE, Nomic, Stella, Jina). Each has its own training corpus, language coverage, and benchmark profile. The MTEB benchmark (Massive Text Embedding Benchmark) is the standard reference; recent leaders for English retrieval include Voyage 3.5, Cohere Embed v4, OpenAI text-embedding-3-large, and BGE-M3 (buildmvpfast.com, retrieved 2026-05-26).

**Dimensionality.** How many numbers does each vector contain? Larger dimensions usually mean better recall but linearly more storage and slower retrieval. The right number is corpus-dependent and budget-dependent. See §3.4 for the trade-off table.

**Domain coverage.** Was the model trained on text that resembles your corpus? A general-English model on legal text, scientific papers, or clinical notes will be measurably weaker than a domain-tuned alternative. Domain-tuned embeddings are a real lever; they are also typically a later-iteration investment because off-the-shelf general models are usually "good enough" to ship the first version.

### 3.3 Matryoshka Representation Learning (MRL)

A 2022 training technique, now mainstream in 2026 embedding models, that changes the dimensionality conversation. In a standard model, the output is a single vector at a fixed dimensionality — to get a smaller vector, you'd have to train a separate model. In a **Matryoshka-trained** model (named after Russian nesting dolls), the model is deliberately trained so that the **first N dimensions of its output form a useful, self-contained smaller embedding** — N can be 64, 128, 256, 512, or any prefix.

What this means operationally: you embed your corpus once at the model's full dimensionality (say, 1024), and you can store and retrieve at any truncated dimensionality by simply slicing the first K numbers off each vector. No re-embedding required. The Hugging Face write-up (retrieved 2026-05-26) reports that 128-dim MRL embeddings often match or beat 512-dim standard embeddings trained independently.

The Towards Data Science 2026 piece (retrieved 2026-05-26) reports that the combination of **256-dim MRL embeddings plus scalar quantization** can reduce vector-store infrastructure cost by ~71% while sacrificing only ~4.6% in Recall@10 — a trade-off most production systems happily make.

Many modern managed embedding models (including AWS Bedrock Titan Text Embeddings V2, Cohere Embed v3+, OpenAI text-embedding-3 family) ship with MRL built in. The practical implication for a Monday ADR: pick the model first, then pick the dimensionality. With MRL, the dimensionality is a runtime parameter, not a training-time commitment.

### 3.4 The dimensionality trade-off table

A summary of what 2026 benchmarks report across common dimensionality choices (AWS Bedrock Titan v2 docs + Towards Data Science survey, retrieved 2026-05-26):

| Dimensions | Storage cost | Query latency | Reported recall | When to choose |
|---|---|---|---|---|
| 256 | 1× | 1× | ~96–97% of 1024 baseline | High-volume, cost-sensitive, latency-sensitive |
| 512 | 2× | 1.5× | ~99% of 1024 baseline | Strong default for most enterprise corpora |
| 1024 | 4× | 2× | baseline | Quality-sensitive, moderate corpora |
| 1536 | 6× | 2.5× | ~+1–2% over 1024 | Quality-critical, smaller corpora |
| 3072 | 12× | 3–4× | ~+2–4% over 1024 | Specialised; rarely worth the cost |

A few honest qualifiers. The numbers above are reported across multiple corpora; your corpus may behave differently. The "storage cost" column counts raw vector storage; vector-store overhead, index structures, and metadata add a roughly constant overhead on top. The "query latency" column counts the similarity-search step only; embedding the query and orchestrating retrieval are roughly constant.

The cost-conscious 2026 default is **MRL-aware models at 512 dim**, and you up-shift to 1024 only if held-out evaluation says you need it.

### 3.5 Index-time vs query-time embedding

Two practical considerations that often confuse newcomers:

- **The same model must embed both the corpus AND the query.** Asymmetry breaks the geometry. If you index with model A and query with model B, the vector space is incoherent and retrieval is random.
- **Some models distinguish "passage" and "query" embedding modes.** This is NOT model asymmetry — it's the same model with different input prefixes optimised for the two roles. Voyage AI, Cohere, and some open-weights families do this. If your model supports it, use it; recall improvements are typically 2–5%.

The Monday ADR commits the embedding *model*. Dimensionality and passage/query mode are runtime parameters that you can tune in week 2's Tue–Fri exercises.

## 4. Generic Implementation

A worked example outside federal acquisitions. Here is a Python sketch for embedding a corpus and a query using a generic managed embedding service that supports MRL. Specific calls are illustrative; the shape is what matters.

```python
# Configuration — committed on the ADR.
EMBEDDING_MODEL = "managed-embed-v2"    # supports MRL truncation
INDEX_DIMENSIONS = 512                   # truncation choice — runtime parameter
SIMILARITY_METRIC = "cosine"             # match to model's documented preference

def embed_chunks(chunks):
    """Embed each chunk as a 'passage' (corpus-side semantic role)."""
    payload = [{"input": c.text, "input_type": "passage"} for c in chunks]
    response = embedding_client.embed(model=EMBEDDING_MODEL, payload=payload)
    return [
        {
            "id": c.id,
            "vector": truncate(v.embedding, INDEX_DIMENSIONS),  # MRL truncation
            "metadata": c.metadata,
        }
        for c, v in zip(chunks, response.embeddings)
    ]

def embed_query(query_text):
    """Embed the user query as a 'query' (search-side semantic role)."""
    response = embedding_client.embed(
        model=EMBEDDING_MODEL,
        payload=[{"input": query_text, "input_type": "query"}],
    )
    return truncate(response.embeddings[0].embedding, INDEX_DIMENSIONS)

def truncate(vector, n):
    """MRL truncation — slice the first n dimensions. Re-normalise to unit length."""
    truncated = vector[:n]
    norm = sum(x * x for x in truncated) ** 0.5
    return [x / norm for x in truncated]
```

Four things this sketch demonstrates. First, **the model is committed once** at the top — that's the ADR decision. Second, **dimensionality is a runtime parameter** — change `INDEX_DIMENSIONS` to 256 or 1024 and the code still works (assuming MRL support and a re-built index at the new dimensionality). Third, **passage and query inputs use different `input_type` tags** — same model, different semantic role. Fourth, **re-normalise after MRL truncation** — most retrieval systems assume unit-length vectors, and truncation breaks that without re-normalisation.

## 5. Real-world Patterns

**E-commerce — multilingual product search.** A retailer's search system handles queries in 12 languages over an English-catalogued corpus. The team chose **Cohere Embed v3 multilingual** for its MIRACL benchmark score (51.4) on cross-lingual retrieval. At 1024 dim they hit storage budget; MRL truncation to 512 saved 50% storage with negligible quality loss. Lesson: language coverage is a model choice; storage is a dimensionality choice.

**Healthcare — clinical literature retrieval.** A pharmacovigilance team over PubMed needed embeddings distinguishing close medical concepts. General-English embeddings (OpenAI text-embedding-3-large) gave acceptable recall but confused "acute kidney injury" and "chronic kidney disease" — clinically very different. A domain-tuned biomedical embedding (BGE variant on PubMed) resolved the confusion. Domain-tuning is sometimes the only path past a quality ceiling.

**Fintech — transaction search.** A bank's customer-facing transaction-search assistant needed to handle queries like "show me my coffee purchases last month." General embeddings handled this acceptably; the team's quality unlock came from **passage/query mode separation** rather than model choice. Embedding transactions in passage mode and the user query in query mode lifted recall@10 by 4% with no change to the underlying model. Cheapest possible win.

**Logistics — driver complaint analysis.** A delivery-routing company indexed customer complaint emails to surface recurring problem patterns. The corpus was small (~120k complaints) but the team initially used a 3072-dim model "because biggest is best." Retrieval latency was acceptable but storage costs grew faster than expected with corpus growth. Switching to 768-dim MRL truncation cut storage 75% with no measurable quality drop on the team's held-out probe set. The lesson: dimensionality choice is a real cost lever; default to the smaller dimension and up-shift only when held-out eval forces it.

## 6. Best Practices

- **Commit the model choice on the ADR; defer the dimensionality choice.** With MRL-aware models, dimensionality is a runtime parameter and can be tuned on held-out evaluation.
- **Default to 512 dimensions on MRL-aware models.** ~99% recall of the full dimensionality at half the storage. Up-shift only with evidence.
- **Use passage/query mode separation when the model supports it.** Cheap recall lift, no architectural change required.
- **Match the similarity metric to the model's documented preference.** Cosine is the most common default, but not universal. Read the model card before committing.
- **Domain-tune at iteration 2, not iteration 1.** General models are usually "good enough" to ship the first version. Spend the early effort on chunking and retrieval mode; come back to domain-tuned embeddings when held-out eval names the gap.
- **Plan for re-embedding from day 1.** Embedding models get superseded. The runbook for "we are switching from model A to model B" should exist before you need it — typical operations: spin up new index, embed corpus to new index, A/B retrieval against a held-out probe set, cut over when confidence is high.
- **Never mix vectors from different models in the same index.** Vector spaces are not interoperable. This is the single most common embedding mistake on a team's first re-indexing exercise.

## 7. Hands-on Exercise

**Time: 10–15 minutes — whiteboarding + small calculation.**

Imagine a corpus of 1,000,000 chunks, each ~256 tokens. You're choosing between:

- Option A: Voyage 3.5 large (1024 dim, $0.18 per 1M tokens at embed time)
- Option B: Bedrock Titan Embed v2 (1024 dim, truncatable via MRL to 512 or 256, $0.02 per 1M tokens)
- Option C: A domain-tuned open-weights model run on your own GPU (768 dim, free per-token but ~$2k/month GPU)

For each option, sketch:

1. The one-time cost to embed the corpus (option A and B only — option C is GPU rental).
2. The monthly storage cost assuming the vector store charges $0.10 per million stored 4-byte floats.
3. The query-time embedding cost assuming 100k user queries per month.

Then write one sentence explaining which option you'd pick AT WHAT DIMENSIONALITY, and what evidence would change your answer.

**What good looks like:** Your dimensionality choice should be explicitly named (e.g., "Option B at 512 dim"). Your reasoning should mention at least one trade-off (cost vs recall, vendor lock-in vs operational simplicity, latency vs quality). Your "evidence that would change my answer" sentence should be specific — e.g., "if held-out eval shows < 90% recall@10 at 512 dim, up-shift to 1024." If you can't name evidence that would flip the decision, your ADR is not falsifiable and isn't actually an ADR.

## 8. Key Takeaways

- Can you explain what an embedding model produces and what a vector dimension represents (and does not represent)?
- Can you name the three operational levers (model, dimensionality, domain coverage) and explain why model choice is the most expensive to reverse?
- Can you describe **Matryoshka Representation Learning** and explain how it changes the dimensionality conversation from a training-time commitment to a runtime parameter?
- Can you read the dimensionality trade-off table and pick a starting point appropriate to a given corpus size and budget?
- Can you avoid the most common embedding mistake — mixing vectors from different models in the same index?

## Sources

1. [Amazon Titan Text Embeddings models — Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/titan-embedding-models.html) — retrieved 2026-05-26
2. [Amazon Titan Text Embeddings V2 now available in Amazon Bedrock — AWS News Blog](https://aws.amazon.com/blogs/aws/amazon-titan-text-v2-now-available-in-amazon-bedrock-optimized-for-improving-rag/) — retrieved 2026-05-26
3. [Introduction to Matryoshka Embedding Models — Hugging Face Blog](https://huggingface.co/blog/matryoshka) — retrieved 2026-05-26
4. [Voyage 3.5 vs OpenAI vs Cohere Embedding Models 2026 — BuildMVPFast](https://www.buildmvpfast.com/blog/best-embedding-model-comparison-voyage-openai-cohere-2026) — retrieved 2026-05-26
5. [Scaling Vector Search: Comparing Quantization and Matryoshka Embeddings for 80% Cost Reduction — Towards Data Science](https://towardsdatascience.com/649627-2/) — retrieved 2026-05-26

Last verified: 2026-05-26
