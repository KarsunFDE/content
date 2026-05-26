---
week: W02
day: Mon
phase: Foundation
topic: "RAG architecture & Scenario Design Planning — pre-Plan-Day vocabulary"
estimated_total_minutes: 60
last_verified: 2026-05-26
fde_situations: [1, 3, 6, 9, 11]
tech: [retrieval-augmented-generation, mongodb-atlas-vector-search, langchain-v1, aws-bedrock, embedding-models, adr-discipline]
sources_research_briefs:
  - research/langchain-v1-20260522.md
  - research/mongodb-atlas-vector-search-20260522.md
  - research/bedrock-claude-catalog-20260522.md
  - research/federal-ambient-far-20260525.md
author: instructor
---

# Pre-session reading — RAG architecture & Scenario Design Planning (pre-Plan-Day)

Week 2, Day 1 (Mon). Estimated total time on task: ~60 minutes across 6 topics. Last verified: 2026-05-26.

> Read **before** W2 Mon morning. Mon is the **first formal Plan Day of the programme** (per D-029). You will leave Mon with a graded Scenario Design Planning artifact (six required sections — see §6), not code. This pre-read gives you the vocabulary you'll need before 09:00.
>
> **§0 Plan retrospective is deferred this week.** The §0 retro pattern formally starts W3 Mon per D-036 — W1 Fri was Day 1 of Phase 1, not a multi-day plan-spec worth retro-ing. Do not arrive expecting one; the Plan Day flow opens with the CO call, not with last-week reflection.

## 1. Stakeholder demand framing — what the contracting officer actually asked for (10 min)

W1 Fri ended with the Contracting Officer flagging a hallucinated citation ("48 CFR 47.305-2") that doesn't exist. Mon morning the same CO returns with a sharper question: *"You told me grounding ships this week. I have to walk into the agency CIO's office Wednesday morning to get the budget commit, and she's going to ask me what 'grounding' actually means. OIG just told her any AI-assisted contracting tool that can't produce a citation trail won't pass their next audit cycle. Walk me through what you're building this week — and what it definitively WILL NOT do by Friday. I'd rather under-promise."*

That is the input to your Mon afternoon Scenario Design Planning artifact. Notice what the CO is *not* asking — she is not asking "is RAG cool" or "which vector DB is best." She is asking two operational questions: (a) what does grounding mean as a contractual deliverable, and (b) what's the OIG-defensible citation trail. Translate her ask into the six RAG decision dimensions in topic 2 before 09:00 — that's the framing the war-room expects you to bring.

The federal-acquisitions stake: contracting officers carry personal accountability for the solicitations they sign (FAR Subpart 1.602 — Contracting Officers). If `acquire-gov` drafts a Section M evaluation factor with a wrong clause citation and the CO signs it, the CO owns the audit finding — not the platform. Grounding + citation trail = the platform earning the CO's signature.

## 2. RAG architecture overview — six dimensions to plan against (10 min)

RAG splits the LLM's job into two phases: **retrieve** (pull relevant chunks from an indexed corpus) and **generate** (the LLM composes a response using *those chunks only* as grounding). The system prompt instructs the model to cite only what it sees in the retrieved context and say "I don't know" otherwise. Each architectural piece is a coupled decision — the **six dimensions** you must argue through in this afternoon's artifact:

1. **Corpus shape + chunking** — what is the corpus (FAR + DFARS, ~3,500 pages per D-060), what is a "chunk" (full clause / sub-paragraph / sentence), and are FAR and DFARS in one index or two?
2. **Embedding model** — which model turns chunks into vectors? Bedrock Titan Embeddings v2 is the training-repo default (1024-dim).
3. **Vector store** — Atlas Vector Search is the stack default per D-031 (collocated with the document store). Pinecone shows up in `acquire-gov`'s `requirements.txt` but is dead code (Item 7 — you'll close it this week).
4. **Retrieval mode** — dense only, sparse (BM25) only, or hybrid? Tue exercises all three over the FAR/DFARS corpus.
5. **Citation grounding surface** — what does the JSON response envelope look like so every quote ties back to `clause_id` + `far_part` + `last_revised` + `corpus_source`? This is the OIG citation-trail deliverable.
6. **Evaluation pattern** — how do we know if a RAG change shipped a regression? Flat-file QA harness + LLM-as-judge lands Fri (LangSmith deferred to W5 per D-031).

These six dimensions are **coupled** — changing the chunking strategy (1) changes what the embedding model (2) is asked to represent, which constrains what the retrieval mode (4) can recover, which determines what the citation surface (5) can prove. Mon ADRs commit on three of these (chunking + embedding model + vector store + LangChain v1.0 posture); Tue–Fri force the rest.

## 3. Chunking strategies — pre-vocabulary (10 min)

