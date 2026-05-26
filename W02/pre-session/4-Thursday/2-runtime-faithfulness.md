---
week: W02
day: Thu
topic_slug: runtime-faithfulness
topic_title: "Runtime faithfulness — how a grounded model still lies"
parent_overview: W02/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://snorkel.ai/blog/retrieval-augmented-generation-rag-failure-modes-and-how-to-fix-them/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.digitalocean.com/community/conceptual-articles/why-rag-systems-fail-in-production
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.confident-ai.com/blog/rag-evaluation-metrics-answer-relevancy-faithfulness-and-more
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.digitalapplied.com/blog/rag-system-metrics-recall-precision-faithfulness-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Runtime faithfulness — how a grounded model still lies

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define **faithfulness** in the RAGAS sense as a *response-to-chunks* metric (not chunks-to-query), and explain why that direction matters.
- Distinguish the three production RAG failure modes — **retrieval miss**, **faithfulness drop**, and **wrong-chunk retrieval** — and which metric catches each.
- Explain why a high faithfulness score (e.g., 0.92) is **not sufficient** evidence that an answer is correct.
- Name the second metric needed alongside faithfulness to catch wrong-chunk retrieval, and where in the pipeline it gets computed.
- Articulate the production threshold convention (≈ 0.85–0.90) and the false-positive vs false-negative trade-off in tuning it.

## 2. Introduction

When teams first ship retrieval-augmented generation (RAG) to production, they often assume that **grounding the model in retrieved context solves the hallucination problem**. A few months of incidents teach otherwise. A grounded model still produces confident-sounding wrong answers — only now the failure mode is harder to detect, because the answer cites a source. The source is just the wrong source.

This is the gap between a model that *invents* facts (the obvious hallucination) and a model that *faithfully composes* an answer from material that does not actually answer the question. The composition step is honest. The retrieval step was not. From the outside, both look like the same defect — a wrong answer with a citation — but the diagnostic, the metric that catches it, and the fix are completely different.

The discipline that has emerged across regulated industries — finance, healthcare, legal, government — is to evaluate RAG output along at least two orthogonal axes at runtime, on every response, before the response is shown to a user. One axis (faithfulness) catches the model going off-source. The other axis (relevance or context-precision) catches the retriever pulling the wrong source. A response that scores well on one and poorly on the other is a different bug than a response that fails both, and the routing decision the platform makes downstream depends on knowing which.

The Snorkel team's production telemetry, summarised in their 2025 failure-mode write-up, found that **roughly 12% of failures originally attributed to "hallucination" were caused by corrupted or wrong retrieval snippets**, not by the generator at all. That is the headline number to hold: a non-trivial fraction of what reads like a hallucination is actually a retrieval defect that faithfulness alone will never surface.

## 3. Core Concepts

### 3.1 What faithfulness measures (and what it does not)

The RAGAS faithfulness metric is defined as follows: take the response, extract every atomic factual claim it makes, and for each claim ask the judge model whether that claim is entailed by the retrieved context chunks. The score is the fraction of claims that are entailed. It ranges from 0 (no claim is supported) to 1 (every claim is supported).

The crucial property of this definition is its **direction**: it goes *from* the response *to* the chunks. It does not ask whether the chunks were the right chunks. It only asks whether the response was honest about whatever chunks it received. That is a useful question — but it is a different question than "did the system answer correctly."

### 3.2 The three failure modes

Production RAG failures cluster into three modes, distinguishable by which metric catches them:

| # | Failure mode | What's broken | Detected by |
|---|---|---|---|
| 1 | **Retrieval miss** | No retrieved chunk passes the similarity threshold | Low top-k similarity score; out-of-distribution query embedding |
| 2 | **Faithfulness drop** | Model makes claims not present in retrieved chunks | Low RAGAS faithfulness (response → chunks) |
| 3 | **Wrong-chunk retrieval** | Retrieved chunk's metadata or scope does not match the query's intent | Low RAGAS context-precision / answer-relevance (chunks → query) |

