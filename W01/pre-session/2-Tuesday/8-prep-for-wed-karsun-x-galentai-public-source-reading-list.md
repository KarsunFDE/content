---
week: W01
day: Tue
topic_slug: prep-for-wed-karsun-x-galentai-public-source-reading-list
topic_title: "Prep for Wed — public-source comparative discipline (vendor due diligence as FDE skill)"
parent_overview: W01/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://www.smartsheet.com/content/vendor-assessment-evaluation
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.investmentbankingcouncil.org/blog/competitive-benchmarks-strengthening-manda-due-diligence
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://dealroom.net/faq/commercial-due-diligence
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://itoaction.com/competitor-analysis-framework-know-your-competition-and-identify-opportunities/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://legal.thomsonreuters.com/en/insights/articles/five-best-practices-for-vendor-due-diligence
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Prep for Wed — public-source comparative discipline (vendor due diligence as FDE skill)

> A separate research artefact at `~/KarsunFDE/content/research/galentai-vs-karsun-redux-comparative-20260522.md` contains the **applied** comparative for Wednesday's session (Karsun ReDuX vs GalentAI specifically). **This reading does not duplicate that artefact.** This reading walks the **generic discipline** of vendor comparison from public sources — the skill you'll exercise on every client engagement, applied to any two vendors in any industry.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Articulate why **vendor-supplied comparative materials are structurally biased** and what an FDE must do instead.
- List the **standard public-source channels** for vendor due diligence (filings, blogs, marketplace listings, customer reviews, conference talks, archived web) and name what each tells you.
- Apply a **6-dimension comparative matrix** (capability, market, deployment model, pricing transparency, ecosystem fit, regulatory posture) to two vendors and name where the matrix is thin.
- Recognise the **disciplines borrowed from M&A due diligence** that translate to vendor comparison and explain when each is appropriate.

## 2. Introduction

Vendor comparison is a recurring task on every client engagement. The naive answer is to read each vendor's website and accept their framing. This is the dominant antipattern. Thomson Reuters 2024 makes the diagnosis: vendor-supplied materials are *marketing*, optimised for the vendor's narrative — they tell you what the vendor wants you to know, not what you need to know. The Smartsheet 2024 corrective: **a structured, dimension-by-dimension comparative from public sources the vendor doesn't control** — SEC filings, customer reviews on G2 / TrustRadius, regulatory filings, engineering blog cross-references, archived web from before recent rebrands.

The 2024 commercial-due-diligence literature (Dealroom, IBCA) treats this as a transferable discipline from M&A practice. An acquirer runs *commercial due diligence* — a structured assessment of market position, competitive landscape, customer base, operational maturity, sourced from public information plus customer interviews. The same toolkit, scaled down, is what an FDE applies to vendor comparison. The 6 dimensions, the public-source channels, the bias-detection checks all transfer.

Wednesday's session models this discipline applied to GalentAI vs Karsun ReDuX specifically. **At every federal engagement, you will run this analysis** — RAG vendors, AIOps platforms, observability tools, sub-vendors in teaming arrangements. Treat Wednesday as the demonstration; this reading is the technique.

## 3. Core Concepts

### 3.1 Why vendor-supplied materials are structurally biased

Three categories of bias appear in essentially every vendor's marketing content:

| Bias | What it looks like | Counter |
|------|--------------------|---------|
| **Selection** | The vendor publishes case studies where they won. Lost deals, deployments that stalled, customers that churned — invisible. | Look at customer reviews on G2 / TrustRadius / Gartner Peer Insights, where dissatisfied customers are over-represented. |
| **Framing** | The vendor names the comparison axes most favourable to them. "We're the only platform that does X" — where X is something the competition doesn't claim because it isn't a primary buying criterion. | Build your own axes from the *buyer's* perspective. Buyer-led requirement docs (client RFPs, federal procurement requirements) name the real axes. |
| **Confounding** | The vendor cites stats that don't actually compare like-for-like. "We saw a 40% improvement" — over what baseline? Compared to what? | Demand baselines explicitly. "Compared to manual processes" is not the same as "compared to our top competitor." |

