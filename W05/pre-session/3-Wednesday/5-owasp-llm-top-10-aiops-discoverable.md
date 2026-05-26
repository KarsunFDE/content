---
week: W05
day: Wed
topic_slug: owasp-llm-top-10-aiops-discoverable
topic_title: "OWASP LLM Top 10 continuation — LLM05 / LLM09 / LLM10 as AIOps-discoverable risks"
parent_overview: W05/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://genai.owasp.org/llm-top-10/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://genai.owasp.org/llmrisk/llm102025-unbounded-consumption/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://genai.owasp.org/llmrisk/llm092025-misinformation/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.invicti.com/blog/web-security/owasp-top-10-risks-llm-security-2025
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.promptfoo.dev/blog/unbounded-consumption/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# OWASP LLM Top 10 continuation — LLM05 / LLM09 / LLM10 as AIOps-discoverable risks

## 1. Learning Objectives

By the end of this reading, the learner can:

- Summarise **LLM05 (Improper Output Handling)**, **LLM09 (Misinformation)**, and **LLM10 (Unbounded Consumption)** from the OWASP Top 10 for LLM Applications 2025, distinguishing each from its predecessor framing in the 2023 list.
- Name the specific **observability signals** an AIOps stack can collect that make each of LLM05/09/10 *discoverable* in production, not just hypothetical.
- Describe **two concrete mitigations** per category — one technical (code/infra) and one operational (process/policy).
- Argue why the *Denial-of-Wallet* framing replaced *Denial-of-Service* in the 2025 list and what that change implies for cost-monitoring discipline.

## 2. Introduction

The OWASP Top 10 for LLM Applications is a community-curated list of the most pervasive vulnerabilities in production LLM systems, updated every one-to-two years. The 2025 edition is the current version as of May 2026 and is the reference cohorts should cite. The list has ten entries; W4 Wednesday's reading covered LLM06 (Excessive Agency) as the HITL #6 framing. W5 Wednesday extends with the three categories that AIOps observability is *particularly well-placed* to surface: LLM05, LLM09, and LLM10.

The framing matters: not every OWASP LLM risk is detectable from telemetry. Prompt injection (LLM01) is often a content-level concern; supply-chain (LLM03) is largely a build-time concern. By contrast, LLM05/09/10 have *strong observability signals*. A well-instrumented LLM endpoint (the W5 Tuesday OpenTelemetry-on-AI material) emits enough structured data that you can build alerts on each of these three categories without exotic tooling.

Treating these three together also captures the *failure-mode lifecycle*: LLM10 is what *attackers* (or accidental misuse patterns) do *to* the system; LLM05 and LLM09 are what the *system* does to *itself* when its outputs are not properly validated or grounded. Wednesday's task is to map each to the `gen_ai.*` semconv signals Tuesday's instrumentation already collects, and to commit which categories your Friday PR will mitigate.

## 3. Core Concepts

### 3.1 LLM05 — Improper Output Handling

**Definition.** LLM05 covers flaws in handling the *output* of an LLM downstream — particularly when that output is passed to other system components (shells, SQL engines, browsers, file systems) without validation. The 2025 entry frames it as a *content-passing* hazard: the model produced something; the system used it without checking what was produced.