The first two are catchable with faithfulness plus a confidence threshold on retrieval. The third is the silent failure that ships wrong answers with sources — and the one that the most-publicised RAG postmortems describe.

### 3.3 Why relevance is the missing dimension

If faithfulness asks "did the response stay on-source," relevance (sometimes called *context precision* in the RAGAS taxonomy) asks the inverse: "were the chunks themselves on-question." A judge call compares the query intent against each retrieved chunk's content and metadata. If the query is scoped to a specific regulation, contract clause, or product line, and the retrieved chunk belongs to a *different* scope, the relevance score is low even though similarity was high enough to retrieve it.

The two scores are independent. A response can be:

- **High faithfulness, high relevance:** the happy path. Ship it.
- **Low faithfulness, high relevance:** model went off-grounding. Re-prompt or escalate.
- **High faithfulness, low relevance:** the dangerous case — confident answer from the wrong source. Escalate.
- **Low faithfulness, low relevance:** double failure. Escalate.

Three of four quadrants need human review. Only one ships automatically. That is the structural reason the runtime gate is a *conjunction*, not a single threshold.

### 3.4 Threshold tuning and the false-positive cost

A common production starting point is faithfulness ≥ 0.85 AND relevance ≥ 0.85 → auto-publish; anything else routes to a review queue. The 2026 RAG-metrics guidance from DigitalApplied positions 0.90 as a production target and below 0.70 as unsafe to ship without a human in the loop.

Tightening the threshold raises the false-positive rate (more drafts queued that the reviewer would have approved anyway). Loosening it raises the false-negative rate (more wrong-but-confident answers escape). The right setting depends on the **cost asymmetry** of the domain. In a domain where a wrong answer creates legal or financial exposure, false negatives dominate and the threshold sits high. In a domain where reviewer bandwidth is the binding constraint, false positives dominate and the threshold sits lower with periodic sampling instead.

### 3.5 The judge model itself

Both faithfulness and relevance are typically computed by an LLM-as-judge call — a separate model invocation that scores the response against the chunks. This adds two LLM calls to the hot path. Production systems use a *smaller* judge model (cheaper, faster, calibrated against ground truth) rather than running the judge with the same frontier model that generated the response. The 2026 "LLM-as-Judge Best Practices" review reports that distilled small judges achieve 10–50× lower cost than frontier judges while maintaining ≈ 85% agreement with human reviewers — higher than two humans agree with each other on borderline cases.

## 4. Generic Implementation

A minimal runtime gate, framework-agnostic, looks like the following. The example uses generic naming — `Document`, `judge_model`, `primary_model` — to keep the pattern domain-independent.

```python
# Generic two-axis RAG gate. Both scores must clear the threshold,
# or the response is routed to a review queue rather than published.

from dataclasses import dataclass

THRESHOLD_FAITHFULNESS = 0.85
THRESHOLD_RELEVANCE = 0.85

@dataclass
class GatedResponse:
    status: str            # "publish" | "review_queue"
    response_text: str
    faithfulness: float
    relevance: float
    failure_mode: str | None

def score_faithfulness(response: str, chunks: list[str], judge) -> float:
    # Decompose response into claims; for each, ask judge if entailed by chunks.
    claims = extract_claims(response, judge)
    if not claims:
        return 1.0
    supported = sum(judge.entails(claim, chunks) for claim in claims)
    return supported / len(claims)

def score_relevance(query: str, chunks: list[str], judge) -> float:
    # For each chunk, ask judge if it is on-scope for the query.
    if not chunks:
        return 0.0
    on_scope = sum(judge.relevant(query, chunk) for chunk in chunks)
    return on_scope / len(chunks)

def gate(query: str, response: str, chunks: list[str], judge) -> GatedResponse:
    # Run both judges in parallel in real code (asyncio.gather);
    # serial here for legibility.
    f = score_faithfulness(response, chunks, judge)
    r = score_relevance(query, chunks, judge)

    if f >= THRESHOLD_FAITHFULNESS and r >= THRESHOLD_RELEVANCE:
        return GatedResponse("publish", response, f, r, None)

    failure = (
        "low_faithfulness" if f < THRESHOLD_FAITHFULNESS and r >= THRESHOLD_RELEVANCE
        else "low_relevance" if r < THRESHOLD_RELEVANCE and f >= THRESHOLD_FAITHFULNESS
        else "double_failure"
    )
    return GatedResponse("review_queue", response, f, r, failure)
```

