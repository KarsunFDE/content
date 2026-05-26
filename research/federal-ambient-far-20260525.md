---
topic: Ambient FAR baseline (Parts 7, 11, 12, 15, 19) — acquire-gov training-project regulatory context
domain: federal-acquisitions
last_verified: 2026-05-25
recency_window: federal-regulatory 6mo
source_count: 5
last_verified_via: /web-research (Firecrawl self-hosted)
fde_situations:
  - federal-acquisitions
  - multi-stakeholder
  - long-cycle-contract
  - regulated-data
pair_project: acquire-gov (shared training-project — not pair-specific)
karsun_aspect: federal-acquisitions-baseline
brownfield_anchor: acquire-gov (solicitation + evaluation services)
---

# Ambient FAR baseline — domain brief (acquire-gov context)

> Curriculum-shaped brief for Karsun-FDE Cohort #1. Covers the FAR slice the cohort touches via the shared `acquire-gov` training-project: pre-solicitation (Part 7), requirements definition (Part 11), commercial items (Part 12), negotiated procurement (Part 15), and small-business set-asides (Part 19). Distinct from post-award (`federal-post-award-admin-20260525.md`) — this brief is the **pre-award + source-selection** half.

Last verified: 2026-05-25

---

## 1. Regulatory framework — pre-award + source-selection

| Regulation | Scope | Owner | Karsun framing |
|------------|-------|-------|----------------|
| **FAR Part 7 — Acquisition Planning** | Acquisition plans, requirements consolidation, brand-name justifications, make-or-buy, contractor inventory | FAR Council | Where every contract begins. Acquisition plans drive solicitation content. Brownfield modernization scope often starts here. |
| **FAR Part 11 — Describing Agency Needs** | Performance work statements (PWS), statements of objectives (SOO), statements of work (SOW), specifications, brand-name + technical-data references, market-research-driven specification | FAR Council | Specification-language is where AI most directly augments KO/PCO work. PWS quality drives every downstream artifact. |
| **FAR Part 12 — Acquisition of Commercial Products and Commercial Services** | Streamlined procedures for commercial items + commercial services; expanded post-2017 with NDAA changes | FAR Council | "Default-first" for federal IT modernization — most cloud + SaaS buys live here |
| **FAR Part 15 — Contracting by Negotiation** | Source-selection methods (LPTA, tradeoff, value adjustment), proposal evaluation, discussions, source-selection authority (SSA), debriefings | FAR Council | The full source-selection rulebook. Where evaluation rubrics + proposal scoring live. |
| **FAR Part 19 — Small Business Programs** | Set-asides, 8(a), HUBZone, WOSB, EDWOSB, SDVOSB, NAICS size-standard mapping, subcontracting plans | FAR Council / SBA | Affects who can bid. Set-aside determinations are early-acquisition decisions with downstream effects. |
| **FAR Part 1 — Federal Acquisition System** | Governing principles, definitions, KO warrants + authorities | FAR Council | Reference — defines all the role-terminology used across other parts |
| **FAR Part 6 — Competition Requirements** | Full + open competition default, exceptions thereto, J&A requirements | FAR Council | When NOT competing requires explicit justification |
| **FAR Part 9 — Contractor Qualifications** | Responsibility determinations, debarment + suspension, FAPIIS | FAR Council | "Is the bidder responsible?" — pre-award gate |
| **FAR Part 13 — Simplified Acquisition Procedures (SAP)** | Below SAT ($250K typically) — simplified procedures | FAR Council | Lower threshold; fewer formalities |
| **FAR Part 14 — Sealed Bidding** | IFB process (sealed-bid) | FAR Council | Rare for IT — mostly construction/commodities |
| **FAR Part 16 — Types of Contracts** | FFP, T&M, cost-reimbursement, IDIQ, GWAC, OASIS, etc. | FAR Council | Choice of contract type drives risk allocation + admin complexity |

**Authority hierarchy:** Statute (Competition in Contracting Act, Small Business Act, Truth in Negotiations Act, Federal Acquisition Streamlining Act, etc.) → FAR (government-wide) → DFARS / agency FAR supplements → agency PILs/SOPs → solicitation-specific clauses.

---

## 2. Workflow stages — pre-award acquisition lifecycle

