---
week: W02
day: Tue
topic_slug: retrieval-evaluation-and-ragas-metrics
topic_title: "Retrieval evaluation as a flat-file pattern + RAGAS metrics"
parent_overview: W02/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://blog.premai.io/rag-evaluation-metrics-frameworks-testing-2026/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.digitalapplied.com/blog/rag-system-metrics-recall-precision-faithfulness-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://deepchecks.com/rag-evaluation-metrics-answer-relevancy-faithfulness-accuracy/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://testquality.com/llm-regression-testing-pipeline/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Retrieval evaluation as a flat-file pattern + RAGAS metrics

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define the four RAGAS-style metrics — faithfulness, context recall, context precision, answer relevance — and explain what each one fails to detect on its own.
- Build a flat-file golden dataset in JSONL that supports automated regression evaluation of a RAG pipeline.
- Wire an eval harness into a CI gate so retriever, prompt, and model changes are caught before they ship.
- Argue why "RAGAS faithfulness is enough" is the wrong answer, and identify which failure modes a single-metric eval misses.

## 2. Introduction

You cannot fix what you cannot measure. Most production RAG teams in 2026 have the same history: shipped without an eval harness, hit a silent regression, instrumented in a panic, kept the instrumentation forever. Building the eval harness *before* the regression is cheaper than building it after.

This reading covers the modern RAG-evaluation discipline: the four-dimensional RAGAS-style metric set, the JSONL-flat-file golden dataset pattern, and how to wire both into a CI gate. RAGAS is the lingua franca not because it is the only framework — DeepEval, Anyscale, Confident-AI occupy the same space — but because the *metric definitions* are the field-standard reference ([Ragas — Available Metrics](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/), retrieved 2026-05-26). The flat-file pattern is deliberately unfashionable. Managed eval platforms (LangSmith, Weights & Biases Weave, Arize) are the right tool *after* you have lived with a flat-file harness and felt what an eval actually does.

## 3. Core Concepts

### Why four metrics, not one

The four RAGAS-style metrics each fail differently. Using them as a set — not one — is the discipline that catches the failure surface that any single metric would miss.

> [!instructor-review]
> Anti-pattern guarded against: known-bad-pattern `ragas-faithfulness-only`. Any source that says "RAGAS faithfulness is sufficient" is teaching a single-metric eval that misses context-recall failures (you cited correctly from the wrong corpus) and context-precision failures (you cited 30 chunks when 3 were needed). All four dimensions are required per the programme spec and per the original RAGAS authoring intent.

**Faithfulness.** Does the generated answer make claims supported by the retrieved chunks? Implemented by decomposing the answer into atomic claims using an LLM judge, then for each claim asking whether the retrieved context supports, contradicts, or is silent about it. Supported claims count toward faithfulness; unsupported claims count against ([Ragas — Available Metrics](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/), retrieved 2026-05-26).

What faithfulness misses on its own: the case where the answer is well-grounded in the *wrong* retrieved chunks. If the retriever pulled documents about cats and the model writes a perfectly faithful summary of those cat documents, faithfulness scores high — but the user asked about dogs.

