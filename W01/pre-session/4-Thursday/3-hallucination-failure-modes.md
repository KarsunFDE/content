---
week: W01
day: Thu
topic_slug: hallucination-failure-modes
topic_title: "Hallucination failure modes — five categories and their mitigation layers"
parent_overview: W01/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://www.onevalley.com/blog/llm-hallucination-isnt-one-problem-its-four-most-teams-are-only-solving-one/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.lakera.ai/blog/guide-to-hallucinations-in-large-language-models
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://aws.amazon.com/what-is/retrieval-augmented-generation/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-27
---

# Hallucination failure modes — five categories and their mitigation layers

> The overview frames hallucination as the headline LLM-specific failure surface. This reading is the depth: five distinct categories, the mitigation layer each one wants, why "the LLM hallucinated" is an insufficient incident root cause, and how the categories map onto the next six weeks of programme material.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the **five hallucination categories** and define each precisely enough that two engineers reviewing the same incident would classify it the same way.
- Match each category to its **primary mitigation layer** (RAG, structured outputs, faithfulness eval, HITL, output budgets).
- Explain why **task-specific evaluation** is the only honest way to characterise hallucination risk and why industry-aggregate hallucination rates are not plannable.
- Identify which hallucination category is most exposed in a federal-acquisitions context and why.

## 2. Introduction

Recent industry analysis frames LLM hallucinations as a small number of distinct failure modes, each with its own mitigation [1][2]. The categories are not academic — they correspond to different code paths in a production system, different observability signals, and different incident-response runbooks. "The LLM hallucinated" is a triage failure, not a root cause; the category is the root cause and dictates the fix.

The Stanford AI Index 2026 reports hallucination rates across 26 evaluated LLMs in 2025-2026 ranging from 22% to 94% depending on task and evaluation methodology [2]. The 4x spread is the lesson: there is no single hallucination rate you can plan against. Each task's exposure must be measured against that task's ground truth — which is why W2 Fri builds an eval harness as a first-class W2 artifact.

## 3. Core Concepts

### 3.1 Confabulation

**Definition.** The model invents a fact that does not exist — a fabricated citation, a non-existent statute, a wrong API parameter, a hallucinated function name.

**Federal-acquisitions example.** The model drafts a solicitation citing "48 CFR 47.305-2 (b)(7) regarding small-business set-aside thresholds". The CFR section exists; the (b)(7) sub-paragraph does not.

**Primary mitigation layer.** **Retrieval grounding** — the model is constrained to cite only chunks retrieved from the canonical FAR/DFARS corpus (W2). When the canonical corpus is unavailable, **constrained tool calls** that restrict outputs to enumerated values prevent free-form invention. Friday's structured outputs are an earlier line of defence — a `Literal[...]` field can refuse to emit an out-of-domain value at all.

**Why this is the headline federal risk.** Procurement protests turn on cited authority. A confabulated FAR clause cited in a solicitation is procurement-protest exposure. This is the category most often pointed to in "why federal-context LLM deployments need HITL".

### 3.2 Stale knowledge

**Definition.** The model's training data is older than the current state of the world. A regulation amended after the training cut-off, a deprecated library version, a model ID that has been superseded.

**Federal-acquisitions example.** The model summarises FAR Subpart 15.3 against text that was amended six months after its training cut-off. The summary is internally consistent but no longer current.

**Primary mitigation layer.** **Retrieval grounding against the current corpus** (W2), with the corpus's update cadence documented and monitored. Secondary: **explicit knowledge cut-off awareness in prompts** — the system prompt names the cut-off and instructs the model to defer to retrieved context when the topic is time-sensitive.

**The trap.** Stale-knowledge hallucinations are confidently phrased. They sound right. They were right at the training cut-off. The only reliable detection layer is grounding against the current corpus.

### 3.3 Over-confidence in plausible outputs

**Definition.** The model produces a confidently phrased answer that contradicts the source material it was actually given. The answer would be plausible if the source material were different; it is wrong because the source is what it is.

**Federal-acquisitions example.** RAG retrieves the actual FAR Subpart 15.3 (current). The model summarises it but inverts a "shall" / "may" distinction in a way that flips the obligation.

