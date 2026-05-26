---
template: research-brief
tech: OWASP LLM Top 10
version_pinned: "2025 (v2.0), published March 2025 — current canonical revision as of 2026-05-22"
last_verified: 2026-05-22
last_verified_via: WebSearch+WebFetch (fallback; web-research unavailable)
recency_window: hot-tech 3mo
sources_count: 3
target_weeks: [W01-Thu-Fri, W02]
candidates_deferred:
  - source: "OWASP Top 10 for Agents 2026 (DeepTeam framework page)"
    url: "https://www.trydeepteam.com/docs/frameworks-owasp-top-10-for-agentic-applications"
    published: "unclear; appears post-2026-02 but the OWASP-published canonical list for LLMs remains 2025"
    reason: "Third-party reframe of an in-progress 2026 update. Not citable as the canonical list until OWASP publishes 2026 themselves."
known_bad_patterns_flagged:
  - note: "User-supplied scope mentioned 'LLM07 Insecure Plugin Design' — that was the 2023 v1.x list. In the current 2025 (v2.0) list, LLM07 is 'System Prompt Leakage'. Brief surfaces this delta explicitly so the curriculum author corrects it before the W1 deck ships."
known_bad_patterns_checked: true
author: research-subagent
---

# Tech research brief — OWASP LLM Top 10

Last verified: 2026-05-22 · Recency window applied: hot-tech 3mo · Pinned version: 2025 (v2.0), pub. March 2025

## 1. What it is (3–5 sentences, no jargon)

The OWASP Top 10 for Large Language Model Applications is the security-community-vetted list of the highest-risk failure modes for LLM-backed systems. The current canonical revision is the **2025 list (sometimes called v2.0)**, published in March 2025 — it superseded the 2023/v1.x list. The cohort uses it as the threat-modelling spine for W2 (RAG security) and as the backdrop for the W1 introduction to LLM-application risk.

## 2. Current stable state (as of `last_verified`)

