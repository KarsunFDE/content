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

## At a glance

| Surface | Format | Duration | When |
|---|---|---|---|
| **Exam (in-person)** | Scenario-grounded MCQ (Entry tier) or MCQ + short scenario (Senior tier). Both tiers run simultaneously. | 90 min | Tue 9 Jun, inside the 2-hr block |
| **Cohort debrief** | Instructor walks the flow, no grades surfaced, schedules audit slots | 30 min | Same 2-hr block, after the exam |
| **Audit interview (1:1)** | Zoom, two auditors splitting the cohort 3-and-3. Tech-grounded opener, then 7-thinking-dim follow-ups. | 30 min per candidate | Async slot you pick that week; you book your own time |

Two different surfaces, two different things being tested. The **exam** asks *"can you demonstrate the named competencies?"*. The **audit** asks *"can you defend the tech you've been working with under interview pressure — and shift frames when the question presses sideways?"*

---

## What the audit auditor is doing

Most audit questions start from **a concrete piece of tech** — a model invocation, a chunking decision, a vector store choice, a citation pattern. The auditor opens with a tech-grounded scenario, lets you work the surface for 3-5 min, then **presses sideways** through follow-ups that ladder across the 7 thinking dimensions. Each press is testing whether you can shift frames, not whether you remember a definition.