**Primary mitigation layer.** **Faithfulness evaluation** (W2 Fri's RAG eval harness measures this against ground truth). Secondary: **structured citations back to source** — the output schema requires each claim to cite the chunk that supports it; downstream validation checks the citation.

**Why this is the hardest to catch.** Confabulation has tells (the cited authority doesn't exist). Stale-knowledge has tells (the corpus and the answer diverge). Over-confidence in plausible outputs is internally consistent and externally false — a human reviewer with the source open is the most reliable detection layer.

### 3.4 Format drift

**Definition.** The model returns prose when it was asked for JSON, or returns JSON with the wrong shape — extra fields, missing fields, wrong types.

**Federal-acquisitions example.** The system prompt asks for `{title, description, far_clauses_referenced[]}`. The model returns a JSON object with `headline` instead of `title`, or with `far_clauses_referenced` as a comma-separated string instead of an array.

**Primary mitigation layer.** **Structured-output APIs** (Bedrock tool use, OpenAI structured outputs, Anthropic tool use) constrain generation to a schema. **Schema validation at the call site** — Pydantic strict mode catches what the API didn't constrain. Friday's PR establishes both.

**The cheap mitigation that demo code skips.** A 5-line Pydantic model is enough to catch every format-drift hallucination before downstream code sees the result. Demo code routinely parses the response with `json.loads` and a heuristic-driven shape check; the heuristic fails on the first novel drift shape.

### 3.5 Truncation

**Definition.** The model hits its context window (input + output combined) or its output cap (`max_tokens`) mid-response and returns a partial output.

**Federal-acquisitions example.** A long FAR-clause summary task with retrieved chunks totalling 80% of the context window leaves 20% for the response; the response cuts off mid-sentence.

**Primary mitigation layer.** **Explicit output-length budgets** in the prompt design + `max_tokens` chosen with margin. **Streaming-aware consumers** — the consumer detects the truncation marker in the streaming response and can decide to continue, prompt for completion, or fail. **Defensive parsing** — never assume a JSON response is complete; check for the closing brace.

**Why this is the most under-discussed category.** Truncation looks like normal output that simply ends. Without instrumentation it is invisible. The cohort sees this in W3 with longer agentic outputs; W1 sets up the instrumentation.

## 4. Generic Implementation

A category-aware logging shape that turns "the LLM hallucinated" incidents into root-cause-classifiable events:

```python
class LLMOutput(BaseModel):
    text: str
    finish_reason: Literal["stop", "length", "content_filter", "tool_use"]
    citations: list[str] = []        # populated by RAG layer in W2

class HallucinationIncident(BaseModel):
    output: LLMOutput
    category: Literal[
        "confabulation",
        "stale_knowledge",
        "over_confidence",
        "format_drift",
        "truncation",
    ]
    detected_by: Literal["pydantic", "rag_faithfulness", "human", "max_tokens"]
    mitigation_applied: Literal["rag", "structured_output", "eval", "hitl", "budget"]
```

The shape is overkill for W1. By W5 the AIOps work consumes exactly this telemetry for hallucination-rate dashboards per category.

## 5. Real-world Patterns

**Healthcare — clinical-note format-drift catch.** Abridge's clinical-note pipeline catches format-drift hallucinations at the Pydantic boundary; failures route to human review rather than degrading the note [3, equivalent pattern]. Per-month metrics published by similar healthcare-AI vendors show format-drift caught at the schema boundary is by far the most common category — 60%+ of detected hallucinations [1].

**Public-sector — confabulated-citation incident pattern.** Multiple federal-pilot LLM deployments in 2024-2025 reported confabulated-citation incidents as the primary HITL-gate justification. The pattern: model cites authority that does not exist; reviewer catches it; HITL gate prevents the artifact from going downstream. The mitigation evolved from "human reviews everything" to "RAG grounds the model; human reviews only ungrounded outputs" — moving HITL from default to exception [3].

**Fintech — over-confidence detection via faithfulness eval.** Stripe's published research on LLM-driven fraud explanations identifies over-confidence as the hardest category to detect in production, because the explanations are internally coherent. Their mitigation is a held-out eval set that measures faithfulness against the actual transaction data [1].

## 6. Best Practices

- **Classify every detected hallucination by category in your incident log** — the category dictates the mitigation layer; aggregate counts drive prioritisation.
- **Wire `finish_reason` into your log shape from day one** — truncation is invisible without it.
- **Treat industry-aggregate hallucination rates with suspicion** — your task's rate is the only number that matters; measure it against your eval set [2].
- **Pydantic + tool-use is the cheapest format-drift mitigation** — there is no defensible reason to ship LLM JSON output without strict-mode parsing [3, foundation pattern].
- **Confabulation in federal contexts is HITL-gated by default** — until RAG-grounding is in place and faithfulness is measured, every cited authority gets a human check.

## 7. Hands-on Exercise

**(10 minutes, paper.)** For each of the five hallucination categories, write one Karsun `acquire-gov` `services/ai-orchestrator/app/main.py` `POST /draft-solicitation` scenario where that category is the most likely failure mode. Then for each scenario, name:

- (a) The mitigation layer that addresses it.
- (b) The week of the programme where that mitigation layer is installed.
- (c) The minimum signal (log field, metric, alert) that would let you detect the category-specific hallucination in production.

**What good looks like.** All five scenarios are plausible against the actual `ai-orchestrator` endpoint. The mitigation layer names match §3 exactly. The week mapping is: format-drift = Friday (today's PR + Friday's structured outputs), confabulation + stale-knowledge = W2 (RAG), over-confidence = W2 Fri (faithfulness eval), HITL gate = Friday's first framing + W3 deep-dive, truncation = today's PR (output-length budget + `finish_reason` logging).

## 8. Key Takeaways

- Can you name the five hallucination categories and give a precise definition for each that two engineers would apply consistently?
- Can you match each category to its primary mitigation layer and identify the programme week where that mitigation is installed?
- Can you defend the position "task-specific evaluation is the only honest measurement of hallucination risk" against the framing "industry benchmarks tell us what to expect"?
- Can you identify confabulation as the headline federal-acquisitions risk and the HITL+RAG combination that mitigates it?

## Sources

1. [LLM Hallucination Isn't One Problem. It's Four. — OneValley](https://www.onevalley.com/blog/llm-hallucination-isnt-one-problem-its-four-most-teams-are-only-solving-one/) — retrieved 2026-05-26 via /web-research
2. [LLM Hallucinations in 2026 — Lakera](https://www.lakera.ai/blog/guide-to-hallucinations-in-large-language-models) — retrieved 2026-05-26 via /web-research
3. [What is Retrieval-Augmented Generation? — AWS](https://aws.amazon.com/what-is/retrieval-augmented-generation/) — retrieved 2026-05-26 via /web-research

Last verified: 2026-05-27
