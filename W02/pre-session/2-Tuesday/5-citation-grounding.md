---
week: W02
day: Tue
topic_slug: citation-grounding
topic_title: "Citation grounding — every quote ties to a chunk-ID + source-ID + last_revised"
parent_overview: W02/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://haruiz.github.io/blog/improve-rag-systems-reliability-with-citations
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://www.firecrawl.dev/glossary/web-search-apis/rag-grounding
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://apxml.com/courses/getting-started-rag/chapter-4-rag-generation-augmentation/attributing-sources
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://air-governance-framework.finos.org/mitigations/mi-13_providing-citations-and-source-traceability-for-ai-generated-information.html
    retrieved_on: 2026-05-26
    recency_category: federal-regulatory-6mo
  - url: https://medium.com/@Nexumo_/rag-grounding-11-tests-that-expose-fake-citations-30d84140831a
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Citation grounding — every quote ties to a chunk-ID + source-ID + last_revised

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define citation grounding and distinguish it from plain retrieval-augmented generation.
- Describe the minimum metadata that must travel with a chunk from ingestion to UI to make citations real (not theatre).
- Identify the difference between a *grounded* citation, an *unsupported* citation, and a *fabricated* citation — and design a test that distinguishes them.
- Specify a citation-bearing response envelope that downstream HITL gates and audit systems can read.

## 2. Introduction

The difference between a real RAG product and an impressive demo is almost always citations. A demo shows the LLM "knows" something. A product shows you **where** the LLM got it from — which document, which paragraph, when it was last revised, and how the retriever found it. Without that, every answer is a black box, and the only thing standing between the model's confidence and a customer's trust is good vibes.

