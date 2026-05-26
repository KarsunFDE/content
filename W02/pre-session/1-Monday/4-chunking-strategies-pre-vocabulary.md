---
week: W02
day: Mon
topic_slug: chunking-strategies-pre-vocabulary
topic_title: "Chunking strategies — pre-vocabulary for the Monday ADR"
parent_overview: W02/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://www.firecrawl.dev/blog/best-chunking-strategies-rag
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://sureprompts.com/blog/chunking-strategies-for-rag
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.pinecone.io/learn/chunking-strategies/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.ibm.com/think/tutorials/chunking-strategies-for-rag-with-langchain-watsonx-ai
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://weaviate.io/blog/chunking-strategies-for-rag
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Chunking strategies — pre-vocabulary for the Monday ADR

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define **chunking** in the RAG context and explain why it's the highest-leverage design decision in the pipeline.
- Distinguish the four canonical chunking strategies — **fixed-size**, **semantic**, **structural/recursive**, and **parent-child (hierarchical)** — and name a corpus shape each is best suited to.
- Explain the **precision-vs-context** trade-off and how parent-child chunking resolves it.
- Name two operational parameters that materially affect chunking quality (chunk size, chunk overlap) and the default values reported by 2026 benchmarks.

## 2. Introduction

Chunking is the act of splitting a source document into the units that will be embedded, indexed, and retrieved. It sounds mechanical. It is, in practice, the single decision that constrains every subsequent stage of the RAG pipeline. The 2026 production RAG literature is unusually consistent on this point: **80% of RAG failures trace to the ingestion and chunking layer**, not to the language model itself (premai.io, retrieved 2026-05-26).

The reason is straightforward. Chunking determines what the retrieval layer is *physically capable of returning*. If a fact lives in a chunk that the embedding model can't represent well, no amount of LLM tuning will recover it. If a chunk is so small that it loses its surrounding context, the LLM gets a fragment and is asked to reason as if it had the whole. If a chunk is so large that it spans multiple topics, its embedding becomes a diffuse average of those topics and matches no query strongly. Chunking decides the ceiling; everything else fights for the percentage points below it.

This reading gives you just enough vocabulary to vote in this afternoon's chunking ADR. It is not the deep dive — Tuesday's session is. It is the framing you need before 09:00 Mon so the chunking conversation doesn't start from zero.

## 3. Core Concepts

### 3.1 What a chunk is, and what it gets used for

A chunk is the unit you embed. Each chunk gets passed through an embedding model and stored as a vector alongside whatever metadata you decide to track (source document ID, section heading, page number, publication date, ACL tags). At query time, the retriever finds the chunks whose vectors are nearest to the query vector and hands them to the language model as grounding context.

Two things follow from this. First, the chunk is the **smallest unit the LLM can ever see** — if a chunk is poorly bounded, the LLM is reading garbage. Second, the chunk's metadata is the only filtering signal the retrieval layer has — if you don't capture a piece of metadata at chunking time, you cannot filter on it at query time.

### 3.2 The four canonical chunking strategies

The current literature converges on four chunking families, with named variants under each.

**Fixed-size chunking.** Split the document into fixed-token (or fixed-character) windows, optionally with a small overlap between adjacent chunks. The simplest possible strategy. Cheap, deterministic, easy to reason about. Its weakness: it ignores document structure. Fixed-size routinely breaks sentences mid-thought, splits clauses across chunks, separates a list header from its list items. Firecrawl's 2026 chunking survey (retrieved 2026-05-26) calls it "a fine baseline and a poor ceiling."

A common default is **512 tokens with 50-token overlap**, but the right answer depends entirely on the corpus.

**Semantic chunking.** Split where the meaning of the text changes. Implementations compute embeddings for sentences and identify the points where adjacent-sentence similarity drops below a threshold; those are the split points. The result is chunks that respect topic boundaries. The cost: every ingestion requires embedding every sentence, which is computationally heavy at scale. Weaviate's chunking guide notes that semantic chunking can also produce wildly uneven chunk sizes, which hurts downstream retrieval calibration.