The Thomson Reuters guide reinforces: even when a vendor is honest, the *aggregate* of their marketing materials produces a skewed picture because every individual artefact is selected for the vendor's interests. This is not malice; it is the structural property of self-promotion. The FDE's job is to operate alongside it, not deny it.

### 3.2 Standard public-source channels

The vendor-due-diligence literature converges on a small set of canonical public sources, each with a different epistemic value:

| Source | What it tells you | Limits |
|--------|--------------------|--------|
| **SEC filings (10-K, 10-Q) for public companies** | Revenue, growth, risk factors, segment reporting, customer concentration, named competitors | Only for public companies; redactions; filed quarterly so lags reality |
| **Acquisition / fundraising announcements** | Strategic direction, owned vs partner-led capabilities, M&A integration debt | Often spun by the announcing party; investigate the *terms*, not the marketing |
| **Government contract awards** (USAspending.gov, FPDS) | Federal-buyer adoption, contract size, agency mix, prime vs sub-contractor pattern | Federal-only; lags by months |
| **Marketplace listings** (AWS Marketplace, Azure Marketplace, GCP Marketplace) | Pricing transparency, supported regions, GovCloud availability, FedRAMP authorisation | Marketing copy under platform constraints — still less filtered than the vendor's own site |
| **Customer reviews** (G2, TrustRadius, Gartner Peer Insights, Capterra) | Real-user pain points, integration friction, support quality | Reviewer self-selection bias; vendor-incentivised reviews skew positive |
| **Engineering blogs (vendor + customer)** | Technical posture, what they actually built vs what they ship, what's hard | Vendor blogs are still marketing; customer engineering blogs are gold but rare |
| **Conference talks (recorded)** | Engineers describing real architectures, not marketers describing aspirational ones | Conference selection bias — only successes get presented |
| **Archived web (Wayback Machine)** | What the vendor *used* to claim — useful for spotting strategic pivots, abandoned products, rebrandings | Snapshots may be incomplete |
| **Open positions** | What the vendor is actively building — engineering job titles reveal strategy | Lags 3–6 months and noisy |

The IBCA 2024 M&A piece adds a discipline rule: **triangulate across at least three independent channels for any claim**. A vendor's own claim + their conference talk are not independent. A vendor's claim + a customer engineering blog + a customer review on G2 are three independent triangulation points.

### 3.3 The 6-dimension comparative matrix

The Smartsheet and Dealroom 2024 sources converge on a near-standard set of comparison dimensions. Adapted for software-vendor comparison (rather than M&A target evaluation), the canonical six are:

| Dimension | What you're answering | Public-source channels |
|-----------|------------------------|------------------------|
| **Capability scope** | What does each vendor actually *do*? Which features are real, which are claimed but unevidenced? | Engineering blogs, conference talks, marketplace listings, customer reviews |
| **Market positioning** | Who is each vendor selling to? Federal CIOs? Commercial SMB? Hyperscaler partners? | Government-contract data, public-customer logos, analyst reports |
| **Deployment model** | SaaS-only? Self-hostable? On-prem? GovCloud / FedRAMP? Air-gap-capable? | Marketplace listings, FedRAMP marketplace, ATO precedent disclosures |
| **Pricing transparency** | Public pricing? Volume tiers? Negotiable? "Contact us"? | Marketplace listings, customer reviews, RFP responses on public-sector contract sites |
| **Ecosystem fit** | What does this integrate with? Cloud providers? LLM providers? Observability vendors? | API documentation, partnership announcements, marketplace integrations |
| **Regulatory posture** | FedRAMP status (Low/Moderate/High)? StateRAMP? CMMC? FISMA? ATO precedent? | FedRAMP marketplace, SAM.gov, CMMC accreditation registry |

A seventh dimension worth adding for AI-platform comparisons specifically: **LLM provider neutrality** — does the platform lock you into one provider (Bedrock-only, OpenAI-only) or support multi-provider? For federal procurement, multi-provider stance is increasingly load-bearing because Anti-Lock-In language appears in 2024-2026 procurement requirements.

### 3.4 What "good" looks like as a deliverable

A defensible comparative matrix has three properties beyond the dimensions themselves:

