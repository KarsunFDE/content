---
week: W01
day: Wed
topic_slug: reading-for-the-comparative-session
topic_title: "Reading for the comparative session — what to bring an opinion on"
parent_overview: W01/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 15
sources:
  - url: https://futurumgroup.com/press-release/should-saas-vendors-prioritize-ai-for-vertical-or-horizontal-use-cases/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.glean.com/perspectives/horizontal-ai-platform
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://lsvp.com/stories/the-next-wave-of-enterprise-ai-from-horizontal-models-to-vertical-solutions/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://en.wikipedia.org/wiki/Competitive_intelligence
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://guides.libraries.psu.edu/berks/CI
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Reading for the comparative session — what to bring an opinion on

> Generic deep-dive on **how to compare two enterprise AI platforms when you do not have access to either company's internals**. The day's overview (`1-DailyTopicOverview.md` §2) gives you the specific GalentAI vs Karsun ReDuX framing and points you at the canonical research artifact at `~/KarsunFDE/content/research/galentai-vs-karsun-redux-comparative-20260522.md`. This file teaches the *skill*: public-source competitive analysis, dimension selection, and the discipline of citation-anchored claims you will need on every federal engagement.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define **public-source competitive intelligence** and distinguish it from insider analysis, restating the four-stage CI process (plan → collect → analyse → disseminate).
- Choose between **horizontal** and **vertical** AI platform categorisations and name at least three buyer-side evaluation criteria that change depending on which category a product sits in.
- Construct a **multi-dimensional comparative matrix** (8–12 dimensions) for two competing enterprise AI platforms when only public information is available.
- Apply **citation discipline** — every claim traces to a public URL with a retrieval date — and articulate why "modesty in claims" is itself the lesson at a federal client engagement.
- Recognise common analyst frames (Forrester Wave, Gartner Magic Quadrant) and explain why being absent from them is a finding, not necessarily a defect.

## 2. Introduction

Engineers who join a forward-deployed engagement quickly discover that the most consequential evaluations they will write are **about products they have never seen the inside of**. A client asks: should we adopt Vendor A or Vendor B? You have access to two websites, three press releases, a marketplace listing, an analyst quadrant, and a podcast interview. That is the dataset. You will be expected to produce a recommendation that holds up under scrutiny from a Chief Information Officer who will, several months later, either thank you or remember that you were wrong.

This is the discipline of **public-source competitive intelligence**. It is older than the AI industry — Penn State's library guides describe a four-stage process (planning, data collection, analysis, dissemination) used by competitive-intelligence professionals across pharmaceuticals, defence procurement, and consumer goods long before any language-model platform existed [4][5]. The methodology is unglamorous: define what you need to know, find every public source that bears on it, weigh the sources against one another, write down what you concluded and what you did not. Done well, it is the most defensible kind of analysis a forward-deployed engineer produces, because every conclusion ships with an audit trail.

The reason this matters specifically in the **enterprise AI platform** market is that the market itself has bifurcated along a horizontal/vertical axis, and the buyer-side evaluation criteria are not the same on both sides [1][2][3]. Horizontal platforms (think general-purpose conversational AI, enterprise knowledge assistants, multi-LLM orchestrators) compete on breadth, deployment speed, and standardisation. Vertical platforms (industry-specific — fintech, healthcare, federal acquisitions, mainframe modernisation) compete on domain accuracy, workflow embedding, and regulatory posture. Walking into a comparison with the wrong frame produces the wrong recommendation.

The Wednesday morning session is not asking you to predict which platform "wins" — that is a question for analysts with surveying budgets. It is asking you to build the *frame* a client would use, defend the *dimensions* you chose, and surface the *unknowns* honestly. The output is rarely a verdict. The output is a comparative matrix and a list of follow-up questions that, if answered, would change the recommendation.

## 3. Core Concepts

### 3.1 The four-stage CI process

Wikipedia's competitive-intelligence entry and Penn State's library guide both converge on the same four stages [4][5]:

1. **Plan.** Define the decision the analysis serves. "Should we adopt X or Y" is a question; "What dimensions matter to the buyer, and on which can we get evidence?" is the plan.
2. **Collect.** Pull from public sources — vendor websites, press releases, marketplace listings, analyst reports, conference talks, customer testimonials, regulatory filings, patent filings, job postings, source code if open, technical documentation if public.
3. **Analyse.** Score each vendor on each dimension. Note where evidence is strong, weak, or absent. Absence is itself a data point.
4. **Disseminate.** Produce the artifact the decision-maker will read. Citations are non-negotiable: every claim has a source URL and a retrieval date.