**Structural / recursive chunking.** Split on the document's own structure. LangChain's `RecursiveCharacterTextSplitter` is the canonical example — it tries a sequence of separators (paragraph break, line break, sentence boundary, word boundary) and falls back to the next-finer one only when the current grain produces chunks too large to fit the target size. For documents with native structure (headings, sections, clauses, code blocks), structural chunking preserves the author's intended units of meaning at almost no runtime cost.

Recursive character splitting at **400–512 tokens with 10–20% overlap** is the most commonly cited 2026 default for general-purpose corpora (kuriko-iwai.com, retrieved 2026-05-26; Chroma research cited there reports 85–90% recall at 400 tokens).

**Parent-child (hierarchical) chunking.** Index small "child" chunks (typically 128–256 tokens) optimised for **retrieval precision**, but when a child chunk is matched, return the larger "parent" chunk (typically 512–1024 tokens) to the LLM as **generation context**. Resolves the precision-vs-context trade-off described in §3.3.

Parent-child requires the vector store to support metadata pointers from child chunks back to their parents, but most modern vector stores handle this natively.

### 3.3 The precision-vs-context trade-off

The central tension in chunking is well-documented in the 2026 literature (Brenndoerfer, retrieved 2026-05-26):

- **Smaller chunks → higher retrieval precision.** Each chunk's embedding represents a narrower idea, so the dense vector match is more discriminating. The query "what is the dosing schedule for amoxicillin in pediatric strep?" matches a small, focused chunk on pediatric amoxicillin much more cleanly than a large, mixed chunk covering pediatric antibiotics generally.
- **Larger chunks → richer generation context.** When the chunk reaches the LLM, the model has the surrounding paragraphs available to disambiguate, qualify, or cite. The same query, given a small focused chunk, gets a precise but context-poor answer; given a larger chunk, gets a precise AND well-qualified answer.

Single-tier chunking forces a trade-off. **Parent-child resolves it** by separating the unit of retrieval from the unit of generation. You match on the small child and present the parent — best of both.

### 3.4 Chunk size and overlap as operational parameters

Two parameters dominate chunking-strategy outcomes:

- **Chunk size** — too small, you lose context; too large, you dilute the embedding. 2026 benchmarks across multiple corpora settle around **256–512 tokens for child/retrieval chunks** and **512–1024 tokens for parent/generation chunks**.
- **Chunk overlap** — small overlaps (5–20% of chunk size) help preserve information that would otherwise be cut at a chunk boundary. Larger overlaps trade storage and retrieval cost for marginal recall gains. The typical 2026 default is **10–20% overlap** for general corpora.

Both numbers are *defaults*, not laws. The right values depend on the corpus's natural unit size — for legal text where clauses are short and self-contained, the right child chunk may be one clause regardless of token count. For narrative text where paragraphs flow into each other, the right overlap may be higher to preserve cross-paragraph references.

### 3.5 Two corpora, two right answers

A useful intuition check. Suppose you have two corpora.

- **Corpus A** is a software API reference — short, structured sections, each documenting one function with parameters, return value, examples. Recursive structural chunking at the section boundary is almost certainly right. Each function's documentation IS the natural chunk. No overlap needed.
- **Corpus B** is a collection of long-form analyst reports — 30–60 page PDFs with running narrative arguments, where conclusions in section 5 depend on data introduced in section 2. Recursive chunking would shatter the cross-reference structure. Parent-child with paragraph-level children and section-level parents preserves it. Overlap matters because key transitional sentences ("As we showed earlier...") need to be findable from both sides.

Same RAG architecture, two completely different chunking ADRs. The corpus drives the chunking, not the framework.

## 4. Generic Implementation