| Stage | When | FAR anchor | What happens | Where AI typically lands |
|-------|------|-----------|--------------|-------------------------|
| 1. **Requirements identification** | Pre-acquisition | Customer + Part 11 | Program office identifies need. Documents in CRD (Capability Requirements Document) or similar. | Need-statement structuring; requirements-traceability |
| 2. **Market research** | Pre-acquisition | Part 10 + 11.103 | Sources sought, RFI, industry days. Documented in market research report. | Industry capability mapping; RFI response analysis |
| 3. **Acquisition planning** | Pre-acquisition | Part 7.1 | Acquisition plan documents strategy: type of contract, set-aside, evaluation approach, milestones | AP-draft generation from need + market research |
| 4. **Requirements documentation** | Pre-solicitation | Part 11 | PWS / SOO / SOW drafted. Specifications written. | Spec-language quality scoring; PWS draft assist; consistency checking across artifacts |
| 5. **Set-aside determination** | Pre-solicitation | Part 19 + 19.502-2 | NAICS code assigned; "rule of two" small-business analysis; set-aside category selected | NAICS suggestion from PWS; rule-of-two market check |
| 6. **Source-selection plan (SSP)** | Pre-solicitation (Part 15 acquisitions) | Part 15.3 | SSP defines evaluation factors + subfactors + relative weighting + source-selection method (LPTA / tradeoff / value-adjusted) | SSP-template generation; factor-weighting consistency checks |
| 7. **Solicitation issuance** | Solicitation phase | Part 15.2 + Part 12 (commercial) | RFP/RFQ posted to SAM.gov contract opportunities | Solicitation-package consistency checking |
| 8. **Industry Q&A** | During solicitation | Solicitation amendments | Offerors ask questions; KO publishes consolidated Q&A | Q&A pattern detection; amendment generation |
| 9. **Proposal receipt** | Closing date | Part 15.207 | Proposals received via SAM.gov / agency portal | Intake routing + completeness pre-check |
| 10. **Initial evaluation** | Post-receipt | Part 15.305 | Source Selection Evaluation Board (SSEB) evaluates per SSP. Strengths/weaknesses/risks/deficiencies. | Strength/weakness extraction; technical-rating consistency |
| 11. **Competitive range determination** | Mid-evaluation | Part 15.306(c) | KO determines which offerors are in competitive range (those most highly rated, may exclude others) | Range-rationale documentation; defensibility checks |
| 12. **Discussions + final proposal revisions (FPR)** | If discussions opened | Part 15.306(d) + 15.307 | Discussions on weaknesses/deficiencies; offerors submit FPRs | Discussion-question generation; FPR-comparison analysis |
| 13. **Final evaluation + SSAC** | Pre-award | Part 15.308 | Source Selection Advisory Council (SSAC) consolidates SSEB findings; SSA makes selection decision | Tradeoff-narrative drafting; selection-rationale evidence |
| 14. **Award + debriefing** | Award | Part 15.503 + 15.506 | Notice to unsuccessful offerors; debriefings for those requesting | Debriefing letter draft; pre/post-award debriefings |
| 15. **Bid protest (if filed)** | Within 10 days of award (typically) | Part 33 + GAO bid-protest regs | Protests filed at GAO or COFC; agency report due in ~30 days | Agency report assembly; protest-merit triage |

**Brownfield concentration for acquire-gov:** Stages 4 (requirements documentation), 6 (SSP), 10 (initial evaluation), 13 (selection rationale). These are where evaluation-services + solicitation-services touch real work.

---

## 3. Key terminology — glossary for cohort

