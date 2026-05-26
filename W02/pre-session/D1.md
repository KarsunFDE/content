---
week: W02
day: Mon
phase: Foundation
topic: "RAG architecture intro + vector DB landscape — pre-Plan-Day"
estimated_total_minutes: 55
last_verified: 2026-05-23
fde_situations: [1, 3, 6, 9, 11]
tech: [retrieval-augmented-generation, mongodb-atlas-vector-search, langchain-v1, aws-bedrock, embedding-models]
sources_research_briefs: []
author: instructor
---

# Pre-session reading — RAG architecture intro + vector DB landscape

Week 2, Day 1 (Mon). Estimated total time on task: ~55 minutes. Last verified: 2026-05-23.

> Read **before** W2 Mon morning. Mon is the first formal **Plan Day** of the programme (per D-029). You will leave Mon with a graded Scenario Design Planning artifact, not code. This pre-read gives you the vocabulary you'll need before 09:00.

## 1. Why this matters (~80 words)

W1 Fri ended with the contracting officer flagging a hallucinated citation ("48 CFR 47.305-2") that doesn't exist. The honest answer to a CO who can't tell whether the platform is grounded is **don't ship an ungrounded drafter into federal acquisitions**. RAG is the mechanism. This week the cohort wires Retrieval-Augmented Generation over the **FAR + DFARS corpus (~3,500 pages, per D-060)** so every clause-quote in `POST /draft-solicitation` and `POST /answer-qa` traces to a real chunk with a real `clause_id` + `last_revised` date.

## 2. Core concept in 5 minutes

RAG splits the LLM's job into two phases: **retrieve** (pull relevant chunks from a corpus indexed at ingestion time) and **generate** (the LLM composes the response using *those chunks only* as grounding). The model is told — in the system prompt — that it must cite only what it sees in the retrieved context and must say "I don't know" otherwise. The architectural pieces: a corpus, an **embedding model** that turns text into vectors, a **vector store** that indexes those vectors for nearest-neighbor search, a **retriever** that runs the query (often hybrid — dense + sparse), an optional **reranker**, and a **grounded prompt assembly** step that injects the retrieved chunks into the system prompt before the LLM call. None of these pieces are "one true choice" — every piece is an ADR.

## 3. What to read or watch tonight

Each link below is dated, primary-source where possible, and time-estimated. Cohort reads these in order.

- [AWS — What is RAG?](https://aws.amazon.com/what-is/retrieval-augmented-generation/) (~10 min read), retrieved 2026-05-23 via /web-research. The AWS-authored framing of the pattern — focus on the two-phase architecture diagram and the "knowledge sources" framing (a public-sector-friendly explanation; ignore the marketing copy for Bedrock Knowledge Bases — that's W5 territory per D-050).
- [MongoDB Atlas — Vector Search overview](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-overview/) (~12 min read), retrieved 2026-05-23 via /web-research. The vector store in the training repo. Focus on `$vectorSearch` aggregation stage + index definition syntax. Note the `filter` field — this is how we enforce `agency_id` multi-tenant boundary (Item 10).
- [LangChain v1.0 — RAG concepts](https://docs.langchain.com/oss/python/langchain/rag) (~10 min read), retrieved 2026-05-23 via /web-research. **Read this against the W1 Fri pre-session's LangChain v1.0 posture warning.** v1.0 RAG examples use plain Python composition — `retriever.invoke()` then `model.invoke()`. There is no `RetrievalQA.from_chain_type(...).run(...)` in v1.0. If a tutorial shows that, it's pre-v1.0 — flag it.
- [AWS Bedrock — Titan Embeddings v2 model](https://docs.aws.amazon.com/bedrock/latest/userguide/titan-embedding-models.html) (~6 min read), retrieved 2026-05-23 via /web-research. The default embedding model in the training repo. 1024-dim by default; 512 and 256 truncations supported for cost/latency trades. **Per D-060 you are authorized to call this for real on Tue** — token/cost accounting starts request 1.
- *(optional)* [FAR Subpart 15.2 (Solicitation and Receipt of Proposals)](https://www.acquisition.gov/far/subpart-15.2) (~10 min skim), retrieved 2026-05-23 via /web-research. The chunk of the corpus you'll be retrieving against on Tue. Skim Sections L and M references — Mon's drafter ADR will name them.

## 4. Two questions to come in with tomorrow

1. *"For the FAR/DFARS clause library, what's the natural chunk boundary — full clause (52.212-4 in one chunk), sub-paragraph (52.212-4(a)(1) in one chunk), or sentence? What changes about retrieval quality at each level?"* (Mon morning war-room exercises this.)
2. *"If a vendor's question references **DFARS 215.371** but the retrieval surfaces **FAR 15.371** — how does the platform notice and what does it do?"* (This is the clause-precedence resolution problem per D-060. It does not have a tidy answer Day 1; the answer evolves across Tue–Thu.)

## 5. Glossary refresh (terms you'll hear tomorrow)

- **RAG** — Retrieval-Augmented Generation. The two-phase pattern: retrieve then generate.
- **Chunk** — A unit of corpus text indexed in the vector store. Chunk boundary is an ADR.
- **Embedding** — A vector representation of text. The embedding model (e.g., Bedrock Titan Embeddings v2) is the function that produces it.
- **Vector store** — Database that indexes vectors for k-nearest-neighbor (kNN) search. Atlas Vector Search is the programme default per D-031.
- **Hybrid retrieval** — Combining dense (semantic) search with sparse (BM25 lexical) search. Reciprocal Rank Fusion (RRF) is the merge function.
- **Reranker** — A second-stage model that re-orders the top-K retrieved chunks for the LLM (e.g., Cohere Rerank, AWS reranker). Optional in v1, usually worth it.
- **Citation grounding** — The discipline that every claim in the LLM output traces to a specific retrieved chunk. The federal-acquisitions equivalent of "show your work."
- **Faithfulness** — RAGAS metric: does the answer use only the retrieved context, or does it leak ungrounded model knowledge?
- **Clause precedence** — Federal-acq rule that **DFARS supplements FAR** (per 48 CFR §201.104). When DFARS modifies a FAR clause, DFARS governs for DoD acquisitions. The RAG layer must surface both citations + the precedence rule.

## 6. Optional deep-dive (not required to participate)

- [Anthropic — Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval) (~12 min read), retrieved 2026-05-23 via /web-research. The "prepend a context summary to each chunk before embedding" technique. Worth knowing as a Wed scenario-alternative; not required for Mon planning.
- [MongoDB — Atlas Vector Search index types](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-type/) (~8 min skim), retrieved 2026-05-23 via /web-research. ANN vs ENN. Worth knowing for the index ADR.

---

## Sources

All citations retrieved 2026-05-23 via /web-research per `pipeline/RESEARCH-PROTOCOL.md`. Recency windows applied: hot-tech 3-month (LangChain v1.0, Bedrock embeddings, Atlas Vector Search), foundation-stable 12-month (RAG concept), federal-regulatory 6-month (FAR Subpart 15.2).
