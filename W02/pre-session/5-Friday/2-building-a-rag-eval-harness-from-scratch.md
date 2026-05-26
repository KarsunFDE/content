---
week: W02
day: Fri
topic_slug: building-a-rag-eval-harness-from-scratch
topic_title: "Building a RAG eval harness from scratch — flat-file qa.jsonl"
parent_overview: W02/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://github.com/RulinShao/RAG-evaluation-harnesses
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://blog.premai.io/rag-evaluation-metrics-frameworks-testing-2026/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://docs.ragas.io/en/latest/concepts/metrics/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.getmaxim.ai/articles/the-5-best-rag-evaluation-tools-you-should-know-in-2026/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://medium.com/@steveinatorx_49018/building-a-financial-rag-system-pt-5-how-i-fixed-chunking-to-reach-90-recall-7f1158e934a9
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# Building a RAG eval harness from scratch — flat-file qa.jsonl

## 1. Learning Objectives

- By the end of this reading, the learner can describe the minimum-viable RAG eval harness as a flat JSONL file plus a runner, and explain why this beats a hosted platform on day one.
- By the end of this reading, the learner can name the fields a curated QA row must carry so it can grade retrieval and generation together rather than separately.
- By the end of this reading, the learner can explain what "curation discipline" means — where rows come from, why fabrication is forbidden, and how the QA set grows over time.
- By the end of this reading, the learner can articulate the cost-and-velocity trade-off between flat-file harnesses and managed eval platforms, and the threshold at which graduation makes sense.

## 2. Introduction

For most teams shipping retrieval-augmented generation in 2026, the eval harness arrives months after the first agent does. That is the wrong order. An eval harness is the contract that lets you make a change to a chunking strategy, prompt, or retrieval pipeline and answer the question "did that actually help?" with evidence rather than vibes. Without one, every PR is an argument and every regression is a surprise found by a user.

The minimum-viable harness is small: a flat-file dataset of question-and-answer pairs (typically JSONL — one JSON object per line, easy to grep, easy to diff in git), a script that runs each row through the system under test, a grader (either deterministic checks or an LLM-as-judge), and a results writer that produces a per-row scoreboard. No platform, no UI, no dashboard. Just files in a repo that ship and run wherever the code does.

The reason this is the right starting point — and the reason mature shops still keep a flat-file harness around even after adopting a hosted platform — is governance. A JSONL file under version control is a reviewable artifact. Every row's history is in `git log`. Adding a row is a PR. Removing a row is a PR. The discipline of editing the harness like code is the same discipline you want around the system the harness measures.

This reading is the generic shape of that harness. Section 4 lays out the file format and runner; Section 5 shows how three industries unrelated to federal acquisitions have used the same pattern.

## 3. Core Concepts

### 3.1 Anatomy of a QA row

The smallest useful row carries five fields:

- `query` — the input string, exactly as it would arrive from a user.
- `ground_truth_answer` — the answer a domain expert would accept. Not necessarily the only acceptable answer, but the canonical one.
- `expected_chunks` — identifiers (IDs, hashes, or section labels) of the corpus chunks that *should* be retrieved to answer this query. Lets you score retrieval independently of generation.
- `corpus_sources_expected` — which corpora (e.g., which document collections) are expected to contribute. Catches "retrieved from the wrong corpus" failure modes that pure semantic similarity scores miss.
- `metadata` — provenance for the row itself: where it came from, who curated it, when it was added. Rows without provenance get retired.

