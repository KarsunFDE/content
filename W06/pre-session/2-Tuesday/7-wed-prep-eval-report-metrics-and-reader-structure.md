---
week: W06
day: Tue
topic_slug: wed-prep-eval-report-metrics-and-reader-structure
topic_title: "Wed-prep — Eval report metrics and reader structure"
parent_overview: W06/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 20
sources:
  - url: https://www.getmaxim.ai/articles/complete-guide-to-rag-evaluation-metrics-methods-and-best-practices-for-2025/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://atlan.com/know/llm-evaluation-frameworks-compared/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.getmaxim.ai/articles/building-a-golden-dataset-for-ai-evaluation-a-step-by-step-guide/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://inference.net/content/llm-observability-monitoring-production-deployments/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://costhawk.ai/glossary/p95-p99-latency
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://ecp.engineering.utoronto.ca/resources/online-handbook/components-of-documents/abstracts-and-executive-summaries/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://courses.lumenlearning.com/sunyulster227technicalwriting/chapter/6-reports/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://en.wikipedia.org/wiki/Inverted_pyramid_(journalism)
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.gao.gov/assets/130024.pdf
    retrieved_on: 2026-05-26
    recency_category: federal-regulatory
  - url: https://ohiostate.pressbooks.pub/feptechcomm/chapter/5-2-executive-summary-abstract/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Wed-prep — Eval report metrics and reader structure

> [!instructor-review]
> **Known-bad-pattern callout (`ragas-faithfulness-only`):** the tradition of reporting only RAGAS *faithfulness* and treating it as sufficient for production sign-off is on the curriculum's blocklist (`skills/tech-research/references/known-bad-patterns.yml`). This reading deliberately requires all four RAGAS dimensions plus a layer of agentic-flow metrics plus a layer of operational metrics. If a sub-team's draft eval report reports faithfulness alone, that is a structural defect, not a stylistic one. Flag during instructor review.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the three layers of a production LLM eval report (RAG quality / agentic flow / operational) and the metrics that live in each.
- Explain why single-metric eval reports fail and what failure modes each RAGAS dimension catches that the others miss.
- Specify the dataset-provenance metadata an eval report must include for auditable reproducibility.
- Design a report so a 5-minute skimmer and a 30-minute deep-reader extract value from the *same* document without contradiction.
- Apply the inverted-pyramid principle (most important information first) to technical eval reports, and place executive-summary content at the size and position the technical-writing literature recommends.
- Decide which content belongs in the body vs the appendix based on reader-role, not author-convenience.

## 2. Introduction

An LLM eval report is the quantitative half of a system's defensibility story. The qualitative half — runbook, ADR catalog, architecture narrative — answers "what does the system do and why?" The quantitative half answers "how well does it do it, and how do we know?" Two things determine whether the report lands: **what's in it** (metrics, layered correctly) and **how it's laid out** (so readers of very different time-budgets can both extract value from one document).

