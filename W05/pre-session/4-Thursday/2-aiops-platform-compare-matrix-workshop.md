---
week: W05
day: Thu
topic_slug: aiops-platform-compare-matrix-workshop
topic_title: "AIOps Platform Compare-Matrix Workshop — Thu morning shape"
parent_overview: W05/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://www.gartner.com/reviews/market/aiops-platforms
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.dynatrace.com/platform/artificial-intelligence/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://docs.datadoghq.com/watchdog/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://newrelic.com/platform/ai-monitoring
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://coralogix.com/ai-observability/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# AIOps Platform Compare-Matrix Workshop — How to Run a Cross-Vendor Evaluation

## 1. Learning Objectives

By the end of this reading, the learner can:

- Articulate why a compare-matrix beats a single-vendor pilot when selecting an AIOps platform.
- Name the dimensions that distinguish AI-native anomaly detection products (causation graph, anomaly-detection model class, LLM-observability native, cost shape, deployment model).
- Identify which dimensions are vendor-marketing fluff and which are operationally load-bearing.
- Facilitate a structured comparison workshop where each participant defends a vendor they did not choose.
- Output a written matrix where every row carries a citation and a confidence rating.

## 2. Introduction

AIOps — the Gartner-defined category that fuses observability data with machine-learning and topology reasoning — has become a vendor scrum. Datadog, Dynatrace, New Relic, Splunk (now Cisco), Coralogix, PagerDuty, BigPanda, ScienceLogic, and BMC Helix all publish overlapping product pages. Each claims AI-native anomaly detection. Each claims causation reasoning. Each claims LLM observability. The marketing surface is high; the operational difference is real but subtle.

A compare-matrix workshop is the antidote. Instead of running a single-vendor proof-of-concept and finding out at month three that the chosen platform doesn't model your topology well, a team spends one half-day building a decision matrix — one row per dimension, one column per candidate, every cell a one-paragraph justification with a citation. The matrix forces a debate that the marketing surface tries to suppress: where does each platform actually *lose*? On what shape of system does its core algorithm break down?

The workshop format is borrowed from federal-acquisitions trade-study practice (and from architecture-decision records in general): you pick the dimensions first, before any vendor demonstration, so the dimensions are not gerrymandered to favour an incumbent. Then you populate the matrix with the help of `/web-research`-style primary-source review — vendor docs, third-party reviews, and Gartner-style market reports — not with a sales-engineer deck.

The output is two artifacts. First, the matrix itself, which becomes the architecture-decision record for the platform choice. Second, the implicit decision rule: which dimensions can the team *not* compromise on, and which are nice-to-haves. The second artifact is more valuable than the first because it survives vendor-product churn — the dimensions stay; the leaders rotate.

## 3. Core Concepts

### 3.1 What an AIOps platform actually does

Gartner defines the category as **event-intelligence solutions**: tools that apply AI and analytics to signals from digital services, then accelerate or automate the response. The key behaviours are: cross-domain event ingestion (logs + metrics + traces + topology), event correlation and enrichment, pattern recognition (anomaly detection), and accelerated remediation (automated runbooks, ticket suppression, intelligent paging) [Gartner — Event Intelligence Solutions, retrieved 2026-05-26].

Note the framing shift: Gartner has been moving the market name away from "AIOps platforms" toward "event-intelligence solutions" because the old name implied a single product class when the actual products differ on which of those behaviours they emphasise. Datadog leans into ingestion-plus-anomaly. Dynatrace leans into topology-plus-causation. PagerDuty leans into event-grouping-plus-routing. The compare-matrix dimensions need to surface those emphases, not paper them over.

### 3.2 Core algorithmic patterns

There are roughly three families of AI-native methodology in this market, and a vendor usually combines two:

1. **Deterministic causal graphs.** A topology is assembled from instrumentation (services, endpoints, infrastructure dependencies), and a causal reasoner walks the graph to find the upstream change that explains a downstream symptom. Dynatrace's Davis AI is the canonical example: "deterministic AI delivers answers, not guesses" framed around their Smartscape real-time dependency graph [Dynatrace Intelligence, retrieved 2026-05-26].

2. **Statistical anomaly detection.** A time-series model — sometimes a forecasting model (Prophet-like), sometimes a density estimator, sometimes a clustering pass — flags metrics and traces that deviate from the learnt baseline. Datadog's Watchdog is the canonical example: it monitors APM and infrastructure for "outlier behaviour" without explicit thresholds [Datadog Watchdog, retrieved 2026-05-26].

3. **Generative or agentic reasoning.** A large language model summarises an incident, drafts a root-cause hypothesis, or proposes a runbook step. New Relic's AI Monitoring extends this to LLM-app stacks, monitoring the model layer alongside the infrastructure layer (including Model Context Protocol calls) [New Relic AI Monitoring, retrieved 2026-05-26]. Coralogix's AI Center positions for AI-app observability — catching hallucinations and behavioural drift in agent fleets [Coralogix AI Observability, retrieved 2026-05-26].