A worked example outside federal acquisitions. Here is a Python sketch using a generic structural splitter against a hypothetical product-documentation corpus (e.g., an e-commerce platform's developer docs). The example is framework-agnostic in shape; specific library calls are illustrative.

```python
# Index-time: structural recursive chunking with overlap, plus parent-child indexing.

# Generic separators ordered coarse-to-fine.
SEPARATORS = ["\n## ", "\n### ", "\n\n", "\n", ". ", " "]
CHILD_CHUNK_SIZE = 256       # tokens — optimised for retrieval precision
CHILD_OVERLAP = 40           # ~15% of chunk size
PARENT_CHUNK_SIZE = 1024     # tokens — optimised for generation context

def recursive_split(text, separators, target_size, overlap):
    """Split text by trying separators in order; recurse if a piece is still too large."""
    if estimated_tokens(text) <= target_size:
        return [text]
    if not separators:
        # No separator left — fall back to hard split at target_size with overlap.
        return hard_split(text, target_size, overlap)
    sep, rest = separators[0], separators[1:]
    pieces = text.split(sep)
    chunks = []
    for p in pieces:
        chunks.extend(recursive_split(p, rest, target_size, overlap))
    return apply_overlap(chunks, overlap)

def chunk_document(doc):
    """Produce both child and parent chunks for the same document, with metadata pointers."""
    parents = recursive_split(doc.text, SEPARATORS, PARENT_CHUNK_SIZE, overlap=0)
    records = []
    for parent_id, parent_text in enumerate(parents):
        children = recursive_split(parent_text, SEPARATORS, CHILD_CHUNK_SIZE, CHILD_OVERLAP)
        for child_id, child_text in enumerate(children):
            records.append({
                "id": f"{doc.id}::p{parent_id}::c{child_id}",
                "text": child_text,           # what gets embedded
                "parent_text": parent_text,   # what gets passed to the LLM at query-time
                "doc_id": doc.id,
                "section_path": doc.section_path_at(parent_id),
                "published_on": doc.published_on,
            })
    return records
```

Three things this sketch demonstrates without committing to any specific framework. First, the **separators list is ordered coarse-to-fine** — that's the recursive-splitter pattern. Second, **child and parent are produced from the same source text** but with different size targets and overlaps. Third, **metadata travels with the chunk** — `doc_id`, `section_path`, and `published_on` are captured at chunking time because that's the only point at which the source information is in scope.

The retrieval-time counterpart embeds only `text` (the child), but returns `parent_text` to the LLM. The vector store needs to support both columns; almost all modern stores do.

## 5. Real-world Patterns

**Healthcare — radiology report indexing.** A 2024 RAG system at a US academic medical centre indexed historical radiology reports for clinician lookup. The team initially used fixed-size 512-token chunks and saw mediocre recall — radiology reports have a predictable structure ("FINDINGS", "IMPRESSION", "RECOMMENDATIONS") and arbitrary token-window splits routinely cut across these sections. Switching to structural chunking on the section headers immediately moved recall from ~60% to ~85%. The corpus had structure; the team had been ignoring it.

**E-commerce — product reviews.** A large online retailer indexed customer reviews to power a Q&A feature ("does this product run small?"). The team's first chunking pass was per-review — one chunk per review. Recall was poor because individual reviews often mentioned multiple attributes (fit, fabric, durability) and the per-review embedding diluted the signal. Switching to sentence-level child chunks with the full review as parent context immediately fixed it: queries about "fit" matched the relevant sentence inside the review, and the LLM saw the whole review for context. Classic parent-child.

**Logistics — driver SOPs.** A delivery company indexed driver SOPs written as numbered steps (1. Park truck. 2. Engage parking brake. ...). Fixed-size chunking shattered step sequences across boundaries. Structural chunking on the step boundary with small overlap was the obvious fix once they looked at the corpus structure. The lesson: **read your corpus before you chunk it**.

**Gaming — community Q&A.** A multiplayer game's wiki had inconsistent structure — some pages had clean headers, others were stream-of-consciousness. Semantic chunking handled both shapes because it split on actual topic shifts, not on formatting. The cost (embedding every sentence at ingestion) was acceptable because re-indexing was infrequent.

## 6. Best Practices

- **Read the corpus before you chunk it.** Spend 30 minutes looking at 20 random documents. The right chunking strategy is usually obvious from the corpus structure.
- **Default to recursive structural chunking with overlap** for general-purpose corpora. 400–512 tokens, 10–20% overlap, separators ordered coarse-to-fine.
- **Use parent-child when you need both precision AND context.** This is most enterprise corpora. The implementation cost is small; the recall + context gain is large.
- **Capture metadata at chunking time.** Anything you want to filter on at query time (tenant, ACL, publication date, document type) must be on the chunk record. You cannot retro-fit metadata cheaply.
- **Match overlap to the corpus's transitional-reference density.** Narrative corpora with lots of "as we discussed earlier" need more overlap. Independent-clause corpora need almost none.
- **Test chunking with a held-out QA set, not by reading chunks.** Eyeballing chunks tells you they're "fine"; running retrieval against real questions tells you whether they're useful.
- **Treat chunking as the first ADR.** Every downstream choice (embedding model, retrieval mode, citation surface) is partially determined by it. Commit chunking before you commit anything else.

## 7. Hands-on Exercise

**Time: 10–15 minutes — small code task.** Take a single Markdown file from a domain you know (other than federal acquisitions) — README, blog post, technical doc, recipe collection, anything with native section structure.

Write (or pseudocode) a recursive splitter that:

1. Splits on `## ` headings first, then `### `, then paragraph breaks, then sentences.
2. Targets 256-token child chunks with 15% overlap.
3. Tags each chunk with the heading path it lives under (e.g., `"## Setup > ### Database"`).

Run it. Inspect the output. Then ask yourself: if I queried this corpus for a specific fact, does the chunk that contains the fact carry enough surrounding context that the LLM could answer correctly? Or does it carry so much extra material that the embedding dilutes?

**What good looks like:** Your chunks should mostly respect heading boundaries — a chunk that crosses from one `##` section to another is a bug. Your heading-path tag should let you reconstruct where the chunk came from without re-reading the source. Token counts should cluster around your 256 target with a tail; very small chunks (< 64 tokens) suggest the splitter ran out of structure to split on and you may need a finer-grained separator.

## 8. Key Takeaways

- Can you explain why **chunking determines the retrieval ceiling** for the whole pipeline, and why 80% of RAG failures trace here?
- Can you name the four canonical chunking strategies (fixed-size, semantic, structural/recursive, parent-child) and a corpus shape each is best suited to?
- Can you describe the **precision-vs-context trade-off** and how parent-child chunking resolves it?
- Can you state reasonable defaults for chunk size and overlap, and explain why those numbers are starting points rather than laws?
- Can you read a corpus and pick a chunking strategy from its structure, rather than picking a chunking strategy and forcing the corpus through it?

## Sources

1. [Best Chunking Strategies for RAG (and LLMs) in 2026 — Firecrawl](https://www.firecrawl.dev/blog/best-chunking-strategies-rag) — retrieved 2026-05-26
2. [Chunking Strategies for RAG: Fixed, Semantic, Recursive, and Parent-Document — SurePrompts](https://sureprompts.com/blog/chunking-strategies-for-rag) — retrieved 2026-05-26
3. [Chunking Strategies for LLM Applications — Pinecone](https://www.pinecone.io/learn/chunking-strategies/) — retrieved 2026-05-26
4. [Chunking strategies for RAG tutorial using Granite — IBM](https://www.ibm.com/think/tutorials/chunking-strategies-for-rag-with-langchain-watsonx-ai) — retrieved 2026-05-26
5. [Chunking Strategies to Improve LLM RAG Pipeline Performance — Weaviate](https://weaviate.io/blog/chunking-strategies-for-rag) — retrieved 2026-05-26

Last verified: 2026-05-26