The forward-deployed engineer version of this is intentionally light on stage 2 (you do not have weeks) and heavy on stage 4 (the comparative ADR is the artifact that will follow the engagement around for months).

### 3.2 Source tiers (high-value vs moderate-value vs flag-with-caveat)

Not every public source is equal. Competitive-intelligence practitioners stratify [3][4]:

- **High-value, low-effort:** vendor's own marketing site, marketplace listings, public press releases on funding/awards/certifications, regulatory filings (SEC 10-K, FedRAMP marketplace listings, GSA Schedule entries), analyst quadrants (Gartner Magic Quadrant, Forrester Wave).
- **Moderate-value, moderate-effort:** conference talks (re:Invent, KubeCon, Black Hat — recorded sessions reveal architecture decisions), customer review sites (G2, Gartner Peer Insights, Capterra), public-cloud-blog deep-dives by the vendor's own platform team, job postings (which roles a company hires for telegraph what they are building).
- **Flag-with-caveat:** social media posts by employees, podcast interviews with founders, tweetstorms — useful for signals, dangerous as primary citations because they are unverified and easily contradicted.

A good public-source matrix tries to anchor every claim to a high-value source; if only a moderate-value source supports it, the claim is softened ("vendor appears to support X" rather than "vendor supports X"); if only a flag-with-caveat source supports it, the claim is excluded or noted as a follow-up question.

### 3.3 Horizontal vs vertical: why the dimensions differ

Recent industry analysis [1][2][3] has documented a clear bifurcation:

| Platform shape | What wins buyers | What you compare on |
|----------------|------------------|---------------------|
| **Horizontal** (general-purpose) | Rapid deployment, standardisation across departments, multi-vendor LLM portability, broad use-case coverage | Breadth (how many engines/agents), integration surface (how many connectors), LLM-vendor neutrality, time-to-first-value, total cost of ownership across departments |
| **Vertical** (industry-specific) | Domain accuracy, regulatory fit, workflow embedding, defensible moats | Domain-specific accuracy on industry tasks, certifications (FedRAMP, HIPAA, PCI), workflow integration with industry-standard systems, data-residency posture, exit cost / lock-in |

Lightspeed Venture Partners has framed the transition as "from horizontal models to vertical solutions" [3], arguing that the next wave of enterprise AI value comes from products that own a workflow end-to-end. Glean — itself a horizontal player — counters that horizontal platforms are the standardisation layer that lets enterprises avoid a sprawl of vertical point solutions [2]. Both are right for different buyers; the analyst's job is to identify which frame fits the client in the room.

For the Wednesday AM session: GalentAI is horizontal (9 engines, 125+ agents, multi-LLM, no federal vertical posture). Karsun ReDuX is vertical (federal mainframe modernisation, 3-agent loop, Bedrock-only). They are not direct competitors. A buyer who is comparing them is asking the wrong question — and surfacing that *is* a legitimate finding.

### 3.4 Standard dimensions for an enterprise-AI-platform matrix

Across the public material, a usable 8–12 dimension matrix typically covers:

1. **Capability scope** — what does the product do, in one sentence?
2. **Target customer** — who buys this, by industry and company size?
3. **Deployment model** — SaaS, self-hosted, hybrid, on-customer-cloud?
4. **Regulatory posture** — FedRAMP / HIPAA / SOC 2 / ISO 27001 / FISMA?
5. **LLM provider lock-in** — single-LLM, multi-LLM, BYO-LLM?
6. **Integration surface** — what connects to what?
7. **Exit cost** — what does it cost to leave?
8. **Strengths the vendor claims** — verbatim from marketing, flagged as vendor claim.
9. **Gaps or risks** — derived from absence of evidence on common dimensions.
10. **Analyst recognition** — Gartner / Forrester / IDC / Everest mentions.
11. **Sales motion** — direct, channel partner, marketplace, prime contractor?
12. **Pricing transparency** — published list price, request-a-quote, opaque?

You do not need all twelve every time. The art is choosing the six to eight that the buyer in the room cares about and being explicit about which you set aside and why.

### 3.5 Analyst frames as sources and as evidence-of-absence