The key shape: two scores, computed independently, ANDed against thresholds. The `failure_mode` field is preserved on the gated response so downstream — the review UI, the audit log, the telemetry pipeline — can route differently for retrieval bugs vs generation bugs.

> [!instructor-review]
> **Known-bad-pattern reminder.** The blocklist entry `ragas-faithfulness-only` (see `~/fde-10-week/skills/tech-research/references/known-bad-patterns.yml`, last reviewed 2026-05-11) flags any source teaching faithfulness as sufficient. This reading explicitly contradicts that pattern — the whole point is that faithfulness alone is insufficient and the second axis (relevance / context precision) is what catches wrong-chunk retrieval.

## 5. Real-world Patterns

**Fintech — transaction-anomaly explanation.** A consumer bank deployed a RAG-grounded assistant that explains flagged transactions to fraud analysts by retrieving policy snippets. Early in production, analysts reported the assistant "confidently citing the wrong policy" on edge cases. Postmortem revealed the embeddings clustered policies by tone (formal vs. customer-facing) more strongly than by topic, so a query about wire-transfer thresholds occasionally pulled a customer-FAQ chunk that *mentioned* wire transfers. Faithfulness was high (the answer faithfully reflected the FAQ); relevance was low (the FAQ was not the authoritative policy). Adding a relevance axis dropped the wrong-policy rate by an order of magnitude.