1. **Every cell has a source.** Not the vendor's website (or rather: the vendor's website *plus* an independent corroboration). Comparative matrices without sources are slides; comparative matrices with sources are evidence.
2. **Gaps are named, not hidden.** If a dimension can't be filled from public sources for one vendor, the cell says "Not publicly disclosed" — not "Better" or "Worse" or "Unknown" without qualifier. Explicit gaps invite the client conversation about *how* to fill them.
3. **The buyer's criteria drive the dimensions, not the vendor's.** If the client cares about FedRAMP authorisation timeline above all else, that dimension is row 1. Adapting the matrix to the *buyer's* priorities is the FDE's value-add.

The Thomson Reuters 2024 piece names the test for a defensible deliverable: "Could you defend this matrix in front of the vendor's own salesperson?" If yes, you have an FDE-grade artefact. If the vendor's salesperson could trivially refute it ("you're citing a 2019 blog post"), you don't.

### 3.5 Disciplines borrowed from M&A due diligence

Three pieces of the M&A toolkit translate cleanly to vendor comparison:

**Customer reference triangulation.** Find publicly-available customer voices — G2 reviews, conference talks by named customers, government-contract awards — and triangulate the vendor's claims against them. Bar: "do at least three customers, sourced independently, corroborate this claim?"

**Risk-factor enumeration.** M&A diligence produces "risk factors" — claims that, if untrue, change the deal terms. For vendor comparison, the equivalent is a "deal-breaker watch list" that exposes where verification matters most.

**Market-context sizing.** Size the *category* — a vendor with 80% share of a stagnating category is a different bet than 10% share of a growing one.

## 4. Generic Implementation

A worked example: comparing two hypothetical AI-observability vendors for a non-federal **fintech** client decision. The same shape applies regardless of industry.

**Vendors:** Vendor Alpha (long-established APM player adding LLM features) vs Vendor Beta (LLM-native startup).

**Step 1 — gather public sources** (a 2-hour bounded research session):
- Both vendors' marketplace listings (AWS Marketplace, GCP Marketplace).
- G2 reviews for both, last 12 months only.
- Engineering blogs: vendor's own + any customer blog mentioning either.
- Most recent fundraising / earnings announcements.
- API documentation (publicly accessible).
- 2 conference talks each (YouTube, search by vendor name + 2025).

**Step 2 — populate the 6-dimension matrix:**

```markdown
|                          | Vendor Alpha                       | Vendor Beta                          |
|--------------------------|------------------------------------|--------------------------------------|
| Capability scope         | APM + traces + logs + LLM eval     | LLM eval + traces (no general APM)   |
|   sources                | G2 reviews, AWS Marketplace listing| G2 reviews, GitHub README             |
| Market positioning       | Enterprise (10K+ FTE buyers)       | Startup-to-mid-market AI teams        |
|   sources                | Customer logos, earnings call      | Customer logos, fundraising deck      |
| Deployment model         | SaaS + self-host (enterprise tier) | SaaS-only                             |
|   sources                | AWS Marketplace, docs              | Pricing page                          |
| Pricing transparency     | Volume-tiered, contact sales       | Public per-event pricing              |
|   sources                | AWS Marketplace                    | Pricing page                          |
| Ecosystem fit            | OTel-native; 100+ integrations     | OTel-native; 12 integrations          |
|   sources                | Docs index                         | Docs index                            |
| Regulatory posture       | SOC2, ISO27001                     | SOC2 (pursuing ISO27001)              |
|   sources                | Trust center page                  | Trust center page                     |
```

**Step 3 — name gaps**: Vendor Beta has no enterprise-tier deployment model disclosed; Vendor Alpha's LLM-specific eval depth is not separable in public sources from their generic APM eval.

**Step 4 — write the recommendation paragraph for the client**, citing the matrix, naming the gaps, and proposing which gaps to close via vendor RFP / customer-reference call. A defensible recommendation runs ~250 words and points to the matrix as the evidence.

That entire artefact takes ~3-4 hours total. It is the deliverable shape you'll repeat throughout your FDE career.

## 5. Real-world Patterns

**Fintech (CTO-level platform selection).** Mid-stage fintech CTOs run vendor-comparison exercises routinely — payments orchestration, fraud detection, KYC providers. The CTO delegates the public-source comparative to an embedded engineer for the technical-evidence layer, then layers commercial diligence on top. Pattern: **engineer-led technical comparative feeds executive-led commercial decision**.

