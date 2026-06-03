---
checkpoint: 1
title: "QC Audit Expectations — Checkpoint 1: LLM Essentials + RAG Architecture"
covers: ["W01 LLM Engineering Essentials", "W02 RAG Architecture"]
delivery_date: 2026-06-09
last_verified: 2026-06-03
read_time_min: 12
audience: learner
---

# Checkpoint 1 — What to Expect (Tue 9 Jun 2026)

> Learner-facing prep brief. Covers **W1 LLM Engineering Essentials** + **W2 RAG Architecture**. The 2-hr in-person event is the **90-min written exam** + 30-min cohort debrief. The **30-min 1:1 audit interview** (Zoom) is scheduled separately across the same week. Read this once when it lands and again the morning of.

## Topics in scope

Everything in `W01/pre-session/` + `W02/pre-session/`. Linked entry points below — these are *your* pre-session readings, the same ones you've already worked.

### W1 — LLM Engineering Essentials

- [LLMs as engineering systems — probabilistic vs deterministic, production discipline](../W01/pre-session/4-Thursday/2-llms-as-engineering-systems.md)
- [Hallucination failure modes](../W01/pre-session/4-Thursday/3-hallucination-failure-modes.md)
- [Model selection criteria](../W01/pre-session/4-Thursday/4-model-selection-criteria.md)
- [AWS Bedrock model invocation](../W01/pre-session/4-Thursday/5-aws-bedrock-model-invocation.md)
- [Streaming responses](../W01/pre-session/4-Thursday/6-streaming-responses.md)
- [Retry logic & strategies](../W01/pre-session/4-Thursday/7-retry-logic-and-strategies.md)
- [Cost structure of LLMs — per-token economics, telemetry, caching levers](../W01/pre-session/4-Thursday/8-cost-structure-of-llms.md)
- [Structured JSON output (Pydantic)](../W01/pre-session/5-Friday/2-structured-json-output-pydantic.md)
- [Output validation gates](../W01/pre-session/5-Friday/3-output-validation-gates.md)
- [Context engineering — system prompts, dynamic assembly, compression](../W01/pre-session/5-Friday/4-context-engineering-system-prompts-dynamic-assembly-compression.md)
- [Prompt evaluations & lifecycle management](../W01/pre-session/5-Friday/5-prompt-evaluations-lifecycle-management.md)
- [Explicit HITL framing (HITL touchpoint #1)](../W01/pre-session/5-Friday/6-explicit-hitl-framing.md)
- [Async FastAPI integration with Spring Boot](../W01/pre-session/5-Friday/7-async-fastapi-integration-with-spring-boot.md)

### W2 — RAG Architecture

- [RAG architecture overview](../W02/pre-session/1-Monday/3-rag-architecture-overview.md)
- [Chunking strategies & vocabulary](../W02/pre-session/1-Monday/4-chunking-strategies-pre-vocabulary.md)
- [Embedding generation — model & dimensionality](../W02/pre-session/1-Monday/5-embedding-generation-model-and-dimensionality.md)
- [MongoDB Atlas Vector Search as RAG store](../W02/pre-session/1-Monday/6-mongodb-atlas-vector-search-as-rag-store.md)
- [LangChain v1.0 posture & ADR discipline](../W02/pre-session/1-Monday/7-langchain-v1-posture-and-adr-discipline.md)
- [Why naive RAG breaks](../W02/pre-session/2-Tuesday/2-why-naive-rag-breaks.md)
- [Retrieval strategies — dense / sparse / hybrid](../W02/pre-session/2-Tuesday/3-retrieval-strategies-dense-sparse-hybrid.md)
- [Reranking fundamentals](../W02/pre-session/2-Tuesday/4-reranking-fundamentals.md)
- [Citation grounding](../W02/pre-session/2-Tuesday/5-citation-grounding.md)
- [Retrieval evaluation & RAGAS metrics](../W02/pre-session/2-Tuesday/6-retrieval-evaluation-and-ragas-metrics.md)
- [Atlas index management & design artifacts](../W02/pre-session/2-Tuesday/7-atlas-index-management-and-design-artifacts.md)
- [Advanced RAG retrieval patterns](../W02/pre-session/3-Wednesday/2-advanced-rag-retrieval-patterns.md)
- [Reranking cascade designs](../W02/pre-session/3-Wednesday/3-reranking-cascade-designs.md)
- [Multi-tenant retrieval boundaries](../W02/pre-session/3-Wednesday/4-multi-tenant-retrieval-boundaries.md)
- [RAG caching & index management](../W02/pre-session/3-Wednesday/5-rag-caching-and-index-management.md)
- [Deterministic fallback patterns](../W02/pre-session/3-Wednesday/6-deterministic-fallback-patterns.md)
- [RAG pipeline composition & observability](../W02/pre-session/3-Wednesday/7-rag-pipeline-composition-and-observability.md)
- [RAG anti-patterns](../W02/pre-session/3-Wednesday/8-rag-anti-patterns.md)
- [Runtime faithfulness](../W02/pre-session/4-Thursday/2-runtime-faithfulness.md)
- [HITL #2 — RAG fallback](../W02/pre-session/4-Thursday/3-hitl-2-rag-fallback.md)
- [RAG security primer](../W02/pre-session/4-Thursday/4-rag-security-primer.md)
- [Latency tuning for RAG](../W02/pre-session/4-Thursday/5-latency-tuning-rag.md)
- [Audit trail for retrieval](../W02/pre-session/4-Thursday/6-audit-trail-for-retrieval.md)
- [Building a RAG eval harness from scratch](../W02/pre-session/5-Friday/2-building-a-rag-eval-harness-from-scratch.md)
- [LLM-as-judge for retrieval eval](../W02/pre-session/5-Friday/3-llm-as-judge-for-retrieval-eval.md)
- [Eval as test fixture & regression framing](../W02/pre-session/5-Friday/4-eval-as-test-fixture-and-regression-framing.md)
- [Iterative improvement loops — eval / fix / re-eval](../W02/pre-session/5-Friday/5-iterative-improvement-loops-eval-fix-re-eval.md)
- [Security eval extension — prompt-injection probes](../W02/pre-session/5-Friday/6-security-eval-extension-prompt-injection-probes.md)

### Cross-cutting (carried from both weeks)

- HITL placement: where to put the human, why there and not earlier/later
- Eval as a CI/CD discipline, not a one-time exercise
- Citation grounding + faithfulness vs answer-quality vs hallucination rate
- Multi-tenant retrieval safety — *"agency DLA must never see GSA's draft clause"*

---

## Example scenarios (illustrative — distinct from anything you'll see)

These are deliberately *different* scenarios than your pre-session readings AND the actual audit Q bank — but they are the same *shape* as what your auditor will hand you. Use them to test how you'd open, not as a Q&A bank. The auditor will have entirely different concrete tech anchors.

### Example A — Eval discipline at programme start

*Supporting reading:* [Prompt evaluations & lifecycle management](../W01/pre-session/5-Friday/5-prompt-evaluations-lifecycle-management.md) · [Eval as test fixture & regression framing](../W02/pre-session/5-Friday/4-eval-as-test-fixture-and-regression-framing.md) · [Explicit HITL framing](../W01/pre-session/5-Friday/6-explicit-hitl-framing.md)

> *"Your pair lead says: 'evals are a Phase-2 concern. Right now we need to ship the drafter MVP. We'll add a proper eval suite next sprint.' Defend or push back."*

**What an opening answer should touch (not exhaustively, not in order):**

- The premise is wrong. Without eval gates, you can't tell whether a change made the drafter better or worse — you're shipping on vibes
- "MVP" doesn't mean "no eval"; it means "minimum viable" — and viability *requires* you to know whether output quality is regressing. The minimum eval is a hand-curated set of representative inputs + expected-shape checks, not a fancy harness
- Frame it as a SDLC discipline question, not a tooling question — the same conversation as "we'll add tests in a follow-up sprint." That ages badly for the same reasons
- What *could* legitimately be deferred: LLM-as-judge tooling, RAGAS-style automated metrics, observability platform integration. What can't be: a curated set of inputs the team agrees represent "working" + a CI step that compares output against expectation

**Likely follow-up presses:**
- *What does the eval set look like for an LLM-driven drafter that's by nature non-deterministic?* (Schema/shape assertions + golden-set human-judged outputs + drift-detection on aggregate metrics.)
- *What eval signal would have detected a regression that ships unfaithful answers with valid-looking citations?* (Distinguish citation validity from claim entailment; press on whether the answer can be discussed in those terms.)
- *Where does the eval live? In the ai-orchestrator? In the Spring Boot service? In a separate eval-service?* (Argues for a separable concern.)

### Example B — Diagnostic tree for "confidently wrong" output

*Supporting reading:* [Hallucination failure modes](../W01/pre-session/4-Thursday/3-hallucination-failure-modes.md) · [Runtime faithfulness](../W02/pre-session/4-Thursday/2-runtime-faithfulness.md) · [Citation grounding](../W02/pre-session/2-Tuesday/5-citation-grounding.md)

> *"A contracting officer flags an LLM-generated response that's confidently incorrect — clean prose, plausible structure, wrong content. The retrieval logs look healthy, the response shape passed validation, the citation pointer resolves to a real clause. Walk me through how you'd narrow down what actually went wrong."*

**What an opening answer should touch:**

- Separate the failure-mode tree before you start picking branches. At least four distinct possibilities: (a) retrieval pulled the wrong chunks but they looked plausible, (b) retrieval pulled correct chunks and generation drifted, (c) prompt contained a leading assumption that biased generation, (d) the user query itself was ambiguous and the model "completed" it in an unhelpful direction
- The diagnostic move is *not* "look at the prompt and tweak it" — that's the symptom, not the cause. The diagnostic move is to ask: *if I held everything constant and changed exactly one input, which change would tell me which failure mode I'm in?*
- Reproducibility comes first. Can you re-run the exact same query and get the same output? If not, you've also got a determinism problem stacked on the correctness problem
- Production response: this isn't a "fix the bug" question; it's a "what would have caught this *before* the CO saw it" question. Land there before the auditor presses

**Likely follow-up presses:**
- *What's the smallest reproducer you can build for this specific case?*
- *What's the right user-facing behaviour when the system *suspects* this class of failure but hasn't proven it?* (Surface uncertainty; don't auto-publish; HITL gate.)
- *Is "the model was just wrong" an acceptable root cause to write in a post-incident note?* (No — that's giving up on the diagnostic question. Always at least name which of the failure-mode branches you couldn't rule out.)

### Example C — Build vs adopt at the retrieval layer

*Supporting reading:* [RAG architecture overview](../W02/pre-session/1-Monday/3-rag-architecture-overview.md) · [LangChain v1.0 posture & ADR discipline](../W02/pre-session/1-Monday/7-langchain-v1-posture-and-adr-discipline.md) · [Advanced RAG retrieval patterns](../W02/pre-session/3-Wednesday/2-advanced-rag-retrieval-patterns.md)

> *"You're scaffolding the W2 retrieval layer. You can either compose retrieval out of primitives (an embedding model + a vector index + your own ranking) or pull in a higher-level retrieval framework that handles chunking, retrieval, and reranking as one black box. Defend a choice."*

**What an opening answer should touch:**

- This is a build-vs-adopt-at-the-right-level question. The wrong frame is "framework yes/no"; the right frame is "what does this team need to *understand* deeply by W6 vs what should be reliable infrastructure"
- A higher-level framework hides choices that turn out to matter for federal-acquisitions workloads — chunking strategy near clause boundaries, citation surface, multi-source corpus handling. If you can't peek inside, you can't diagnose
- Composing from primitives takes longer week 1 but pays back when something breaks at week 4 — you know the parts, you can swap one without rebuilding all of them
- Trade you'd defend: framework for the *plumbing* (vector store client, embedding model invocation, batch ingestion), hand-rolled for the *decisions* (chunking, reranking strategy, citation grounding). Don't black-box the parts you'll be asked to defend in Phase 1 Gate

**Likely follow-up presses:**
- *How does this choice show up in your CI?* (Framework version pins, breaking-changes risk; primitive composition has more code to test but fewer external moving parts.)
- *What does adopting the framework lock you out of by W5?* (Custom KG-augmented retrieval, hybrid scoring across non-supported retrievers.)
- *Your pair partner says "but the framework just works." What's the diagnostic-thinking answer to that?* (Working is not the same as understanding; you can't operate what you can't diagnose.)

---

## What "meets bar" looks like

You don't need to land every keyword. You need to:

1. **Open by naming the failure mode or trade-off** — don't restate the scenario back, name what's actually being decided
2. **Show you can compare** — when the auditor offers two options or you produce them, defend *why this one over that one* in the specific scenario, not in general
3. **Handle a sideways press without losing the thread** — when the question shifts to reliability/security/SDLC, you should be able to follow without re-framing from scratch
4. **Know where the human goes** — for any AI workflow, you should be able to point to where the HITL gate sits and defend its placement

Common stuck points (don't get stuck here):

- **Reciting a bullet list without prioritising** — auditors read this as "I memorised this but didn't internalise it." Lead with the load-bearing fact, not the textbook order
- **Conflating adjacent concepts** — RAG vs fine-tuning, citation-validity vs faithfulness, prefix-cache vs response-cache — these are diagnostic confusions for the auditor; they press the lens harder if you mix them up
- **Treating LLMs as deterministic in your answer** — temperature, sampling, and prefix-conditioning all live in the answer. *"It should be deterministic"* is a wrong frame
- **Skipping the eval question** — if the auditor describes a production failure and you don't mention *what eval would have caught it in CI*, you missed half the lens
- **Citing OWASP LLM categories you don't recognise** — better to name "this looks like a prompt-injection-via-retrieval class issue" than "LLM01" if you don't remember the number

### Tier calibration (informational)

Both tiers see the same question bank in the audit. Each question has two bars — a Senior bar and an Entry bar. **Entry candidates clearing the Entry bar confidently meet bar.** **Senior candidates landing only at the Entry bar are approaching bar — that's a flag, not a fail.** Tier doesn't change the question; it changes the bar the auditor reads you against.

---

## How to prep

The four highest-leverage hours of prep this week, ordered:

1. **Re-read your own Phase-1 ADRs** (chunking strategy, embedding model, vector store choice, LangChain v1.0 posture). The auditor will press on the *trade-offs you committed to* — not on the abstract space of options. Know what you decided and why
2. **Walk the W1 Fri + W2 failure scenarios** in your head — for each, what was the failure mode, what would you try first, what's the load-bearing fix. Most audit openers anchor in a scenario shape you've already worked
3. **Pick one tech you didn't touch deeply** and read the relevant `research/` brief in the working repo end-to-end — auditors press on modernization tech (Spring Boot upgrade path, FastAPI patterns, AWS SDK v2) even at Checkpoint 1 because it's what you're working in daily. Don't be surprised by a question on async patterns or lifecycle handlers

What *not* to do:

- Don't binge new content the night before. You know what you know — sleep matters more
- Don't go in expecting yes/no questions. Every question lands with at least one follow-up

---

## Logistics

- **Exam:** in-person, both tiers in the same room, 90 min, you'll have the format walked at exam start
- **Audit interview:** Zoom, 30 min, two auditors split the cohort. You'll get a scheduling link in the cohort debrief. Pick a slot that's your *normal energy* — late afternoon is generally fine, but pick *your* high-energy window
- **Recording:** the Zoom is auto-recorded for instructor reference. Not shared outside the cohort
- **Composite:** the auditor scores each question on a 4-point scale; your per-checkpoint verdict is *meets / partial / below*. Per-question feedback comes from your instructor afterwards, not from the auditor on the call

If you have a calendar conflict that week, raise it with your instructor by **Mon 8 Jun EOD** so audit slots can be re-flowed.

---

*Brief last verified 2026-06-03. Re-verify any hot-tech reference (Bedrock model IDs, LangChain v1.0 API, Atlas Vector Search features) against your `research/` brief if you're prepping more than 30 days post-write date.*
