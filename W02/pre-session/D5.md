---
week: W02
day: Fri
phase: Foundation
topic: "RAG evaluation dimensions + W3 Mon Plan Day prep (agentic preview)"
estimated_total_minutes: 50
last_verified: 2026-05-23
fde_situations: [1, 3, 6, 7, 9, 11, 12]
tech: [ragas, llm-as-judge, eval-harness, regression-testing, github-actions, langchain-v1, multi-agent-preview]
sources_research_briefs: []
author: instructor
---

# Pre-session reading — RAG evaluation dimensions + W3 Mon Plan Day prep

Week 2, Day 5 (Fri). Estimated total time on task: ~50 minutes. Last verified: 2026-05-23.

> Read **before** W2 Fri morning. Fri is two things: (1) build the RAG eval harness from scratch (morning war-room + practical), and (2) **first Live Defense of the programme** (one learner per pair defends one W02 scenario-alternative for 10 minutes) + **W2 standard two-tier MCQ** released 16:30. Also: this pre-read previews W3 Mon's Plan Day (agentic systems — first §0 retro on a multi-day plan-spec).

## 1. Why this matters (~85 words)

Thursday wired HITL #2 with a faithfulness threshold of 0.85 — a conservative guess. Friday's eval harness is what gives that threshold a defensible empirical basis. Without a harness, every PR change is a faith-based commit and the OIG auditor's question *"how do you know this is working?"* has no answer. The harness also closes debt **Item 12** (GHA lint disabled) at the rag-eval suite scope — the first piece of CI that actually runs. The Fri eval pattern is the muscle memory the cohort carries into W3 (eval-as-test-fixture for agentic flows) and W5 (production observability on the same dimensions).

## 2. Core concept in 5 minutes

A RAG eval harness has four parts:

**Part 1: held-out QA set.** Flat JSONL file at `tests/rag-eval/qa.jsonl`. Each row: `{query, expected_chunks: [clause_ids], expected_answer: "natural language", expected_citations: [clause_ids]}`. Seeded by the cohort from real federal-acq Q&As (the Tue vendor question, Thu's failing query, plus 20 more curated from real OASIS+ / SAM.gov Q&A archives that we've grounded in `/web-research`). This is the FIRST test in CI for this suite.

**Part 2: four RAGAS dimensions.** Per `ragas-faithfulness-only` known-bad-pattern, faithfulness alone is insufficient. The four dimensions to measure on each QA row:
- **Faithfulness** — Do the answer's claims appear in the retrieved chunks?
- **Context recall** — Of the expected_chunks, how many are in the retrieved set?
- **Context precision** — Of the retrieved chunks, how many are relevant?
- **Answer relevance** — Does the answer address the question?

Each scored 0–1 by an LLM-as-judge (Bedrock Claude Haiku — cheap, fast, good enough at scoring against rubrics).

**Part 3: eval-as-test-fixture in CI.** The eval harness runs in GitHub Actions on every PR against `ai-orchestrator` or `solicitation-service`. Output: a markdown summary commenting on the PR with per-dimension scores + a regression delta vs main. Any PR dropping any dimension >5% blocks merge.

