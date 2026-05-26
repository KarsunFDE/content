---
week: W05
day: Wed
topic_slug: ai-sre-pattern-walkthrough
topic_title: "AI-SRE Pattern Walkthrough — 6 patterns"
parent_overview: W05/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://www.datadoghq.com/blog/bits-ai-sre/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://incident.io/blog/what-is-ai-sre-complete-guide-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://rootly.com/sre/ai-sre-agent-ai-changing-incident-response-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://devops.com/part-1-death-of-the-toil-how-ai-agents-are-replacing-traditional-runbooks/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://ciroos.ai/blogs/ai-for-sres-the-power-of-cross-domain-correlation-in-root-cause-analysis
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# AI-SRE Pattern Walkthrough — 6 patterns

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the **six canonical AI-SRE patterns** (investigation, correlation, root-cause, fix-PR generation, runbook augmentation, auto-remediation) and explain what each *adds* beyond traditional observability.
- Distinguish the *diagnostic* patterns (1–3) from the *action-producing* patterns (4–6) and explain why human-in-the-loop concerns concentrate on the latter.
- Describe the **fix-PR generation pattern** (Pattern 4) and explain why it is the signature pattern of AI-SRE — the one that distinguishes it from older AIOps.
- Sketch the **learning loop** between Pattern 5 (runbook augmentation) and Pattern 1 (investigation), and explain why each incident makes the next investigation faster.

## 2. Introduction

The phrase *"AI-SRE"* gets used to mean many things in 2026 — sometimes a vendor product, sometimes a job title, sometimes a generic banner over any LLM-flavoured operations tool. To make the term useful you need a pattern vocabulary: a small set of named behaviours an AI-SRE system can exhibit, separately from any one vendor's marketing.

The pattern set that has consolidated in the practitioner literature has six entries: **investigation, correlation, root-cause, fix-PR generation, runbook augmentation, and auto-remediation**. The first three are diagnostic: they answer *what's happening* and *why*. The next three are action-producing: they propose, codify, or execute a response. The split matters because the trust and governance questions concentrate almost entirely on patterns 4–6. Patterns 1–3 are essentially "smarter dashboards" — they may be wrong, but they cannot mis-fire a fix.

Walking through all six patterns once gives you a shared vocabulary for the rest of W5 and for Friday's Final Adversarial Review PR. The exercise is *not* "memorise the list" — it's "for any AI-SRE tool you evaluate this week, classify which patterns it implements, at what stage of the graded-autonomy ladder, and with which authority boundaries". That classification is the deliverable.

## 3. Core Concepts

### 3.1 Pattern 1 — Investigation

**What it does.** When an alert fires, an investigation pattern automatically: pulls the alert's metric, traces, and log context; queries adjacent services; forms a hypothesis ("this looks like a connection-pool starvation event"); and writes a summary into the incident channel.

