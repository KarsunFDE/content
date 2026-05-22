---
title: GalentAI vs Karsun ReDuX — Comparative Research Brief
purpose: Source material for W1 Wed 27 May 2026 — 3hr instructor-led comparative session (Karsun-FDE Cohort #1)
last_verified: 2026-05-22
researcher: claude-opus-4-7
tech: [galentai, karsun-redux, aws-bedrock, langgraph, fedramp, govcloud, cdao-tradewinds]
fde_situations: [federal-modernization, ai-platform-selection, brownfield-modernization]
recency_window: federal-regulatory-6mo, hot-tech-3mo
---

# GalentAI vs Karsun ReDuX — Deep-Dive Brief for Cohort Comparative Session

**Audience:** Karsun-FDE Cohort #1 (Wed 27 May 2026, 3hr instructor-led session)
**Goal:** Frame "this is the end-goal capability the cohort is building toward" — ground in public, citable claims only.
**Scraping window:** 2026-05-22.
**Constraint:** No model-internal claims; every fact has a URL + scrape date. Marketing voice flagged.

---

## 1. GalentAI deep dive

### One-paragraph summary

Galent (legal entity name "Galent Management Consulting" per Crunchbase) is positioned publicly as an "AI-native digital engineering" services firm whose product layer is the GalentAI Platform — an opinionated execution framework that wraps LLM access behind proprietary engines, a knowledge/context graph layer, and pre-built agents. Its commercial wedge against hyperscaler-native managed-agent runtimes (e.g., Anthropic's Claude Managed Agents, public beta Apr 2026) is the claim that "managed agents optimize for ease of entry but lack enterprise-grade infrastructure for production at scale" — see Galent's May 2026 blog ([source](https://galent.com/insights/blogs/claude-managed-agents-vs-enterprise-ai-platforms/), scraped 2026-05-22). Headquartered in Chennai, India per their July 2025 AIStudio launch ([source](https://galent.com/newsroom/galent-unveils-aistudio-in-chennai-to-accelerate-ai-native-digital-engineering/), scraped 2026-05-22). **No FedRAMP or US federal posture is disclosed publicly** — vertical messaging on galent.com is Banking, Healthcare, Insurance, Communication/Media/Tech.

### Architecture — the engines

Galent's homepage lists **9 proprietary engines** total; the public materials emphasize four prominently ([source](https://galent.com/), scraped 2026-05-22):

| Engine | Claim |
|--------|-------|
| **NeuroQL Engine** | "Patent-pending prompt engine achieving 70% fewer context window failures"; uses a "MicroPrompt architecture and deterministic execution model" |
| **RCM Engine** | Neurosymbolic AI — "fuses neural reasoning with rule enforcement"; "every decision validated, auditable, and compliant" |
| **Knowledge Graph** | "Maps entire systems including code, data, logs, and dependencies before AI execution" |
| **Context Graph** | "Encodes business logic, architecture standards, and compliance rules governing AI actions" |

> **Important reconciliation:** the user-supplied prior-session framing of "four engines = deterministic execution / knowledge integration / agent orchestration / AIOps signal layer" does **not** map cleanly to Galent's currently public taxonomy. The public taxonomy is NeuroQL / RCM / KG / CG. The closest mapping: deterministic execution ≈ NeuroQL, knowledge integration ≈ Knowledge Graph + Context Graph, agent orchestration ≈ RCM, AIOps signal layer ≈ EveryOps (see below — a service offering, not an engine). This is worth surfacing in session — public framing has shifted.

Additional platform features publicly claimed:
- "125+ Reusable Agents across delivery, engineering, operations, and modernization"
- "70+ validated use cases"
- "50+ enterprise projects, 70+ production deployments"
- Platform runs "inside your cloud boundary, routes across any LLM"
- "Flat-priced — engineers alongside your team from pilot through production"
([source](https://galent.com/), scraped 2026-05-22)

### EveryOps positioning

"EveryOps" is a **service offering**, not an engine — sits under Managed Services. Claims ([source](https://galent.com/), scraped 2026-05-22):
- 99.99% uptime
- 70% MTTR reduction
- Bundled with AI-driven ITSM and SRE/Observability

### Federal posture

| Question | Public answer |
|----------|--------------|
| FedRAMP authorization? | **Not disclosed.** No federal-compliance language found across galent.com, the newsroom, or insights pages. |
| GovCloud availability? | **Not disclosed.** |
| AWS Marketplace listing? | **Not found** as of 2026-05-22 scrape. |
| Federal customer references? | **None named publicly.** Galent's Chennai AIStudio press release (24 Jul 2025) names no government clients. |
| Closest "regulated industries" signal | The "Claude Managed Agents vs. Enterprise AI Platforms" blog references PHI/PII/ITAR-controlled content in its diagnostic checklist — positioning toward heavily regulated sectors generally, not US federal specifically ([source](https://galent.com/insights/blogs/claude-managed-agents-vs-enterprise-ai-platforms/), scraped 2026-05-22) |

### Pricing model

Public site claims "flat-priced" engagements ([source](https://galent.com/), scraped 2026-05-22). No tiers, per-seat, or per-call rates are published. The May 2026 "Claude Managed Agents vs. Enterprise AI Platforms" post contrasts managed-agent linear consumption pricing (their example: "20 agents running 8 hours/day = ~$5,300/month") against "predictable, flat-rate structures" Galent positions itself toward — but does not publish actual Galent rate cards ([source](https://galent.com/insights/blogs/claude-managed-agents-vs-enterprise-ai-platforms/), scraped 2026-05-22).

### AIOps / signal layer

The AIOps/observability story is bundled under **EveryOps**. Public claims are operational outcomes (99.99% uptime, 70% MTTR reduction) rather than an architecturally described signal layer. The platform page references "Knowledge Graph maps entire systems including code, data, **logs**, and dependencies" — implying a telemetry-integration capability, but no AIOps platform partnerships (Datadog/Dynatrace/New Relic) are disclosed ([source](https://galent.com/), scraped 2026-05-22).

### Recent news (rolling 6mo from 2026-05-22)

| Date | Headline | Source | Notes |
|------|----------|--------|-------|
| May 2026 (path-dated) | "Claude Managed Agents vs. Enterprise AI Platforms" — competitive-positioning blog | [galent.com](https://galent.com/insights/blogs/claude-managed-agents-vs-enterprise-ai-platforms/) | Positions GalentAI vs Anthropic's Apr 2026 Managed Agents beta |
| 2026 | "Fourthwards" thought-leadership series launched | [galent.com](https://galent.com/newsroom/galent-launches-fourthwards-series/) | AI's role in elevating GCC (Global Capability Center) value |
| 2026 | Jina Priya appointed Global Head of Services | galent.com/newsroom | Leadership build-out |
| 2026 | Vijayakumar Anbu named Chief AI Architect | galent.com/newsroom | Engineering build-out |
| 2026 | Ramakrishnan Venkatasubramanian named CTO | galent.com/newsroom | C-suite build-out |
| 24 Jul 2025 | Galent AIStudio Chennai opens | [galent.com](https://galent.com/newsroom/galent-unveils-aistudio-in-chennai-to-accelerate-ai-native-digital-engineering/) | Delivery-hub launch; no federal customers named |

**Notable absence:** no funding rounds, no named customer logos, no analyst reports (Gartner/Forrester) found publicly for Galent.

### Public reviews summary

Searched G2, Gartner Peer Insights, Forrester (2026-05-22): **no Galent-specific customer reviews found**. One Fortune-500-telecom CTO testimonial appears on galent.com itself (no name attached). Crunchbase profile exists ([source](https://www.crunchbase.com/organization/galent-management-consulting), 403 on direct fetch 2026-05-22) but funding/employee detail not extractable without paid access. ZoomInfo and TheOrg list Galent but neither provides analyst-grade review content.

This is itself a finding: Galent is a young-ish (founded ~mid-2010s based on LinkedIn/Crunchbase listings), services-first firm wrapping a product layer — not a Gartner-tracked AI platform vendor in the same category as Glean, Cohere, or Anthropic.

### 10+ specific facts with URLs + dates

1. NeuroQL Engine claims 70% fewer context window failures — [galent.com](https://galent.com/), 2026-05-22
2. RCM Engine uses neurosymbolic AI (neural + rule enforcement) — [galent.com](https://galent.com/), 2026-05-22
3. Platform claims "9 Core Engines" + "125+ Reusable Agents" — [galent.com](https://galent.com/), 2026-05-22
4. EveryOps service line claims 99.99% uptime and 70% MTTR reduction — [galent.com](https://galent.com/), 2026-05-22
5. Platform claims "4X to 10X faster time to value" — [galent.com](https://galent.com/), 2026-05-22
6. Platform claims "Lower TCO by 25% and free 20% of engineering capacity" — [galent.com](https://galent.com/), 2026-05-22
7. Platform claims "Cut PoC-to-Production failure rate by 85-90%" — [galent.com](https://galent.com/), 2026-05-22
8. Verticals listed: Banking & Finance, Healthcare & Lifesciences, Insurance, Communication/Media/Tech (no Government/Public Sector) — [galent.com](https://galent.com/), 2026-05-22
9. Galent's May 2026 blog frames managed-agent runtimes as "infrastructure streamlined for convenience — but not designed for control" — [galent.com](https://galent.com/insights/blogs/claude-managed-agents-vs-enterprise-ai-platforms/), 2026-05-22
10. Galent recommends "Build your core on enterprise AI platforms. Use managed agents to accelerate at the edges." — [galent.com](https://galent.com/insights/blogs/claude-managed-agents-vs-enterprise-ai-platforms/), 2026-05-22
11. Galent uses "Second-Gen AI" and "Reverse Conway's Maneuver" as positioning language — [galent.com Chennai press release](https://galent.com/newsroom/galent-unveils-aistudio-in-chennai-to-accelerate-ai-native-digital-engineering/), 2025-07-24
12. Leadership: Ashwin Bharath (CEO), Shankar Iyer (President & COO), Sriram Rajagopal (Founder), Ramakrishnan Venkatasubramanian (CTO) — [galent.com](https://galent.com/), 2026-05-22
13. Platform claims "runs inside your cloud boundary, routes across any LLM" — [galent.com](https://galent.com/), 2026-05-22
14. 60% faster app development claim under App Dev & Modernization service line — [galent.com](https://galent.com/), 2026-05-22

---

## 2. Karsun ReDuX deep dive

### One-paragraph summary

Karsun Solutions (Herndon, VA — founded 2009, ~800 employees per AWS Partner case study) is a federal-modernization services firm whose product is **ReDuX** ("REfactor-driven Digital transformation of User Experience"), an agentic-AI modernization platform built on AWS Bedrock. ReDuX is **not a horizontal enterprise-AI platform** like GalentAI — it is a vertical, opinionated COBOL/legacy-mainframe-modernization workflow that uses Bedrock-hosted Claude models, knowledge graphs, and RAG-grounded code generation to drive the discovery → blueprinting → transformation → verification loop. ReDuX shipped on AWS Marketplace in July 2025, achieved AWS Mainframe Modernization Competency, and was assessed "Awardable" by the DoW CDAO Tradewinds Solutions Marketplace in October 2025 — a load-bearing federal-defense procurement signal. Origin story is grounded in Karsun's GSA and FAA modernization work.

### Capabilities + architecture

From the canonical AWS Public Sector Blog post (published 19 Mar 2024, [source](https://aws.amazon.com/blogs/publicsector/karsun-solutions-builds-modernization-platform-using-amazon-bedrock/), scraped 2026-05-22):

**Three-agent core** (the public framing has tightened over time — earlier materials show more, the AWS Marketplace listing lands on three named agents):

1. **Blueprint Agent** — discovery: analyzes COBOL/Assembly/Natural/ADABAS code; constructs domain knowledge graphs (user flows, APIs, data structures, jobs); includes a RAG-grounded chatbot for business-rule exploration with hallucination controls
2. **Modernization Agents** (a.k.a. AppPilot / Transformation Agent) — generates production-grade code from Blueprint outputs; supports incremental migration; flexible foundation-model switching for cost optimization
3. **Verifier Agent** — auto-generates test scripts, organizes data into user stories, builds data-sync/ETL pipelines and migration workflows

**Infrastructure stack (AWS-only, per public AWS Public Sector Blog):**
- Amazon EKS (Kubernetes) for the runtime
- Amazon Bedrock for foundation-model access — Claude 3.5 Sonnet named as a default in the AWS Marketplace listing
- AWS PrivateLink to keep Bedrock traffic off the public internet (this is the data-privacy claim)
- Amazon Aurora PostgreSQL (relational)
- Amazon Neptune (knowledge graph)
- Amazon S3 (storage)

**Visualization + auxiliary:** C4 architecture diagrams for architects; ETL pipeline generation for data engineers; automated test scripts ([source](https://aws.amazon.com/blogs/publicsector/karsun-solutions-builds-modernization-platform-using-amazon-bedrock/), scraped 2026-05-22).

### Federal posture

| Question | Public answer |
|----------|--------------|
| FedRAMP authorization? | **No FedRAMP ATO disclosed.** The AWS Marketplace listing does not claim FedRAMP. ReDuX's defensible federal claim is **CDAO Tradewinds Awardable status (Oct 2025)** + AWS Government/Migration/Modernization/DevOps Competencies + CMMI v2.0 Level 5 — not a FedRAMP authorization. |
| GovCloud availability? | **Not specified** on either Marketplace listing scraped (prodview-ovm4yupjljktk and prodview-ewysxj3be4vra). |
| AWS Marketplace listing? | **Yes — two listings.** ([Self-hosted/Cloud-Tenant SaaS](https://aws.amazon.com/marketplace/pp/prodview-ewysxj3be4vra) — $1,500/mo self-hosted, $4,000/mo cloud-tenant, $0.09/Input Unit, $0.05/Output Unit; and [Professional services](https://aws.amazon.com/marketplace/pp/prodview-ovm4yupjljktk) — custom pricing). Listed July 30, 2025. |
| ATO precedent | **CDAO Tradewinds Awardable status, October 2025** — Department of War's (DoW) procurement vehicle for AI/ML/data capabilities. Karsun was assessed Awardable via competitive scoring rubric ([source](https://goredux.ai/karsun-redux-assessed-awardable-for-department-of-war-work-in-the-cdaos-tradewinds-solutions-marketplace/), [Karsun mirror](https://karsun-llc.com/news/karsun-redux-assessed-awardable-for-department-of-war-work-in-the-cdaos-tradewinds-solutions-marketplace/), 2026-05-22). |
| Federal customers named | **GSA and FAA** named as origin-of-platform customers (ReDuX "emerged from this experience" / "developed to tackle massive modernization challenges at the GSA and FAA") — multiple sources. **No specific customer named in the AWS Public Sector Blog post** itself. Karsun's $145M FAA AIT IDIQ (Oct 2017) and $110M GSA CAMEO contract are the institutional anchors. |

### Pricing model (disclosed)

ReDuX has two AWS Marketplace listings — one is unusually transparent for a federal-targeting platform:

| SKU | Monthly | Per-Input | Per-Output | 12mo savings |
|-----|---------|-----------|------------|--------------|
| Self-Hosted | $1,500 | $0.09/Input Unit | $0.05/Output Unit | up to 17% |
| Cloud Tenant | $4,000 | $0.09/Input Unit | $0.05/Output Unit | up to 17% |
| Professional services | Custom (private offer) | — | — | — |

Source: [AWS Marketplace ReDuX listing](https://aws.amazon.com/marketplace/pp/prodview-ewysxj3be4vra), scraped 2026-05-22.

### Customer wins + case studies

| Date | Customer / Story | Source |
|------|------------------|--------|
| Origin | GSA + FAA — ReDuX "developed to tackle massive modernization challenges at the GSA and FAA" — [washingtonexec / multiple secondary sources], 2026-05-22 |
| 2017 (anchor contract) | FAA AIT (Office of IT) — $145M 5-year IDIQ — [Karsun press release](https://karsun-llc.com/press-release/karsun-solutions-awarded-145-million-contract-with-federal-aviation-administration/), 2026-05-22 |
| Ongoing | DHS Balanced Workforce Strategy modernization — search-surfaced, not deep-cited |
| Ongoing | GSA CAMEO Small Business contract — $110M ceiling; acquisition planning, solicitation writing, contract award/admin functions |
| Apr 2025 | Unnamed government mainframe-to-microservices case study — [Karsun blog](https://karsun-llc.com/blog/ai-accelerated-mainframe-migration-from-karsun-on-the-aws-public-sector-blog/), 2026-05-22 |
| Jul 2025 | ReDuX on AWS Marketplace — [Karsun press release](https://karsun-llc.com/news/karsun-redux-now-available-on-aws-marketplace/), 2026-05-22 |
| Oct 2025 | ReDuX Awardable on CDAO Tradewinds — [Karsun press release](https://karsun-llc.com/news/karsun-redux-assessed-awardable-for-department-of-war-work-in-the-cdaos-tradewinds-solutions-marketplace/), 2026-05-22 |

### Karsun corporate context

- **Founded** 2009, **HQ** Herndon, VA ([AWS Partner Success page](https://aws.amazon.com/partners/success/karsun-solutions/), 2026-05-22)
- **Headcount** ~800 staff + contractors (AWS Partner Success)
- **AWS Partnership** — held: AWS Government, Migration, Modernization, DevOps competencies; AWS Well-Architected Partner; achieved **AWS Mainframe Modernization Competency** ([source](https://karsun-llc.com/), 2026-05-22). Tier: Advanced Consulting Partner per WebSearch summary 2026-05-22.
- **Quality certifications** — **CMMI v2.0 Level 5 (DEV)**, third reappraisal ([source](https://orangeslices.ai/karsun-solutions-receives-cmmi-level-5-dev-maturity-appraisal-for-the-third-time/), referenced 2026-05-22)
- **Analyst recognition** — 2026 ISG Provider Lens® Mainframes Services and Solutions: **Contender** in U.S. Public Sector Application Modernization Services quadrant; **Product Challenger** in Global Mainframe Application Modernization Software quadrant ([source](https://www.businesswire.com/news/home/20260414955677/en/U.S.-Public-Sector-Applies-AI-to-Mainframe-Modernization), 2026-05-22)
- **Other recognition** — Named to 2026 AI50 list by Northern Virginia Technology Council ([karsun-llc.com](https://karsun-llc.com/), 2026-05-22)
- **Reseller channel** — Available through Carahsoft ([source](https://www.carahsoft.com/karsun-solutions), 2026-05-22)
- **Leadership** — Sundar Vaidyanathan (CEO, founder), Badri Sriraman (SVP Karsun Innovation Center — owns ReDuX) ([karsun-llc.com leadership pages](https://karsun-llc.com/leadership/badri-sriraman/), 2026-05-22)

### Recent news (rolling 6mo from 2026-05-22)

| Date | Headline | Source |
|------|----------|--------|
| Apr 2026 | ISG Provider Lens 2026 — Karsun named Contender (US Public Sector) + Product Challenger (Global Software) for mainframe modernization | [businesswire 14 Apr 2026](https://www.businesswire.com/news/home/20260414955677/en/U.S.-Public-Sector-Applies-AI-to-Mainframe-Modernization) |
| Jan 2026 | CEO Sundar Vaidyanathan profile on shaping next phase of federal IT (note: 403 on direct fetch) | [WashingtonExec](https://washingtonexec.com/2026/01/how-karsun-solutions-sundar-vaidyanathan-is-shaping-the-next-phase-of-federal-it/) |
| Oct 2025 | ReDuX assessed Awardable for DoW work in CDAO Tradewinds Solutions Marketplace | [goredux.ai](https://goredux.ai/karsun-redux-assessed-awardable-for-department-of-war-work-in-the-cdaos-tradewinds-solutions-marketplace/) |
| Jul 2025 | ReDuX available on AWS Marketplace (announced 30 Jul 2025) | [karsun-llc.com](https://karsun-llc.com/news/karsun-redux-now-available-on-aws-marketplace/) |
| Apr 2025 | "AI Accelerated Mainframe Migration from Karsun on the AWS Public Sector Blog" — Karsun mirror post (1 Apr 2025) | [karsun-llc.com](https://karsun-llc.com/blog/ai-accelerated-mainframe-migration-from-karsun-on-the-aws-public-sector-blog/) |

### 10+ specific facts with URLs + dates

1. ReDuX runs on AWS Bedrock with foundation-model traffic via AWS PrivateLink — [AWS PS Blog 19 Mar 2024](https://aws.amazon.com/blogs/publicsector/karsun-solutions-builds-modernization-platform-using-amazon-bedrock/), 2026-05-22
2. Knowledge graph uses Amazon Neptune; relational data in Aurora PostgreSQL; objects in S3; runtime on EKS — same source
3. ReDuX claims **4x faster project delivery, 50% fewer resources, 22.5% productivity increase, 2x code quality improvement** (vs prior baseline) — [AWS Partner Success](https://aws.amazon.com/partners/success/karsun-solutions/), 2026-05-22
4. Carahsoft variant: "transforms legacy applications 4x faster with 2x less rework" — [carahsoft.com](https://www.carahsoft.com/karsun-solutions), 2026-05-22
5. Blueprint Agent uses Bedrock + Claude 3.5 Sonnet for system analysis — [AWS Marketplace listing](https://aws.amazon.com/marketplace/pp/prodview-ewysxj3be4vra), 2026-05-22
6. Self-hosted ReDuX = $1,500/mo + $0.09/Input Unit + $0.05/Output Unit; Cloud Tenant = $4,000/mo + same per-unit — same source
7. CDAO Tradewinds Awardable status (DoW) — Oct 2025 — [karsun-llc.com](https://karsun-llc.com/news/karsun-redux-assessed-awardable-for-department-of-war-work-in-the-cdaos-tradewinds-solutions-marketplace/), 2026-05-22
8. CEO Sundar Vaidyanathan quote (Jul 2025): "We are thrilled to see this platform developed in-house by our Karsun Innovation Center now available to the commercial market through the AWS Marketplace" — [Karsun press release](https://karsun-llc.com/news/karsun-redux-now-available-on-aws-marketplace/), 2026-05-22
9. SVP Badri Sriraman quote (Jul 2025): "Acquiring ReDuX via the AWS Marketplace is ideal for public sector teams looking to quickly access products that will accelerate the modernization of legacy systems" — same source
10. ReDuX positioned for COBOL, Assembly, Natural, ADABAS — same source
11. ISG 2026 Provider Lens: Contender (US PubSec) + Product Challenger (Global Software) — [businesswire](https://www.businesswire.com/news/home/20260414955677/en/U.S.-Public-Sector-Applies-AI-to-Mainframe-Modernization), 2026-05-22
12. CMMI v2.0 Level 5 (DEV), third appraisal — [orangeslices.ai](https://orangeslices.ai/karsun-solutions-receives-cmmi-level-5-dev-maturity-appraisal-for-the-third-time/), 2026-05-22
13. Karsun anchor federal contracts: FAA AIT $145M IDIQ (2017), GSA CAMEO $110M (acquisitions) — [Karsun FAA press release](https://karsun-llc.com/press-release/karsun-solutions-awarded-145-million-contract-with-federal-aviation-administration/), 2026-05-22
14. AWS Mainframe Modernization Competency achieved — [karsun-llc.com homepage](https://karsun-llc.com/), 2026-05-22

---

## 3. Comparative matrix (publicly grounded)

| Dimension | **GalentAI** | **Karsun ReDuX** |
|-----------|--------------|------------------|
| **Capability scope** | Horizontal enterprise-AI platform: 9 engines, 125+ reusable agents across delivery/engineering/ops/modernization ([galent.com](https://galent.com/), 2026-05-22) | Vertical: legacy COBOL/mainframe → cloud-native microservices via 3-agent loop (Blueprint / Modernization / Verifier) ([AWS PS Blog](https://aws.amazon.com/blogs/publicsector/karsun-solutions-builds-modernization-platform-using-amazon-bedrock/), 2026-05-22) |
| **Target customer** | Banking, Healthcare, Insurance, Comms/Media/Tech — large-enterprise; "Global Capability Centers" framing ([galent.com](https://galent.com/), 2026-05-22) | US federal agencies; mainframe-modernization-led ([Karsun homepage](https://karsun-llc.com/), 2026-05-22) |
| **Deployment model** | "Runs inside your cloud boundary, routes across any LLM" — implies private-VPC / BYO-cloud ([galent.com](https://galent.com/), 2026-05-22) | AWS Marketplace SaaS — Self-Hosted ($1,500/mo) or Cloud Tenant ($4,000/mo) ([AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-ewysxj3be4vra), 2026-05-22) |
| **FedRAMP / federal ATO** | **None disclosed publicly** | **No FedRAMP**; CDAO Tradewinds Awardable (Oct 2025); AWS Government/Migration/Modernization/DevOps competencies; CMMI L5 — see source links above |
| **LLM provider** | "Routes across any LLM" — multi-provider ([galent.com](https://galent.com/), 2026-05-22) | Bedrock-only; Claude 3.5 Sonnet referenced; "flexible foundation model switching" within Bedrock ([AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-ewysxj3be4vra), 2026-05-22) |
| **Federal fit (GovCloud / ATO precedent / pricing)** | No GovCloud, no ATO, no federal pricing posture publicly | CDAO Tradewinds Awardable; GSA/FAA origin; transparent per-Input/per-Output pricing on AWS Marketplace |
| **Exit cost / lock-in** | Multi-LLM and private-cloud posture *reduces* lock-in; flat pricing claim implies predictable exit ([galent blog](https://galent.com/insights/blogs/claude-managed-agents-vs-enterprise-ai-platforms/), 2026-05-22) | Bedrock-coupled architecture (EKS, Aurora, Neptune, S3, PrivateLink) implies AWS-stack lock-in ([AWS PS Blog](https://aws.amazon.com/blogs/publicsector/karsun-solutions-builds-modernization-platform-using-amazon-bedrock/), 2026-05-22) |
| **Public strengths claimed** | 4-10x faster time to value; 25% TCO reduction; 85-90% PoC-to-prod failure reduction; deterministic execution via NeuroQL + neurosymbolic reasoning ([galent.com](https://galent.com/), 2026-05-22) | 4x faster delivery, 50% fewer resources, 22.5% productivity, 2x code quality ([AWS Partner Success](https://aws.amazon.com/partners/success/karsun-solutions/), 2026-05-22) |
| **Gaps / risks (public)** | No federal posture; no Gartner/Forrester coverage; no named customer logos; "Fortune 500 Telecom" testimonial is anonymous; product-vs-services boundary blurry | No FedRAMP authorization (only Awardable status); ISG ranks as "Contender" not "Leader"; Bedrock-only coupling; per-unit input/output pricing can drift with usage |
| **Analyst recognition** | None found (G2, Gartner, Forrester) as of 2026-05-22 | ISG Provider Lens 2026 — Contender (PubSec) + Product Challenger (Global Software) |
| **Sales motion** | Direct + services-led; flat-priced engagements | Direct AWS Marketplace + Carahsoft + CDAO Tradewinds + competitive contracts |

---

## 4. "End-goal and capabilities" framing for the cohort

A Karsun-FDE engineer working on a federal-acquisitions Pair Project for this cohort is building a Phase-2 modernization deliverable that — if it shipped through Karsun's actual production pipeline — would end up as a **ReDuX-orchestrated workflow** with these characteristics:

1. **Discovery is an agent, not a person-month.** What the cohort does manually in W1 Tue brownfield-debt inventory, ReDuX's Blueprint Agent does at scale: ingest COBOL/Java/Angular code, populate a Neptune knowledge graph of "user flows, APIs, data structures, jobs," and expose a RAG-grounded chatbot for business-rule Q&A ([AWS PS Blog](https://aws.amazon.com/blogs/publicsector/karsun-solutions-builds-modernization-platform-using-amazon-bedrock/), 2026-05-22). The cohort's W3 Knowledge Graph + Context Graph work in Phase 1 is literally building the muscle ReDuX automates.

2. **Code generation is grounded.** The Modernization Agent uses RAG against reference implementations to mitigate hallucination — Karsun's documented control. The cohort's W2 RAG week and W4 AI-security week (OWASP LLM Top 10) directly map to "what controls do you put around a code-generating agent before it ships against federal workloads?"

3. **The whole pipeline routes through Bedrock + PrivateLink.** Karsun's federal-defensibility claim is data-handling: no foundation-model training exposure, all Bedrock traffic via AWS PrivateLink. The cohort's training-project uses Bedrock for the same reason — this is the real-world deployment topology, not an academic choice.

4. **Verification is an agent.** ReDuX's Verifier Agent auto-generates test scripts and data-migration workflows. The cohort's W5 AIOps + W6 client-deliverability work (runbook, ADR catalog, eval report, handoff) builds the surrounding human-readable artifacts that a Verifier-agent loop *cannot* generate but a senior FDE owns end-to-end.

5. **The federal procurement reality has its own loop.** ReDuX's path-to-federal-deployment goes through CDAO Tradewinds Awardable status (Oct 2025), AWS Marketplace listing (Jul 2025), Carahsoft reseller channel, and AWS Government/Migration/Modernization competencies — *not* through FedRAMP authorization. The cohort needs to know: in the current federal-AI procurement environment, an Awardable assessment via Tradewinds + AWS competency stack is a legitimate alternate path to "ATO-equivalent" credibility for early-stage AI platforms. FedRAMP is the gold standard; CDAO Tradewinds is the express lane.

The frame for the cohort: **"You are not building a from-scratch federal-AI platform. You are building the thing that, if validated, becomes a ReDuX accelerator on your pair's specific federal-acquisitions aspect."**

---

## 5. Discussion-prompt seeds for the 3-hour instructor session

1. **The pricing transparency gap.** Karsun publishes ReDuX self-hosted at $1,500/mo + per-Input/Output Unit on AWS Marketplace ([source](https://aws.amazon.com/marketplace/pp/prodview-ewysxj3be4vra)). Galent says "flat-priced" but publishes no rates ([source](https://galent.com/)). Which is the federal-buyer-friendlier posture, and why? When does opacity hurt vs. help a federal sale?

2. **The "Awardable" question.** ReDuX is CDAO Tradewinds Awardable (Oct 2025) but **not** FedRAMP-authorized. Karsun is currently selling into the DoW (Department of War) via this path. Why does CDAO Tradewinds exist as an alternate procurement path? What does it tell us about how federal AI procurement is changing in 2025-2026?

3. **The "every LLM" claim vs. the Bedrock-only reality.** Galent claims "routes across any LLM" ([source](https://galent.com/)). ReDuX is Bedrock-only with Claude 3.5 Sonnet as default ([source](https://aws.amazon.com/marketplace/pp/prodview-ewysxj3be4vra)). For a federal acquisitions modernization, which architectural choice is more defensible to an ATO reviewer — and why does Karsun's narrower choice *help* federal positioning instead of hurting it?

4. **Knowledge graph as architectural primitive.** Both ReDuX (Neptune) and GalentAI (Knowledge Graph + Context Graph engines) lead with a graph layer ([AWS PS Blog](https://aws.amazon.com/blogs/publicsector/karsun-solutions-builds-modernization-platform-using-amazon-bedrock/), [galent.com](https://galent.com/)). Why does **every** serious enterprise-AI modernization platform converge on a graph layer? What does the graph give you that vector RAG alone cannot?

5. **The hallucination control claim.** Both platforms claim hallucination mitigation. ReDuX uses "RAG against reference implementations" ([AWS PS Blog](https://aws.amazon.com/blogs/publicsector/karsun-solutions-builds-modernization-platform-using-amazon-bedrock/)). GalentAI uses "neurosymbolic RCM Engine — rule enforcement on neural reasoning" ([galent.com](https://galent.com/)). Which is the stronger control for a code-generation agent operating on a regulated federal codebase? Where does each break down?

6. **The Karsun "4x faster, 50% fewer resources, 22.5% productivity" claim.** ([AWS Partner Success](https://aws.amazon.com/partners/success/karsun-solutions/)) These are reported as *outcomes* without a comparator baseline. As a Karsun-FDE engineer, how would you defend or attack these numbers in a customer presentation? What's the missing context in any "Nx faster" modernization claim?

7. **Services-wrapped product vs. product-wrapped services.** Galent leads with services and back-loads the product (GalentAI Platform as enablement layer). Karsun is a services firm whose Innovation Center *spun out* ReDuX as a productized SKU. Both shapes are common in federal modernization. Which delivery shape is more sustainable when your buyer is a federal agency — and what does that imply about the cohort's career arc?

8. **The Tradewinds Marketplace as procurement signal.** CDAO Tradewinds was built explicitly to "accelerate the procurement and adoption of AI/ML capabilities" for DoW ([source](https://karsun-llc.com/news/karsun-redux-assessed-awardable-for-department-of-war-work-in-the-cdaos-tradewinds-solutions-marketplace/)). Awardability is assessed via "competitive scoring rubrics." What does this tell us about how the federal AI-procurement timeline is collapsing? Is FedRAMP becoming a luxury authorization for a slower era?

9. **Galent's "managed agents vs. enterprise platforms" framing — is it a real distinction?** In May 2026 Galent published a counter-positioning blog vs. Anthropic's Claude Managed Agents beta ([source](https://galent.com/insights/blogs/claude-managed-agents-vs-enterprise-ai-platforms/)). Their thesis: managed agents are "infrastructure streamlined for convenience — but not designed for control." Is this a defensible technical claim, or a vendor-defensive move? How would a federal CISO actually decide?

10. **The lock-in trade.** ReDuX's defensibility is *because* it leans into the AWS stack (Bedrock, EKS, Aurora, Neptune, S3, PrivateLink). Galent's defensibility is *because* it abstracts the LLM layer. Both work. When does each strategy break for a federal customer with a 5-10 year horizon?

11. **Where ReDuX stops.** Karsun's three-agent loop ends at Verifier (test + data-sync). It does not include observability, AI-SRE, or auto-remediation. The cohort's W5 AIOps anchor week covers exactly that gap. Is this a missing capability in ReDuX, or a deliberate scope boundary — and where do you draw it on your Pair Project?

12. **The ISG "Contender" rank.** Karsun is named **Contender** (US Public Sector) and **Product Challenger** (Global Software) in ISG Provider Lens 2026 — not **Leader** ([source](https://www.businesswire.com/news/home/20260414955677/en/U.S.-Public-Sector-Applies-AI-to-Mainframe-Modernization)). What does that ranking gap actually mean? What would close it?

13. **The federal acquisitions cohort hook.** Karsun has a $110M GSA CAMEO acquisitions contract — the same domain the cohort's Pair Projects are anchored to. If a cohort engineer's Phase-2 modernization deliverable shipped tomorrow, where in the GSA CAMEO scope would it slot? (And if it doesn't slot — what does that tell you about your Pair's federal-aspect choice?)

14. **The "Fortune 500 Telecom CTO" testimonial gap.** Galent's homepage testimonial is anonymous-Fortune-500. Karsun's customer references are agency-named (GSA, FAA, DHS). What does this difference in disclosure tradition tell us about commercial-AI vs. federal-AI marketing — and which is more useful to the cohort as practitioners?

15. **Hot-tech: Claude Managed Agents (Apr 2026 public beta).** Anthropic shipped a competing layer in the same quarter as this session. Both ReDuX and GalentAI compete with it in different ways. As a federal modernization engineer, how do you decide when to use a managed-agent runtime vs. a platform layer vs. a custom LangGraph build? (This is also the cohort's W3 multi-agent design exercise — bring it back here.)

---

## 6. Source bibliography

### Target 1 — GalentAI

| URL | Content type | Date scraped | Confidence |
|-----|--------------|--------------|------------|
| [galent.com](https://galent.com/) | 1st-party homepage | 2026-05-22 | high (1st-party) |
| [galent.com — Claude Managed Agents blog](https://galent.com/insights/blogs/claude-managed-agents-vs-enterprise-ai-platforms/) | 1st-party blog (positioning) | 2026-05-22 | high (1st-party, but commercial) |
| [galent.com — insights index](https://galent.com/insights/) | 1st-party blog index | 2026-05-22 | high (1st-party) |
| [galent.com — newsroom](https://galent.com/newsroom/) | 1st-party press releases | 2026-05-22 | high (1st-party) |
| [galent.com — AIStudio Chennai](https://galent.com/newsroom/galent-unveils-aistudio-in-chennai-to-accelerate-ai-native-digital-engineering/) | 1st-party press release | 2026-05-22 (post dated 2025-07-24) | high (1st-party) |
| [Crunchbase — Galent Management Consulting](https://www.crunchbase.com/organization/galent-management-consulting) | 3rd-party (403 on fetch) | 2026-05-22 | medium (gated) |

### Target 2 — Karsun ReDuX

| URL | Content type | Date scraped | Confidence |
|-----|--------------|--------------|------------|
| [AWS Public Sector Blog — Karsun ReDuX builds platform on Bedrock](https://aws.amazon.com/blogs/publicsector/karsun-solutions-builds-modernization-platform-using-amazon-bedrock/) | 1st-party (AWS+Karsun co-authored) | 2026-05-22 (post dated 2024-03-19) | high (canonical) |
| [AWS Marketplace — ReDuX SaaS](https://aws.amazon.com/marketplace/pp/prodview-ewysxj3be4vra) | 1st-party (AWS Marketplace listing) | 2026-05-22 | high (commercial filing) |
| [AWS Marketplace — Karsun professional services for ReDuX](https://aws.amazon.com/marketplace/pp/prodview-ovm4yupjljktk) | 1st-party (AWS Marketplace listing) | 2026-05-22 | high (commercial filing) |
| [AWS Partner Success — Karsun Solutions](https://aws.amazon.com/partners/success/karsun-solutions/) | 1st-party (AWS-curated) | 2026-05-22 | high |
| [karsun-llc.com homepage](https://karsun-llc.com/) | 1st-party | 2026-05-22 | high |
| [karsun-llc.com — ReDuX AI innovation page](https://karsun-llc.com/innovation-center/innovation-center-projects/go-redux-ai/) | 1st-party | 2026-05-22 | high |
| [Karsun — ReDuX AWS Marketplace announcement](https://karsun-llc.com/news/karsun-redux-now-available-on-aws-marketplace/) | 1st-party press release (2025-07-30) | 2026-05-22 | high |
| [Karsun — ReDuX Awardable on CDAO Tradewinds](https://karsun-llc.com/news/karsun-redux-assessed-awardable-for-department-of-war-work-in-the-cdaos-tradewinds-solutions-marketplace/) | 1st-party press release (Oct 2025) | 2026-05-22 | high — load-bearing federal claim |
| [goredux.ai — Awardable announcement](https://goredux.ai/karsun-redux-assessed-awardable-for-department-of-war-work-in-the-cdaos-tradewinds-solutions-marketplace/) | 1st-party (ReDuX product site) | 2026-05-22 | high |
| [Karsun — AI mainframe migration blog](https://karsun-llc.com/blog/ai-accelerated-mainframe-migration-from-karsun-on-the-aws-public-sector-blog/) | 1st-party blog (2025-04-01) | 2026-05-22 | high |
| [Karsun — $145M FAA AIT IDIQ press release](https://karsun-llc.com/press-release/karsun-solutions-awarded-145-million-contract-with-federal-aviation-administration/) | 1st-party press release (2017-10-23) | 2026-05-22 | high |
| [Carahsoft — Karsun Solutions](https://www.carahsoft.com/karsun-solutions) | 3rd-party reseller page | 2026-05-22 | medium (commercial summary) |
| [BusinessWire — ISG Provider Lens 2026 mainframes](https://www.businesswire.com/news/home/20260414955677/en/U.S.-Public-Sector-Applies-AI-to-Mainframe-Modernization) | 3rd-party (analyst press release) | 2026-05-22 (dated 2026-04-14) | high (analyst-sourced) |
| [WashingtonExec — Sundar Vaidyanathan profile](https://washingtonexec.com/2026/01/how-karsun-solutions-sundar-vaidyanathan-is-shaping-the-next-phase-of-federal-it/) | 3rd-party (trade press, Jan 2026) | 2026-05-22 (403 on direct fetch — known via search snippets) | medium (search-summarized) |
| [OrangeSlices — CMMI L5 (3rd appraisal)](https://orangeslices.ai/karsun-solutions-receives-cmmi-level-5-dev-maturity-appraisal-for-the-third-time/) | 3rd-party trade press | 2026-05-22 | medium |
| [Karsun — Badri Sriraman leadership page](https://karsun-llc.com/leadership/badri-sriraman/) | 1st-party | 2026-05-22 (via search) | high |

### General federal-context

| URL | Content type | Date scraped | Notes |
|-----|--------------|--------------|-------|
| [FedRAMP.gov](https://www.fedramp.gov/) | 1st-party (US federal) | 2026-05-22 | Background — neither ReDuX nor GalentAI is FedRAMP-listed |
| [DoW CDAO Tradewinds program](https://goredux.ai/karsun-redux-assessed-awardable-for-department-of-war-work-in-the-cdaos-tradewinds-solutions-marketplace/) | Federal procurement context | 2026-05-22 | Alternate path for AI/ML federal procurement |

---

## 7. Research gaps

The cohort should know what we **could not find publicly** as of 2026-05-22 — this is itself instructive about how federal-AI competitive intelligence works in practice.

1. **GalentAI's pricing.** "Flat-priced" is the marketing claim. No tiers, no per-seat, no per-call published. The instructor should acknowledge this and frame: in federal procurement, opacity-of-pricing is often a *blocker* — the buyer needs an aligned-cost projection for budget cycles. Galent's posture suits enterprise SOWs but not federal Schedule sales.

2. **GalentAI's federal posture.** No FedRAMP, no GovCloud, no named federal customer, no Carahsoft listing, no Tradewinds Awardable status. The instructor should be explicit: **Galent does not currently appear positioned for US federal sales.** Whether that is a strategic choice (focus on commercial first) or a gap, we cannot determine from public materials.

3. **GalentAI's actual architecture beyond the four named engines.** The homepage claims 9 engines but only names four prominently. The remaining five are not disclosed publicly. Treat this as a known unknown.

4. **ReDuX's FedRAMP plan.** Karsun does not publicly disclose FedRAMP-in-process status. Whether Karsun intends to pursue FedRAMP authorization for ReDuX, or whether the CDAO Tradewinds path + AWS competency stack is the deliberate strategy, is not publicly answered. The Sundar Vaidyanathan WashingtonExec profile (Jan 2026) might contain this signal but was 403 on direct fetch.

5. **ReDuX customer names beyond GSA/FAA origin.** The AWS Public Sector Blog post deliberately does not name customers. The CDAO Tradewinds awardability is a procurement vehicle, not a customer. Specific deployments at named agencies under named contract numbers are not disclosed publicly.

6. **Both platforms' actual eval results.** Both claim modernization metrics (4x faster, 22.5% productivity, etc.) without published methodology, baselines, or third-party validation. The cohort should treat all of these as marketing-claims-requiring-validation, not benchmarks.

7. **Gartner / Forrester coverage.** Neither GalentAI nor ReDuX has Gartner Magic Quadrant or Forrester Wave placement found in public search as of 2026-05-22. ISG Provider Lens covered ReDuX (Contender + Product Challenger in 2026). No equivalent ranking exists for GalentAI in any analyst report we surfaced.

8. **GalentAI customer count and revenue.** Crunchbase profile exists but 403'd on direct fetch. Public materials claim "50+ enterprise projects" but no revenue, no headcount, no funding history accessible without paid analyst data.

9. **Whether GalentAI is competing in federal-modernization at all.** This is the deepest open question. The May 2026 "Claude Managed Agents" blog uses generic regulated-industries language (PHI/PII/ITAR) — ITAR is the only US-federal-adjacent signal — but no federal-specific positioning. The instructor should treat the comparison as **commercial-enterprise-AI vs. federal-AI-modernization**, not as two head-to-head federal platforms.

10. **Karsun ReDuX integration story with end-to-end DevSecOps tooling.** ReDuX terminates at Verifier Agent; the connection to downstream CI/CD, security-scanning, monitoring/AIOps is not described in public materials. The cohort's W5 AIOps work would build into this gap.

---

**Total: ~4,000 words of cited material. Built strictly from public sources. Last verified 2026-05-22.**
