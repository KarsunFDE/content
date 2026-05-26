---
week: W04
day: 3-Wednesday
topic_slug: hitl-6-excessive-agency-authority-boundary-design
topic_title: "HITL #6 — Excessive Agency (LLM06) as authority-boundary design"
parent_overview: W04/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 14
sources:
  - url: https://genai.owasp.org/llmrisk/llm062025-excessive-agency/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.indusface.com/learning/owasp-llm-excessive-agency/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://galileo.ai/blog/prevent-excessive-agency-llms
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.agentpatterns.tech/en/architecture/human-in-the-loop-architecture
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.strata.io/blog/agentic-identity/practicing-the-human-in-the-loop/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# HITL #6 — Excessive Agency (LLM06) as authority-boundary design

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define OWASP LLM06:2025 Excessive Agency in terms of its three root causes (excessive functionality, excessive permissions, excessive autonomy) and recognise the pattern in a tool-calling LLM application.
- Apply the five canonical OWASP mitigation strategies (minimize extensions, minimize functionality, avoid open-ended functions, minimize permissions, execute in user's context) to a concrete endpoint inventory.
- Build the **8-endpoint × 4-column authority-boundary table** (rows = the eight tools the agent can invoke; columns = `Caller` / `AI Authority` / `Human Gate` / `Rollback`) for a system under test, and recognise that this artifact IS the deliverable HITL #6 teaches.
- Distinguish appropriate gating from over-gating — recognise that "human approval everywhere" is itself an LLM06 finding because it floods reviewers and erodes the trust they place in the gate.
- Recognise the alignment between OWASP LLM06, the EU AI Act Article 14 (effective August 2026), and NIST AI RMF 1.0 — three different frames, same artifact.

## 2. Introduction

OWASP LLM06:2025 Excessive Agency is, in a single sentence: an LLM-based system has been granted authority to interface with other systems, take actions, or make decisions that exceed what is required for its purpose ([OWASP GenAI, 2025](https://genai.owasp.org/llmrisk/llm062025-excessive-agency/)). The mitigation is *not* "make the LLM less smart" — it is **authority-boundary design**. For each tool the LLM can invoke, you name: who can call the LLM endpoint, what authority the LLM exercises through it, what gate requires a human's approval, and what the rollback path looks like if the LLM was wrong.

This is the **sixth of seven HITL touchpoints** in the programme arc (W1 Fri / W2 Thu / W3 Mon / W3 Wed / W3 Thu / W4 Wed / W5 Wed). What makes it distinct is the *frame*. Earlier touchpoints framed HITL as a workflow concern (where does a human review fit) or as an agent-orchestration concern (where does the graph pause for approval). The W4 Wed frame is **HITL as a security control**, expressed in compliance language — the same artifact, but spoken to a security architect or a FedRAMP auditor rather than a UX designer.

**Bridging HITL #5 (W3 Thu) and HITL #7 (W5 Wed).** HITL #5 was the LangGraph hard-interrupt + FAR 15.308 audit-row implementation — the *runtime mechanics* of pausing an agent and emitting the audit event. HITL #7 is the AIOps auto-remediation authority-closure — what authority an SRE-style remediation agent may exercise without paging a human. HITL #6 sits between them: the **authority-boundary table itself is the artifact** that names, per tool, which gate type HITL #5's mechanics enforce at runtime and which decisions HITL #7's auto-remediation logic is allowed to make. The three touchpoints form one chain — table (#6) → runtime gate (#5) → auto-remediation authority closure (#7).

The regulatory weather matters. **EU AI Act Article 14**, enforceable from August 2, 2026, mandates human oversight capabilities for high-risk AI systems — and is explicit that the oversight must be "trained, measurable, and provable" ([Strata, 2026](https://www.strata.io/blog/agentic-identity/practicing-the-human-in-the-loop/)). NIST AI RMF 1.0 says the same thing in different words. The authority-boundary table you build tomorrow is the artifact that satisfies both frames simultaneously.

## 3. Core Concepts

### 3.1 The three root causes of Excessive Agency

OWASP names three ([OWASP GenAI, 2025](https://genai.owasp.org/llmrisk/llm062025-excessive-agency/)):

| Root cause | Example | Diagnostic question |
|---|---|---|
| **Excessive functionality** | An email tool that exposes read + send + delete when only read is needed | "What's the smallest tool that does the job?" |
| **Excessive permissions** | A database reader granted UPDATE/DELETE when only SELECT is needed | "Does the underlying service grant exceed the tool's stated purpose?" |
| **Excessive autonomy** | A tool that mutates state without human verification when the action is irreversible or expensive | "What's the smallest blast-radius gate that satisfies the trust requirement?" |

All three are about **calibration**. The LLM should be granted exactly the authority it needs — no more, no less. Over-grant = LLM06 finding. Under-grant = product doesn't work. The discipline is naming the right line per tool.

### 3.2 The five canonical mitigation strategies

OWASP's mitigation set ([OWASP GenAI, 2025](https://genai.owasp.org/llmrisk/llm062025-excessive-agency/); [Indusface, 2025](https://www.indusface.com/learning/owasp-llm-excessive-agency/)):

1. **Minimize extensions.** The LLM has access to only the tools strictly required for the task. Two-tool agents pass; ten-tool agents need re-examination.
2. **Minimize functionality.** Each tool exposes only the operations needed. An "email-reader" tool does not include send. A "calendar-viewer" does not include delete.
3. **Avoid open-ended functions.** No `execute_shell`, no `fetch_arbitrary_url`, no `query_arbitrary_sql`. Replace with purpose-built extensions: `get_user_by_id`, `list_calendar_for_user_today`, `send_templated_notification`.
4. **Minimize permissions.** The service account the tool authenticates with is scoped to the operation. A product-catalog reader has SELECT-only access to specific tables. Not the whole database.
5. **Execute in user's context.** The action runs as the user, not as a privileged service account. OAuth flows with user-scoped tokens, not system credentials.

These map 1:1 onto the four columns of the authority-boundary table — each mitigation surfaces as a column-level constraint per row.

### 3.3 The 8-endpoint × 4-column authority-boundary table

The artifact every team should be able to produce — **eight rows (one per tool the agent can invoke), four columns (`Caller` / `AI Authority` / `Human Gate` / `Rollback`)**. The table below is the canonical worked example for the `acquire-gov` agent surface; the cohort builds the same shape against their own pair-project endpoints in the hands-on exercise below.

| # | Tool / Endpoint | Caller (who's authorized to invoke?) | AI Authority (what action does the LLM take?) | Human Gate (when does a human approve?) | Rollback (what undoes a wrong action?) |
|---|---|---|---|---|---|
| 1 | `search_documents` | Authenticated user | Returns retrieved chunks — no synthesis | None — read-only retrieval | N/A (no side effect) |
| 2 | `view_document(id)` | Authenticated user | Fetches one document by ID | None — read-only, scoped by tenant | N/A (no side effect) |
| 3 | `summarize_document(id)` | Authenticated user | LLM synthesises summary from one fetched document | None — read-only output; downstream gate applies on use | N/A; summary not persisted unless explicit save call |
| 4 | `draft_email(to, subject, body)` | Authenticated user | Composes draft text only — no send | None — draft is non-side-effecting | N/A (no side effect until `send_email`) |
| 5 | `send_email(draft_id)` | Authenticated user via UI | Executes send to allowlisted recipients | Pre-send confirmation modal; recipient allowlist enforced | "Recall" within 5 minutes; immutable audit-log event |
| 6 | `share_document(id, recipient_email)` | Authenticated user (recipient ∈ tenant allowlist) | Grants read-access to recipient | Pre-share confirmation; recipient-allowlist check | Revoke share within 24h; audit-log event |
| 7 | `delete_document(id)` | Admin role only | Marks document deleted (soft-delete) | Two-person approval (initiator + admin) | Restore from audit-log within 30 days |
| 8 | `bulk_archive(filter)` | Service role only | Archives N documents matching filter | Two-person approval + dry-run preview of affected count + blast-radius cap | Reverse-archive script per archive batch; audit row per affected record |

Every row has all four columns filled in — or it's not done. A "Human Gate: TBD" row is an open security finding.

Two rows in the table are the most likely to be flagged in an OWASP LLM06 audit: row 8 (`bulk_archive`) because its `filter` parameter is open-ended in shape (LLM06 root cause: excessive functionality), and row 6 (`share_document`) because if the recipient list is not restricted to a tenant-scoped allowlist it becomes an exfiltration vector (LLM06 root cause: excessive permissions). Both are caught not by adding more gates but by *narrowing the tool's contract* — `bulk_archive` should expose a small set of named filter shapes rather than a free-form filter, and `share_document` should reject any recipient not in the caller's tenant allowlist before the gate even fires.

### 3.4 The over-gating anti-pattern

A common first-draft authority-boundary table puts "human approves before action" on every row. This is wrong for two reasons:

- **Read-only retrievals don't need a gate at retrieval time.** The gate is downstream, where the retrieved content gets *used*. Gating retrieval clutters the human queue with no risk reduction.
- **Gate fatigue.** When a reviewer sees 100 approval requests per day and 99 are trivial, they rubber-stamp all 100 — and the one that mattered gets approved with the rest. The literature calls this "approval-button fatigue" and treats it as a leading cause of HITL failure ([Galileo, 2025](https://galileo.ai/blog/prevent-excessive-agency-llms); [Agent Patterns, 2025](https://www.agentpatterns.tech/en/architecture/human-in-the-loop-architecture)).

The discipline: **naming what does NOT need a gate is itself an artifact**. A compliance package cluttered with hollow controls fails its own audit. Risk-based routing — low-risk reversible actions execute with audit trails; medium-risk actions get confidence-based routing; only high-risk irreversible actions require synchronous approval — is the 2025-2026 consensus.

### 3.5 Three frames, same artifact

The authority-boundary table satisfies three regulatory frames simultaneously:

- **OWASP LLM06.** Security frame. The table is the proof that excessive agency has been bounded.
- **EU AI Act Article 14.** Compliance frame, enforceable August 2026. The table is the proof that high-risk-AI human oversight is "trained, measurable, and provable" ([Strata, 2026](https://www.strata.io/blog/agentic-identity/practicing-the-human-in-the-loop/)).
- **NIST AI RMF 1.0.** Risk-management frame. The table is the input to the "Govern" function — risk-categorising each LLM-mediated action.

One artifact, three frames. That's the value of producing it correctly.

## 4. Generic Implementation

A generic LangGraph-style HITL gate that enforces authority-boundary metadata at runtime, expressed in plain Python (no LCEL pipes, no Chain class — per LangChain v1.0 guidance):

```python
# app/agents/hitl_gate.py
# Generic authority-boundary gate enforced at tool-call time.

from dataclasses import dataclass
from enum import Enum
from typing import Callable, Any

class GateType(Enum):
    NONE        = "no_gate"           # read-only or no side effect
    LOGGED      = "logged_no_gate"    # mutates but cheap to reverse; audit only
    CONFIRM     = "user_confirm"      # synchronous user confirmation modal
    TWO_PERSON  = "two_person_appr"   # initiator + separate approver
    OUT_OF_BAND = "out_of_band"       # async approval queue, not real-time

@dataclass(frozen=True)
class ToolPolicy:
    tool_name: str
    allowed_callers: frozenset[str]   # role names
    gate: GateType
    rollback_strategy: str            # human-readable, audited
    max_blast_radius: int             # affected records cap

# Centralised registry. Tools that are NOT in the registry cannot be called.
TOOL_REGISTRY = {
    "search_knowledge_base": ToolPolicy(
        tool_name="search_knowledge_base",
        allowed_callers=frozenset({"user", "admin"}),
        gate=GateType.NONE,
        rollback_strategy="N/A — read-only",
        max_blast_radius=0,
    ),
    "send_notification": ToolPolicy(
        tool_name="send_notification",
        allowed_callers=frozenset({"user", "admin"}),
        gate=GateType.CONFIRM,
        rollback_strategy="recall window 5min; audit log",
        max_blast_radius=1,
    ),
    "delete_record": ToolPolicy(
        tool_name="delete_record",
        allowed_callers=frozenset({"admin"}),
        gate=GateType.TWO_PERSON,
        rollback_strategy="restore from audit log within 30d",
        max_blast_radius=1,
    ),
}

class AuthorityViolation(Exception):
    """Raised when a tool call exceeds its authority boundary — a finding."""

def invoke_tool(tool_name: str, caller_role: str, args: dict[str, Any],
                approve: Callable[[ToolPolicy, dict], bool]) -> Any:
    policy = TOOL_REGISTRY.get(tool_name)
    if policy is None:
        raise AuthorityViolation(f"tool '{tool_name}' not in registry")

    if caller_role not in policy.allowed_callers:
        audit_log.security_event("authority_violation",
                                 tool=tool_name, caller=caller_role)
        raise AuthorityViolation(f"caller '{caller_role}' not allowed for {tool_name}")

    if policy.gate is not GateType.NONE:
        # Synchronous gate — approve() returns False if reviewer rejects or times out.
        if not approve(policy, args):
            audit_log.event("gate_rejected", tool=tool_name, args=args)
            raise AuthorityViolation(f"gate {policy.gate.value} rejected for {tool_name}")

    # Final blast-radius enforcement
    if args.get("affected_count", 1) > policy.max_blast_radius:
        raise AuthorityViolation(
            f"blast radius {args['affected_count']} > cap {policy.max_blast_radius}")

    result = _execute(tool_name, args)
    audit_log.event("tool_executed",
                    tool=tool_name, caller=caller_role, rollback=policy.rollback_strategy)
    return result
```

What each section does:

- **`GateType` enum** encodes the *menu* of gate strengths so policy is declarative, not buried in conditional code.
- **`TOOL_REGISTRY` is allowlist semantics.** A tool that isn't in the registry cannot be called. New tools require a security review — the registry entry IS the review artifact.
- **Caller allowlist** enforces the OWASP "execute in user's context" mitigation — not all callers may invoke all tools.
- **Gate enforcement** is synchronous and fail-closed. A timeout or rejection raises; there is no "approve by default" path.
- **Blast-radius cap** catches the "bulk-update" excessive-agency variant where an action that's safe at N=1 becomes catastrophic at N=10,000.
- **Audit log on every event** — execution, rejection, violation — produces the trail that proves the gate worked.

## 5. Real-world Patterns

**E-commerce — refund-bot authority bounding.** A 2025 e-commerce platform built an LLM agent to process refund requests. First-draft authority let the agent issue refunds up to $500 autonomously. Within a week, prompt-injection in customer-message replies tricked the agent into a $50,000 refund batch. Fix: per-action human-approval for any refund > $50, daily aggregate cap, and a separate "summarise the refund queue" tool that does NOT have refund-issuing authority. The lesson named in their incident review: "the LLM having the *right* to act and the LLM *needing* to act are different decisions" ([Galileo, 2025](https://galileo.ai/blog/prevent-excessive-agency-llms)).

**Logistics — shipment-routing agent.** A 2026 logistics platform deployed an LLM-mediated shipment-routing agent. The agent had open-ended access to a "modify_route" tool. A semantic-cloaking prompt injection (in a customer's shipping instructions) caused the agent to re-route hundreds of shipments to an attacker-controlled address. The fix replaced `modify_route(any_route)` with three purpose-built tools: `add_intermediate_stop`, `change_carrier`, `update_delivery_window`. Each tool had its own authority-boundary entry. Open-ended functions removed entirely ([Agent Patterns, 2025](https://www.agentpatterns.tech/en/architecture/human-in-the-loop-architecture)).

**SaaS productivity — calendar-deletion incident.** A 2025 productivity SaaS gave an LLM assistant calendar-management authority including delete. A misinterpreted user message ("clear out my old meetings") caused the assistant to delete six months of recurring events including team meetings. The fix added a two-person approval (assistant + user confirms with a "preview the changes" step) for any delete affecting > 5 events. The post-mortem named the over-gating risk explicitly: "we considered requiring confirmation for every change but knew users would tune it out — we put the gate at blast-radius thresholds instead" ([Strata, 2026](https://www.strata.io/blog/agentic-identity/practicing-the-human-in-the-loop/)).

**Voice-AI banking — high-risk action confirmation.** A 2026 voice-AI deployment in retail banking implemented HITL as a *voice* confirmation: any transfer over $1,000 required the user to speak a one-time spoken code aloud before the action executed. Failure to respond within 30 seconds aborted the action. The lesson: the gate doesn't have to be a "click here to approve" modal — it can be a modality-appropriate confirmation, but it must be present, synchronous, and audited ([Agent Patterns, 2025](https://www.agentpatterns.tech/en/architecture/human-in-the-loop-architecture)).

## 6. Best Practices

- **Build the authority-boundary table BEFORE shipping the tool.** Reverse-engineering boundaries from an existing deployment misses the easy wins (tools that should not have existed).
- **Allowlist tools, not denylist.** A tool that isn't in the registry cannot be called.
- **Replace open-ended functions with purpose-built ones.** `execute_shell` is never an answer.
- **Risk-tier the gate, not bypass-or-gate-everything.** Low-risk reversible → log only. Medium → confidence routing. High-risk irreversible → synchronous human approval.
- **Cap blast radius per tool.** A safe action at N=1 can be catastrophic at N=10,000.
- **Audit every gate event.** Approvals, rejections, timeouts, violations. The audit log is your compliance proof.
- **Re-test gate latency under load.** A gate that times out and falls open on heavy load isn't a gate.
- **Name what does NOT need a gate.** The "no gate, read-only" row is as load-bearing as the "two-person approval" row.

## 7. Hands-on Exercise

**Whiteboarding prompt (10–15 min):** The worked example in §3.3 covered a generic document-management agent surface. For this exercise, build the **8-endpoint × 4-column** authority-boundary table for **your pair-project's agent endpoints** (grants-portal, contract-payment, or FOIA-response — whichever your pair owns). If your pair has fewer than eight named agent tools today, list the eight you would expect by W5; if you have more than eight, pick the eight with the largest authority surface.

For each row: name the allowed callers (user / admin / service), the AI authority (what action the LLM takes through this tool), the gate type (none / logged / confirm / two-person), and the rollback strategy.

Then answer two follow-ups:

1. Identify the two tools in your table most likely to be flagged for over-grant in an OWASP LLM06 audit, and explain which of the three LLM06 root causes (excessive functionality / excessive permissions / excessive autonomy) each falls under.
2. Identify one tool in your table that should have **no gate** despite touching production data, and explain why adding a gate would be over-gating (gate fatigue, no risk reduction, downstream gate exists, etc.).

**What good looks like:** the candidate produces a complete eight-row table whose shape matches §3.3's worked example. Read-only retrievals are correctly un-gated; mutating actions have gates calibrated to blast radius (single-record mutation = confirm, multi-record or delete = two-person, bulk = two-person + blast-radius cap + dry-run). The two over-grant flags name a tool whose authority is broader than its purpose (excessive functionality), and a tool whose service-account scope exceeds what it needs (excessive permissions). The candidate also names that the table itself — not just the gates — is the deliverable HITL #6 teaches, and recognises it as the input artifact to HITL #5's runtime gate mechanics and HITL #7's auto-remediation closure.

## 8. Key Takeaways

- LLM06 has three root causes (excessive functionality, permissions, autonomy) — can I name which one applies to a specific finding?
- The five canonical mitigations (minimize extensions, minimize functionality, avoid open-ended, minimize permissions, execute in user context) map 1:1 onto authority-boundary table columns — am I producing the table?
- A complete authority-boundary table names the human gate AND what does NOT need a gate — am I avoiding the gate-fatigue anti-pattern?
- The same artifact satisfies OWASP LLM06, EU AI Act Article 14 (Aug 2026), and NIST AI RMF — am I producing it once, citing it three ways?
- Blast-radius caps, allowlist tool registries, and audited synchronous gates are the runtime enforcement — are my gates fail-closed on timeout, and do my logs prove the boundary held?

## Sources

1. [OWASP GenAI Security Project — LLM06:2025 Excessive Agency (canonical entry)](https://genai.owasp.org/llmrisk/llm062025-excessive-agency/) — retrieved 2026-05-26
2. [OWASP LLM06: Excessive Agency in AI Systems (Indusface, 2025)](https://www.indusface.com/learning/owasp-llm-excessive-agency/) — retrieved 2026-05-26
3. [Tame Excessive Agency in Your LLMs to Avoid Costly Mistakes (Galileo, 2025)](https://galileo.ai/blog/prevent-excessive-agency-llms) — retrieved 2026-05-26
4. [Human-in-the-Loop Architecture: When Humans Approve Agent Decisions (Agent Patterns, 2025)](https://www.agentpatterns.tech/en/architecture/human-in-the-loop-architecture) — retrieved 2026-05-26
5. [Human-in-the-Loop: A 2026 Guide to AI Oversight (Strata, 2026)](https://www.strata.io/blog/agentic-identity/practicing-the-human-in-the-loop/) — retrieved 2026-05-26

Last verified: 2026-05-26