Just enough taxonomy to vote between strategies in this afternoon's chunking ADR. Detail expansion lives in Tue's pre-session §1.

- **Fixed-size chunking** — split by token count (e.g., 512 tokens with 50-token overlap). Cheap, deterministic, breaks at arbitrary boundaries (mid-clause, mid-sentence). Wrong default for legal text.
- **Semantic chunking** — split where sentence-embedding similarity drops below a threshold. Respects topic boundaries but is non-deterministic and computationally heavy at ingestion.
- **Structural chunking** — split on the corpus's native structure (FAR clauses split at `52.NNN-N` boundaries; sub-clauses at `(a)`, `(a)(1)`, etc.). Highest fidelity for regulated corpora. The natural default for FAR/DFARS.
- **Parent-child chunking** — index at small grain (sub-paragraph) but retrieve the parent (full clause) for LLM context. Best-of-both-worlds for precision-vs-context trade-off. Wed pre-session §1 expands.

The federal-acquisitions angle: a vendor question like *"what does FAR 52.212-4(a)(1) require for invoice timing?"* needs sub-paragraph precision at retrieval but full-clause context at generation (the model must read the surrounding `(a)` to interpret `(a)(1)`). Pick your strategy with that example in hand.

## 4. Embedding generation — model and dimensionality choice (10 min)

The training repo defaults to **Bedrock Titan Embeddings v2** — 1024 dimensions by default, with 512 and 256 truncations supported for cost/latency trade-offs. The model card: max input 8,192 tokens, optimised for English retrieval, ~\$0.00002 per 1K input tokens. Per **D-060 you are authorized to call this for real on Tue** — token/cost accounting starts request 1, no mock-mode safety net.

Two trade-off levers your Mon ADR must name:

- **Dimensionality**: 1024 = best recall, highest storage + retrieval cost. 512 = ~15% recall loss in benchmarks, half the storage. 256 = ~30% recall loss, quarter the storage. For a 3,500-page corpus that's the difference between an Atlas M10 index fitting comfortably or pushing the tier.
- **Model choice vs domain-tuning**: Titan v2 is general-English. Cohere Embed Multilingual + OpenAI text-embedding-3-large are alternatives the Wed `/web-research` slot evaluates. None of them are domain-tuned on federal-acquisitions text — domain-tuned embeddings are a Wed scenario-alternative, not a Mon default.

Embedding dimensionality is **fixed at index creation in Atlas** (per Atlas Vector Search docs, retrieved 2026-05-22 via /web-research) — changing embedding model means rebuilding the index. The Mon ADR you commit on embedding model is therefore irreversible-ish; treat it as such.

## 5. MongoDB Atlas Vector Search as RAG store (8 min)

Atlas Vector Search lets you store vector embeddings alongside operational documents in MongoDB and query them with approximate-nearest-neighbour (ANN) search via the `$vectorSearch` aggregation stage. Per the Atlas docs (retrieved 2026-05-22 via /web-research):

- **`$vectorSearch` must be the first stage** of any aggregation pipeline where it appears. Cannot be used inside `$lookup` sub-pipelines or `$facet`. Operational gotcha most teams hit on day 1.
- **Index signature:** one `vector` field (path + numDimensions + similarity = cosine / euclidean / dotProduct) plus zero or more `filter` fields declared by path. Fields used in `filter:` at query time MUST be declared as `filter`-type fields in the index — otherwise the query errors.
- **Multi-tenant pre-filter:** use the `filter:` clause **inside** `$vectorSearch`. Adding `$match` after `$vectorSearch` is a correctness footgun — the vector search runs against the full ANN graph first; post-filter wastes candidates and may leak cross-tenant near-neighbours.
- **`numCandidates >= 20*k`** heuristic for ANN recall. Too low → poor recall; too high → latency.
- **Similarity metric** matches your embedding model's normalisation (Atlas defaults to cosine in its UI — that's flagged as a known-bad-pattern in our research brief; verify against Titan v2's docs, don't take the default).

The load-bearing field this week: **`agency_id` as a `filter`-type field in the index** (closes debt **Item 10** at the schema level, not at query time). Tue afternoon you bake this in. Wed morning you'll see what happens when it's missing.

## 6. LangChain v1.0 posture + ADR discipline — the two non-negotiable Mon ADRs (12 min)

Your Mon afternoon Scenario Design Planning artifact must commit ADRs on **chunking + embedding model + vector store + LangChain v1.0 posture** (per D-033). The first three are the "what" decisions from topics 3–5. The fourth — LangChain v1.0 posture — is the **how-do-we-compose** decision, and it has known-bad-patterns the cohort must explicitly avoid.

**LangChain v1.0 posture (per /web-research, retrieved 2026-05-22):**