- **Canonical list version: 2025 (v2.0), published 2025-03.** No OWASP-published 2026 revision as of 2026-05-22. Third-party "2026 update" posts exist (Repello, ScanMyLLM, DeepTeam Agents) but these reflect community speculation / agent-specific reframes, not an OWASP release.
- Active list (LLM01:2025 – LLM10:2025):

  | ID | Title | One-line |
  |----|-------|----------|
  | LLM01 | Prompt Injection | User or retrieved-content prompts alter LLM behaviour; **indirect/RAG-borne** variant is the most-exploited subclass |
  | LLM02 | Sensitive Information Disclosure | PII / credentials / business logic leaking through completions or system-prompt exposure |
  | LLM03 | Supply Chain | Compromised model weights, datasets, embeddings, plugins, or model-hosting provider |
  | LLM04 | Data and Model Poisoning | Pre-training, fine-tuning, or embedding-corpus tampering |
  | LLM05 | Improper Output Handling | Downstream systems trusting LLM output without validation/sanitisation (SQLi, XSS via LLM) |
  | LLM06 | Excessive Agency | Tools/permissions wider than the model's task requires |
  | LLM07 | **System Prompt Leakage** *(new in 2025; replaces 2023's "Insecure Plugin Design")* | Internal system prompts leak via prompt-injection or model introspection |
  | LLM08 | Vector and Embedding Weaknesses *(new in 2025)* | RAG-store poisoning, cross-tenant embedding leakage, inversion attacks |
  | LLM09 | Misinformation | Confident-sounding hallucinations consumed as fact |
  | LLM10 | Unbounded Consumption | Resource exhaustion / DoS / cost-runaway via prompt amplification |
- Breaking changes in last 3 months: none from OWASP itself. The list has been stable since 2025-03.
- Known-bad-pattern check: pass (no KBP file entries for OWASP). **But:** user's supplied scope referenced "LLM07 Insecure Plugin Design" — that's the 2023 list ID. The 2025 LLM07 is "System Prompt Leakage". Curriculum author must update.

## 3. What we teach (and what we deliberately don't)

- **In scope (W1–W2):** All ten LLM01–LLM10:2025 entries as headline cards; deep-dive on LLM01 (prompt injection, especially indirect via retrieved RAG content), LLM03 (supply chain — model + embedding + plugin provenance), LLM06 (excessive agency — directly applies to LangChain `create_agent` tool design), LLM07 (system prompt leakage — relevant to system-prompt cache design in W2), LLM08 (vector/embedding weaknesses — directly applies to W2 Atlas Vector Search multi-tenant setup).
- **Out of scope:** OWASP Top 10 for Agentic Applications (in-progress, not canonical), OWASP ASVS LLM cross-walk (Phase 3 material).
- **Misconceptions to pre-empt:** (a) "There's a 2026 list." — not from OWASP itself, as of 2026-05-22. Cite 2025. (b) "LLM07 is Insecure Plugin Design." — outdated 2023 framing; LLM07:2025 is System Prompt Leakage. (c) "LLM06 Excessive Agency is about model autonomy." — partly; it's specifically about *over-broad tool/permission grants*, which the cohort directly designs in `create_agent`.

## 4. Recommended primary sources

- [OWASP Top 10 for LLM Applications (OWASP project page)](https://owasp.org/www-project-top-10-for-large-language-model-applications/) — accessed 2026-05-22. Authoritative OWASP project landing; references current 2025 list.
- [OWASP GenAI Security Project — LLM Top 10 archive](https://genai.owasp.org/llm-top-10/) — accessed 2026-05-22. Canonical entry-by-entry detail for LLM01–LLM10:2025.
- [OWASP project GitHub repo](https://github.com/OWASP/www-project-top-10-for-large-language-model-applications/) — accessed 2026-05-22. Source-of-truth for any in-progress 2026 work; check before W2 in case OWASP publishes during the cohort.

## 5. Code reference snippets (idiomatic, current API)

N/A — OWASP Top 10 is a taxonomy, not an API. Cohort artefacts cite by ID:

> "This control mitigates **LLM01:2025 Prompt Injection (indirect, RAG-borne)** by sanitising retrieved-document content before injection into the system prompt, and **LLM08:2025 Vector and Embedding Weaknesses** by enforcing `tenant_id` pre-filtering in the `$vectorSearch` filter clause."

## 6. Risks and watch-items

- OWASP is widely expected to publish a 2026 update; check `genai.owasp.org/llm-top-10/` before W2 (2026-06-01).
- Third-party "2026 OWASP LLM Top 10" posts already circulate — instructors should NOT cite them as canonical.
- Agentic-applications list (separate OWASP track) may merge into the LLM list or stay separate — watch through 2026.

## 7. Alternatives / companion frameworks

- MITRE ATLAS — adversarial ML threat matrix; pairs with OWASP for a STRIDE-style threat model.
- NIST AI RMF 1.0 — risk-management framework, less prescriptive than OWASP; relevant for the federal-modernization angle in W3+.

## 8. Brief sign-off

- Drafted by: research-subagent (2026-05-22)
- Reviewed against `known-bad-patterns.yml`: pass (no entries). Flagged: user-supplied scope referenced the deprecated 2023 LLM07 framing — surfaced in frontmatter.
- 1-month-release check: not triggered (2025 list is the canonical revision; no OWASP-published change since 2025-03).
- Approved for downstream artifact authoring: 2026-05-22

## Sources

- OWASP Top 10 for LLM Applications project page, owasp.org, retrieved 2026-05-22 via WebFetch. <https://owasp.org/www-project-top-10-for-large-language-model-applications/>
- OWASP GenAI Security Project — LLM Top 10, genai.owasp.org, retrieved 2026-05-22 via WebFetch. <https://genai.owasp.org/llm-top-10/>
- OWASP project GitHub repo, github.com, retrieved 2026-05-22 via WebSearch. <https://github.com/OWASP/www-project-top-10-for-large-language-model-applications/>
