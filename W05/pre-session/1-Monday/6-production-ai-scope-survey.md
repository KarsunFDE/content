---
week: W05
day: Mon
topic_slug: production-ai-scope-survey
topic_title: "Production AI Scope Survey — the bridge from discovery to instrumentation"
parent_overview: W05/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://www.braintrust.dev/articles/best-ai-observability-tools-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://coralogix.com/ai-blog/agentic-ai-observability/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://azure.github.io/AI-in-Production-Guide/chapters/chapter_12_keeping_log_observability
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.elastic.co/blog/2026-observability-trends-generative-ai-opentelemetry
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.pwc.com/us/en/tech-effect/ai-analytics/ai-observability.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Production AI Scope Survey — the bridge from discovery to instrumentation

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define a "Production AI Scope Survey" — a per-surface inventory of where AI is in production, what its observability needs are, and what governance applies.
- Distinguish three layers that any AI scope survey must enumerate: the agent/LLM layer, the retrieval/data layer, and the downstream integration layer.
- Justify why scope discipline is the load-bearing virtue of the survey — what gets *excluded* matters as much as what's included.
- Apply the survey pattern to map a discovery-phase finding to a concrete instrumentation commitment for the next sprint.

## 2. Introduction

A Production AI Scope Survey is a per-surface inventory that names every place AI is running (or will run) in production for a given system, what observability the surface gets, and what governance applies to it. It's the artifact that bridges *discovery* (we found these AI surfaces) and *instrumentation* (we will monitor these surfaces, in this way, this iteration).

The discipline matters because AI surfaces don't stand still. A SaaS that launched with one LLM-powered recommender ends a year later with: that recommender, a chat support agent, a few embeddings-backed search endpoints, a content-moderation classifier, a code-suggestion sidecar in the admin console, and three internal tools the platform team built last quarter. Without a scope survey, the observability team treats this estate as "the AI stuff" — which means the actual surfaces are tracked by tribal knowledge, governed by whoever shipped them, and monitored unevenly.

