---
week: W02
day: Thu
topic_slug: audit-trail-for-retrieval
topic_title: "Audit trail for retrieval + E2E RAG integration"
parent_overview: W02/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://www.digitalapplied.com/blog/agent-audit-trail-design-7-best-practices-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://devsecopsschool.com/blog/audit-trails/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://agnitestudio.com/blog/designing-tamper-resistant-audit-trails-compliance-systems/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://docs.langchain.com/oss/python/releases/langchain-v1
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://articles.chatnexus.io/knowledge-base/audit-trails-and-compliance-reporting-for-regulate/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Audit trail for retrieval + E2E RAG integration

## 1. Learning Objectives

By the end of this reading, the learner can:

- Explain what a **correlation ID** is and why it is load-bearing for tying a multi-stage RAG request (retrieval → generation → judges → human review → publish) into a single auditable chain.
- Describe the **append-only event-store discipline** and why "update" or "delete" of audit rows is the failure mode regulators check for.
- Name the **canonical fields** of a retrieval audit event — actor, action, timestamp, correlation ID, payload — and what each enables on the read side.
- Articulate the **migration target away from legacy LangChain `LLMChain.run()`** as plain-Python composition with explicit invocation and validation, citing the v1.0 framing.
- Map the **three RAG endpoints** of a typical service — clause/document search, grounded Q&A, drafter — to the parts of the audit story (retrieval row vs. response row vs. reviewer row).

## 2. Introduction

The audit trail is the artifact that lets a team defend a grounded-AI system months or years later. A regulator, auditor, internal incident-response team, or a customer with legal recourse opens an investigation against a specific past interaction, and the question they need answered is: *what exactly happened, with what data, in what order, decided by whom, and is the evidence preserved unaltered?* Building a system that can answer this question after the fact is harder than building one that produces correct answers in the moment.

For RAG specifically, the audit trail has a multi-stage shape. A single request fans out through query embedding, vector retrieval, reranking, primary-model completion, judge evaluation on two axes, and either auto-publish or human-review routing. Each stage produces a decision that must be reconstructible from log evidence alone. The technique that ties this fan-out into one chain is the **correlation ID** — a unique identifier minted at the request boundary, threaded through every downstream call, and emitted on every audit row.

The second load-bearing discipline is **append-only storage**. Rows enter the log; rows never leave. No updates, no deletes, no in-place edits. Corrections emit a *new* row that supersedes the original — both stay. Regulators check for this because in-place edits can hide misbehaviour after the fact. Append-only is also what enables time-travel queries — "show me the state of this decision as the system saw it at 14:32" — which are the operationally interesting queries during an investigation.

## 3. Core Concepts

### 3.1 The correlation ID

A correlation ID is a unique value — typically a UUID — minted at the request boundary, attached to the request as a header or context-local variable, and propagated to every downstream call. Each audit row emitted during the request carries the same ID. Queries filter by correlation ID to reconstruct the full decision path.

Two conventions: (a) **mint at the gateway, propagate through every call** — via `X-Correlation-Id` header or async context variable; a service that forgets to propagate creates an audit gap. (b) **Distinguish from the request ID** — some teams use one value for both; others keep them separate so the correlation ID can span retries (new request ID, same correlation).

### 3.2 Append-only event-store discipline

The audit store is structurally an **immutable log**:

- **Inserts only.** No `UPDATE`, no `DELETE` at the database level — often enforced by revoking those permissions for the application role on the audit collection.
- **Corrections by superseding row.** A row that needs amendment emits a new row referencing the original (`supersedes: <previous_row_id>`). Both rows are retained.
- **No purges within the regulatory retention horizon** (six years federal, seven SOX-adjacent, longer in some healthcare contexts).
- **Tamper-evidence** layered on top — periodic row-hash checkpoints, anchoring to an external append-only system, or merkle-tree summaries published to a third party.

The first three are table stakes; the fourth is domain-dependent.

### 3.3 Canonical fields of a retrieval audit row

A retrieval audit event has roughly the following shape. The field names vary by team; the *information content* is what matters.