Gartner Magic Quadrant, Forrester Wave, IDC MarketScape, and Everest PEAK Matrix evaluations are themselves the output of public-source comparative analysis at industrial scale. Coverage by an analyst is high-value evidence: an entry in the most recent Forrester Wave: Conversational AI Platforms means a vendor met the analyst's inclusion criteria (revenue floor, customer count, product completeness). Absence is also evidence. A vendor that has never appeared in a relevant analyst frame is either too small, too new, too narrow, or a deliberate non-participant. Each of those is a different finding. Do not assume "absent from quadrant" means "bad product" — many excellent niche platforms intentionally skip analyst submission cycles.

## 4. Generic Implementation

This section walks through building a public-source comparative matrix for two **generic** enterprise AI platforms — call them *NorthStar AI* (a hypothetical horizontal e-commerce-search platform) and *RiverFlow* (a hypothetical vertical AI platform for clinical documentation). The mechanics translate directly to any pair of AI platforms in any industry.

**Step 1 — Frame the buyer.** Before opening a browser tab, write down the client's situation in two sentences. *"A 4,000-bed hospital network is choosing an AI documentation assistant. Buyer is the CMIO; the budget is a one-year pilot."* That framing tells you which dimensions matter (clinical accuracy, EHR integration, HIPAA posture) and which do not (general developer-platform extensibility).

**Step 2 — Identify the source set.** For each vendor, list every public source you can find before reading any of them. A typical set:

```
NorthStar AI:
  - northstar.ai (marketing site)
  - AWS Marketplace listing (if present)
  - LinkedIn company page (employee count proxy)
  - latest Forrester Wave: Commerce AI (2026)
  - founder podcast on Acquired
  - re:Invent talk 2025
  - G2 reviews (50+)

RiverFlow:
  - riverflowhealth.com
  - HIMSS conference talk 2026
  - JAMA case study (peer-reviewed)
  - KLAS Research report citation
  - HIPAA attestation page
  - Job postings (Workday + LinkedIn)
```

**Step 3 — Tag every source.** Decide its tier (high / moderate / flagged) and its retrieval date. Save the URL with a timestamp. A spreadsheet column for `source_tier` and `retrieved_on` becomes the audit trail.

**Step 4 — Score the matrix.** For each dimension, write a *cell* of the form:

```
Dimension: Regulatory posture
NorthStar: SOC 2 Type II claimed on website [src 1, retrieved 2026-05-20].
           No HIPAA attestation found. No HITRUST. Absence flagged.
RiverFlow: HIPAA BAA available; HITRUST CSF v11 certified per attestation
           page [src 6, retrieved 2026-05-20]; SOC 2 Type II [src 7].
Comparative note: RiverFlow is the only viable choice for the hospital
buyer on regulatory grounds alone; the analysis can stop here unless
NorthStar produces HIPAA evidence in a discovery call.
```

Notice three things: (a) every factual claim has a source citation, (b) absence is documented explicitly, (c) the analyst's interpretation is separated from the evidence — the buyer can disagree with the interpretation without disagreeing with the facts.

**Step 5 — Surface the unknowns.** End the matrix with a "questions to ask in discovery" list. *"Does NorthStar have any HIPAA-capable deployment topology under NDA? What is RiverFlow's exit cost if the hospital decides not to renew at year-2?"* These are the questions a CIO will ask the vendors directly; surfacing them is part of the analyst's value.

**Step 6 — Write the one-sentence finding.** Not "RiverFlow wins" — but: *"For this CMIO with this budget on this timeline, the regulatory posture asymmetry makes RiverFlow the default. The only condition that flips the recommendation is if NorthStar produces a HIPAA-attested deployment in discovery."* Conditional findings travel better than verdicts.

## 5. Real-world Patterns

**Healthcare — AI clinical documentation (Nuance DAX vs Abridge vs Suki).** When the U.S. health systems began evaluating AI scribe vendors in 2024–2026, the most-cited buyer reports used a public-source matrix with HIPAA posture, EHR integration depth (Epic vs Cerner vs MEDITECH), per-clinician pricing transparency, and accuracy benchmarks against in-house transcription. Buyers who started with feature-count comparisons consistently ended up rebuilding the matrix around regulatory posture and EHR fit — the lesson being that *dimension selection* dominates the analysis [2,3].

**Fintech — fraud-detection platforms (Sift vs Featurespace vs DataVisor).** Fintech CISOs evaluating fraud-detection AI typically anchor the matrix on false-positive rate, integration with the customer's identity-graph data, and model-explainability for regulator audits. The pattern of "vertical wins on accuracy, horizontal wins on integration breadth" is visible in every published case study [1,3]. Sift wins on multi-product breadth; Featurespace wins on banking-specific accuracy; neither buyer is wrong — the question is which dimension dominates the client's risk model.