**Healthcare IT (HIE platform selection).** A regional Health Information Exchange uses the same 6-dimension toolkit weighted heavily toward regulatory posture (HIPAA, HITRUST, state privacy laws). Pattern: **dimension weighting by regulatory primacy** — the regulatory must-have is row 1 and the disqualifier.

**E-commerce (CDP selection).** A retailer comparing Customer Data Platforms cites peer-retailer engineering blogs heavily — "Macy's wrote up their CDP migration." Pattern: **peer-customer engineering-blog content as the highest-trust public source**.

**Logistics (TMS / WMS evaluation).** Mid-size 3PLs leaning on marketplace listings + customer reviews because the SaaS-vs-self-host dimension is operationally critical (warehouses lose internet; SaaS-only TMS is a non-starter for some). Pattern: **deployment-model dimension as architectural disqualifier**.

## 6. Best Practices

- **Never accept the vendor's own comparative as the artefact.** Build your own.
- **Triangulate every load-bearing claim across three independent channels.** Vendor + 1 = not enough; vendor + 2 independents = the floor.
- **Source every cell, even when the source is "vendor's own docs" — make that explicit so the client knows where corroboration is missing.**
- **Name gaps as "not publicly disclosed" rather than guessing.** Honest gaps invite useful conversations; bad guesses destroy the matrix's credibility.
- **Adapt dimension weights to the buyer's priorities, not the vendor's strengths.** If FedRAMP is the buyer's must-have, that's row 1.
- **Build the matrix in plain markdown / a Confluence table, not slideware.** Slides hide reasoning; tables expose it.
- **Time-box public-source research to 2–4 hours per vendor for an initial comparative.** Beyond that, you're optimising the matrix instead of running the next decision.

## 7. Hands-on Exercise

**Whiteboarding-the-discipline drill (12-15 min, no code):**

Pick two AI-observability vendors of your choice from the wider industry (Datadog vs Honeycomb is one possible pairing; Arize vs Langfuse is another). Don't actually research them — instead, **plan the research session**:

1. List the **public-source channels** you'd use, in order of expected information density.
2. List the **6 dimensions** of the matrix you'd populate.
3. For each dimension, name **which channel(s)** you'd lean on most heavily.
4. Identify **two load-bearing claims** you'd expect each vendor to make about themselves, and **two ways you'd triangulate** each.
5. Set a **time budget** (how long would this take to do well? to do quickly?).

**What good looks like:** a one-page plan that names channels, dimensions, sources-per-dimension, triangulation targets, and time budget. If you can write this for an industry pair you don't know, you can write it for Wednesday's comparative and for every engagement after.

## 8. Key Takeaways

- *Can I articulate the three structural biases in vendor-supplied materials and the counter for each?* (Maps to LO 1.)
- *Can I name the 8+ standard public-source channels and what each tells you?* (Maps to LO 2.)
- *Can I apply the 6-dimension comparative matrix to two vendors and name the gaps?* (Maps to LO 3.)
- *Can I name three M&A-due-diligence disciplines and how each translates to vendor comparison?* (Maps to LO 4.)

## Sources

1. [Smartsheet — Vendor Assessment & Evaluation: How to Choose the Best](https://www.smartsheet.com/content/vendor-assessment-evaluation) — retrieved 2026-05-26
2. [Investment Banking Council — Competitive Benchmarks Strengthening M&A Due Diligence](https://www.investmentbankingcouncil.org/blog/competitive-benchmarks-strengthening-manda-due-diligence) — retrieved 2026-05-26
3. [Dealroom — Commercial Due Diligence: Types, Process & Checklist](https://dealroom.net/faq/commercial-due-diligence) — retrieved 2026-05-26
4. [Insight to Action — Competitor Analysis Framework: Know Your Competition and Identify Opportunities](https://itoaction.com/competitor-analysis-framework-know-your-competition-and-identify-opportunities/) — retrieved 2026-05-26
5. [Thomson Reuters — Know your vendor: 5 best practices to reduce risk](https://legal.thomsonreuters.com/en/insights/articles/five-best-practices-for-vendor-due-diligence) — retrieved 2026-05-26

Last verified: 2026-05-26