A well-run compare-matrix surfaces which combination a vendor leans on. A "Davis AI" answer is not the same shape as a "Watchdog AI" answer; calling them both "AIOps" hides the difference.

### 3.3 The dimensions worth comparing

Useful dimensions tend to be operational, not marketing-coded. A short, defensible set:

- **AI-native anomaly detection** — does the platform learn the baseline automatically, or does it need static thresholds? What's the false-positive rate during a deploy?
- **LLM observability first-class** — is there a `gen_ai.*` semantic-convention surface, or are LLM calls bolted onto generic spans? (OpenTelemetry's `gen_ai.*` conventions are the current standard.)
- **Token / cost-per-request visibility** — for LLM apps, can you slice cost by tenant, by route, by model version?
- **Causation graph** — is there an explicit topology, or are correlations purely temporal?
- **Authorization boundary** (regulated workloads only) — FedRAMP, HIPAA, PCI: which controls does the vendor hold, and at what level?
- **Pricing model** — per-host, per-ingest-GB, per-query, or hybrid. Pricing shape often dictates which signals get cut when budget tightens.
- **Hands-on familiarity** — what the team already runs. A "best" tool nobody knows costs more to adopt than a "good" tool one engineer already configures in their sleep.
- **OTel-friendliness** — does the platform consume OpenTelemetry data natively, or does it require its proprietary agent? OTel-friendliness is portability.
- **Deployment model** — SaaS-only, self-managed, hybrid; for regulated workloads the in-boundary deployment story is decisive.

The dimensions you leave OUT are diagnostic too. Don't compare on "number of integrations" (everyone claims 600+). Don't compare on "ease of dashboards" (subjective and demo-rigged). Don't compare on "AI quality" without a concrete signal class (anomaly precision on a defined workload).

### 3.4 The forced-defence rule

The workshop's load-bearing rule: each participant is assigned a vendor they did **not** pre-pick. They argue for that vendor's strengths — including the dimensions where it would beat their own first choice. This is the same epistemic move as Red-Team / Blue-Team or formal devil's-advocate review: it surfaces the dimensions where the favoured vendor is weaker, which the team would otherwise rationalise away.

When done well, the workshop ends with at least one dimension where the team's pre-workshop favourite did not win. That row gets a written justification for why the team accepts the trade-off. The justification, not the chosen vendor, is the durable output.

## 4. Generic Implementation

A compare-matrix is a markdown table backed by a per-cell justification document. The skeleton — borrowed from architecture-decision-record templates — looks like:

```markdown
# AIOps Platform Selection Matrix — <project> — <date>

## Candidates
- Vendor A — version evaluated, eval date
- Vendor B — version evaluated, eval date
- Vendor C — version evaluated, eval date

## Decision dimensions (locked before vendor review)
1. Anomaly precision on deploy days
2. Topology / causation graph quality
3. LLM observability native (gen_ai.* spans)
4. Pricing model shape
5. Authorization boundary (regulatory)
6. OTel-friendliness
7. Existing team familiarity

## Matrix
| Dimension | Vendor A | Vendor B | Vendor C |
|-----------|----------|----------|----------|
| 1. Anomaly precision on deploys | High — see §1.A | Med — see §1.B | High — see §1.C |
| 2. Topology / causation graph | Implicit — see §2.A | Explicit graph — §2.B | Implicit — §2.C |
| 3. LLM observability | Beta gen_ai.* — §3.A | Generic spans — §3.B | First-class — §3.C |
| ... | ... | ... | ... |

## Per-cell justifications
### §1.A Vendor A anomaly precision on deploys
<one paragraph; one or more citations with retrieval date>
### §1.B ...
```

The per-cell justification is non-optional. A matrix cell that reads "High" without a paragraph and a citation is a marketing claim, not an evaluation.

Two procedural notes:

- The candidate set is **closed** before evaluation. If you keep adding vendors, the dimensions get re-shaped to match the new entrant, which defeats the workshop. Limit to three or four serious candidates.
- The dimensions are **frozen** before vendor demos. If a vendor argues for adding a dimension during a demo, capture it as a follow-up question — do not edit the matrix mid-flight.

## 5. Real-world Patterns

**Healthcare — choosing a SIEM-plus-AIOps combination for a hospital network.** A regional health system evaluated three vendors against eight dimensions, including HIPAA boundary, FHIR-native log parsing, and integration with their existing PagerDuty incident-management surface. The chosen vendor was not the highest-rated technically; it was the one whose AI-anomaly model had the lowest false-positive rate on deploy days, because the hospital's deploy cadence was already triggering 40-plus alerts per release elsewhere. The matrix surfaced the operational cost of alert noise, which dominated the algorithmic-quality comparison [Gartner — Event Intelligence Solutions reviews, retrieved 2026-05-26].

**Gaming — anomaly detection for a live-service title.** A studio running a free-to-play title compared Datadog Watchdog, Dynatrace Davis, and an in-house anomaly detector. They evaluated on detection lead-time during a "downloadable content" launch event, where load patterns deviated sharply from baseline. The compare-matrix surfaced that Watchdog's statistical model needed eight hours to learn the new baseline, while Davis's graph-based reasoner caught the upstream CDN issue in minutes. The studio chose Dynatrace for production and kept the in-house anomaly detector for narrow-domain signals not covered by either commercial platform [Dynatrace Intelligence, retrieved 2026-05-26].

**E-commerce — LLM observability for a generative-search feature.** A retail platform rolled out an LLM-powered search assistant and needed to track token cost, hallucination rate, and citation-mismatch rate alongside the rest of their observability stack. They evaluated New Relic AI Monitoring, Coralogix AI Center, and a custom OpenTelemetry-only setup against five LLM-specific dimensions. The matrix made the cost-per-tenant slicing requirement non-negotiable; only one vendor surfaced the breakdown out-of-the-box, which made the choice trivial despite the vendor lagging on three other dimensions [New Relic AI Monitoring, retrieved 2026-05-26; Coralogix AI Observability, retrieved 2026-05-26].

**Logistics — multi-cloud anomaly correlation.** A parcel-delivery network used a compare-matrix to evaluate AIOps platforms that could correlate signals across AWS, Azure, and on-prem warehouse infrastructure. The decisive dimension was OpenTelemetry-friendliness: the team had standardised on OTel and refused to install a proprietary collector per cloud. Two of the three candidates failed that dimension, ending the evaluation early and saving four weeks of pilot work [Dynatrace Intelligence, retrieved 2026-05-26].

## 6. Best Practices

- Lock the dimensions before any vendor demo; do not edit them mid-flight even when a vendor's pitch makes a missing dimension look appealing.
- Assign each participant a vendor they did not pre-pick; force-defence beats unforced consensus.
- Cite every cell. A cell without a citation is a marketing claim.
- Keep the candidate set small (three or four). If your shortlist is longer, run a pre-filter dimension first to cut to the serious candidates.
- Capture the per-cell *confidence*, not just the rating. "High, confidence Low" tells future-you that the cell needs re-verification under load.
- Re-run the matrix annually. AIOps platforms iterate aggressively; a 2026 matrix is stale by 2027.
- Record the rejected candidates and the dimensions they lost on; future evaluations save time by skipping vendors that lost on still-relevant dimensions.

## 7. Hands-on Exercise

Pick three observability platforms you have access to documentation for (any three: Datadog, Dynatrace, New Relic, Coralogix, Splunk, Grafana Cloud, Honeycomb, etc.). In 15 minutes:

1. Write down five operational dimensions you would compare them on for a workload of your choice (LLM app, microservice fleet, batch pipeline, etc.). Lock the list.
2. For dimension 1, write a single-paragraph evaluation per vendor with one citation each.
3. Identify the dimension where your favourite vendor *loses*. Write the trade-off justification.

**What good looks like.** The matrix has dimensions that are operationally specific to your workload (not generic marketing dimensions like "ease of use"). Each cell paragraph cites a primary source — a vendor doc page, a third-party review, an OTel-conformance test, a regulatory boundary listing — with retrieval date. The trade-off justification names a real operational consequence ("we accept slower LLM observability because the topology graph cuts our MTTR by 30%") and not a wishful claim.

## 8. Key Takeaways

- Why does a compare-matrix workshop beat a single-vendor pilot for AIOps selection?
- What are the three families of AI methodology — causal graph, statistical anomaly, generative — and how does each one surface a different shape of incident?
- What dimensions are operationally load-bearing versus marketing-coded?
- Why is the forced-defence rule (defend a vendor you didn't pick) the load-bearing technique of the workshop?
- Which artifact is more durable — the matrix itself, or the per-cell trade-off justifications?

## Sources

1. [Gartner — Event Intelligence Solutions Reviews and Ratings](https://www.gartner.com/reviews/market/aiops-platforms) — retrieved 2026-05-26
2. [Dynatrace Intelligence (Davis AI) overview](https://www.dynatrace.com/platform/artificial-intelligence/) — retrieved 2026-05-26
3. [Datadog Watchdog documentation](https://docs.datadoghq.com/watchdog/) — retrieved 2026-05-26
4. [New Relic AI Monitoring](https://newrelic.com/platform/ai-monitoring) — retrieved 2026-05-26
5. [Coralogix AI Observability](https://coralogix.com/ai-observability/) — retrieved 2026-05-26

Last verified: 2026-05-26