**E-commerce — search and personalisation (Algolia vs Bloomreach vs Coveo).** A widely cited pattern in commerce-AI evaluation is that buyers initially compare on relevance benchmarks (a moderate-value source — vendor-supplied benchmarks against vendor-chosen corpora) and only later realise that the deciding dimension was *merchandiser workflow* (how does a non-technical marketer tune the search experience?). The lesson translates directly: vendor-supplied accuracy benchmarks should be flagged as moderate-value sources, never treated as ground truth [2].

**Logistics — last-mile-routing AI (Bringg vs DispatchTrack vs Onfleet).** Public-source competitive analyses of logistics AI repeatedly find that the "platform" framing masks fundamentally different products — one is a SaaS dashboard, one is an API-first developer tool, one is a managed-service offering. Discovering that mid-analysis (rather than mid-pilot) is the value of a 1-day matrix-build exercise.

## 6. Best Practices

- **Cite every factual claim with a URL and retrieval date** — if you cannot, soften the language ("appears to support" rather than "supports") or move the claim to a discovery-questions list.
- **Document absence as explicitly as presence** — "No FedRAMP attestation found on public site as of 2026-05-26" is a finding, not a gap in the matrix.
- **Separate vendor claims from analyst interpretation** — a dimension cell has two parts: what the vendor says, and what you conclude. Conflating them is the most common analyst error.
- **Choose dimensions for the buyer in the room, not for completeness** — eight well-chosen dimensions beat twelve generic ones; surface what you excluded and why.
- **Treat analyst-quadrant absence as a question, not a verdict** — many strong niche products are absent from Gartner/Forrester coverage by choice or by inclusion-criteria thresholds.
- **End every analysis with a conditional finding** — "X is the default unless Y" travels better than "X wins" and resists the most common rebuttal ("but what about Y").
- **Refuse to claim certainty on internals you cannot see** — "Modesty in claims" is itself the federal-engagement lesson; over-confident claims about an opaque competitor are how analysts lose credibility.

## 7. Hands-on Exercise

**(Whiteboarding, 12 minutes.)** Pick any two enterprise software products you have never used — example pairs: Snowflake vs Databricks, Datadog vs New Relic, Notion vs Confluence. Without opening either vendor's website, draft an 8-dimension comparative matrix using only the dimensions you would care about as a buyer. Then open each vendor's homepage for *2 minutes only* and fill in what you can confirm. For every cell you cannot fill, write a one-line discovery question. Time-box the whole exercise to 12 minutes.

**What good looks like.** A finished sketch has: (1) eight dimensions chosen with one-line justifications, (2) most cells either filled-with-citation or flagged-as-unknown, (3) a discovery-questions list of 5–8 items, (4) a one-sentence conditional finding. A common failure mode is filling in cells from memory — every cell that lacks a URL is a cell you are guessing on. Mark those explicitly.

## 8. Key Takeaways

- Can you describe the four-stage CI process and why every claim must ship with a citation and a retrieval date?
- Can you explain why the horizontal-vs-vertical platform distinction changes the *dimensions* you compare on, not just the *vendors* in scope?
- Can you build an 8-dimension comparative matrix for an enterprise AI platform pair you have never used, anchored entirely in public sources?
- Can you articulate why "absence of evidence" is a finding rather than a gap, and how to communicate it to a CIO without overstating?
- Can you write a one-sentence conditional finding (the form "X is the default unless Y") for a comparative analysis?

## Sources

1. [Should SaaS Vendors Prioritize Verticalized or Horizontal AI? — Futurum Group](https://futurumgroup.com/press-release/should-saas-vendors-prioritize-ai-for-vertical-or-horizontal-use-cases/) — retrieved 2026-05-26
2. [Vertical vs horizontal AI: Which platform fits your needs? — Glean](https://www.glean.com/perspectives/horizontal-ai-platform) — retrieved 2026-05-26
3. [The Next Wave of Enterprise AI: From Horizontal Models to Vertical Solutions — Lightspeed Venture Partners](https://lsvp.com/stories/the-next-wave-of-enterprise-ai-from-horizontal-models-to-vertical-solutions/) — retrieved 2026-05-26
4. [Competitive intelligence — Wikipedia](https://en.wikipedia.org/wiki/Competitive_intelligence) — retrieved 2026-05-26
5. [Competitive Intelligence: Tools and Techniques — Penn State Library Guides](https://guides.libraries.psu.edu/berks/CI) — retrieved 2026-05-26

Last verified: 2026-05-26