By 2026 the AI observability category has matured enough that tooling is no longer the bottleneck — platforms specializing in everything from prompt-level tracing to multi-agent workflow visibility are widely available ([Braintrust AI observability buyer's guide, 2026-05-26](https://www.braintrust.dev/articles/best-ai-observability-tools-2026)). The bottleneck is now the *survey* — the act of naming, per surface, what's there and what gets monitored.

This reading walks the survey generically. The day's overview pins it to the cohort's specific surfaces and feature-inventory items; here we look at the shape across industries.

## 3. Core Concepts

### 3.1 The three layers a survey must enumerate

A scope survey enumerates each AI surface across three layers ([Coralogix agentic AI observability guide, 2026-05-26](https://coralogix.com/ai-blog/agentic-ai-observability/)):

| Layer | What it covers | Survey question |
|-------|----------------|-----------------|
| Agent / LLM | Model calls, prompts, completions, tool invocations | Which models, which providers, what token budget? |
| Retrieval / data | Vector store calls, document fetch, external lookup | What knowledge is grounded? What's its freshness? |
| Downstream integration | Outbound API calls, message publishing, system actions | What does the AI surface *do* in the world? |

A survey that names only the agent/LLM layer is incomplete; the retrieval layer is where grounding bugs live, and the integration layer is where authority leaks (the AI calls a downstream that has more privileges than the AI was supposed to wield).

### 3.2 Scope discipline — what's out matters

The survey lists what's *out of scope this iteration*, not just what's in. Out-of-scope items are not "ignored" — they're acknowledged and deferred with a reason and a re-evaluation trigger. Self-service observability discipline at scale rests on this — telemetry sprawls when nothing is ever explicitly out ([Platform Engineering self-service observability, 2026-05-26](https://platformengineering.org/blog/self-service-observability)).

Common out-of-scope categories:

- Surfaces in staging or pre-prod (separate spec, separate cadence).
- Internal tools used only by the platform team (lower bar, log-only).
- Experimental surfaces behind a feature flag (in scope when the flag flips for any user).
- Surfaces owned by a partner team (in their scope survey; this survey references theirs).

### 3.3 The "lights up under what" column

For each in-scope surface, the survey names what observability lights it up — the specific OTel auto-instrumentation, the framework SDK, the custom span emission. This isn't a vendor selection (that's the platform-choice ADR's job); it's the wiring-level commitment ([Azure AI in Production Guide, observability, 2026-05-26](https://azure.github.io/AI-in-Production-Guide/chapters/chapter_12_keeping_log_observability)).

Example rows:

- Frontend SPA → RUM SDK + `traceparent` propagation.
- Java order-service → OTel Java auto-agent.
- Python AI orchestrator → `opentelemetry-instrumentation-fastapi` + `gen_ai.*` semconv spans.
- LLM provider → InvokeModel calls instrumented with OTel GenAI semantic conventions.

The discipline is that each row commits a *concrete* mechanism. "Use OTel" is not a row; "use OTel Java auto-agent v1.X, configured with the gen_ai-context propagator" is.

### 3.4 Required vs candidate

The survey separates *required* commitments (must land this iteration) from *candidate* commitments (might land this iteration, decision pending). The distinction maps onto the team's load-bearing work for the sprint:

- Required → directly tied to a discovery finding, a customer commitment, or a compliance deadline.
- Candidate → tied to an ADR still in proposed state; flips to required when the ADR is accepted.

Candidate rows that don't get promoted within the iteration are deferred to the next survey with a noted reason. The pattern keeps the scope honest — items don't squat as ambiguous forever.

### 3.5 Cost-and-tenant surfaces

AI-specific observability adds a dimension that traditional APM didn't carry: per-tenant cost attribution. The survey names which surfaces emit `tenant_id` tags on cost-relevant spans, and which surfaces don't yet. Non-multi-tenant deployments can skip this; multi-tenant deployments cannot, because without it the SaaS economics become a forensic exercise rather than a dashboard query ([Elastic 2026 observability trends, 2026-05-26](https://www.elastic.co/blog/2026-observability-trends-generative-ai-opentelemetry)).

## 4. Generic Implementation

A generic Production AI Scope Survey for a hypothetical multi-tenant SaaS adopting LLM features. Structure carries across domains.

```markdown
# Production AI Scope Survey — Q3 iteration

## In-scope surfaces this iteration

| Surface | Layer | Lights up under | Required / Candidate | Rationale |
|---------|-------|-----------------|----------------------|-----------|
| /recommend (catalog) | Agent + Retrieval | OTel Python + gen_ai.* + vector-store exporter | Required | Q2 discovery: token-cost variance per tenant > 4x |
| /search (semantic) | Retrieval | Embeddings client wrapped with custom spans | Required | Customer commitment: search-latency SLO |
| /chat-support | Agent + Tool | OTel Python + tool-call span emission | Candidate | Pending ADR-014 (auto-resolve authority) |
| /moderate (classifier) | Agent | Span emission at boundary; outputs sampled at 10% | Required | Q2 incident: drift on jailbreak class |
| /code-suggest (admin SPA) | Agent | Frontend RUM + agent-side gen_ai.* | Candidate | Pending ADR-015 (admin-tier governance) |

## Required cost-and-tenant surfaces
- All Required rows above MUST carry tenant_id on their cost-relevant spans
  this iteration. Backend dashboards enforce tenant partitioning via RBAC.

## Out of scope this iteration (acknowledged + deferred)
- Internal tools the platform team built for ops use (log-only; revisit Q4
  once external customer adoption begins).
- Experimental "smart-tagging" feature behind feature flag (in scope when
  the flag opens to any external tenant).
- Partner team's AI-driven QA bot (covered by their scope survey; we
  reference theirs from our integration map, do not duplicate here).

## Re-evaluation triggers
- New AI surface lights up in production → survey is re-opened, not amended.
- Candidate row promoted to Required by ADR → survey is re-opened.
- Any out-of-scope item flips status → re-open.
```

The discipline is that the survey is *versioned* — each iteration produces a new survey, the old one stays as history. A team can read the survey trail back and see which surfaces lit up when, which moved required vs candidate, which were deferred.

## 5. Real-world Patterns

**Healthcare — agent + retrieval inventory before audit**. A health-tech company preparing for HIPAA-aligned external review wrote a Production AI Scope Survey expressly to give auditors a single-page answer to "where is AI in your production system?" The survey separated agent/LLM surfaces (where the model emits content) from retrieval surfaces (where regulated data is grounded) — auditors cared more about the retrieval layer because that's where PHI flowed. The survey's most useful property turned out to be the *out-of-scope* section: it named the experimental internal-only surfaces the auditors did *not* need to inspect, focusing the review where it mattered ([PwC AI observability, 2026-05-26](https://www.pwc.com/us/en/tech-effect/ai-analytics/ai-observability.html)).

**Fintech — required vs candidate as sprint discipline**. A fintech platform with an aggressive LLM-feature roadmap used the required/candidate split as its sprint-planning anchor. Each scope survey iteration's "required" rows were the sprint's committed observability work; the "candidate" rows were the sprint's stretch goals. Required slipped → scope survey re-opened. Candidate stayed un-promoted → noted with reason and deferred. The discipline kept the observability backlog visible alongside the feature backlog instead of being treated as overhead.

**E-commerce — cost-and-tenant surfaces as economics gate**. A multi-brand e-commerce SaaS treated per-tenant cost attribution as a gate: no AI surface went live without the tenant_id tag on cost-relevant spans. The survey enforced this — the column was required, not optional. The result was that per-tenant unit economics for AI features were dashboard queries from day one rather than forensic exercises after the bill arrived. Without the gate, the platform's CFO would have been blind to which tenants were profitable on AI features ([Elastic 2026 observability trends, 2026-05-26](https://www.elastic.co/blog/2026-observability-trends-generative-ai-opentelemetry)).

**Gaming — survey as integration-with-partner-team contract**. A games-publishing platform with AI moderation features had two teams owning overlapping surfaces (one owned moderation, one owned chat). The scope survey from each team referenced the other's — moderation's survey marked the chat-side spans as "covered by chat team's survey, do not duplicate." The cross-reference pattern kept both surveys honest and prevented telemetry duplication that would have doubled the bill.

## 6. Best Practices

- **Inventory all three layers, not just the model layer.** Agent/LLM, retrieval/data, downstream integration — a survey that names only the model layer misses where grounding bugs and authority leaks live.
- **Make out-of-scope a first-class section.** Acknowledged + deferred + re-evaluation trigger beats unspoken omission; what's not in scope must be a positive statement, not silence.
- **Separate required from candidate.** Required is the iteration's committed work; candidate is pending an ADR. The split keeps the observability backlog readable alongside the feature backlog.
- **Commit concrete instrumentation mechanisms per row.** "Use OTel" is not a row; "OTel Java auto-agent v1.X with gen_ai-context propagator" is.
- **Treat per-tenant cost attribution as a gate, not a nice-to-have**, in any multi-tenant deployment. Cost-relevant spans carry `tenant_id` before the surface goes live.
- **Version the survey by iteration, don't amend in place.** Each iteration produces a new survey; the history of which surfaces lit up when is the value.
- **Cross-reference partner surveys**; do not duplicate. Overlapping team boundaries are resolved by naming whose survey owns which span, not by both teams instrumenting the same surface twice.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** Pick a system you've worked on (real or invented). Draft a single-iteration Production AI Scope Survey using the structure in §4:

1. List 3–5 in-scope surfaces, classified by layer (agent / retrieval / integration).
2. For each, name the instrumentation that lights it up (concrete mechanism).
3. Mark each as required or candidate, with a one-sentence rationale.
4. List 2–3 explicitly out-of-scope items with deferral reasons.
5. State the re-evaluation triggers that would re-open the survey before its scheduled cadence.

**What good looks like.** A finished page reads like a contract for the iteration — required rows are testable ("did we land OTel on the order-service this sprint?"), out-of-scope rows are positive statements with reasons, and the re-evaluation triggers are events not feelings ("when X surface lights up", not "if things change"). If any in-scope row is missing the instrumentation mechanism, the row isn't ready — flag for clarification.

## 8. Key Takeaways

- *What does a Production AI Scope Survey commit?* A per-iteration, per-surface inventory of where AI runs in production, how it's instrumented, what governance applies, and what's out of scope.
- *Which three layers must every survey enumerate?* Agent/LLM, retrieval/data, downstream integration. Surveys that name only the model layer miss grounding bugs and authority leaks.
- *Why is the out-of-scope section load-bearing?* Without explicit deferral, telemetry sprawls; what's not in scope is acknowledged + dated + given a re-evaluation trigger.
- *How does the required-vs-candidate split help?* It keeps the observability backlog visible alongside the feature backlog: required is committed work, candidate is pending an ADR, deferral is explicit.

## Sources

1. [AI observability tools — buyer's guide 2026 (Braintrust)](https://www.braintrust.dev/articles/best-ai-observability-tools-2026) — retrieved 2026-05-26
2. [Agentic AI Observability — practical guide (Coralogix)](https://coralogix.com/ai-blog/agentic-ai-observability/) — retrieved 2026-05-26
3. [Chapter 12 — Observability (Azure AI in Production Guide)](https://azure.github.io/AI-in-Production-Guide/chapters/chapter_12_keeping_log_observability) — retrieved 2026-05-26
4. [Observability trends for 2026 — GenAI + OpenTelemetry (Elastic)](https://www.elastic.co/blog/2026-observability-trends-generative-ai-opentelemetry) — retrieved 2026-05-26
5. [AI observability for enterprise AI agents (PwC)](https://www.pwc.com/us/en/tech-effect/ai-analytics/ai-observability.html) — retrieved 2026-05-26

Last verified: 2026-05-26