The 7 lenses the auditor uses (you don't need to memorise them — recognise the *shape* of the press):

| # | Lens | What a press from this lens sounds like |
|---|------|----------------------------------------|
| T1 | System | *"What does this change for the team that owns the audit log?"* |
| T2 | Architecture | *"What does this lock you out of in 6 months?"* |
| T3 | Design | *"Sketch the request/response — what's the failure shape?"* |
| T4 | Data & Cloud | *"Postgres or Atlas — defend it. What's the IAM blast radius?"* |
| T5 | Reliability & Security | *"What breaks first under load? OWASP exposure?"* |
| T6 | SDLC & Delivery | *"What test would you write? How do you roll back at 3am?"* |
| T7 | AI Thinking | *"Where's the HITL gate, and why there and not earlier?"* |

For Checkpoint 1, **T7 (AI Thinking) is the spine** — at least 4 of the questions your auditor picks for you will be tagged primary-T7. Expect heavy pressure on grounding, eval discipline, HITL placement, and the cost/latency/quality triangle.

---

## Topics in scope

Everything in `W01/pre-session/` + `W02/pre-session/` + the war-room incidents you actually worked. Linked entry points below — these are *your* pre-session readings, the same ones you've already worked.

### W1 — LLM Engineering Essentials

- [Hallucination failure modes](../W01/pre-session/4-Thursday/3-hallucination-failure-modes.md)
- [Model selection criteria](../W01/pre-session/4-Thursday/4-model-selection-criteria.md)
- [AWS Bedrock model invocation](../W01/pre-session/4-Thursday/5-aws-bedrock-model-invocation.md)
- [Streaming responses](../W01/pre-session/4-Thursday/6-streaming-responses.md)
- [Retry logic & strategies](../W01/pre-session/4-Thursday/7-retry-logic-and-strategies.md)
- [Structured JSON output (Pydantic)](../W01/pre-session/5-Friday/2-structured-json-output-pydantic.md)
- [Output validation gates](../W01/pre-session/5-Friday/3-output-validation-gates.md)
- [Context engineering — system prompts, dynamic assembly, compression](../W01/pre-session/5-Friday/4-context-engineering-system-prompts-dynamic-assembly-compression.md)
- [Prompt evaluations & lifecycle management](../W01/pre-session/5-Friday/5-prompt-evaluations-lifecycle-management.md)
- [Explicit HITL framing (HITL touchpoint #1)](../W01/pre-session/5-Friday/6-explicit-hitl-framing.md)
- [Async FastAPI integration with Spring Boot](../W01/pre-session/5-Friday/7-async-fastapi-integration-with-spring-boot.md)

### W2 — RAG Architecture

- [W2 PLAN — week shape](../W02/PLAN.md)
- W2 pre-session: Mon (RAG architecture intro + vector DB landscape), Tue (chunking + embedding selection), Wed (advanced RAG patterns + multi-tenant retrieval), Thu (RAG fallback patterns + HITL #2), Fri (RAG eval dimensions). Files land in `W02/pre-session/` as the week opens.
- W2 war-room incidents (Mon→Fri) — these *are* the scenarios you'll be pressed on; the wording will differ, the shape will not. Reference [`W02/war-room/`](../W02/war-room/).

### Cross-cutting (carried from both weeks)

- Probabilistic vs deterministic systems — when each is appropriate, where the contract sits
- HITL placement: where to put the human, why there and not earlier/later
- Eval as a CI/CD discipline, not a one-time exercise
- Citation grounding + faithfulness vs answer-quality vs hallucination rate
- Multi-tenant retrieval safety — *"agency DLA must never see GSA's draft clause"*

> **Additional QC-team prep packets** — your instructor will share two reference documents from the QC team: **FDE_week1_LLM_Essentials** and **FDE_week2_RAG_Architecture**. These are higher-density scenario banks the QC team has flagged as representative of the *flavour* of question you'll meet — work through them as practice, not as a leak. The actual checkpoint questions are authored from the same content surface but worded differently. Memorising answers from those packets is the wrong move; internalising the *thinking shape* is the right one.

---

## Example scenarios (illustrative — distinct from anything you'll see)

These are deliberately *different* scenarios than your week's war-rooms, your pre-session readings, the QC prep packets, AND the actual audit Q bank — but they are the same *shape* as what your auditor will hand you. Use them to test how you'd open, not as a Q&A bank. The auditor will have entirely different concrete tech anchors.

### Example A — Eval discipline at programme start (T6 / T7)

> *"Your pair lead says: 'evals are a Phase-2 concern. Right now we need to ship the drafter MVP. We'll add a proper eval suite next sprint.' Defend or push back."*

**What an opening answer should touch (not exhaustively, not in order):**

- The premise is wrong. Without eval gates, you can't tell whether a change made the drafter better or worse — you're shipping on vibes
- "MVP" doesn't mean "no eval"; it means "minimum viable" — and viability *requires* you to know whether output quality is regressing. The minimum eval is a hand-curated set of representative inputs + expected-shape checks, not a fancy harness
- Frame it as a SDLC discipline question, not a tooling question — the same conversation as "we'll add tests in a follow-up sprint." That ages badly for the same reasons
- What *could* legitimately be deferred: LLM-as-judge tooling, RAGAS-style automated metrics, observability platform integration. What can't be: a curated set of inputs the team agrees represent "working" + a CI step that compares output against expectation

**Likely follow-up presses:**
- *T7 — what does the eval set look like for an LLM-driven drafter that's by nature non-deterministic?* (Schema/shape assertions + golden-set human-judged outputs + drift-detection on aggregate metrics.)
- *T5 — what eval signal would have detected a regression that ships unfaithful answers with valid-looking citations?* (Distinguish citation validity from claim entailment; press on whether the answer can be discussed in those terms.)
- *T2 — where does the eval live? In the ai-orchestrator? In the Spring Boot service? In a separate eval-service?* (Argues for a separable concern.)

### Example B — Diagnostic tree for "confidently wrong" output (T7 / T3)

> *"A contracting officer flags an LLM-generated response that's confidently incorrect — clean prose, plausible structure, wrong content. The retrieval logs look healthy, the response shape passed validation, the citation pointer resolves to a real clause. Walk me through how you'd narrow down what actually went wrong."*

**What an opening answer should touch:**

- Separate the failure-mode tree before you start picking branches. At least four distinct possibilities: (a) retrieval pulled the wrong chunks but they looked plausible, (b) retrieval pulled correct chunks and generation drifted, (c) prompt contained a leading assumption that biased generation, (d) the user query itself was ambiguous and the model "completed" it in an unhelpful direction
- The diagnostic move is *not* "look at the prompt and tweak it" — that's the symptom, not the cause. The diagnostic move is to ask: *if I held everything constant and changed exactly one input, which change would tell me which failure mode I'm in?*
- Reproducibility comes first. Can you re-run the exact same query and get the same output? If not, you've also got a determinism problem stacked on the correctness problem
- Production response: this isn't a "fix the bug" question; it's a "what would have caught this *before* the CO saw it" question. Land there before the auditor presses

**Likely follow-up presses:**
- *T6 — what's the smallest reproducer you can build for this specific case?*
- *T5 — what's the right user-facing behaviour when the system *suspects* this class of failure but hasn't proven it?* (Surface uncertainty; don't auto-publish; HITL gate.)
- *T7 — is "the model was just wrong" an acceptable root cause to write in a post-incident note?* (No — that's giving up on the diagnostic question. Always at least name which of the failure-mode branches you couldn't rule out.)

### Example C — Build vs adopt at the retrieval layer (T4 / T2 / T7)

> *"You're scaffolding the W2 retrieval layer. You can either compose retrieval out of primitives (an embedding model + a vector index + your own ranking) or pull in a higher-level retrieval framework that handles chunking, retrieval, and reranking as one black box. Defend a choice."*

**What an opening answer should touch:**

- This is a build-vs-adopt-at-the-right-level question. The wrong frame is "framework yes/no"; the right frame is "what does this team need to *understand* deeply by W6 vs what should be reliable infrastructure"
- A higher-level framework hides choices that turn out to matter for federal-acquisitions workloads — chunking strategy near clause boundaries, citation surface, multi-source corpus handling. If you can't peek inside, you can't diagnose
- Composing from primitives takes longer week 1 but pays back when something breaks at week 4 — you know the parts, you can swap one without rebuilding all of them
- Trade you'd defend: framework for the *plumbing* (vector store client, embedding model invocation, batch ingestion), hand-rolled for the *decisions* (chunking, reranking strategy, citation grounding). Don't black-box the parts you'll be asked to defend in Phase 1 Gate

**Likely follow-up presses:**
- *T6 — how does this choice show up in your CI?* (Framework version pins, breaking-changes risk; primitive composition has more code to test but fewer external moving parts.)
- *T2 — what does adopting the framework lock you out of by W5?* (Custom KG-augmented retrieval, hybrid scoring across non-supported retrievers.)
- *T7 — your pair partner says "but the framework just works." What's the diagnostic-thinking answer to that?* (Working is not the same as understanding; you can't operate what you can't diagnose.)

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
2. **Walk the W1 Fri + W2 war-room timelines** in your head — what was the incident, what did the cohort try first, what was the load-bearing fix. Most audit openers anchor in a war-room shape you've already touched
3. **Practice on the QC prep packets** (`FDE_week1_LLM_Essentials` + `FDE_week2_RAG_Architecture`) — *not to memorise answers*, but to feel the question shape and the breadth. Time-box yourself: if a question takes you more than 2 min to find an opening, that's a topic to revisit
4. **Pick one tech you didn't touch deeply** and read the relevant `research/` brief in the working repo end-to-end — auditors press on modernization tech (Spring Boot upgrade path, FastAPI patterns, AWS SDK v2) even at Checkpoint 1 because it's what you're working in daily. Don't be surprised by a question on async patterns or lifecycle handlers

What *not* to do:

- Don't try to memorise the QC packets verbatim. The packets are *flavour*, not the answer key
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
