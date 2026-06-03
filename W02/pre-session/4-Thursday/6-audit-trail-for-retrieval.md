---
week: W02
day: Thu
topic_slug: audit-trail-for-retrieval
topic_title: "Audit trail for retrieval + E2E RAG integration"
parent_overview: W02/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 11
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
last_verified: 2026-06-03
---

# Audit trail for retrieval + E2E RAG integration

> [!NOTE]
> **From Wed (D3):** the audit-log Item 2 race fix established append-only writes per row. This topic threads the correlation ID through retrieval → judges → publish → CO action so six months of `qa_co_*` events stitch into one chain.

## 1. Learning Objectives

- Explain the **correlation ID** as the load-bearing thread tying multi-stage RAG requests into one chain.
- Describe **append-only event-store discipline** and why update/delete are the failure modes OIG checks for.
- Name the canonical fields of a retrieval audit row and what each enables on the read side.
- Migrate legacy `LLMChain.run()` (Item 5) to plain Python composition per LangChain v1.0 + D-033.

## 2. Introduction

The audit trail is the artifact that lets a team defend a grounded-AI system six months later. OIG opens an investigation against a past interaction: *what happened, with what data, in what order, decided by whom, and is the evidence preserved unaltered?* Building a system that answers this after the fact is harder than building one that produces correct answers in the moment. A RAG request fans out — embedding, retrieval, rerank, generation, two judges, publish-or-queue, reviewer action. Each stage must be reconstructible from log evidence alone. Two disciplines tie it together: the **correlation ID** threaded through every call, and **append-only storage** enforced at the database role level. Today is also where Item 5's `LLMChain.run()` migration lands (D-033) — v1.0 composition is plain Python, not a framework abstraction.

## 3. Core Concepts

### 3.1 The audit-event row shape

```jsonc
{ "event_id":      "evt-...",            // unique row
  "correlation_id":"req-abc123",          // ties retrieval → judge → publish → CO action
  "actor":         "ai-orchestrator:answer-qa",
  "actor_type":    "system",              // vs "user" for CO actions
  "action":        "qa_retrieval",        // taxonomy below
  "timestamp":     "2026-06-04T15:47:00Z",
  "payload": {
    "tenant_id": "agency-xyz",
    "query":     "...",
    "retrieved": [
      { "chunk_id":           "...",
        "chunk_content_hash": "sha256:...",   // reconstruction property
        "doc_version":        "v3",
        "clause_id":          "FAR 15.304",
        "similarity":         0.81,
        "rerank_score":       0.74 }
    ],
    "faithfulness_score": 0.91,
    "relevance_score":    0.88 },
  "metadata": { "model_version":  "anthropic:claude-sonnet-4-5",
                "embedding_model":"text-embed-v3" } }
```

`chunk_content_hash` + `doc_version` are the **reconstruction property** — six months later the chunk may be edited or gone, but the hash + version let an investigator recover what the system actually saw at request time.

### 3.2 Action taxonomy per RAG path