Richer rows add `difficulty` (`easy` / `medium` / `hard` / `adversarial`), `failure_modes_targeted` (e.g., `cross-corpus-precedence`, `numeric-precision`, `tenant-isolation`), and `notes` (free-text for the curator's reasoning). The richer the metadata, the easier it is to slice the scoreboard later.

### 3.2 Flat-file vs. hosted

Hosted eval platforms (LangSmith, Braintrust, Confident AI, Maxim, and similar) offer dashboards, run history, judge versioning, and dataset management features that flat files cannot. The cost is platform lock-in, an additional surface for a security review, and — most importantly — a workflow that is decoupled from the code-review loop where engineering decisions actually happen ([Maxim — Best RAG Evaluation Tools 2026](https://www.getmaxim.ai/articles/the-5-best-rag-evaluation-tools-you-should-know-in-2026/), retrieved 2026-05-26).

The decision criterion that has held up across many teams: flat-file is sufficient until at least one of three thresholds is hit — the QA set exceeds roughly 500 rows, more than three contributors are editing it concurrently, or the eval needs to run against production traffic samples rather than a fixed set. Below all three thresholds, the JSONL file in `tests/eval/qa.jsonl` is the right answer.

### 3.3 The curation rule

The rule that separates a harness that catches real bugs from one that scores well on synthetic data: **never fabricate rows**. Every QA pair traces to a real source — a real user query that surfaced in support tickets, a real document an expert annotated, a real regulatory ambiguity from an archive. Fabricated rows tend to encode the model's existing biases (because the curator is often the model) and the harness becomes a self-fulfilling prophecy.

In practice this looks like: a curation log alongside the JSONL file, citing the source URL or ticket ID for each row. Rows without provenance are quarantined to a separate `qa-unverified.jsonl` and don't gate merges. Rows with provenance gate merges. This split is the single highest-leverage governance choice ([Building a Financial RAG System — How I Fixed Chunking to Reach 90% Recall](https://medium.com/@steveinatorx_49018/building-a-financial-rag-system-pt-5-how-i-fixed-chunking-to-reach-90-recall-7f1158e934a9), retrieved 2026-05-26).

### 3.4 Growth strategy

A QA set is never finished — the question is how it grows. Three signals add rows automatically (with curator review):

- A production failure surfaced by a user → triage → if the failure mode isn't represented in the current set, add a row.
- A near-miss caught by a sibling check (e.g., a pin-test or smoke test) → if the failure mode generalises, add a dimensional row.
- A new corpus or new document type arriving in the system → at least one row per new document class.

The compounding effect is the point: a 50-row harness from week one that grows to 250 rows over a quarter — every row earned by real evidence — outperforms a 500-row synthetic harness shipped on day one. The flat-file format makes this growth visible in the repo's commit history.

## 4. Generic Implementation

A minimal harness runner, in plain Python — no framework abstractions, no `Chain` class, no `|` pipe composition:

```python
# tests/eval/run_eval.py
import json
from pathlib import Path
from system_under_test import answer_query  # the RAG pipeline being graded

DATASET = Path("tests/eval/qa.jsonl")
RESULTS = Path("tests/eval/results.jsonl")

def load_rows(path):
    with path.open() as f:
        for line in f:
            line = line.strip()
            if line and not line.startswith("#"):
                yield json.loads(line)

def grade_retrieval(retrieved_chunks, expected_chunks):
    """Recall: of the expected, how many were retrieved? Precision: of the retrieved, how many were expected?"""
    retrieved_ids = {c["id"] for c in retrieved_chunks}
    expected_ids = set(expected_chunks)
    if not expected_ids:
        return {"recall": None, "precision": None}
    recall = len(retrieved_ids & expected_ids) / len(expected_ids)
    precision = (
        len(retrieved_ids & expected_ids) / len(retrieved_ids)
        if retrieved_ids else 0.0
    )
    return {"recall": recall, "precision": precision}

def main():
    with RESULTS.open("w") as out:
        for row in load_rows(DATASET):
            response = answer_query(row["query"])  # returns answer + retrieved_chunks
            retrieval_scores = grade_retrieval(response.retrieved_chunks, row["expected_chunks"])
            out.write(json.dumps({
                "row_id": row.get("id"),
                "query": row["query"],
                "answer": response.answer,
                "retrieval": retrieval_scores,
                # generation grading (LLM-as-judge) is added in the next reading
            }) + "\n")

if __name__ == "__main__":
    main()
```

The runner is intentionally boring. It loads rows, calls the system, scores retrieval, writes results. The LLM-as-judge layer is a separate concern (covered in the next topic reading). Keeping the runner this small is what lets it survive multiple refactors of the system under test.

## 5. Real-world Patterns

**Fintech — credit-card chargeback dispute generation.** A team at a mid-size payments processor built a flat-file QA harness for an internal RAG agent that helps support reps draft chargeback responses. Curation came from the existing dispute archive (1,200 historical disputes with adjudicated outcomes). They achieved a 40% improvement in recall — from 0.526 to 0.738 — by iterating chunking strategies against the harness; without the harness they would have shipped the first chunking strategy and never noticed the regression in section-spanning regulatory citations ([Building a Financial RAG System Pt 5 — Steven B, Feb 2026](https://medium.com/@steveinatorx_49018/building-a-financial-rag-system-pt-5-how-i-fixed-chunking-to-reach-90-recall-7f1158e934a9), retrieved 2026-05-26).

**Healthcare — medication-safety decision support.** Researchers evaluating LLMs for medication-safety triage built a 91-row scenario harness covering 40 clinical vignettes across 16 specialties. The rows came from real adverse-event reports, not from synthetic generation. The harness exposed a 13.3% performance gap in high-risk scenarios versus low-risk ones — a gap that would have been invisible to an aggregate score ([Large language model as clinical decision support system augments medication safety, PMC 12629785](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12629785/), retrieved 2026-05-26).

**E-commerce — product-question answering.** A large marketplace shipped a flat-file harness covering 320 historical "is this product compatible with X" questions where customer-service had recorded the correct answer. The set was small enough to grep, small enough that every team member could read it in an afternoon. When they later moved to a hosted platform, they kept the JSONL as the canonical authoring surface and synced to the platform — the platform was the dashboard, not the source of truth ([Maxim — Best RAG Evaluation Tools 2026](https://www.getmaxim.ai/articles/the-5-best-rag-evaluation-tools-you-should-know-in-2026/), retrieved 2026-05-26).

## 6. Best Practices

- Curate rows from real evidence (tickets, archives, expert annotations) and reject fabricated rows — quarantine unverified rows in a separate file that does not gate merges.
- Keep the runner small enough that any team member can read it in five minutes; complexity belongs in the system under test, not the harness.
- Version the dataset in git alongside the code it grades; every row addition or removal is a reviewable diff.
- Add a row for every production failure that surfaces; the harness grows by evidence, not by synthetic expansion.
- Separate retrieval scoring (deterministic, fast) from generation scoring (LLM-as-judge, slower); they fail for different reasons and need different fix paths.
- Tag rows with `difficulty` and `failure_mode` metadata so the scoreboard can be sliced — an aggregate average hides the high-risk subset.
- Decide upfront when you would graduate to a hosted platform (typically: >500 rows, >3 contributors, or production-traffic eval); flat-file until then.

## 7. Hands-on Exercise

**Whiteboard exercise (15 min):** You are starting a new RAG agent that helps internal employees answer "is this expense reimbursable" questions against a company's expense policy + IRS guidance corpus. You have access to (a) the last 6 months of help-desk tickets, (b) the policy documents, (c) a domain expert (the corporate controller) for 2 hours per week.

Sketch:

1. The `qa.jsonl` row schema you would use (which fields, why).
2. Where the first 25 rows would come from, in priority order.
3. One row you would deliberately put in `qa-unverified.jsonl` rather than the main set, and why.
4. The first three failure-mode tags you would attach to rows, and why those three.

**What good looks like:** Your row schema includes provenance (`source_ticket_id` or `source_section`) and at least one slice key (`difficulty` or `failure_mode`). Your first-25 plan starts with help-desk tickets where the controller already adjudicated the answer (real evidence with ground truth) before touching synthetic expansion. Your unverified row is one where the controller hasn't yet confirmed — you separate it so it does not gate merges. Your failure-mode tags are concrete enough to slice on (e.g., `policy-vs-irs-precedence`, `numeric-threshold-edge`, `multi-receipt-claim`) rather than vague (e.g., `tricky` or `edge-case`).

## 8. Key Takeaways

- Why is JSONL-in-the-repo the right minimum-viable eval harness, and what governance properties does it have that a hosted platform doesn't?
- Which fields does a QA row need to grade retrieval and generation independently, and why does separating them matter?
- What is the curation rule that distinguishes a harness that catches real bugs from one that scores well on synthetic data?
- At what thresholds would you graduate from flat-file to hosted, and what would you keep flat-file even after graduation?

## Sources

1. [RAG Evaluation Harnesses — RulinShao GitHub](https://github.com/RulinShao/RAG-evaluation-harnesses) — retrieved 2026-05-26
2. [RAG Evaluation: Metrics, Frameworks & Testing (2026) — Prem AI](https://blog.premai.io/rag-evaluation-metrics-frameworks-testing-2026/) — retrieved 2026-05-26
3. [RAGAS — Overview of available metrics](https://docs.ragas.io/en/latest/concepts/metrics/) — retrieved 2026-05-26
4. [The 5 Best RAG Evaluation Tools You Should Know in 2026 — Maxim AI](https://www.getmaxim.ai/articles/the-5-best-rag-evaluation-tools-you-should-know-in-2026/) — retrieved 2026-05-26
5. [Building a Financial RAG System Pt 5 — How I Fixed Chunking to Reach 90% Recall (Steven B, Feb 2026)](https://medium.com/@steveinatorx_49018/building-a-financial-rag-system-pt-5-how-i-fixed-chunking-to-reach-90-recall-7f1158e934a9) — retrieved 2026-05-26
6. [Large language model as clinical decision support system augments medication safety in 16 clinical specialties — PMC 12629785](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12629785/) — retrieved 2026-05-26

Last verified: 2026-05-26
