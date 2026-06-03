---
week: W02
day: Thu
topic_slug: audit-trail-for-retrieval
topic_title: "Audit trail for retrieval + E2E RAG integration"
parent_overview: W02/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 6
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
> **From topic 3:** the envelope returns one of four `qa_co_*` reviewer-action events. This topic is where those events land + how they tie back to retrieval rows via a correlation ID.

## The audit-event row shape

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

## Action taxonomy per RAG path

| Endpoint | Action types emitted | What's reconstructible 6 months later |
|---|---|---|
| `POST /rag/clause-search` (retrieval-only) | `qa_retrieval` | What chunks were surfaced, at what filter |
| `POST /answer-qa` (HITL #2 gate today) | `qa_retrieval` → `qa_generated` → `qa_judged` → `qa_published` OR `qa_queued` + (`qa_co_approved`/`qa_co_edited`/`qa_co_rejected`/`qa_co_sla_breach`) | Full chain from query through CO action |
| `POST /draft-solicitation` (drafter) | `qa_retrieval` → `qa_generated` → CS workflow rows (no inline judge gate — implicit HITL) | Retrieval + draft + CS-review workflow trail |

> [!WARNING]
> **Anti-pattern: partial-log = no-log.** OIG findings treat **incomplete audit trails as no audit trail**. A propagation gap (one service forgets `X-Correlation-Id`), an in-place edit on a misclassified row, or a chunk-ID-without-content-hash when the corpus has since mutated — each makes the chain unreconstructible. Common internet patterns ("we'll log on errors", "we'll backfill the hash later") fail this bar. Enforce `INSERT`-only at the database role level; revoke `UPDATE` and `DELETE` for the app role on the audit collection. CI test: inject a known correlation ID through the pipeline, then reconstruct from the audit store and assert the chain matches.

## Item 5 — `LLMChain.run()` migration (D-033)

Three call sites in `legacy_chain.py` ship off the `Chain` API today: `DraftingWizardChain`, `AmendmentEditorChain`, `notification-copy generator`. Target = plain Python composition.

```python
# v1.0 target — plain Python; no Chain subclass, no |, no .run()
from pydantic import BaseModel

class DraftResponse(BaseModel):
    response_text: str
    citations: list[str]

def generate_draft(query, retrieved, model_client) -> DraftResponse:
    prompt = build_prompt_template(query=query, chunks=retrieved)
    raw = model_client.invoke(prompt, model="anthropic:claude-sonnet-4-5")
    return DraftResponse.model_validate_json(raw)
```

Codex Adversarial Review is at **Ramping** strictness (D-034) — any PR re-introducing `LLMChain.run()` after today is blocked, not coached. Pair-project teams: sweep your pair-domain code for `Chain` subclasses + `.run()` calls and migrate in the PM PR.

## Self-check

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

<details>
<summary>Deeper dive — correlation ID propagation + tamper-evidence</summary>

- **Mint at the gateway**, propagate via `X-Correlation-Id` header or async context-local. Any service that forgets to propagate creates an audit gap. Distinguish from `request_id` so the correlation ID can span retries.
- **Reconstruction query**: `audit.find({"correlation_id": X}).sort([("timestamp", 1), ("event_id", 1)])` — must be performant. Index on `correlation_id`.
- **Tamper-evidence** (domain-dependent): periodic row-hash checkpoints; merkle-tree summaries published to a third party; anchoring to an external append-only system (S3 Object Lock with compliance mode, QLDB). Threat model: insider with database access modifying historical rows.
- **Retention horizon**: 6 years federal, 7 SOX-adjacent, longer in healthcare. No purges within horizon.

</details>

<details>
<summary>Thu PM E2E targets — endpoints landing today</summary>

| Endpoint | What lands |
|---|---|
| `POST /rag/clause-search` | RAG benchmark surface — retrieval only |
| `POST /answer-qa` | **HITL #2 gate** — retrieval → completion → judge → envelope |
| `POST /draft-solicitation` | RAG-grounded Section C SOW + Section L drafter (implicit HITL via CS workflow) |

Plus Item 5 migration. EOD verification: SSA can see no auto-publish without faithfulness ≥ 0.85; review queue exists; audit log shows the full chain.

</details>

<details>
<summary>Sources (retrieved via /web-research per D-046)</summary>

1. Agent Audit Trail Design — 7 Best Practices 2026: <https://www.digitalapplied.com/blog/agent-audit-trail-design-7-best-practices-2026> — 2026-05-26
2. DevSecOps School — Audit Trails 2026 Guide: <https://devsecopsschool.com/blog/audit-trails/> — 2026-05-26
3. Agnite Studio — Tamper-Resistant Audit Trails: <https://agnitestudio.com/blog/designing-tamper-resistant-audit-trails-compliance-systems/> — 2026-05-26
4. LangChain v1.0 release notes: <https://docs.langchain.com/oss/python/releases/langchain-v1> — 2026-05-26
5. ChatNexus — Audit Trails for Regulated Industries: <https://articles.chatnexus.io/knowledge-base/audit-trails-and-compliance-reporting-for-regulate/> — 2026-05-26

Bad-pattern blocklist: `langchain-chain-class`, `langchain-lcel-pipe`, `langchain-chaining-verb`, `langchain-pre-v1-advocacy` (last reviewed 2026-05-12).

</details>

Last verified: 2026-06-03