The current stable release is `langchain==1.3.0` (released 2026-05-12). v1.0 GA (2025-10-22) made three deliberate architectural choices that bite anyone learning from older tutorials:

- The **`Chain` class is removed** from the main package — moved to `langchain-classic` for backwards compatibility. **You will not subclass `Chain` and you will not call `.run()`** on a chain object. Any tutorial showing `RetrievalQA.from_chain_type(...).run(...)` is pre-v1.0 — flag it for instructor review, do not adopt.
- **LCEL `|` pipe syntax is no longer the foundational composition pattern.** Sources advocating LCEL as "the foundation" or "the standard way" are stale. Sequential composition in v1.0 is **plain Python** — assign-to-variables or nested function calls. Example: `retrieved = retriever.invoke(q); response = model.invoke(build_prompt(q, retrieved))`. Not `chain = retriever | model`.
- The **`create_agent` entry point** is the canonical way to build an agent (W3 territory — not today). For pure RAG composition this week, we don't need it.

**Why this matters for Mon's ADR**: if you commit to "use LCEL pipes everywhere" in your scenario design planning artifact, Codex Adversarial Review (Ramping strictness on Fri per D-034) will flag it as architectural drift and the Mon ADR becomes Thu rework. Commit to plain Python composition + `create_agent` only when you cross into agent territory.

**ADR discipline (the framing that survives all six weeks):** an ADR captures an **irreversible-ish decision** + its **rationale at the time** + the **alternatives considered**. The Mon ADR template is in `templates/scenario-design-planning.md`. Each ADR carries: title, status (Proposed / Accepted / Superseded), context, decision, consequences, alternatives. The afternoon artifact requires 2–3 ADRs committed in your pair-project repo by 17:00.

**Iterative spec-driven dev framing (the rhythm, not the topic):** W2 Mon is the first time you produce a plan-spec. W3 Mon onward, every Plan Day opens with §0 — a retrospective on last week's plan-spec (per D-036). You iterate on planning **six times** before W6 — spec-driven dev is a *discipline* you practise from this Monday, not a topic introduced fresh in W4.

**HITL thread continuity:** Mon does not own a HITL touchpoint, but your citation-grounding ADR (dimension 5 in topic 2) is what makes HITL #2 wireable on Thu — the `needs_human_review` envelope hangs off the citation surface you commit today. The full 7-touchpoint thread: W1 Fri (LLM Essentials, done) → **W2 Thu (RAG fallback, next)** → W3 Mon-Wed-Thu → W4 Wed → W5 Wed.

---

## What you'll do W2 Mon

- Morning war-room (09:00–12:00): six RAG decision dimensions surfaced at the whiteboard, framed against the CO's question + OIG citation-trail requirement.
- Afternoon (Plan Day): graded Scenario Design Planning artifact authored — requirements synthesis, 6R/system map, 2–3 ADRs (chunking + embedding model + vector store + LangChain v1.0 posture), estimate, risk register, open questions.
- EOD (17:00): plan-spec integrity check with instructor. If your plan can't survive 10 minutes of pushback, it's not ready.
- Tomorrow's prep: read `pre-session/2-Tuesday/1-DailyTopicOverview.md` — vendor Q&A scenario drops at 09:00 Tue and your Mon ADRs get their first lived test.

## Sources

All citations retrieved via /web-research per `pipeline/RESEARCH-PROTOCOL.md`. Recency windows applied: hot-tech 3mo (LangChain v1.0, Atlas Vector Search, Bedrock Titan v2), foundation-stable 12mo (RAG concept), federal-regulatory 6mo (FAR Subpart 15.2, 48 CFR §201.104).

- LangChain v1.0 research brief — `research/langchain-v1-20260522.md`, retrieved 2026-05-22. Pinned v1.3.0; known-bad-patterns flagged (Chain class, LCEL pipe as foundation).
- MongoDB Atlas Vector Search research brief — `research/mongodb-atlas-vector-search-20260522.md`, retrieved 2026-05-22. `$vectorSearch` first-stage rule, `filter:` clause semantics, vector index definition.
- AWS Bedrock Claude catalog + Titan Embeddings v2 — `research/bedrock-claude-catalog-20260522.md`, retrieved 2026-05-22. Titan Embeddings v2 1024-dim default, truncation modes.
- FAR Subpart 15.2 (Solicitation and Receipt of Proposals) — `research/federal-ambient-far-20260525.md`, retrieved 2026-05-25 via /web-research. The corpus chunk Tue retrieves against.
- 48 CFR §201.104 (DFARS supplements FAR — clause precedence rule), retrieved 2026-05-25 via /web-research, captured in `research/federal-ambient-far-20260525.md`.

Last verified: 2026-05-26