**Part 4: regression framing.** When Mon's chunking ADR is revised in W3, the harness re-runs against the same QA set. Did the change improve or regress? The harness is the cohort's instrument for answering this. (W5 wires the same dimensions into production observability via OpenTelemetry — Fri's harness is the seed.)

**LLM-as-judge tuning is its own discipline.** The judge prompt matters. Common failure: the judge is more lenient than the human evaluator. Mitigation: pin a small set (~5 rows) of ground-truth scored-by-instructor examples; calibrate the judge against them; refuse to ship a judge prompt that disagrees with instructor scoring by more than 0.1 on average.

## 3. What to read or watch tonight

- [RAGAS — Overview of available metrics](https://docs.ragas.io/en/latest/concepts/metrics/) (~12 min read), retrieved 2026-05-23 via /web-research. Focus on the four metrics named above + the "Metric calculation" pages for each. Note: RAGAS the *library* is one implementation; the cohort can use it as a reference but the Fri harness rolls its own LLM-as-judge calls against Bedrock for tighter integration. Treat RAGAS docs as conceptual canonical, code as one impl.
- [Anthropic — Building Effective LLM-as-Judge Systems](https://www.anthropic.com/news/evaluating-ai-systems) (~10 min read), retrieved 2026-05-23 via /web-research. Anthropic-authored guidance on prompt engineering for LLM-as-judge. Focus on: rubric-style prompts beat freeform "rate this 0-10" prompts; calibration against human-scored examples is non-negotiable; chain-of-thought in the judge's output is worth the tokens.
- [GitHub Actions — Reusable workflow for matrix testing](https://docs.github.com/en/actions/sharing-automations/reusing-workflows) (~10 min skim), retrieved 2026-05-23 via /web-research. The Fri harness wires into a reusable workflow that runs the eval matrix on every PR. Note: the workflow is the FIRST piece of GHA CI that actually runs since debt Item 12 disabled the lint workflow last quarter. The cohort opens an OIG-style finding against the repo (debt Item 12) per inventory line 387.
- [LangSmith — Evaluation overview](https://docs.smith.langchain.com/evaluation) (~8 min skim — for context, NOT for implementation), retrieved 2026-05-23 via /web-research. **DO NOT IMPLEMENT THIS WEEK.** LangSmith production tracing is W5 (per D-031). Fri's harness is flat-file + GHA. Knowing what we'd graduate to in W5 is useful framing for the eval ADR.
- *(optional)* [SWE-bench — Evaluation methodology](https://www.swebench.com/) (~10 min skim), retrieved 2026-05-23 via /web-research. Production-scale eval framework. Worth knowing as comparative context — federal-acq evals don't need this depth, but the discipline of "every change runs against a held-out set" is identical.

## 4. Two questions to come in with tomorrow

1. *"What's the rubric prompt for the faithfulness judge? Specifically: what does the judge return when the answer says 'According to FAR 15.208(a)...' and FAR 15.208(a) IS in the retrieved chunks but the answer paraphrases it wrong? Is that faithful (cited the right chunk) or unfaithful (paraphrased wrong)?"* (This is the friction the judge prompt needs to resolve. Cohort defends their answer.)
2. *"If the eval harness shows 0.78 faithfulness on the current implementation but Thu's HITL threshold is 0.85 — does the harness imply the threshold is too high, or that the current implementation is too weak?"* (Both are defensible answers; the discipline is naming what the harness measures vs what it doesn't.)

## 5. Glossary refresh (terms you'll hear tomorrow)

- **Eval harness** — Codified test suite measuring RAG quality on a held-out QA set.
- **Held-out QA set** — Queries + expected results NOT used in any system prompt; the eval can't be "leaked" to the system at runtime.
- **Faithfulness** — RAGAS metric: claims-in-answer vs claims-supported-by-retrieved-context.
- **Context recall** — RAGAS metric: ground-truth chunks present in retrieval set / total ground-truth chunks.
- **Context precision** — RAGAS metric: relevant chunks in retrieval set / total chunks in retrieval set.
- **Answer relevance** — RAGAS metric: does the answer address the question (irrespective of correctness)?
- **LLM-as-judge** — Pattern: second LLM call scores the first LLM's output on a rubric.
- **Judge calibration** — Discipline: instructor scores ~5 examples; LLM judge prompt iterated until average disagreement < 0.1.
- **Regression delta** — Per-dimension score change PR vs main. >5% drop = block merge.
- **Eval-as-test-fixture** — Eval harness runs in CI on every PR, output annotates the PR.

## 6. W3 Mon Plan Day preview — first §0 retrospective on a multi-day plan-spec (5 min)

W3 Mon is the **first §0 plan retrospective** in the programme (per D-036). W3 Mon morning will retro the W2 Mon plan-spec — did Tue–Thu reality match what each pair planned Mon? Specific dimensions to come prepared to retro:

- **Chunking decision** — sub-paragraph held up? Or did Wed's parent-child indexing land where Mon didn't predict?
- **Vector store choice** — Atlas Vector Search held up against Wed's scenario-alternative research?
- **HITL #2 wiring scope** — Mon's open-questions section named the threshold; did it land where the plan expected?
- **What got dropped from the plan** — the eval harness was W2 Mon's last open question; did it ship before Fri or did Mon under-scope it?

W3 introduces **agentic systems** — single-agent first (Mon-Tue), multi-agent + LangGraph + HITL #5 (Wed-Fri). Two flows from the inventory (per D-060 resolution #2): `POST /agent/intake-triage` Mon-Tue (lighter warmup) + Evaluator → consensus → SSA handoff Wed-Fri. Both flows ship in W3.

## 7. Live Defense Fri — what to expect (3 min)

First Live Defense of the programme. One learner per pair defends one W02 scenario-alternative for 10 minutes, instructor reads from `templates/live-defense-rubric.md`. Format: 2 min defender opens with ADR decision + key trade-off; 5 min instructor question script; 2 min cohort clarifying questions; 1 min defender closing. Scoring rubric pinned to instructor — defender doesn't see scores in real time.

The pair picks which member defends Friday morning (before war-room ends). Defender preps over lunch. Defense runs 14:00.

## 8. Optional deep-dive (not required to participate)

- [Anthropic — Test-driven evaluation development](https://www.anthropic.com/news/evaluating-ai-systems) (~8 min — re-read of the section on iterating evals before iterating systems), retrieved 2026-05-23 via /web-research. The discipline of "build the eval first, then the system." Fri's harness is W2's version; W3-W6 inherit the discipline.

---

## Sources

All citations retrieved 2026-05-23 via /web-research per `pipeline/RESEARCH-PROTOCOL.md`. Recency windows applied: hot-tech 3-month (RAGAS, LangSmith, Anthropic eval guidance, LangChain v1.0), foundation-stable 12-month (GitHub Actions reusable workflows), reference-material (RAGAS docs).

## Known-bad-pattern warnings against the reading

- `ragas-faithfulness-only` blocklist — Fri's harness uses all four dimensions, not just faithfulness. If a pair argues "faithfulness is enough," the Live Defense rubric explicitly probes this.
- LangSmith docs may show v0.x LangChain integration examples — if a pair argues for adopting LangSmith this week, push back (D-031: LangSmith deferred to W5). The flat-file harness is intentional.
- "GHA matrix testing across model versions" patterns sometimes recommend running the eval across N Bedrock models. Don't this week — pin one model (Bedrock Claude Sonnet) for the answer + one model (Bedrock Claude Haiku) for the judge. Cost predictability matters; multi-model eval lands in W5.