| Term | What it means | Why cohort cares |
|------|---------------|------------------|
| **PWS** (Performance Work Statement) | Outcome-focused statement of work (under Part 11) | Default for performance-based service contracts |
| **SOO** (Statement of Objectives) | High-level objectives statement (offerors propose PWS in response) | Hands solution design to offerors |
| **SOW** (Statement of Work) | Detailed how-to (prescriptive) | More common in supplies/construction than services |
| **CRD** (Capability Requirements Document) | DoD-side requirements document | Common in acquire-gov-relevant scope |
| **AP** (Acquisition Plan) | Required for acquisitions over agency threshold | Part 7 artifact |
| **SAT** (Simplified Acquisition Threshold) | Statutory threshold — currently $250K (some categories higher) | Below SAT = simpler procedures |
| **NAICS** | North American Industry Classification System code | Drives small-business size determination |
| **SBSA** (Small Business Set-Aside) | Set-aside reserved for small business | Default consideration when rule-of-two satisfied |
| **8(a)** | SBA's Business Development program (socially + economically disadvantaged) | Sole-source up to $4.5M; competed up to $7M |
| **HUBZone** | Historically Underutilized Business Zone program | Set-aside or price-evaluation preference (10%) |
| **WOSB / EDWOSB** | Women-Owned / Economically Disadvantaged Women-Owned | Set-aside-eligible in specified NAICS |
| **SDVOSB** | Service-Disabled Veteran-Owned Small Business | Set-aside + sole-source authority |
| **LPTA** | Lowest Price Technically Acceptable | Pure price after technical pass/fail |
| **Tradeoff** | Best-value-tradeoff (price + technical + past performance + ...) | Subjective; most common for services |
| **VAS** (Value Adjusted Score) | Price/technical tradeoff via mathematical formula | Emerging alternative to subjective tradeoff |
| **SSEB / SSAC / SSA** | Source Selection Evaluation Board / Advisory Council / Authority | Tiered review structure |
| **SAM** (System for Award Management) | Entity registration + contract opportunities + small-biz cert | All sources route through |
| **CAGE code** | Commercial and Government Entity code | Unique entity ID (older than UEI but still required) |
| **DUNS → UEI** | Old Dun & Bradstreet number → SAM-issued Unique Entity Identifier (transition completed 2022) | Identity primary key |
| **IDIQ** | Indefinite-Delivery / Indefinite-Quantity | Multi-task-order vehicle |
| **GWAC** | Government-Wide Acquisition Contract | IDIQ usable by any agency |
| **MAS / GSA Schedule** | Multiple Award Schedule | GSA's broad commercial-item vehicle |
| **Past Performance Information** | History + CPARS data | Required factor in tradeoff |
| **J&A** | Justification and Approval (for other-than-full-and-open competition) | Required for sole-source above threshold |
| **OCI** (Organizational Conflict of Interest) | Per FAR 9.5 | Pre-award gate; AI-relevant when bidders also do AI consulting |
| **Subcontracting plan** | Required for awards above threshold; per FAR 19.7 | Small-business utilization commitments |

---

## 4. Common stakeholder roles

| Role | Sits where | Workflow involvement |
|------|------------|---------------------|
| **Program Manager (PM) / Requiring Activity** | Customer organization | Owns the need + funding |
| **Technical SME / Requirements Engineer** | Customer / Program office | Writes PWS / SOW / specifications |
| **Contracting Officer (KO/PCO)** | Contracting office | Owns acquisition strategy + signs contracts |
| **Contract Specialist** | Contracting office | KO's deputy; drafts SOPs |
| **Source Selection Authority (SSA)** | Often senior KO or higher | Makes final award decision |
| **Source Selection Evaluation Board (SSEB)** | Cross-functional (technical + price + past-performance evaluators) | Performs initial proposal evaluation |
| **Source Selection Advisory Council (SSAC)** | Senior evaluators | Reviews SSEB + advises SSA |
| **Small Business Specialist (SBS)** | Each agency required to have | Reviews set-aside determinations |
| **SBA Procurement Center Representative (PCR)** | SBA, embedded at major buying activities | Advocates for small-biz set-asides |
| **Cost / Price Analyst** | Contracting office or DCAA | Price reasonableness + cost realism analyses |
| **Legal counsel** | Office of General Counsel | Reviews high-dollar / complex acquisitions; protest defense |
| **OFPP** | OMB | Government-wide acquisition policy |
| **GAO bid-protest forum** | Legislative branch | Adjudicates protests within 100 days |
| **Court of Federal Claims (COFC)** | Judicial | Alternative protest forum + post-award disputes |

---

## 5. Where AI adoption + brownfield modernization typically lands

### Phase-1 seeds (AI adoption against existing brownfield)

| Seed | Brownfield pain | AI-adoption surface |
|------|----------------|---------------------|
| **PWS quality scoring** | Inconsistent specification quality across acquisitions; downstream rework | Rubric-driven scoring + draft-improvement suggestions |
| **Acquisition plan draft assist** | APs are templated but content is bespoke; multi-week drafts | Generate from need + market research + precedent APs |
| **NAICS code suggestion** | Wrong NAICS leads to wrong set-aside + size standard | Suggest from PWS content; explain rationale |
| **Set-aside rule-of-two analysis** | Manual market check by SBS | Auto-pull SAM small-biz registrants in NAICS + capability |
| **Source selection plan consistency** | Factor weights drift between SSP and solicitation language | Cross-document consistency checking |
| **Industry Q&A pattern detection** | Q&A reveals solicitation ambiguities late | Cluster industry questions to anomaly-flag solicitation language |
| **Proposal evaluation strength/weakness extraction** | Evaluators paraphrase proposal content; consistency drifts | Anchored extraction from proposal text |
| **Tradeoff narrative drafting** | SSA's selection statement is the most-protested artifact | Evidence-anchored draft with mandatory KO edit |
| **Debriefing letter assembly** | Manual today; high time burden | Auto-generate from evaluation findings |
| **Bid-protest precedent retrieval** | GAO decisions database is non-trivial to navigate | RAG over GAO + COFC precedent |