**What it adds.** Traditional monitoring gives you the alert. The investigation pattern gives you the *narrative* — a paragraph that lets the on-call engineer skip ten minutes of dashboard-pivoting. By the time the engineer opens their laptop, the agent has already produced a working hypothesis ([Datadog, *Bits AI SRE*](https://www.datadoghq.com/blog/bits-ai-sre/), retrieved 2026-05-26).

**Failure modes.** Confident-but-wrong narratives. The pattern is most dangerous when its summary is plausible but the supporting evidence is thin. Mitigation: every investigation produces a citation trail back to the spans, metrics, or log lines that grounded it.

### 3.2 Pattern 2 — Correlation

**What it does.** Cross-references signals across telemetry domains (metrics, logs, traces, deploys, config changes) to surface causal candidates the human would not notice without a query they didn't know to write. Example: *"latency on service-B spiked at 14:30; service-A had a config push at 14:28; here are the two changes to that configmap."*

**What it adds.** It compresses the *change-correlation* step of incident response — historically the longest manual step — to single-digit seconds.

**Failure modes.** Spurious correlations. A change three minutes before a spike may be unrelated. Good correlation surfaces *candidates* with explicit confidence, not a single "the cause is X" claim ([Ciroos: Cross-Domain Correlation in Root Cause Analysis](https://ciroos.ai/blogs/ai-for-sres-the-power-of-cross-domain-correlation-in-root-cause-analysis), retrieved 2026-05-26).

### 3.3 Pattern 3 — Root-Cause Analysis (RCA)

**What it does.** Goes beyond correlation to a defended causal claim, often combining trace propagation, dependency graphs, and historical similar-incident matching.

**What it adds.** A *process-gap* call-out — not just *which span was slow* but *what about the system's design made this incident possible*. The 2026 practitioner literature consistently emphasises that the RCA's value is naming process gaps, not symptoms.

**Failure modes.** Over-fitting to past incidents. If the agent has only seen the same root cause twice, it will pattern-match the third novel incident into the same bucket. Mitigation: the RCA pattern reports a *similarity score* to past incidents and explicitly flags "novel-shape" incidents for higher human review.

### 3.4 Pattern 4 — Fix-PR Generation

**What it does.** Drafts a code change, configuration change, or migration that would close the root cause. Output is a pull request (or equivalent diff) with a description, linked evidence, and a regression test where applicable. The PR is **not auto-merged**; a human reviews and merges.

**What it adds.** This is the *signature* pattern that distinguishes 2026 AI-SRE from older AIOps. Older AIOps surfaced what was wrong; AI-SRE proposes how to fix it. The fix-PR is the "human-in-the-loop made operational" pattern — the agent moves all the way to a concrete, reviewable change without taking any action on production.

**Failure modes.** Drive-by fixes that mask the underlying issue (e.g., bumping a timeout to silence an alert). Mitigation: a competent fix-PR pattern includes a regression test that would have *caught* the original symptom; if the test cannot be written, the fix is flagged for human re-scoping ([incident.io: AI SRE Explained](https://incident.io/blog/what-is-ai-sre-complete-guide-2026), retrieved 2026-05-26).

### 3.5 Pattern 5 — Runbook Augmentation

**What it does.** After an incident, proposes a new runbook entry (or amends an existing one) capturing the diagnostic path that worked. New incidents draw on the amended runbook in Pattern 1 (investigation), making the next instance of the same problem faster to triage.

**What it adds.** Organisational memory. A team's runbook traditionally goes stale; the augmentation pattern keeps it living. The 2026 trend is to store runbooks alongside post-mortems and recent deploys in a vector index the investigation pattern can query semantically ([DevOps.com: Death of the Toil](https://devops.com/part-1-death-of-the-toil-how-ai-agents-are-replacing-traditional-runbooks/), retrieved 2026-05-26).

**Failure modes.** Runbook bloat. A proposed entry for every incident produces an unusable runbook. Mitigation: the augmentation pattern proposes *condensations* — merging similar new entries into existing sections — and the human accepting the augmentation is responsible for trimming.

### 3.6 Pattern 6 — Auto-Remediation

**What it does.** Executes a bounded action against production (restart a pod, scale a pool, rotate a credential, reset a circuit breaker). Subject to the graded-autonomy ladder discussed in the HITL #7 reading — most teams reserve Pattern 6 to a small set of well-known action classes.

**What it adds.** Throughput. The first five patterns reduce mean-time-to-diagnose. Pattern 6 reduces mean-time-to-restore.

**Failure modes.** All the dimensions named in the HITL #7 reading — reversibility, blast-radius, audit posture, novelty, vendor coupling, authority delegation — concentrate here. A platform that ships Pattern 6 without explicit policy on each dimension is dangerous regardless of the agent's diagnostic quality.

### 3.7 The learning loop — Pattern 5 ↔ Pattern 1

The patterns are not a linear pipeline; they form a closed loop. Pattern 1 produces an investigation. Pattern 3's RCA captures the causal chain. Pattern 5's runbook augmentation codifies the diagnostic path. Pattern 1's *next* run draws on the augmented runbook. Each incident makes the next one faster to triage and, where appropriate, faster to remediate via Pattern 6.

```
       ┌────────────────────────────────────────────┐
       │                                            │
       ▼                                            │
  ┌──────────┐    ┌───────────┐   ┌─────────────┐  │
  │ 1. Invest│ ─► │ 2. Correl │─► │ 3. RCA      │  │
  └──────────┘    └───────────┘   └──────┬──────┘  │
                                          │         │
                                          ▼         │
                                  ┌───────────────┐ │
                                  │ 4. Fix-PR     │ │
                                  └───────┬───────┘ │
                                          │         │
                                          ▼         │
                                  ┌───────────────┐ │
                                  │ 5. Runbook    │─┘
                                  │   augment     │
                                  └───────┬───────┘
                                          │
                                          ▼
                                  ┌───────────────┐
                                  │ 6. Auto-remed │  (bounded; HITL-7-gated)
                                  └───────────────┘
```

## 4. Generic Implementation

A short illustrative pseudocode walkthrough (vendor-agnostic, no Karsun framing):

```python
# pseudo-sre-agent.py — illustrative only
def on_alert(alert):
    # Pattern 1: investigation
    context = telemetry.pull_context(alert)  # metrics + traces + logs around alert window
    hypothesis = reasoner.summarise(context)

    # Pattern 2: correlation
    candidates = telemetry.correlate(window=alert.window, signals=["deploys","config","metrics"])

    # Pattern 3: root cause
    rca = reasoner.rca(hypothesis, candidates, similar_past=memory.search(hypothesis))

    # Pattern 4: fix-PR
    if rca.confidence > 0.7 and rca.is_actionable:
        pr = code.draft_fix(rca, repo=service.repo, tests=True)
        github.open_pr(pr, reviewers=on_call())

    # Pattern 5: runbook augmentation
    if not memory.has_similar_runbook(rca):
        runbook.propose_entry(rca, evidence=context, awaiting_human=True)

    # Pattern 6: auto-remediation (policy-gated)
    action = policy.lookup(rca.class)
    if action.stage == "autonomous" and action.reversible:
        execute(action, audit_to=audit_log, actor="agent")
    elif action.stage == "approval_gated":
        propose(action, approver=on_call())
    else:
        escalate(rca, page=on_call())
```

What the snippet illustrates: each pattern is a distinct *capability*; the agent's *behaviour* on any given alert is the composition the platform's policy permits. The same diagnostic stack can power any of the six patterns or only the first three — the choice is governance, not architecture.

## 5. Real-world Patterns

**E-commerce (logistics platform, parcel-tracking outage).** The platform's published 2026 incident review walks the patterns explicitly: Bits AI–style investigation pulled the failing carrier API into the timeline; correlation surfaced a config push thirty minutes earlier that changed the timeout from 5 s to 1 s; RCA named the process gap (no canary on third-party API config changes); the fix-PR raised the timeout and added a canary check; runbook augmentation added a "always canary third-party config" entry. No Pattern 6 fired — the team's policy is approval-gated for any config push.

**Fintech (real-time payments processor).** The team uses Patterns 1–3 across the board, Pattern 4 in approval-gated mode on stateless services only, Pattern 5 only with human approval on each entry, and Pattern 6 only on two action classes (pod restart on OOM, read-replica failover). Their published rationale: the payments domain absorbs the *diagnostic* benefit of AI-SRE entirely, but the *action* benefit is capped by regulatory governance.

**Healthcare (national hospital network).** Patterns 1–3 are live; Pattern 4 generates PRs but only for non-clinical infrastructure; Pattern 5 augments runbooks but every entry is human-approved before it influences future investigations; Pattern 6 is disabled across the board. The team's reasoning: the *audit posture* of clinical systems makes any non-human writer indefensible. The diagnostic patterns alone are reported to have cut mean-time-to-diagnose by more than half.

**Gaming (live-service platform, matchmaking outage).** Pattern 1 produced an investigation; Pattern 2 correlated the spike with a regional CDN config change; Pattern 3 named the root cause (a CDN rule excluded the matchmaking endpoint from cache); Pattern 4 produced a PR that re-included it; Pattern 6 (region failover) was *not* triggered because the agent's policy held it in approval-gated mode for region-scoped actions. The studio's published 2026 retrospective treats this as a *good* outcome — the diagnostic patterns were fast; the action pattern stayed where the policy put it.

## 6. Best Practices

- **Adopt the patterns in order.** Patterns 1–3 first; trust patterns 4–5 next; pattern 6 last and only on narrow, reversible action classes.
- **Make each pattern's output cite its evidence.** A bare assertion ("the root cause is X") without trace IDs, log lines, or metric points is failure-prone and undefensible in post-mortem.
- **Treat Pattern 4's regression test as load-bearing.** A fix-PR without a test that would have caught the original symptom is a band-aid; the test discipline is what keeps the pattern honest.
- **Cap runbook augmentation with a human curator.** The augmentation pattern proposes; the human prunes. Without pruning, runbooks bloat past usefulness.
- **Separate the policy layer from the diagnostic layer.** The same agent that runs Patterns 1–3 should not also be the actor that executes Pattern 6 — at minimum, the action layer has its own audit trail with `actor=agent` and a trace ID linking back to the diagnosis.
- **Re-evaluate the policy when an incident exits the agent's pattern set.** Every incident the agent couldn't pattern-match is a hint that the runbook or the agent's training corpus needs a new entry.

## 7. Hands-on Exercise

**Whiteboarding prompt (15 min).** Pick a past incident from a system you've worked on — not from federal acquisitions; use a system from your prior experience. For each of the six patterns, fill in *what the AI-SRE behaviour would have been on this incident*:

| Pattern | What the agent would have produced (one sentence) |
|---|---|
| 1. Investigation | |
| 2. Correlation | |
| 3. RCA | |
| 4. Fix-PR | |
| 5. Runbook augmentation | |
| 6. Auto-remediation | (or "blocked by policy — reason:") |

**What good looks like.** A good answer treats each pattern as a *distinct* output, not as one big "AI response". The fix-PR should be one or two concrete files changed, not "the agent fixes it". The Pattern 6 row should *very often* be "blocked by policy — reason:" — that is the realistic answer for most production incidents in 2026, and the *reason* is the load-bearing part.

## 8. Key Takeaways

- *What are the six canonical AI-SRE patterns, and which are diagnostic vs action-producing?*
- *Why is Pattern 4 (fix-PR generation) the signature pattern that distinguishes 2026 AI-SRE from older AIOps?*
- *How does Pattern 5 (runbook augmentation) feed back into Pattern 1 (investigation), and why does that loop matter?*
- *Which pattern carries all the governance weight discussed in the HITL #7 reading, and why?*

## Sources

1. [Introducing Bits AI SRE, your AI on-call teammate — Datadog](https://www.datadoghq.com/blog/bits-ai-sre/) — retrieved 2026-05-26
2. [AI SRE explained: what it is, how it works, and the human vs. AI reality — incident.io](https://incident.io/blog/what-is-ai-sre-complete-guide-2026) — retrieved 2026-05-26
3. [What Is an AI SRE Agent? How AI Is Changing Incident Response in 2026 — Rootly](https://rootly.com/sre/ai-sre-agent-ai-changing-incident-response-2026) — retrieved 2026-05-26
4. [Death of the Toil: How AI Agents Are Replacing Traditional Runbooks — DevOps.com](https://devops.com/part-1-death-of-the-toil-how-ai-agents-are-replacing-traditional-runbooks/) — retrieved 2026-05-26
5. [AI for SREs: Cross-Domain Correlation in Root Cause Analysis — Ciroos](https://ciroos.ai/blogs/ai-for-sres-the-power-of-cross-domain-correlation-in-root-cause-analysis) — retrieved 2026-05-26

Last verified: 2026-05-26