| Field | Purpose |
|---|---|
| `event_id` | Unique row identifier |
| `correlation_id` | Ties this row to all other rows from the same originating request |
| `actor` | Who/what emitted the row — `service-name:endpoint`, or `user:<id>` for review actions |
| `actor_type` | `system` vs `user` |
| `action` | The taxonomy of what happened — `qa_retrieval`, `qa_published`, `qa_reviewed`, etc. |
| `timestamp` | High-precision UTC timestamp |
| `payload` | Structured per-action data (query, retrieved-chunk references, scores, etc.) |
| `metadata` | Cross-cutting context — request source, tenant, version of the model used |
| `prev_event_id` | Optional pointer to the immediately previous event in the chain |

The payload is where the action-specific information lives. For a retrieval row, the payload includes the query, the IDs (not the full content) of retrieved chunks, the similarity and rerank scores, and the judge scores if computed. For a reviewer-action row, the payload includes the reviewer's identifier, the action taken (approved / edited / rejected), and any diff between the draft and the published version.

### 3.4 Time-travel queries and the reconstruction property

The investigation-time query: "for correlation ID *X*, reconstruct the system's state at each stage." Three properties required:

1. **All rows for a correlation ID are findable** (indexed on `correlation_id`).
2. **Rows are ordered** (by `timestamp`, tiebreaker on `event_id`).
3. **Each row's payload is self-sufficient** — does not depend on external state that may have changed since.

Point 3 is subtle. A row recording "we retrieved chunk IDs A, B, C" loses information if A/B/C have since been edited or deleted. Two mitigations, usually both: (a) store the chunk *content hash* in the audit row alongside the ID; (b) keep the corpus itself versioned so chunk A at retrieval time is recoverable as A@version-T.

### 3.5 The LangChain legacy `LLMChain.run()` migration

A common legacy pattern that **does not** survive v1.0 migration: code wrapping the model call inside a `Chain` subclass and invoking via `chain.run(input)`. Removed in v1.0 — the `Chain` class moved to `langchain-classic` for backwards compatibility, and v1.0 docs do not centre LCEL or `|` pipes.

Migration target: **plain Python composition** — build the prompt explicitly, invoke the model client, validate with Pydantic. The sequence *is* the code.

```python
# Legacy pattern (do NOT ship after v1.0 migration):
# response = my_chain.run({"query": query})

# v1.0 target — plain Python composition:
from pydantic import BaseModel

class DraftResponse(BaseModel):
    response_text: str
    citations: list[str]

def generate_draft(query: str, retrieved: list[dict], model_client) -> DraftResponse:
    prompt = build_prompt_template(query=query, chunks=retrieved)
    raw = model_client.invoke(prompt, model="anthropic:claude-sonnet-4-5")
    return DraftResponse.model_validate_json(raw)
```

The composition is `invoke(build_prompt(...))` plus validation. No `Chain` subclass; no `|` pipes; no `.run()`.