### Phase-2 seeds (modernization driven by Phase-1 discoveries)

| Seed | What modernizes | Why it's brownfield-rich |
|------|----------------|--------------------------|
| **Unified pre-award workspace** | One workspace for PM + KO + SBS + counsel | Today each role works in separate tools |
| **Living acquisition plan** | AP that updates as work progresses; auto-feeds solicitation drafts | AP is one-time artifact today |
| **Evidence-anchored evaluation system** | Every evaluation finding tied to proposal text excerpt | Bid-protest defensibility driver |
| **CPARS-feedback-loop modernization** | Past performance flows seamlessly back into next source selection | Today often forgotten until source selection |
| **Federated small-biz capability search** | SAM + state registries + SBA databases unified | Today scattered |
| **Decision-record preservation layer** | Every SSEB decision archived with full audit trail | Drives protest defense + IG access |

### Patterns to flag (curriculum guardrails)

- **Authority boundaries** — same as post-award brief. Only KOs bind the government. Source-selection decisions belong to SSA. AI never decides; AI proposes.
- **Bid-protest defensibility** — every evaluation finding must trace to proposal text. AI-generated extractions need provenance.
- **OCI screening** — bidders that build the AI tool may have OCI concerns when the AI evaluates their proposals. Cohort architecture-decision-relevant.
- **Apparent acceptance avoidance** — AI-assisted Q&A responses must not commit the government to solicitation interpretations beyond KO-approved language.
- **No proprietary-data leakage** — proposals contain confidential bidder info; AI tools must isolate per-proposal context.

---

## 6. Sources

| Source | URL | Last seen | Authority tier |
|--------|-----|----------|----------------|
| Acquisition.gov — FAR Part 7 (Acquisition Planning) | https://www.acquisition.gov/far/part-7 | 2026-05-25 | T1 — primary regulation |
| Acquisition.gov — FAR Part 11 (Describing Agency Needs) | https://www.acquisition.gov/far/part-11 | 2026-05-25 | T1 — primary regulation |
| Acquisition.gov — FAR Part 12 (Commercial Products and Services) | https://www.acquisition.gov/far/part-12 | 2026-05-25 | T1 — primary regulation |
| Acquisition.gov — FAR Part 15 (Contracting by Negotiation) | https://www.acquisition.gov/far/part-15 | 2026-05-25 | T1 — primary regulation |
| Acquisition.gov — FAR Part 19 (Small Business Programs) | https://www.acquisition.gov/far/part-19 | 2026-05-25 | T1 — primary regulation |

### Recency notes (D-046 compliance)

- FAR amendments via Federal Acquisition Circulars (FACs). Check most recent FAC affecting Parts 7/11/12/15/19 before W2 Mon Plan Day.
- Watch for: 2026 RFO (Revolutionary FAR Overhaul) — restructuring proposals would substantially affect this brief.

---

## 7. Cohort use-points

- **W1 Tue PM:** Instructor walks acquire-gov solicitation + evaluation services. Reference §2 lifecycle stages and §3 terminology.
- **W1 Wed AM:** Pair-claim vote — pairs not claiming grants/post-award/FOIA may anchor their pair-project on a different FAR slice; this brief is their starting point.
- **W2 Thu:** RAG essentials war-room — SAM + USAspending + GAO protest decisions = canonical pre-award RAG corpus.
- **W3 Wed:** Multi-agent handoff — PM → SBS → KO → SSA delegation tree models agent boundaries.
- **W3 Thu:** LangGraph HITL deep-dive — SSA selection-decision gate is the canonical HITL-required step.
- **W4 Wed:** AI Security war-room — OCI scenario for bidders that also build AI tools.
- **W4 Mon Plan Day §0:** Phase-2 modernization seeds drive Phase-2 backlog for acquire-gov-adjacent work.
- **W5 Wed:** AIOps + AI-SRE — protest-rate anomalies + evaluation-cycle-time anomalies.

---

**Author note:** Curriculum-shaped brief, not legal advice. FAR is a living document — verify against acquisition.gov live URLs before any cohort-produced ADR cites specific FAR sections.