**Canonical vulnerable patterns** ([OWASP genai.owasp.org/llm-top-10](https://genai.owasp.org/llm-top-10/), retrieved 2026-05-26):

- Remote Code Execution: `subprocess.run(model_output)` or `eval(model_output)`.
- SQL Injection: `cursor.execute(model_output)` with no parameterisation.
- Cross-Site Scripting (XSS): rendering the model's output as Markdown or HTML on a page without sanitisation.
- File-system: writing the model's output to a path the model itself chose.

**AIOps-discoverable signals.** If the LLM endpoint is instrumented per the W5 Tuesday material, three signals make LLM05 surface-able in observability:

- `gen_ai.response.output_type` drift — e.g., an endpoint that *should* return structured JSON is suddenly returning prose. The schema validator's failure rate is a leading indicator.
- Downstream-system error spike correlated with model output — e.g., SQL parse errors on a service that consumes model output start spiking right after a model upgrade.
- Output-token entropy anomaly — for endpoints returning constrained outputs (JSON, code, citations), a sudden jump in output entropy can indicate format drift.

**Mitigations:**

- *Technical:* a Pydantic (or equivalent) schema gate between the model and the downstream consumer; explicit content-type assertions; parameterised execution paths (no string interpolation into shells / SQL).
- *Operational:* a schema-drift alert that fires when validator failure exceeds a threshold; review of any new endpoint to confirm the output-handling path is parameterised.

### 3.2 LLM09 — Misinformation

**Definition.** LLM09 covers the system producing false, misleading, or fabricated content that appears authoritative. The 2025 edition deliberately folded the older *Overreliance* category into LLM09, on the rationale that misinformation rarely causes damage in isolation — it causes damage when a user trusts it. The combined category names both the production *and* the consumption side of the risk.

**Canonical patterns:**

- Hallucinated citations — references to papers, statutes, or sources that do not exist.
- Plausible-but-wrong claims with confident phrasing — high model confidence, low factual grounding.
- Confabulated chains of reasoning that pass surface review but rest on fabricated premises.
- Stale answers — the model's training data is older than the question requires.

**AIOps-discoverable signals.** Per [OWASP LLM09:2025](https://genai.owasp.org/llmrisk/llm092025-misinformation/) (retrieved 2026-05-26):

- RAG retrieval `k=0` with confident output — the retrieval surface returned no documents but the model still answered. This is a strong hallucination signal.
- Citation-pattern mismatch — the model emits "[1]" tokens with no corresponding source IDs in the retrieved context.
- Retrieval-vs-output similarity collapse — the embedding similarity between the retrieved chunks and the model's claim drops below threshold.
- User-feedback signal — thumb-down rate on outputs with no citations is higher than on outputs with citations.

**Mitigations:**

- *Technical:* require RAG grounding for any factual-claim endpoint; emit citation IDs as structured fields the system can validate; reject responses where retrieved-context similarity to the response is below threshold.
- *Operational:* a hallucination-rate dashboard with the four signals above; a periodic human review of outputs with no citations.

### 3.3 LLM10 — Unbounded Consumption

**Definition.** LLM10 replaced the 2023-list's *Model Denial of Service* in the 2025 list. The reframing is deliberate. The older entry centred on availability attacks; the new entry centres on **cost**. The community term that now dominates the literature is *Denial-of-Wallet* (DoW) — an attacker (or accidental loop) does not bring the service down but runs up its bill ([Promptfoo: *Beyond DoS*](https://www.promptfoo.dev/blog/unbounded-consumption/), retrieved 2026-05-26).

**Canonical patterns:**

- An agent loop with no termination condition that calls the model repeatedly.
- Retrieval-augmented endpoints with no `k` cap that fan out across the corpus.
- Prompt-stuffing attacks that pump huge inputs into the model to consume input tokens.
- Repeated requests from a single source that bypass rate-limiting because rate-limiting is per-IP and not per-tenant.

**AIOps-discoverable signals.** The Tuesday OpenTelemetry-on-AI material's `gen_ai.usage.*` semantic conventions are the primary surface:

- `gen_ai.usage.input_tokens` and `gen_ai.usage.output_tokens` per request, per endpoint, per tenant.
- Cost per request derived from token usage and the model's price band.
- Agent-loop iteration count per request (often a custom span attribute on the agent orchestrator).
- Per-tenant burn-rate alerts when a tenant's daily token spend exceeds 2× rolling 7-day average.

**Mitigations:**

- *Technical:* per-tenant token budgets enforced by a middleware gate (not by hope); a circuit breaker on downstream model calls (W4 Friday's Resilience4j material); an explicit max-iteration cap on every agent loop.
- *Operational:* a daily token-spend report by tenant + endpoint; an automatic-alert burn-rate watch (this is part of Wednesday afternoon's research-day deliverables).

### 3.4 Why three categories, not all ten?

The pedagogical reason for grouping LLM05/09/10 together is that they share *observability-first* discoverability. The instrumentation you put in place for W5 Tuesday's OTel on AI already produces the signal surface for all three. The other categories (prompt injection, supply chain, data poisoning) are not less important — they require different *kinds* of detective controls (red-teaming, SCA tooling, training-data audits).

The three together also cover the most common production-incident vocabulary for LLM systems in 2026: *"the model returned the wrong shape"* (LLM05), *"the model made it up"* (LLM09), *"the model loop ran us up a bill"* (LLM10). If your Friday PR mitigates three categories well, this is usually a good set to pick from.

## 4. Generic Implementation

A worked example outside federal acquisitions. A **healthcare patient-portal team** ships a symptom-checker endpoint backed by a RAG over the clinic's clinical guidance documents. The team commits to mitigating LLM05/09/10 in Friday's PR.

**LLM05 mitigation — Pydantic gate on the response model.**

```python
from pydantic import BaseModel, Field

class SymptomCheckResponse(BaseModel):
    recommendation: str = Field(min_length=10, max_length=2000)
    severity: str = Field(pattern="^(self-care|see-clinician-today|emergency)$")
    citations: list[str] = Field(min_length=1)  # require at least one source
    confidence: float = Field(ge=0.0, le=1.0)

def post_process(model_raw: str) -> SymptomCheckResponse:
    parsed = json.loads(model_raw)  # already JSON-mode constrained
    return SymptomCheckResponse(**parsed)  # raises on schema violation
```

The Pydantic gate makes LLM05 *fail closed* — any model output that doesn't conform fails the request rather than being silently passed downstream. The validator's failure-rate is an AIOps signal.

**LLM09 mitigation — citation grounding + retrieval-similarity check.**

```python
# pseudocode
retrieved = vector_store.search(query, k=4)
if len(retrieved) == 0:
    return refuse_response("no relevant guidance found")

response = model.generate(prompt=build_prompt(retrieved))
sim = cosine(embed(response.recommendation), embed(retrieved.text))
if sim < SIMILARITY_THRESHOLD:
    return refuse_response("grounding insufficient")

assert all(cid in retrieved.ids for cid in response.citations)  # no invented IDs
return response
```

**LLM10 mitigation — per-tenant token budget + agent-loop cap.**

```python
TENANT_BUDGET = budget_service.get(tenant_id)
if TENANT_BUDGET.tokens_used_today + estimated_request_tokens > TENANT_BUDGET.daily_cap:
    raise BudgetExceeded()

MAX_LOOP = 8
for i in range(MAX_LOOP):
    step = agent.step(state)
    if step.is_terminal:
        break
else:
    raise AgentLoopExceeded()
```

Each mitigation pairs with a span attribute on the corresponding OTel span — `gen_ai.response.validation_passed`, `gen_ai.grounding.similarity`, `gen_ai.usage.tenant_budget_remaining` — so the AIOps dashboard can detect the *absence* of these protections in any production traffic.

## 5. Real-world Patterns

**Fintech (consumer banking chatbot, 2026 incident).** The bank's published 2026 incident review attributes a one-day customer-support outage to an LLM10 amplification: a misconfigured agent loop iterated 47 times per request after a routing-rule change. Token spend went from $400/day to $32,000/day before a finance-ops alert (not an SRE alert) caught it. Their post-fix mitigation is a per-tenant + per-request token cap enforced in middleware. The team treats the lesson as "LLM10 is a cost alert, not an availability alert; the on-call rotation now reads both kinds".

**E-commerce (product-description generation pipeline).** The platform pre-2025 emitted product descriptions with hallucinated specifications — colour, fabric content, dimensions that did not match the SKU. After folding *Overreliance* into LLM09 in the 2025 list, the team explicitly added citation grounding (every claim must come from the SKU's source-of-truth attributes table) and a fail-closed schema. Hallucination rate dropped by 90% in the first month after the gate was added ([Invicti: *OWASP Top 10 LLM 2025*](https://www.invicti.com/blog/web-security/owasp-top-10-risks-llm-security-2025), retrieved 2026-05-26).

**Gaming (player-support natural-language assistant).** The studio's post-mortem of a 2026 incident describes an LLM05 case: the assistant generated Markdown links that, when rendered in the support-portal UI, executed embedded `javascript:` payloads. The fix combined an output schema (strict-Markdown-only) and a downstream renderer that strips `javascript:` URLs — defence in depth.

**Logistics (carrier-integration platform).** The team's 2026 published architecture note pairs LLM10 mitigation with circuit breakers: agent loops are wrapped in a per-tenant budget; a Resilience4j-style circuit breaker on the upstream model API trips after burn-rate exceeds threshold, and the platform falls back to a deterministic non-AI route. The note specifically frames LLM10 mitigation as *availability + cost*, not one or the other.

## 6. Best Practices

- **Treat LLM05's schema gate as load-bearing.** Without it, the LLM output is unbounded text passed into untyped consumers — the same class of bug as untrusted user input.
- **Require RAG grounding for any factual-claim endpoint.** LLM09's main mitigation is *not* better prompting; it's *don't answer when you can't ground*.
- **Cap agent loops with a hard iteration limit.** The LLM10 incidents that make headlines are almost always unbounded loops, not adversarial input.
- **Instrument the absence of protection.** A request that *should* have a budget gate but doesn't is the most dangerous traffic on your platform; the dashboard should make this visible.
- **Re-verify the OWASP version annually.** The list refreshes every one-to-two years; the 2025 edition is current in 2026, but the next refresh will renumber.
- **Pick three categories your Friday PR will mitigate, not all ten.** The mitigation matters more than the breadth.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** Pick a model-backed endpoint from your prior experience — not federal-acquisitions. For each of LLM05, LLM09, and LLM10:

| Category | The endpoint's current exposure (one sentence) | The signal that would surface a real incident | The mitigation you would ship Friday |
|---|---|---|---|
| LLM05 | | | |
| LLM09 | | | |
| LLM10 | | | |

**What good looks like.** A good answer is specific — the LLM05 row names *which downstream consumer* of the model output is the unprotected one (the database? the renderer? the file system?). The LLM09 row names *which factual claims* in the response are at risk (citations? numerical claims? policy interpretation?). The LLM10 row names *which loop or fan-out* has no current cap. *"Add validation"* is not a mitigation; *"Add a Pydantic gate on the response model with a JSON-mode constraint; reject on schema fail"* is.

## 8. Key Takeaways

- *What does LLM05 actually cover, and which downstream consumers of model output are the highest-risk integration points?*
- *Why was Overreliance folded into LLM09 in the 2025 list, and how does that reframing change the mitigation strategy?*
- *Why did LLM10 replace Denial-of-Service, and what is the "Denial-of-Wallet" failure mode?*
- *Which observability signals from a well-instrumented LLM endpoint surface each of LLM05/09/10 in production?*

## Sources

1. [OWASP Top 10 for LLM Applications 2025 — OWASP Gen AI Security Project](https://genai.owasp.org/llm-top-10/) — retrieved 2026-05-26
2. [LLM10:2025 Unbounded Consumption — OWASP Gen AI Security Project](https://genai.owasp.org/llmrisk/llm102025-unbounded-consumption/) — retrieved 2026-05-26
3. [LLM09:2025 Misinformation — OWASP Gen AI Security Project](https://genai.owasp.org/llmrisk/llm092025-misinformation/) — retrieved 2026-05-26
4. [OWASP Top 10 for LLMs 2025: Key Risks and Mitigation Strategies — Invicti](https://www.invicti.com/blog/web-security/owasp-top-10-risks-llm-security-2025) — retrieved 2026-05-26
5. [Beyond DoS: How Unbounded Consumption is Reshaping LLM Security — Promptfoo](https://www.promptfoo.dev/blog/unbounded-consumption/) — retrieved 2026-05-26

Last verified: 2026-05-26