| Endpoint | Action types emitted | What's reconstructible 6 months later |
|---|---|---|
| `POST /rag/clause-search` (retrieval-only) | `qa_retrieval` | What chunks were surfaced, at what filter |
| `POST /answer-qa` (HITL #2 gate today) | `qa_retrieval` → `qa_generated` → `qa_judged` → `qa_published` OR `qa_queued` + (`qa_co_approved` / `qa_co_edited` / `qa_co_rejected` / `qa_co_sla_breach`) | Full chain from query through CO action |
| `POST /draft-solicitation` (drafter) | `qa_retrieval` → `qa_generated` → CS workflow rows (no inline judge gate — implicit HITL via workflow) | Retrieval + draft + CS-review workflow trail |

### 3.3 Append-only discipline + reconstruction property

Inserts only. No `UPDATE`, no `DELETE` for the application role. Corrections emit a superseding row (`supersedes: <prev_event_id>`); both stay. No purges within retention (6 years federal, 7 SOX-adjacent). The reconstruction query — "all rows for correlation X, in order" — must be performant: index on `correlation_id`, sort by `(timestamp, event_id)`. Each row's payload must be **self-sufficient** — doesn't depend on external state that may have changed. That's why content hash + doc version live on every retrieval row.

> [!IMPORTANT]
> **Test the reconstruction property in CI.** Inject a known correlation ID through the pipeline, then reconstruct from the audit store and assert the chain matches expected stages. A propagation gap (one service forgets `X-Correlation-Id`), an in-place edit on a misclassified row, or a chunk-ID-without-content-hash when the corpus has since mutated — each makes the chain unreconstructible. CI catches all three.

## 4. Generic Implementation

```python
# Generic Item 5 migration target — plain Python composition.
# Lives in acquire-gov at services/ai-orchestrator/drafting/draft_qa.py
# (Replaces legacy DraftingWizardChain / AmendmentEditorChain / notification chain.)
import uuid
from datetime import datetime, timezone
from pydantic import BaseModel

audit = mongo_client["app"]["audit_events"]   # UPDATE+DELETE revoked at role level

class DraftResponse(BaseModel):
    response_text: str
    citations: list[str]

def generate_draft(query, retrieved, model_client, correlation_id) -> DraftResponse:
    # v1.0 target — plain Python; no Chain subclass, no |, no .run()
    prompt = build_prompt_template(query=query, chunks=retrieved)
    raw = model_client.invoke(prompt, model="anthropic:claude-sonnet-4-5")
    response = DraftResponse.model_validate_json(raw)
    emit_audit_event(correlation_id, "qa_generated",
                     {"query": query, "draft": response.model_dump()})
    return response

def emit_audit_event(correlation_id, action, payload):
    audit.insert_one({
        "event_id": str(uuid.uuid4()),
        "correlation_id": correlation_id,
        "actor": "ai-orchestrator:answer-qa",
        "actor_type": "system",
        "action": action,
        "timestamp": datetime.now(timezone.utc),
        "payload": payload,
        "metadata": {"model_version": "anthropic:claude-sonnet-4-5"},
    })

def reconstruct_chain(correlation_id):       # investigation-time query
    return list(audit.find({"correlation_id": correlation_id})
                     .sort([("timestamp", 1), ("event_id", 1)]))
```

Three call sites in `legacy_chain.py` migrate to this shape today: `DraftingWizardChain`, `AmendmentEditorChain`, `notification-copy generator`. Codex Adversarial Review is at **Ramping** strictness (D-034) — any PR re-introducing `LLMChain.run()` after today is blocked, not coached.

## 5. Real-world Patterns

**Healthcare — clinical-decision-support.** A CDS vendor regulated by HIPAA + FDA Class II records correlation ID per clinician interaction, retrieval row per guideline lookup, model output, safety verdict, and the clinician's accept/edit/override. Append-only at the database role level; corrections supersede with explicit pointers. Append-only enforced at the role grant survived a SOC 2-equivalent audit; in-place edits would have been a material finding.

**Fintech — KYC-document assistant.** A bank's KYC workflow uses RAG to summarise customer-uploaded documents. Every retrieval emits an audit row including the document version hash, so investigators verify which document the model reasoned about even if it was later replaced. The hash-on-row pattern made the audit trail defensible in SOC 2.

**SaaS — multi-tenant developer-tools.** A B2B SaaS dev-docs assistant emits an audit row per retrieval with tenant ID, chunk IDs, content hashes. The row is evidence of record for cross-tenant-leakage investigations — what was retrieved, what filter applied, what corpus version active. The reconstruction property is what makes the row defensible when an enterprise customer asks "was our data ever surfaced to another tenant?"

**Pharma — regulatory-submission drafting.** A regulatory team's RAG assistant drafts FDA-submission sections. The audit trail records retrieval evidence, draft, SME review actions, final submitted text — all tied by correlation ID. In an FDA investigation years later, the company reconstructs every drafted sentence's provenance. The audit trail itself is part of the submission package — same posture OIG takes with `acquire-gov` exports.

## 6. Best Practices

- **Mint correlation ID at the outermost API boundary**, propagate through every downstream call.
- **Enforce append-only at the database level**, not in application code — revoke `UPDATE` + `DELETE` for the app role.
- **Store chunk content hashes + doc versions on retrieval rows** — chunk IDs alone are insufficient when the corpus mutates.
- **Distinguish system actors from user actors** — type drives downstream analytics.
- **Migrate `LLMChain.run()` to plain Python composition** (D-033).
- **Treat the audit log as a read surface from day one** — index `correlation_id` for the reconstruction query.
- **Test the reconstruction property in CI** — inject a known ID, reconstruct, assert.

> [!WARNING]
> **Anti-pattern: partial-log = no-log.** OIG findings treat **incomplete audit trails as no audit trail**. A propagation gap (one service forgets `X-Correlation-Id`), an in-place edit on a misclassified row, or a chunk-ID-without-content-hash when the corpus has since mutated — each makes the chain unreconstructible. Common internet patterns ("we'll log on errors", "we'll backfill the hash later") fail this bar. Enforce `INSERT`-only at the database role level; revoke `UPDATE` and `DELETE` for the app role on the audit collection. Per `known-bad-patterns.yml` `partial-audit-log` + `langchain-chain-class`.

## 7. Hands-on Exercise

Design the audit trail for a RAG-backed assistant in a non-acquisitions domain (healthcare clinical-Q&A, fintech research tool, logistics ops assistant). Sketch (a) the event-type taxonomy — 5–8 action names with one-sentence purposes, (b) the canonical fields of your retrieval event with rationale per field, (c) one migration in your design — a legacy pattern (the `Chain`/`.run()` shape or another) and the replacement, (d) the reconstruction query in pseudocode given a correlation ID, (e) one tamper-evidence mechanism layered on top of append-only and the threat model it addresses. War-room block E ports this to `acquire-gov` Item 5 migration + EOD reconstruction test.

> [!NOTE]
> **Self-check** (30s)
>
> 1. Why does a retrieval audit row need `chunk_content_hash` AND `doc_version`, not just `chunk_id`?
> 2. Why enforce append-only at the database role level rather than in application code?

<details>
<summary>Show answers</summary>

1. Because the corpus mutates. Six months later, chunk-ID `A` may point to edited text, or the chunk may be gone. The content hash + doc version let an investigator reconstruct what the system actually saw at request time — the **reconstruction property** is what makes the row defensible. Without it, the audit row records a pointer to mutable state, and the OIG investigation finds an unreconstructible chain.
2. Application-code enforcement is one bypass away from broken — a debug endpoint, a maintenance script, or a future bug can `UPDATE` rows without intent. Revoking `UPDATE` and `DELETE` on the app role at the DB level makes in-place edits structurally impossible. Corrections emit a superseding row referencing the original (`supersedes: <prev_event_id>`); both stay.

</details>

## 8. Key Takeaways

- **Correlation ID** minted at the gateway and propagated through every downstream call ties multi-stage requests into one auditable chain.
- **Append-only at the role grant**, not in application code — `UPDATE` + `DELETE` revoked; corrections emit superseding rows.
- **Chunk content hash + doc version** on retrieval rows guarantee the reconstruction property when the corpus mutates.
- **LangChain v1.0 migration** replaces `Chain.run()` with plain Python composition — `model_client.invoke(build_prompt(...))` + Pydantic validation.
- Test the reconstruction property in CI; partial logs fail OIG scrutiny.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- <https://www.digitalapplied.com/blog/agent-audit-trail-design-7-best-practices-2026> — retrieved 2026-05-26 — hot-tech-3mo
- <https://devsecopsschool.com/blog/audit-trails/> — retrieved 2026-05-26
- <https://agnitestudio.com/blog/designing-tamper-resistant-audit-trails-compliance-systems/> — retrieved 2026-05-26
- <https://docs.langchain.com/oss/python/releases/langchain-v1> — retrieved 2026-05-26
- <https://articles.chatnexus.io/knowledge-base/audit-trails-and-compliance-reporting-for-regulate/> — retrieved 2026-05-26

Bad-pattern blocklist: `langchain-chain-class`, `langchain-lcel-pipe`, `langchain-chaining-verb`, `langchain-pre-v1-advocacy`, `partial-audit-log` (last reviewed 2026-05-12).

</details>

<details>
<summary>Deeper dive — correlation ID propagation + tamper-evidence + Thu PM E2E targets</summary>

**Correlation ID propagation:**

- **Mint at the gateway**, propagate via `X-Correlation-Id` header or async context-local. Any service that forgets to propagate creates an audit gap. Distinguish from `request_id` so the correlation ID can span retries (new request ID, same correlation).
- **Reconstruction query**: `audit.find({"correlation_id": X}).sort([("timestamp", 1), ("event_id", 1)])` — must be performant. Index on `correlation_id`.

**Tamper-evidence layered on append-only:** periodic row-hash checkpoints; merkle-tree summaries published to a third party; anchoring to an external append-only system (S3 Object Lock with compliance mode, QLDB). Threat model: insider with database access modifying historical rows. Append-only at the role level defends against application-bypass writes; tamper-evidence defends against database-administrator-level modification. Federal-acq is far enough up the regulatory ladder that both layers are expected.

**Retention horizon:** 6 years federal, 7 SOX-adjacent, longer in some healthcare contexts. No purges within horizon. Plan storage cost accordingly — append-only over a 6-year window with retrieval-row payloads in the kilobyte range adds up to non-trivial sustained spend.

**Thu PM E2E targets — endpoints landing today:**

| Endpoint | What lands |
|---|---|
| `POST /rag/clause-search` | RAG benchmark surface — retrieval only |
| `POST /answer-qa` | **HITL #2 gate** — retrieval → completion → judge → envelope |
| `POST /draft-solicitation` | RAG-grounded Section C SOW + Section L drafter (implicit HITL via CS workflow) |

Plus Item 5 migration. EOD verification: SSA can see no auto-publish without faithfulness ≥ 0.85; review queue exists; audit log shows the full chain.

</details>

Last verified: 2026-06-03