The 2025–2026 production-LLM literature is unanimous that single-metric eval is insufficient. Multi-metric evaluation matrices are now standard in high-stakes deployments. Any single metric can be gamed; a set in tension surfaces failures a single number hides ([Maxim AI, retrieved 2026-05-26](https://www.getmaxim.ai/articles/complete-guide-to-rag-evaluation-metrics-methods-and-best-practices-for-2025/)).

A report served to multiple audiences usually fails one of them. The manager wants 5 minutes; the auditor wants 30. The fix is structural: write *one* report, engineered so the 5-minute reader's reading is a strict prefix of the 30-minute reader's. Executive summaries should be "about 5% as long as the document proper" and appear "before the body" ([Ohio State Press, retrieved 2026-05-26](https://ohiostate.pressbooks.pub/feptechcomm/chapter/5-2-executive-summary-abstract/); [U of Toronto, retrieved 2026-05-26](https://ecp.engineering.utoronto.ca/resources/online-handbook/components-of-documents/abstracts-and-executive-summaries/)). GAO makes it prescriptive: conclusions before discussions, summaries before details ([GAO Guide, retrieved 2026-05-26](https://www.gao.gov/assets/130024.pdf)).

This reading covers both halves — what to measure (three-layer metric shape) and how to lay it out (5/30 reader structure). It is the input to W6 Wed's authoring session for `eval/REPORT.md`.

## 3. Core Concepts

### 3.1 Three layers of metrics

A production LLM eval report has three layers, each with distinct failure modes:

**Layer 1 — RAG quality (or generative quality).** For systems with retrieval, the RAGAS-family metrics. For systems without retrieval, prompt-and-response quality metrics (faithfulness to source, answer relevance, factuality against ground truth).

**Layer 2 — Agentic flow quality.** For systems with multi-step or multi-agent flows, metrics that capture *flow* behaviour: routing accuracy, agent-to-agent agreement rates, HITL approval rates, tool-call success rates.

**Layer 3 — Operational quality.** Latency percentiles (p50/p95/p99), per-request cost, per-tenant cost, token consumption, error rates, throughput. At production scale, p99 latency degradations and per-tenant cost blowouts cause user-visible failures faster than mild quality regressions — operational data is *not* secondary.

A report that covers layer 1 only is a *capability* report; a report that covers all three is a *production* report. Quality answers "does it work?", flow answers "does it work safely?", operational answers "does it work at scale within budget?"

### 3.2 RAGAS metrics — why all four

RAGAS (Retrieval Augmented Generation Assessment) defines four core dimensions that catch different failure modes:

- **Faithfulness** — does the answer stay grounded in retrieved chunks? Catches hallucination *within* the response.
- **Answer relevancy** — does the answer address the user's question? Catches *off-topic* responses that may be faithful but not on-topic.
- **Context precision** — are the retrieved chunks the *right* ones? Catches retrieval-noise problems where the model gets correct chunks but in a flood of irrelevant ones.
- **Context recall** — did we retrieve everything we needed? Catches *missing* context that leads to incomplete answers.

The dimensions are deliberately in tension. A system that maximises faithfulness alone may over-refuse. A system that maximises answer relevancy may hallucinate to look responsive. Any *single* dimension can be gamed; the *combination* is harder to game and easier to diagnose. The RAGAS library shipped its 50th metric in 2025 and v0.4 (late 2025) expanded coverage beyond pure RAG to agentic flows ([RAGAS, TruLens, DeepEval Compared, Atlan](https://atlan.com/know/llm-evaluation-frameworks-compared/), retrieved 2026-05-26).

> The curriculum blocklist explicitly flags eval reports that present *faithfulness only*. Faithfulness reports near-perfect scores on systems that have severe context-recall problems, because it only measures grounding to what *was* retrieved, not whether enough was retrieved. Always report all four together.

### 3.3 Agentic-flow and operational metrics

For multi-step / multi-agent systems, additional metrics layer on top: routing accuracy (triage routing vs ground truth), agent agreement rate (disagreement signals HITL escalation), HITL approval-rate distribution (approve-as-is / light-revise / heavy-revise / reject — high approve-as-is risks rubber-stamping, high reject signals an unreliable agent), and tool-call success rate.

The 2026 LLM-observability literature converges on a small set of operational metrics ([Inference.net Guide](https://inference.net/content/llm-observability-monitoring-production-deployments/), retrieved 2026-05-26; [CostHawk P95/P99](https://costhawk.ai/glossary/p95-p99-latency), retrieved 2026-05-26): latency p50/p95/p99 per endpoint (never use averages as the primary latency metric), per-request cost (p50 + p99 — the long tail concentrates cost), per-tenant cost (surfaces the one anomalous tenant), error rate per endpoint by class, and token consumption (p50 + p99). Report as time-series, not single points.

### 3.4 Methodology and dataset provenance

The methodology section carries: a **golden dataset description** (size, construction, labellers, populations represented + under-represented — the "golden dataset" pattern is the 2025–2026 standard, [Maxim AI](https://www.getmaxim.ai/articles/building-a-golden-dataset-for-ai-evaluation-a-step-by-step-guide/), retrieved 2026-05-26); **provenance metadata** (source, retrieval date, reviewer identity, consent flags — the dataset is itself an audit-trail artifact); **eval cadence** (per PR, nightly, weekly — what triggers a re-eval); and **CI integration** (blocking, gating, or informational; if gating, the threshold and how it was set). Without these, the numbers are decorative. A complete report also distinguishes **offline evaluation** (golden-dataset, reproducible, the basis for sign-off) from **online monitoring** (production traffic, catches what the golden dataset doesn't represent). When the two diverge, that divergence is itself the most important finding.

### 3.5 The two-reader brief

The first authoring move for layout is to write down explicitly who the two readers are:

- **5-minute reader (skim depth).** Senior decision-maker. Wants: the headline numbers, the trend direction, the named failure modes, the recommendation. Does *not* want: methodology, per-segment breakdowns, raw output.
- **30-minute reader (depth).** Reviewer, auditor, or model-risk specialist. Wants: methodology, dataset provenance, per-segment breakdowns, evidence supporting every §1 claim, unresolved questions.

The structural rule: the 5-minute reader's reading is a *strict prefix* of the 30-minute reader's reading. Anything in the 5-minute version is also in the 30-minute version. Nothing in the 5-minute version contradicts anything in the 30-minute version. This is the same nesting principle as the three-levels rule for presentations, applied to a written document.

### 3.6 The seven-section eval report shape

```
eval/REPORT.md

§1 Executive summary
   - 300 words or fewer. Read in 5 minutes including time to think.
   - Multi-metric headline, trend direction, 2-3 named failure modes
     with Finding IDs, the recommendation.

§2 Methodology
   - Golden dataset, populations represented + under-represented,
     inter-rater agreement. Eval cadence + CI gating thresholds.
     Online-monitoring sampling rate.

§3 Quantitative results — quality layer
   - All four RAGAS dimensions over time. Per-tenant breakdown if relevant.

§4 Quantitative results — agentic flow layer
   - Routing accuracy, agent-agreement rates, HITL approval rates,
     tool-call success rates. Time series, not snapshot.

§5 Known failure modes
   - Each as a named entry: what failed, evidence, root cause if known,
     Finding ID, owner, remediation plan, ETA.

§6 Unresolved questions for next quarter
   - The "what we would test next if we had time" list.
   - Distinct from §5: failure modes are known and named; unresolved
     questions are gaps in *evidence*, not gaps in *system behaviour*.

§7 Appendix
   - Raw eval outputs, per-PR eval diff, dashboard links,
     dataset-card excerpt, full per-segment breakdowns.
```

Each section earns its place against a reader-question: §2 = "how do you know?"; §3 = "what are the numbers?"; §4 = "what about the flow?"; §5 = "what's broken?"; §6 = "what don't you know?"; §7 = "show me."

### 3.7 The executive summary as a contract; inverted pyramid; non-contradiction

The §1 Executive summary is a *contract* between the report and its 5-minute reader. If the body contains a worse number than §1 advertises, the reader has been misled — at the next review the reader will read the report differently, or not at all. §1 must lead with the headline metric set (multi-metric, not single), name the trend direction explicitly, surface the 2–3 most material named failure modes with Finding IDs, and state the recommendation as one sentence. If it cannot fit those four elements in 300 words, the system is more complex than the executive-summary format can compress — split the report or call out only the most material numbers.

Inside each section, the **inverted-pyramid principle** applies again: lead with the conclusion, then evidence, then details ([Inverted pyramid, Wikipedia, retrieved 2026-05-26](https://en.wikipedia.org/wiki/Inverted_pyramid_(journalism)); [SUNY Ulster Reports, retrieved 2026-05-26](https://courses.lumenlearning.com/sunyulster227technicalwriting/chapter/6-reports/)). A §5 entry reads as a finding-shaped summary ("Context recall regressed 0.81 → 0.79; root cause = chunker change; closing via hybrid-search rollout Q3; F-2026-Q2-021"), then evidence underneath. The skimmer reading only first sentences still gets the gist.

**Non-contradiction discipline.** The structural failure that destroys multi-reader reports is contradiction between sections — §1 says "quality on target" but §3 reports a metric below target; §1 says "launch as planned" but §5 lists a launch-blocking gap. After authoring §1, make a pass through the rest of the document looking for sentences that contradict §1; if any exist, rewrite §1 or the contradictory section. Appendices contain content that doesn't fit in the body but cannot be left out — large tables, raw outputs, per-segment breakdowns. Decision rule: specialist-only → appendix; every-reader → body. Every appendix entry must have a body-section pointer; otherwise it should not exist.

## 4. Generic Implementation

A worked example outside federal acquisitions: a streaming-media platform's content-recommendation team publishes a quarterly model-quality report.

**§1 Executive summary (≈ 250 words):**

> Q2 2026 recommendation model update. **Headline metrics:** offline NDCG@10 = 0.412 (target ≥ 0.40, up from Q1 0.398); session-level diversity = 0.61 (target ≥ 0.55, flat); p99 inference latency = 18ms (target ≤ 25ms, improved from 22ms). **Trend:** quality up, diversity flat, latency improved. **Named failure modes:** (a) cold-start performance for accounts < 7 days old remains below target — F-2026-Q2-104, owner cold-start squad, ETA Q3; (b) quality degrades 11% on accounts with > 90% prior-skip behaviour — F-2026-Q2-118, owner ranking team lead, ETA next quarter. **Recommendation:** roll out v17 to 100% in Q3 with the two limitations disclosed; defer cold-start fix to the dedicated Q3 launch.

The skimmer has three headline numbers, the trend, two named failure modes with owners/ETAs, and a one-sentence recommendation. Reading time ≈ 2 minutes.

**§2 Methodology (depth):**

> Offline evaluation against the v3 user-journey dataset (2.4M sessions, stratified across geography × device × tenure × content-affinity; refreshed monthly). Over-represents long-tenure NA/EU; under-represents new LATAM accounts — F-2026-Q1-072 tracks expansion. Eval runs nightly on every PR touching `/recommend/*`. CI gating at NDCG@10 ≥ 0.38 (deploy block) and session-diversity ≥ 0.50 (warning). Online monitoring sampled at 1% of production sessions.

**§5 Known failure modes (excerpt):**

> **F-2026-Q2-104 — Cold-start regression on accounts < 7 days old.** Severity: medium. Evidence: NDCG@10 = 0.31 cold-start vs 0.42 established; flat Q1 → Q2. Root cause: the new contextual-bandit reduces exploration when prior-watch signal is sparse — by design for established accounts, harmful here. Remediation: separate cold-start policy in development for Q3. Mitigation in v17: cold accounts route to the v16 path.

**Non-contradiction check:** §1 says "roll out v17 to 100% in Q3"; §5 says "cold accounts route to v16." Consistent — the report does not say "launch v17 universally," it says "launch with the named limitation disclosed and v16 fallback active for cold accounts." Both sections agree.

## 5. Real-world Patterns

**E-commerce recommendation reporting.** Major platforms combine offline ranking quality (NDCG@K, MRR), online A/B outcomes (CTR, conversion), and operational metrics (p99 latency, per-region serving cost). The three-layer pattern transfers exactly.

**Healthcare — FDA clinical-AI validation.** AI/ML clinical-decision-support submissions include clinical performance (sensitivity, specificity, AUC) plus population characterisation, workflow integration (clinician agreement, override rate), and deployment-environment metrics. Sensitivity-only reports face additional information requests ([medRxiv 2025](https://www.medrxiv.org/content/10.1101/2025.03.04.25323131.full.pdf), retrieved 2026-05-26).

**Scientific journal articles.** Scientific publishing has engineered multi-reader documents for centuries. Abstract = 30-second; intro + conclusion = 5-minute; methods/results/discussion = 30-minute; supplementary = appendix. Each layer stands alone without contradicting the others.

**Investment-research reports.** Equity-research notes serve portfolio managers (skim — recommendation, price target) and investment committees (deep-read — thesis, comp set, model, risks). Analysts who fail this discipline lose readers from the skim layer first ("you buried the lede"), then from the depth layer ("front didn't match back").

**GAO and IG reports.** GAO publishes for layered readership: a "Highlights" page, the full report, and appendices. The writing guide makes layering prescriptive — conclusions and summaries before details ([GAO Guide, retrieved 2026-05-26](https://www.gao.gov/assets/130024.pdf)).

## 6. Best Practices

- **Report all three layers** (RAG/generative quality, agentic flow, operational). A one-layer report is a partial story.
- **Always report all four RAGAS dimensions together.** Single-dimension reporting is on the blocklist for good reason.
- **Use percentile metrics for latency and cost** (p50, p95, p99). Averages hide tail behaviour that drives user-visible failures.
- **Treat the golden dataset as a versioned artifact** with a dataset card (size, construction, populations, inter-rater agreement, known under-represented groups). Cite the version with every metric.
- **Distinguish offline eval from online monitoring** and report both. Divergence is the most important finding when it appears.
- **Tie every regression to a Finding ID** with owner, ETA, and remediation plan. Numbers without action items read as theatre.
- **Write the executive summary last, place it first.** Cap it at 5% of report length (~300 words for a 6,000-word report).
- **Make every section's first paragraph the inverted-pyramid lede.** A skimmer reading only section opens still gets each section's gist.
- **Cross-reference §1 to the depth sections.** "See §5 for full description of F-2026-Q2-104" — the reader knows where to dig.
- **Run the non-contradiction pass before publishing.** No claim in §1 should be silently undermined by a number in §3 or a finding in §5.
- **Use appendices for specialist-only material, with body cross-references.** Appendix entries without a body pointer shouldn't exist.

## 7. Hands-on Exercise

**Time:** 20 minutes. **Format:** combined metric outline + report structure.

You are designing the quarterly eval report for a generic customer-support AI assistant deployed to a SaaS product. The assistant retrieves against the product's help-centre and generates draft responses human agents review before sending. 5-minute reader: VP Customer Experience. 30-minute reader: model-quality team lead.

**Steps:**

1. **List the metrics in each layer:** Layer 1 (all four RAGAS dimensions with target thresholds), Layer 2 (≥ 3 flow-level metrics relevant to human review), Layer 3 (p50/p95/p99 latency for ≥ 2 endpoints, per-request and per-tenant cost).
2. **Draft the §1 Executive summary** in ≤ 300 words: three headline metrics, trend direction, two named failure modes (invent them), one-sentence recommendation.
3. **Outline §2 through §7** with one or two sentences each. For §5, write one full finding-shaped entry tied to a §1 failure mode.
4. **Identify one contradiction risk** — name a §1 sentence and a §3-or-§5 sentence that, if both appeared, would undermine each other. Explain the corrective.
5. **Identify two metrics that, if reported alone, would mislead** the reader. Explain what each would hide.

**Self-check:** All four RAGAS dimensions, not just faithfulness? Latency in percentiles, not averages? §1 leads with the metric set (multi, not single)? §1 names at least two failure modes with IDs? Methodology says what the golden dataset *does not* represent? VP CX could act on §1 alone? §5 reinforces §1 rather than undermining it?

**What good looks like:** three labelled layers, ≥ 4 metrics each; §1 is 250–300 words with three metrics + trend, two failure modes with owners/ETAs, recommendation; §5 entry leads with the finding's headline, then evidence/root cause/owner/ETA; methodology names ≥ 1 under-represented population; offline-vs-online explicit; two "misleading-alone" metrics named (likely candidate: faithfulness — high faithfulness with low context recall reads as a working system that's actually missing answers); contradiction check identifies a plausible leak with corrective.

## 8. Key Takeaways

- Does my eval report cover all three layers (RAG quality / agentic flow / operational) or only one?
- Have I reported all four RAGAS dimensions together so the cross-dimension tension is visible?
- Are my latency and cost metrics percentile-based (p50/p95/p99) rather than averaged?
- Does my methodology section name what the golden dataset *does not* represent?
- Have I separated offline eval from online monitoring, and reported both?
- Is the executive summary ≤ 5% of report length, placed at the top, and complete enough to act on alone?
- Does every section open with an inverted-pyramid lede so skimmers extract the gist from the first paragraph?
- Have I run a non-contradiction pass between §1 and the depth sections?

## Sources

1. [Complete Guide to RAG Evaluation: Metrics, Methods, and Best Practices for 2025 (Maxim AI)](https://www.getmaxim.ai/articles/complete-guide-to-rag-evaluation-metrics-methods-and-best-practices-for-2025/) — retrieved 2026-05-26
2. [RAGAS, TruLens, DeepEval: LLM Evaluation Frameworks (2026) (Atlan)](https://atlan.com/know/llm-evaluation-frameworks-compared/) — retrieved 2026-05-26
3. [Building a "Golden Dataset" for AI Evaluation (Maxim AI)](https://www.getmaxim.ai/articles/building-a-golden-dataset-for-ai-evaluation-a-step-by-step-guide/) — retrieved 2026-05-26
4. [LLM Observability: A Complete Guide to Monitoring Production Deployments (Inference.net)](https://inference.net/content/llm-observability-monitoring-production-deployments/) — retrieved 2026-05-26
5. [P95 / P99 Latency — AI Cost Glossary (CostHawk)](https://costhawk.ai/glossary/p95-p99-latency) — retrieved 2026-05-26
6. [Abstracts and Executive Summaries (University of Toronto Engineering Communication)](https://ecp.engineering.utoronto.ca/resources/online-handbook/components-of-documents/abstracts-and-executive-summaries/) — retrieved 2026-05-26
7. [Reports (SUNY Ulster Technical Writing)](https://courses.lumenlearning.com/sunyulster227technicalwriting/chapter/6-reports/) — retrieved 2026-05-26
8. [Inverted pyramid (journalism) — Wikipedia](https://en.wikipedia.org/wiki/Inverted_pyramid_(journalism)) — retrieved 2026-05-26
9. [Guide for Writing Executive Summaries (GAO)](https://www.gao.gov/assets/130024.pdf) — retrieved 2026-05-26
10. [Executive Summary and Abstract (Ohio State Pressbooks)](https://ohiostate.pressbooks.pub/feptechcomm/chapter/5-2-executive-summary-abstract/) — retrieved 2026-05-26

Last verified: 2026-05-26