Citation grounding is the discipline that makes "where did this come from?" a first-class engineering concern. It begins at ingestion (every chunk gets metadata), travels through retrieval (the metadata stays attached to the chunk), arrives at generation (the LLM is asked to ground its claims in specific chunks), and ends at the UI (the user can see and verify the citation). Skip any one of those stages and the citation becomes either decorative or wrong ([Firecrawl Glossary — RAG Grounding](https://www.firecrawl.dev/glossary/web-search-apis/rag-grounding), retrieved 2026-05-26).

This matters most when stakes attach to the answer: medical, legal, regulatory, financial, government. But the discipline is the same in lower-stakes domains — every developer who has shipped a customer-facing RAG product reports that citations were the difference between user trust and user complaints. This reading is the generic architecture; the day's overview shows you the specific federal-acquisitions instantiation that exercises every part of it.

## 3. Core Concepts

### What "grounded" actually means

"Grounded" is the property that **every factual claim in the generated answer traces to a specific piece of retrieved evidence**. Not "the model used context"; not "the answer references the documents"; specifically: claim X is supported by chunk Y, and you can verify that claim X actually appears in chunk Y.

There is a hierarchy of grounding quality worth knowing:

**Fabricated citation.** The model invented a citation that does not match any retrieved chunk. The cited "document" may not exist at all, or it exists but does not contain what the model claims. This is what makes legal and academic AI tools dangerous when ungrounded — the citations look plausible but are confabulated.

**Unsupported citation.** The model cited a real retrieved chunk, but the chunk does not actually support the claim. The model wrote a confident sentence and tacked a citation onto it for window-dressing.

**Grounded citation.** The model's claim is genuinely supported by the cited chunk. A human (or an LLM judge) reading the chunk would agree the chunk says what the claim says.

A real citation infrastructure has to distinguish all three. Grounding without verification is theatre.

### Required chunk metadata

A chunk that supports citation needs at minimum:

- **`chunk_id`** — a stable identifier for the specific retrieved span. UUID or hash; deterministic so re-ingestion does not break old citations.
- **`source_id`** — the document the chunk came from (URL, filename, document ID, regulation reference, whichever is natural for the corpus).
- **`source_type`** — what *kind* of source this is (`policy`, `regulation`, `support-article`, `product-spec`, `web-page`). Different source types can carry different trust levels, different precedence, different staleness tolerances.
- **`last_revised`** — when the underlying source was last updated. Stale-source detection lives off this field.
- **`section`** or **`anchor`** — where in the source document the chunk lives (heading path, page number, paragraph anchor). This is what makes "verify this citation by clicking through" actually possible.
- **`retrieval_score`** — the retriever's confidence. Useful for diagnostics, not for the UI.

Per industry guidance, the chunk metadata is the load-bearing structure for any future citation, filter, or hierarchical retrieval ([apxml — Attributing Sources in RAG Generated Output](https://apxml.com/courses/getting-started-rag/chapter-4-rag-generation-augmentation/attributing-sources), retrieved 2026-05-26). Without this metadata at ingestion time you cannot retrofit citations later; the data isn't there.

### Where metadata flows in the pipeline

```
INGESTION
  parse source → split into chunks → attach metadata → embed → store
                                       ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
                                       this is the load-bearing step

RETRIEVAL
  query → top-k chunks (each carrying its metadata)
                                  ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
                                  metadata travels alongside the text

GENERATION
  prompt = system + chunks(with IDs visible) + query
  LLM is instructed to ground claims in specific chunk IDs
  response is structured: {answer_with_inline_citations, [{chunk_id, ...}]}

VERIFICATION
  for each claim in the answer:
    check that the cited chunk actually supports the claim (LLM-judge or manual)

UI
  rendered answer shows inline citation markers
  clicking a marker reveals the source chunk, source document, last_revised
```

The discipline is end-to-end. Drop the metadata at ingestion and the rest is decorative. Drop the verification step and you do not know if the model is fabricating. Drop the UI rendering and the user cannot self-verify.

### The response envelope

Modern citation-bearing RAG systems return a structured envelope rather than a free-text answer ([Henry Ruiz — Building Trustworthy RAG Systems with In-Text Citations](https://haruiz.github.io/blog/improve-rag-systems-reliability-with-citations), retrieved 2026-05-26). A typical shape:

```json
{
  "answer": "Refunds are processed within 7–10 business days [1]. International orders may take up to 21 days [2].",
  "citations": [
    {
      "marker": "[1]",
      "chunk_id": "policy-domestic-returns-v3-chunk-04",
      "source_id": "policy-domestic-returns",
      "source_type": "policy",
      "last_revised": "2026-03-14",
      "section": "Section 3.2 — Domestic Refund Timing",
      "url": "https://internal.example.com/policies/returns#domestic-timing"
    },
    {
      "marker": "[2]",
      "chunk_id": "policy-intl-shipping-v2-chunk-09",
      "source_id": "policy-intl-shipping",
      "source_type": "policy",
      "last_revised": "2025-11-02",
      "section": "Section 5 — International Refund Processing",
      "url": "https://internal.example.com/policies/intl#refunds"
    }
  ],
  "needs_human_review": false,
  "confidence_signals": { ... }
}
```

The envelope is the contract between the RAG service and everything downstream — the UI, the HITL gate, the audit log, the eval harness. Lock the envelope shape early; refactoring it later is painful because every consumer assumed the previous shape.

### Verifying citations are real, not theatre

A citation that *looks* right and *is* wrong is worse than no citation — it generates false confidence. Standard verification techniques:

1. **Claim-decomposition + per-claim check.** Split the generated answer into atomic claims, then check each claim against the cited chunk. RAGAS faithfulness implements exactly this. If a claim is not supported by its cited chunk, flag it.
2. **Chunk-presence sanity check.** Every `chunk_id` in the response envelope must exist in the retrieved set. If the model invented a chunk ID, this catches it immediately. Trivially cheap; should be on by default.
3. **Substring presence check.** For exact quotes the model attributes to a source, verify the quote actually appears in the cited chunk. Catches the "I'll add quote marks for confidence" failure mode.
4. **Round-trip check.** Re-retrieve using the answer as a query. If the cited chunks are not in the top-k of that re-retrieval, the model's citation choice is suspicious.

Per recent practitioner write-ups, even a small battery of automated grounding tests at CI catches a high proportion of citation regressions before they ship ([Medium — RAG Grounding: 11 Tests That Expose Fake Citations](https://medium.com/@Nexumo_/rag-grounding-11-tests-that-expose-fake-citations-30d84140831a), retrieved 2026-05-26).

### Source freshness and precedence

`last_revised` exists for a reason. Two patterns it supports:

**Staleness flagging.** When a chunk's `last_revised` is older than a domain-specific threshold (90 days for regulatory; 30 days for product policies; 7 days for inventory), the UI surfaces a freshness warning. The user is told they may be reading an out-of-date answer; the model is not pretending the source is fresh.

**Precedence resolution.** When the corpus contains overlapping or contradictory sources (a primary regulation + a supplement; a public policy + an internal override), the source metadata is the input to precedence rules. The rules themselves live outside the RAG pipeline; the metadata makes the rules expressible. See finance/governance guidance on this ([FINOS AI Governance Framework — Mi-13: Citations and Source Traceability](https://air-governance-framework.finos.org/mitigations/mi-13_providing-citations-and-source-traceability-for-ai-generated-information.html), retrieved 2026-05-26).

## 4. Generic Implementation

A response-envelope schema in Pydantic — domain-neutral:

```python
from datetime import date
from pydantic import BaseModel

class Citation(BaseModel):
    marker: str                # "[1]" — matches the inline marker in `answer`
    chunk_id: str              # stable identifier of the retrieved span
    source_id: str             # the document/regulation/article this chunk came from
    source_type: str           # 'policy' | 'regulation' | 'spec' | 'kb-article'
    last_revised: date         # for staleness detection
    section: str | None        # heading path or paragraph anchor
    url: str | None            # deep link to the source if available

class RagAnswer(BaseModel):
    answer: str                            # inline-cited prose: "...as required [1]..."
    citations: list[Citation]              # one entry per inline marker
    needs_human_review: bool = False       # tripped by faithfulness/precedence rules
    confidence_signals: dict = {}          # diagnostics for the HITL gate

def verify_envelope(env: RagAnswer, retrieved_chunk_ids: set[str]) -> list[str]:
    """
    Cheap structural verification — catches fabricated chunk_ids and missing markers.
    Returns a list of human-readable problems; empty list = passed.
    """
    problems: list[str] = []
    for c in env.citations:
        if c.chunk_id not in retrieved_chunk_ids:
            problems.append(f"Fabricated chunk_id: {c.chunk_id}")
        if c.marker not in env.answer:
            problems.append(f"Citation marker {c.marker} not present in answer text")
    # Reverse direction: any inline markers without a Citation entry?
    import re
    markers_in_text = set(re.findall(r"\[\d+\]", env.answer))
    declared_markers = {c.marker for c in env.citations}
    for orphan in markers_in_text - declared_markers:
        problems.append(f"Orphan marker in answer: {orphan}")
    return problems
```

Notes on what this is doing:

- The `Citation` model is the wire contract — every downstream consumer reads exactly these fields.
- `verify_envelope` is the cheap structural check that should run on every response before it leaves the service. Catching a fabricated `chunk_id` here is one line of defence; the LLM-judge faithfulness check is the heavier downstream layer.
- `needs_human_review` is the single field every HITL gate reads. Keep it boolean and reason-rich (the reasons live in `confidence_signals`).
- This schema is content-agnostic — it works for product policies, regulations, support articles, medical guidelines, or any other domain that wants traceable answers.

## 5. Real-world Patterns

**Healthcare clinical-guidelines assistant.** A regional hospital network's clinical-guidelines assistant returns every recommendation with a citation back to a specific guideline document, the section, and the `last_revised` date. The UI surfaces a freshness warning when the guideline is older than 12 months (the network's clinical-governance window). Physicians reported that the freshness indicator built trust independently of the answer quality — they could see when to call a colleague vs trust the bot. The citation envelope was a precondition for clinical adoption.

**E-commerce policy chatbot.** A large retailer's customer-support chatbot returns answers grounded in their published policy documents. Each citation includes the policy section anchor and a deep link. The team reported that adding the deep links measurably reduced agent-escalation rates — customers could verify the policy themselves rather than escalating to confirm. The envelope shape was the unblocker, not the model itself ([Firecrawl Glossary — RAG Grounding](https://www.firecrawl.dev/glossary/web-search-apis/rag-grounding), retrieved 2026-05-26).

**Legal-research SaaS — citation-fabrication mitigation.** A legal-research platform reported that early versions of their RAG-backed research assistant occasionally produced citations to *plausible-but-nonexistent* cases. They implemented the chunk-presence sanity check (every cited `chunk_id` must be in the retrieved set) and a substring presence check (any direct quote must literally appear in the cited chunk). The combination caught fabricated citations in CI before deploy and at runtime before responses left the service.

**Financial advisory chatbot — precedence resolution.** A consumer-finance chatbot serves answers grounded in both regulatory disclosures and internal product guides. Where the two overlap, the regulatory source carries precedence. The team encoded the precedence in the response-envelope assembly layer (not the LLM prompt) — the LLM produces a draft answer with cited chunks, and a deterministic post-processor selects the regulatory citation when a conflicting product-guide chunk was also retrieved. The `source_type` field made the precedence rule expressible without a custom data model ([FINOS AI Governance Framework — Citations and Source Traceability](https://air-governance-framework.finos.org/mitigations/mi-13_providing-citations-and-source-traceability-for-ai-generated-information.html), retrieved 2026-05-26).

## 6. Best Practices

- Attach chunk metadata at ingestion time, not as an afterthought — you cannot retrofit citations into a corpus that did not carry source IDs through ingestion.
- Lock the response-envelope schema early and version it; every downstream consumer (UI, HITL gate, audit log, eval harness) assumes a stable shape.
- Make every claim in the generated answer carry an inline citation marker, and every marker resolve to exactly one Citation entry — orphans on either side are bugs.
- Run the cheap structural verification (fabricated `chunk_id` check, orphan-marker check) on every response synchronously; this is one of the highest-leverage defences against silent regressions.
- Use `last_revised` as a freshness signal in the UI — surface stale citations to the user rather than hiding them.
- Treat citation verification as a test suite, not a one-off — re-run automated grounding tests on every model swap, every prompt change, and every reranker change.
- Encode source precedence in deterministic post-processing, not in the LLM prompt — precedence rules tend to be domain-specific and audit-relevant, and post-processing is easier to test.

## 7. Hands-on Exercise

**Envelope-design whiteboard (15 min).** Pick any domain you would not normally apply to a federal-acquisitions workflow — a customer-support knowledge base, a developer documentation portal, a recipe database, a sports-stats reference, whichever you can stand up in your head fast.

Then design the response envelope for that domain. Specifically, decide:

1. What fields go in your `Citation` schema for this domain — what would the `source_type` enum be, what would `section` look like, would you need any domain-specific fields (`recipe_version`, `season_year`, `product_revision`)?
2. What is your `last_revised` staleness threshold — what is "fresh enough" in this domain?
3. What deterministic post-processing would you do to enforce precedence — is there one in your domain, and if so what is the precedence rule expressed in plain English?
4. Sketch what the inline citation UI would look like — text-only markers `[1]`, hover-cards, footnotes, sidebar panel?

**What good looks like.** You should be able to fully specify the envelope without saying "and the LLM figures out the rest." Every load-bearing decision — source-type enum, staleness threshold, precedence rule, UI shape — is a deterministic engineering choice, not a model behaviour. If you find yourself relying on the model to "be smart" about citations, push back; that is exactly the failure mode this discipline targets.

## 8. Key Takeaways

- Can I distinguish a grounded citation from an unsupported one and a fabricated one — and do I know how I would test for each?
- Do I know the minimum chunk-metadata set that has to travel from ingestion to UI for citations to be real rather than theatre?
- Can I specify a response-envelope schema for a citation-bearing RAG service and explain why locking it early matters?
- Can I name two automated grounding checks that should run on every response and explain what each one catches?

## Sources

1. [Henry Ruiz — Building Trustworthy RAG Systems with In-Text Citations](https://haruiz.github.io/blog/improve-rag-systems-reliability-with-citations) — retrieved 2026-05-26
2. [Firecrawl Glossary — What is RAG Grounding?](https://www.firecrawl.dev/glossary/web-search-apis/rag-grounding) — retrieved 2026-05-26
3. [apxml — Attributing Sources in RAG Generated Output](https://apxml.com/courses/getting-started-rag/chapter-4-rag-generation-augmentation/attributing-sources) — retrieved 2026-05-26
4. [FINOS AI Governance Framework — Mi-13: Citations and Source Traceability](https://air-governance-framework.finos.org/mitigations/mi-13_providing-citations-and-source-traceability-for-ai-generated-information.html) — retrieved 2026-05-26
5. [Medium / Nexumo — RAG Grounding: 11 Tests That Expose Fake Citations](https://medium.com/@Nexumo_/rag-grounding-11-tests-that-expose-fake-citations-30d84140831a) — retrieved 2026-05-26

Last verified: 2026-05-26