**Healthcare — clinical-trial Q&A.** A pharmaceutical CRO built RAG over its clinical-trial protocols to answer site-coordinator questions. The protocol corpus includes both *current* protocols and *archived superseded* versions. A high-similarity retrieval against an archived protocol scored well on faithfulness (the answer matched the chunk) but was operationally wrong (the trial is now governed by the superseded version's replacement). The fix: relevance scoring keyed on the `protocol_version_status` field — a chunk from an archived protocol is automatically scored zero for relevance regardless of similarity. ([Snorkel 2025](https://snorkel.ai/blog/retrieval-augmented-generation-rag-failure-modes-and-how-to-fix-them/))

**E-commerce — product-spec assistant.** A large electronics retailer's RAG assistant retrieved spec sheets for products. When a customer asked about a specific model number, the retriever occasionally surfaced a spec sheet for a similarly-named *predecessor* model. Faithfulness scored 0.94. Relevance, scoring chunk model-number against query model-number, scored 0.31. The relevance gate caught the mismatch and routed to "ask for clarification" rather than auto-publishing a wrong spec.

**Legal — contract-clause lookup.** A legal-tech vendor reported in their 2026 evaluation write-up ([DigitalApplied](https://www.digitalapplied.com/blog/rag-system-metrics-recall-precision-faithfulness-2026)) that they targeted 90% faithfulness *and* 90% context-precision in production. Below 70% on either dimension, the response is blocked entirely. The two-axis target is what made the system underwritable by their compliance team.

## 6. Best Practices

- **Always compute two axes, not one.** Faithfulness and relevance (or context-precision) are independent; a single metric will hide the wrong-chunk failure mode in production.
- **Use a smaller, distilled judge model on the hot path.** Frontier-model judges add latency and cost without meaningfully improving agreement with humans on calibrated rubrics; 85%+ agreement is achievable at 10–50× lower cost.
- **Run the two judge calls in parallel.** They are independent — `asyncio.gather` or equivalent saves the latency of the second judge call entirely.
- **Preserve `failure_mode` on the response envelope.** Downstream routing (queue priority, audit categorisation, telemetry alerts) needs to distinguish retrieval bugs from generation bugs.
- **Tune thresholds by failure-cost asymmetry, not vibes.** Pick the false-positive vs false-negative trade-off explicitly, document it in an ADR, revisit it monthly with telemetry.
- **Sample published responses for periodic offline re-evaluation.** A small percentage of auto-published responses should be re-scored against a stronger eval set to catch threshold drift over time.
- **Block on judge unavailability rather than defaulting to publish.** If the judge service is down, the safe default is "route to queue," not "ship without evaluation."

## 7. Hands-on Exercise

**Task (whiteboard or pseudocode, 10–15 min):** You are designing the runtime gate for a RAG-grounded customer-support assistant in an industry of your choosing (not federal acquisitions). The assistant retrieves from a corpus of product documentation. Sketch:

1. The two metrics you will compute on every response, with one-sentence definitions.
2. The threshold for each metric, with a one-sentence justification keyed to the cost-asymmetry of your chosen industry.
3. The shape of the `GatedResponse` returned to the calling service — list the fields, mark which are required and which are optional.
4. One additional failure mode beyond the three in §3.2 that your industry would care about, and which metric would catch it.

**What good looks like.** A correct sketch separates faithfulness (response → chunks) from relevance (chunks → query) and is explicit about which catches which failure mode. The thresholds cite the cost asymmetry — a healthcare assistant tunes high (false negatives expensive), a games-FAQ tunes lower (false positives expensive, reviewer bandwidth tight). The envelope includes `failure_mode` for downstream routing. The "additional failure mode" question often surfaces things like *temporal staleness* (chunk is technically relevant but six months out of date) or *jurisdictional mismatch* (right policy, wrong country) — both interesting because they require *third* dimensions beyond faithfulness and relevance.

## 8. Key Takeaways

- **What does "faithfulness" actually measure?** Whether each claim in the response is entailed by the retrieved chunks — a response→chunks check, not a chunks→query check.
- **Why is faithfulness alone insufficient?** It does not detect when the retrieved chunks were themselves wrong; a high-faithfulness response from the wrong source is a wrong answer with a citation.
- **What is the second axis, and what does it catch?** Relevance (or context precision) — chunks→query — catches wrong-chunk retrieval where similarity scored high but scope did not match.
- **How should thresholds be chosen?** By explicit cost-asymmetry analysis (false-positive cost vs false-negative cost), documented in an ADR, revisited on telemetry.
- **Why is the judge model usually a smaller model?** Distilled judges hit ≈85% human agreement at 10–50× lower cost than frontier models on calibrated rubrics; using the frontier model as judge wastes latency and money without raising quality.

## Sources

1. [RAGAS — Faithfulness metric documentation](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/) — retrieved 2026-05-26
2. [Snorkel AI — RAG failure modes and how to fix them](https://snorkel.ai/blog/retrieval-augmented-generation-rag-failure-modes-and-how-to-fix-them/) — retrieved 2026-05-26
3. [DigitalOcean — Why RAG systems fail in production](https://www.digitalocean.com/community/conceptual-articles/why-rag-systems-fail-in-production) — retrieved 2026-05-26
4. [Confident AI — RAG evaluation metrics: answer relevancy, faithfulness, and more](https://www.confident-ai.com/blog/rag-evaluation-metrics-answer-relevancy-faithfulness-and-more) — retrieved 2026-05-26
5. [DigitalApplied — RAG system metrics: recall, precision, faithfulness (2026)](https://www.digitalapplied.com/blog/rag-system-metrics-recall-precision-faithfulness-2026) — retrieved 2026-05-26

Last verified: 2026-05-26