**Context recall.** Did retrieval return all the chunks needed to answer? Computed against a ground-truth set of "chunks that should have been retrieved for this query." Misses the failure mode where retrieval returned all the right chunks *plus* many irrelevant ones (that is context precision's job).

**Context precision.** Of the chunks retrieved, how many were actually used / actually relevant? Catches over-retrieval — when `k = 20` would have been fine and you set `k = 50`, this metric drops.

**Answer relevance.** Does the answer address the question asked? Catches the failure where the model wrote a fluent, correct, well-grounded *paragraph about something else*. Implemented by reverse-engineering candidate questions from the answer and measuring their similarity to the original question.

### The four metrics interact

Two practical examples of why the set matters:

**High faithfulness, low context recall.** The retriever missed a critical chunk. The model wrote a perfectly faithful answer to what it had — but the answer is incomplete. Faithfulness alone reports green; the user gets half an answer.

**High faithfulness, high context recall, low context precision.** Retrieval pulled everything relevant — and a lot more. The answer is grounded; the latency is high; the cost-per-call is bloated; the model is more likely to drift toward irrelevant retrieved chunks. A precision-aware eval catches this before it shows up as a billing surprise ([PremAI — RAG Evaluation Metrics 2026](https://blog.premai.io/rag-evaluation-metrics-frameworks-testing-2026/), retrieved 2026-05-26).

### The flat-file golden dataset

A golden dataset is the held-out set of `(question, expected_answer, expected_chunks)` rows you evaluate against. The flat-file pattern stores them as JSONL — one row per line, machine-readable, diff-friendly in git. A practical structure:

```jsonl
{"id": "qa-001", "question": "What is the refund window for domestic returns?", "expected_chunks": ["policy-domestic-returns-v3-chunk-04"], "expected_answer_substrings": ["7-10 business days"], "tags": ["refund-policy", "easy"]}
{"id": "qa-002", "question": "How long do international refunds take?", "expected_chunks": ["policy-intl-shipping-v2-chunk-09"], "expected_answer_substrings": ["21 days"], "tags": ["refund-policy", "international"]}
{"id": "qa-003", "question": "I bought something on 1 December and want to return it in February — what happens?", "expected_chunks": ["policy-domestic-returns-v3-chunk-04", "policy-holiday-returns-v1-chunk-02"], "expected_answer_substrings": ["holiday return window", "extended"], "tags": ["refund-policy", "hard", "multi-chunk"]}
```

Practical guidance from the field:

- **Size:** 100–300 rows is a typical production size. Smaller (~10–50) is fine for an early scaffold; the cohort starts there. ([TestQuality — LLM Regression Testing Pipeline 2026](https://testquality.com/llm-regression-testing-pipeline/), retrieved 2026-05-26)
- **Diversity:** rows must cover the query-class distribution your real users send. A golden set of only easy single-hop questions is a comfort blanket, not an evaluation.
- **Tags:** every row gets tags so you can slice the metrics — "regression on hard queries" vs "regression on easy queries" tells you very different things.
- **Maintenance:** rotate the set as production drifts. Add representative real queries; retire rows that are no longer representative.

### Synthetic + curated, not synthetic-only

Pure-synthetic golden sets evaluate the model against the kind of questions the model itself produces — a known failure mode. The recommended pattern is **synthetic generation plus human validation**: LLMs propose candidate `(question, expected_chunks)` pairs from the corpus; a domain expert curates and edits the set ([Deepchecks — RAG Evaluation Metrics](https://deepchecks.com/rag-evaluation-metrics-answer-relevancy-faithfulness-accuracy/), retrieved 2026-05-26).

### Wiring into CI

The eval harness should run automatically on every change that could move the metrics — retriever code, prompt, chunking strategy, embedding model, reranker, the LLM itself. A reasonable CI gate:

1. **Pre-commit / PR-open:** run the harness on the golden set, fail the build if any metric drops below a per-metric threshold (e.g., faithfulness ≥ 0.85, context recall ≥ 0.80).
2. **Per-tag thresholds:** an "easy" subset can have a tighter threshold than "hard"; if "hard" regresses badly you may accept that while still gating on the floor.
3. **Diff reporting:** the CI output shows per-row changes from the previous run — which questions newly pass, which newly fail. This is the diagnostic the team actually reads.

The discipline matters because RAG regressions are silent. A retriever-config change that breaks 5% of queries does not throw an exception; it returns a polite, confident, wrong answer. The CI gate is the only thing standing between that change and production ([TestQuality — LLM Regression Testing Pipeline 2026](https://testquality.com/llm-regression-testing-pipeline/), retrieved 2026-05-26).

### LLM-as-judge — when to trust it

Faithfulness, answer relevance, and context relevance use an LLM as judge — necessary (you cannot regex "does this claim contradict this chunk") but with its own failure modes: judge bias toward the model-under-test, domain mismatch, and cost (every metric is one or more LLM calls per row). The common mitigation is a smaller cheaper judge plus a small human-labelled calibration sample to detect judge drift over time ([Digital Applied — RAG System Metrics 2026](https://www.digitalapplied.com/blog/rag-system-metrics-recall-precision-faithfulness-2026), retrieved 2026-05-26).

## 4. Generic Implementation

A minimal flat-file eval harness in Python — domain-neutral, no managed-platform dependencies:

```python
import json
from pathlib import Path

def load_golden(path: Path) -> list[dict]:
    return [json.loads(line) for line in path.read_text().splitlines() if line.strip()]

def evaluate_row(row: dict, rag_response: dict) -> dict:
    """Compute the four metrics for a single golden-set row."""
    expected_chunk_ids = set(row["expected_chunks"])
    retrieved_chunk_ids = {c["chunk_id"] for c in rag_response["citations"]}

    # Context recall — did we retrieve the ground-truth chunks?
    if expected_chunk_ids:
        recall = len(expected_chunk_ids & retrieved_chunk_ids) / len(expected_chunk_ids)
    else:
        recall = 1.0

    # Context precision — of what we retrieved, how much was expected?
    if retrieved_chunk_ids:
        precision = len(expected_chunk_ids & retrieved_chunk_ids) / len(retrieved_chunk_ids)
    else:
        precision = 0.0

    # Substring presence — a cheap proxy for "did the answer contain the expected facts?"
    answer = rag_response["answer"].lower()
    substr_present = sum(
        1 for s in row.get("expected_answer_substrings", [])
        if s.lower() in answer
    )
    expected_count = max(1, len(row.get("expected_answer_substrings", [])))
    substring_score = substr_present / expected_count

    # Faithfulness + answer relevance — LLM-judge calls (sketched here as stubs;
    # in production this is a single LLM call with structured output)
    faithfulness    = llm_judge_faithfulness(rag_response["answer"], rag_response["citations"])
    answer_relevance = llm_judge_answer_relevance(row["question"], rag_response["answer"])

    return {
        "id": row["id"],
        "context_recall": recall,
        "context_precision": precision,
        "substring_score": substring_score,
        "faithfulness": faithfulness,
        "answer_relevance": answer_relevance,
        "tags": row.get("tags", []),
    }

def aggregate(per_row: list[dict]) -> dict:
    """Aggregate per-row metrics into overall + per-tag means."""
    overall = {
        m: sum(r[m] for r in per_row) / len(per_row)
        for m in ["context_recall", "context_precision",
                  "substring_score", "faithfulness", "answer_relevance"]
    }
    per_tag = {}
    for row in per_row:
        for tag in row["tags"]:
            per_tag.setdefault(tag, []).append(row)
    per_tag_metrics = {
        tag: {m: sum(r[m] for r in rows) / len(rows)
              for m in ["context_recall", "context_precision",
                        "substring_score", "faithfulness", "answer_relevance"]}
        for tag, rows in per_tag.items()
    }
    return {"overall": overall, "per_tag": per_tag_metrics}

def ci_gate(aggregated: dict, thresholds: dict) -> tuple[bool, list[str]]:
    """Return (passed, failure_messages)."""
    failures = [
        f"{metric} = {aggregated['overall'][metric]:.3f} below threshold {threshold}"
        for metric, threshold in thresholds.items()
        if aggregated["overall"][metric] < threshold
    ]
    return (len(failures) == 0, failures)
```

Notes on what this is doing:

- The harness is one Python file. Pretty-printed JSON in, structured metrics out. No platform.
- The LLM-judge functions are sketched as `llm_judge_*` — the actual implementations are a single structured-output call to whichever judge model you trust.
- Substring presence is a cheap structural proxy for answer correctness; it does not replace LLM-judged faithfulness, but it catches catastrophic regressions for free.
- `aggregate` produces both an overall summary and a per-tag breakdown — the per-tag breakdown is what catches "you broke the hard queries while making the easy ones look slightly better."
- `ci_gate` is the build-fail step. Pure Python, no framework. The thresholds live in a config file next to the harness.

## 5. Real-world Patterns

**Consumer-app personalisation chatbot.** A consumer-finance product reported wiring a 200-row golden set into their CI pipeline and catching a retriever-config regression that would have shipped silently. The regression was a chunking-strategy change that improved precision on simple queries but tanked context recall on multi-hop queries (e.g., "I bought it during the holiday season but want to return it now") — exactly the failure mode that a single faithfulness metric would have missed because each individual answer was internally consistent. The per-tag breakdown surfaced the bug in the first CI run after the change ([TestQuality — LLM Regression Testing Pipeline 2026](https://testquality.com/llm-regression-testing-pipeline/), retrieved 2026-05-26).

**SaaS developer-docs assistant.** A developer-tools company runs a flat-file `qa.jsonl` of ~300 rows tagged by feature area. When the team upgraded their embedding model, the eval harness showed answer relevance and faithfulness held steady but context precision dropped by 0.08 across the board (more chunks were now competing for relevance). The team pinned down a sub-optimal `numCandidates` setting in the new index, fixed it, and shipped — the regression would have been invisible in a faithfulness-only eval ([PremAI — RAG Evaluation Metrics 2026](https://blog.premai.io/rag-evaluation-metrics-frameworks-testing-2026/), retrieved 2026-05-26).

**Healthcare clinical decision support.** A clinical-decision-support team built their golden set by extracting questions from clinician chart-review logs and having a senior physician curate the expected-answer field. The human-validation step caught about 18% of LLM-generated candidate questions as clinically meaningless or dangerous — the failure mode pure-synthetic golden sets cannot detect.

**Customer-support knowledge base, B2B SaaS.** A B2B SaaS team built a CI gate running eval on every PR touching the retrieval pipeline. The gate caught two production-breaking regressions in six months and blocked an embedding-model swap that benchmarked well on synthetic queries but regressed sharply on real-customer queries ([Digital Applied — RAG System Metrics 2026](https://www.digitalapplied.com/blog/rag-system-metrics-recall-precision-faithfulness-2026), retrieved 2026-05-26).

## 6. Best Practices

- Evaluate against all four RAGAS-style dimensions (faithfulness, context recall, context precision, answer relevance) — single-metric eval is a known anti-pattern.
- Start with a small flat-file golden set (10–50 rows) and grow it deliberately to 100–300 — premature managed-platform adoption hides the metric semantics from the team.
- Build the golden set with synthetic generation + human validation — never pure synthetic; never pure manual either.
- Tag every row so you can slice metrics by query class (easy vs hard, single-hop vs multi-hop, common vs edge-case).
- Wire eval into CI with per-metric thresholds; fail the build on regression, not on a single-row failure.
- Use a cheaper judge model for routine eval; keep a small human-labelled calibration sample to detect judge drift.
- Treat the golden set as a living artifact — rotate stale rows, add representative real queries, refresh quarterly.
- Re-run the full eval after every retriever change, prompt change, embedding-model swap, reranker swap, or LLM upgrade — these are exactly the changes that produce silent regressions.

## 7. Hands-on Exercise

**Build a five-row golden set and a single-row evaluator (15 min).** Pick a domain you can stand up locally (a small documentation site, your own notes folder, a Wikipedia subset).

1. Author **five JSONL rows** in `qa.jsonl` shape — each with `id`, `question`, `expected_chunks` (chunk IDs you choose / fabricate), `expected_answer_substrings`, and `tags`. At least one row should be "easy" (single-hop, single chunk), at least one "hard" (multi-hop or requires multiple chunks).
2. Write a **single Python function** `evaluate_row(row, rag_response)` that computes context recall, context precision, and substring score for that row. Do not implement the LLM-judge metrics — the structural metrics are enough for the drill.
3. Manually construct **three fake `rag_response` envelopes** for one of your rows:
   - **(a)** all expected chunks retrieved, all substrings present;
   - **(b)** only half the expected chunks retrieved;
   - **(c)** all expected chunks plus five extra irrelevant ones.
4. Run your evaluator over the three responses and write down what each of recall, precision, and substring score reports. Predict the numbers before running; check after.

**What good looks like.** Your evaluator should be ~20 lines of Python. Response (a) gives recall = 1.0, precision = 1.0, substring = 1.0. Response (b) gives recall = 0.5, precision = 1.0, substring score depends on the answer. Response (c) gives recall = 1.0, precision = small, substring still high. The point of the drill is to feel — viscerally — that recall and precision do not move together, and that watching only one of them hides the other's failure mode.

## 8. Key Takeaways

- Can I define the four RAGAS-style metrics and name one failure mode each one *fails* to detect on its own?
- Can I structure a JSONL golden-set row with the fields needed to support automated evaluation, and explain why tags matter?
- Do I know how to wire an eval harness into a CI gate and what per-metric thresholds I would set for a heterogeneous query distribution?
- Can I argue persuasively against "RAGAS faithfulness is enough" with a concrete example of a failure mode that only context-recall or context-precision would catch?

## Sources

1. [Ragas — Available Metrics](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/) — retrieved 2026-05-26
2. [PremAI — RAG Evaluation: Metrics, Frameworks & Testing (2026)](https://blog.premai.io/rag-evaluation-metrics-frameworks-testing-2026/) — retrieved 2026-05-26
3. [Digital Applied — RAG System Metrics: Recall, Precision, Faithfulness 2026](https://www.digitalapplied.com/blog/rag-system-metrics-recall-precision-faithfulness-2026) — retrieved 2026-05-26
4. [Deepchecks — RAG Evaluation Metrics: Answer Relevancy, Faithfulness, and Accuracy](https://deepchecks.com/rag-evaluation-metrics-answer-relevancy-faithfulness-accuracy/) — retrieved 2026-05-26
5. [TestQuality — LLM Regression Testing Pipeline for QA Engineers: RAG Triad & Gold Sets in 2026](https://testquality.com/llm-regression-testing-pipeline/) — retrieved 2026-05-26

Last verified: 2026-05-26