> [!instructor-review]
> **Blocklist enforcement.** Per `~/fde-10-week/skills/tech-research/references/known-bad-patterns.yml` (last reviewed 2026-05-12), the IDs `langchain-chain-class`, `langchain-lcel-pipe`, `langchain-chaining-verb`, and `langchain-pre-v1-advocacy` collectively forbid recommending the legacy `Chain` API or LCEL `|` syntax as foundational. The canonical v1.0 framing is `create_agent` for agent-shaped problems and plain Python for sequential composition — see the [LangChain v1 release notes](https://docs.langchain.com/oss/python/releases/langchain-v1).

### 3.6 The three endpoints in a typical grounded service

A typical grounded service exposes three endpoint shapes, each with a slightly different audit story:

| Endpoint | What lands per request | Audit-trail emphasis |
|---|---|---|
| **Retrieval-only** (e.g., `POST /search`) | Query → ranked chunks (no LLM generation) | Retrieval row only; no generation/judge/review rows |
| **Grounded Q&A** (e.g., `POST /answer`) | Query → retrieval → response → judges → publish or queue | Full chain — retrieval, generation, both judges, publish-or-queue, plus any reviewer-action rows downstream |
| **Drafter** (e.g., `POST /draft`) | Query → retrieval → response → human-CS review workflow before publish | Retrieval and generation rows, plus the CS-review workflow rows; no inline judge gate (implicit-HITL via workflow) |

The grounded-Q&A endpoint is where the runtime evaluation gate lives. The drafter endpoint defers the human-review step to an explicit workflow — there is no envelope; the human review is *the entire emit path*.

## 4. Generic Implementation

A generic retrieval-audit emission and the corresponding query, framework-agnostic, in a non-acquisitions domain (a SaaS knowledge-base assistant):

```python
# Generic audit-emit on retrieval. Append-only Mongo collection
# with $UPDATE and $DELETE permissions revoked at the role level.

import uuid
from datetime import datetime, timezone

audit = mongo_client["app"]["audit_events"]   # append-only by role grant

def emit_retrieval_event(
    correlation_id: str,
    tenant_id: str,
    actor: str,
    query: str,
    retrieved_chunks: list[dict],
    f_score: float | None,
    r_score: float | None,
) -> str:
    event_id = str(uuid.uuid4())
    audit.insert_one({
        "event_id": event_id,
        "correlation_id": correlation_id,
        "actor": actor,                      # e.g., "answer-service:answer"
        "actor_type": "system",
        "action": "qa_retrieval",
        "timestamp": datetime.now(timezone.utc),
        "payload": {
            "tenant_id": tenant_id,
            "query": query,
            "retrieved": [
                {
                    "chunk_id": c["chunk_id"],
                    "chunk_content_hash": c["content_hash"],   # for reconstruction
                    "doc_version": c["doc_version"],
                    "similarity": c["similarity"],
                    "rerank_score": c.get("rerank_score"),
                }
                for c in retrieved_chunks
            ],
            "faithfulness_score": f_score,
            "relevance_score": r_score,
        },
        "metadata": {
            "model_version": "anthropic:claude-sonnet-4-5",
            "embedding_model": "text-embed-v3",
        },
    })
    return event_id

# Reconstruction query: full chain for a single correlation ID, in order.
def reconstruct_chain(correlation_id: str) -> list[dict]:
    return list(
        audit.find({"correlation_id": correlation_id})
             .sort([("timestamp", 1), ("event_id", 1)])
    )
```

The two pieces: an emit function that stores enough payload to reconstruct the stage independently of downstream state changes (note the `chunk_content_hash` and `doc_version` — those are the load-bearing fields for the reconstruction property), and a query that retrieves the full ordered chain for an investigation.

## 5. Real-world Patterns

**Healthcare — clinical-decision-support audit.** A clinical decision support vendor's RAG-grounded recommendations engine is regulated by both HIPAA and FDA Class II medical-device requirements. Their audit trail records the correlation ID per clinician interaction, the retrieval row for every guideline lookup, the model output, the safety-classifier verdict, and the clinician's accept/edit/override action. Append-only at the database role level; corrections supersede with explicit pointers. During regulatory audits, investigators routinely sample correlation IDs and verify the chain is complete and unaltered. ([DigitalApplied 2026](https://www.digitalapplied.com/blog/agent-audit-trail-design-7-best-practices-2026))

**Fintech — KYC-document assistant.** A bank's KYC-onboarding workflow uses a RAG-grounded assistant to summarise customer-uploaded documents. Every retrieval emits an audit row including the document version hash, so investigators can verify which document the model was reasoning about even if it was later replaced. Append-only survived a SOC 2 audit; in-place edits would have been a material finding. ([Agnite Studio 2026](https://agnitestudio.com/blog/designing-tamper-resistant-audit-trails-compliance-systems/))

**SaaS — multi-tenant developer-tools assistant.** A B2B SaaS company's developer-docs assistant emits an audit row per retrieval with tenant ID, chunk IDs, and chunk content hashes. The audit row is the evidence of record for cross-tenant-leakage investigations — what was actually retrieved, what filter was applied, what corpus version was active. The reconstruction property is what makes the row defensible six months later. ([DevSecOps School 2026](https://devsecopsschool.com/blog/audit-trails/))

**Pharma — regulatory-submission drafting.** A pharmaceutical company's regulatory team uses a RAG-grounded assistant to draft FDA-submission sections. The audit trail records retrieval evidence, the draft, SME review actions, and the final submitted text — all tied by correlation ID. In an FDA investigation years later, the company can reconstruct every drafted sentence's provenance. The audit trail itself is part of the submission package. ([ChatNexus 2026](https://articles.chatnexus.io/knowledge-base/audit-trails-and-compliance-reporting-for-regulate/))

## 6. Best Practices

- **Mint the correlation ID at the outermost API boundary, propagate through every downstream call.** A propagation gap is an audit gap.
- **Enforce append-only at the database level, not in application code.** Revoke `UPDATE` and `DELETE` for the application role on the audit collection; corrections emit a superseding row.
- **Store chunk content hashes and document versions in retrieval audit rows.** Chunk IDs alone are insufficient if the corpus mutates.
- **Distinguish system actors from user actors.** Retrieval/generation/judge rows are system-emitted; reviewer-action rows are user-emitted; the type drives downstream analytics.
- **Migrate `LLMChain.run()` and `Chain` subclass usage to plain Python composition.** Sequential composition is explicit Python, not a framework abstraction.
- **Treat the audit log as a *read* surface from day one.** "All rows for correlation X, in order" must be performant; design the index accordingly.
- **Test the reconstruction property in CI.** Inject a known correlation ID through the pipeline, then reconstruct from the audit store and assert the chain matches expected stages.

## 7. Hands-on Exercise

**Task (whiteboard or pseudocode, 10–15 min):** You are designing the audit trail for a RAG-backed assistant in a non-acquisitions domain (e.g., a healthcare clinical-Q&A, a fintech research tool, a logistics ops assistant). Sketch:

1. The **distinct event-type taxonomy** you would emit — list the action names and a one-sentence purpose for each. Aim for 5–8 actions.
2. The **canonical fields** of your retrieval event, with one sentence on why each is required.
3. One **migration** in your design — a legacy code pattern (the `Chain`/`run()` shape, or any other legacy you can name) and what you would replace it with.
4. The **reconstruction query** for an investigation, in pseudocode — given a correlation ID, return the ordered chain.
5. One **tamper-evidence** mechanism layered on top of append-only storage, and the threat model it addresses.

**What good looks like.** The event taxonomy is *meaningful* — actions like `qa_retrieval`, `qa_generated`, `qa_judged`, `qa_published`, `qa_reviewer_approved/edited/rejected` — not a single catch-all. The retrieval fields include `correlation_id`, `chunk_content_hash`, `doc_version`, scores, and a `model_version` metadata field. The migration names a specific replacement (e.g., remove `RAGChain` subclass; replace with `generate_draft(query, retrieved, client)` doing prompt-build + invoke + validate). The reconstruction query is correctly indexed (`correlation_id` then `timestamp`). The tamper-evidence layer is realistic — periodic row-hash anchoring, merkle summaries, or write-once storage; the threat model is "insider with database access modifying historical rows."

## 8. Key Takeaways

- **What is a correlation ID?** A unique identifier minted at the request boundary, propagated through every downstream call, and emitted on every audit row — it ties multi-stage RAG requests into a single auditable chain.
- **What is the append-only discipline?** Inserts only; corrections by superseding row; no in-place updates or deletes within the regulatory retention horizon — typically enforced at the database role level.
- **Why store chunk content hashes and doc versions on retrieval audit rows?** Because the corpus mutates over time; chunk IDs alone do not let an investigator reconstruct what the system actually saw.
- **What replaces legacy `LLMChain.run()` in v1.0?** Plain Python composition — `model_client.invoke(build_prompt(...))` with explicit validation; no `Chain` subclass, no `|` pipe operator.
- **Why does the audit trail need to support a reconstruction query, not just an insert path?** Because the operationally interesting question, months after the fact, is "what did the system see and decide at time T for correlation ID X" — answerable only if rows for X are findable, ordered, and individually self-sufficient.

## Sources

1. [Agent Audit Trail Design — 7 Best Practices for 2026](https://www.digitalapplied.com/blog/agent-audit-trail-design-7-best-practices-2026) — retrieved 2026-05-26
2. [What is Audit Trails? Meaning, Architecture, Examples, Use Cases (2026 Guide)](https://devsecopsschool.com/blog/audit-trails/) — retrieved 2026-05-26
3. [Designing Tamper-Resistant Audit Trails for Compliance Systems](https://agnitestudio.com/blog/designing-tamper-resistant-audit-trails-compliance-systems/) — retrieved 2026-05-26
4. [LangChain v1.0 release notes — agent-first pivot, classic-namespace move](https://docs.langchain.com/oss/python/releases/langchain-v1) — retrieved 2026-05-26
5. [Audit Trails and Compliance Reporting for Regulated Industries](https://articles.chatnexus.io/knowledge-base/audit-trails-and-compliance-reporting-for-regulate/) — retrieved 2026-05-26

Last verified: 2026-05-26
