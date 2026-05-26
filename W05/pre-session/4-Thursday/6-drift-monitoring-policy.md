---
week: W05
day: Thu
topic_slug: drift-monitoring-policy
topic_title: "Drift Monitoring Policy — committed-to-prod thresholds and alert routing"
parent_overview: W05/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://docs.aws.amazon.com/sagemaker/latest/dg/model-monitor.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://www.evidentlyai.com/ml-in-production/data-drift
    retrieved_on: 2026-05-26
    recency_category: foundation-stable-12mo
  - url: https://docs.aws.amazon.com/bedrock/latest/userguide/model-lifecycle.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
  - url: https://docs.datadoghq.com/watchdog/
    retrieved_on: 2026-05-26
    recency_category: hot-tech-3mo
last_verified: 2026-05-26
---

# Drift Monitoring Policy — From Signal to Committed Threshold

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish data drift, concept drift, prediction drift, and training-serving skew, and explain when each one matters in an LLM application.
- Identify the per-endpoint shape of a drift-monitoring policy and the four fields each rule must carry (threshold, window, routing, authority).
- Choose between population-level statistics (e.g., PSI, KS) and rule-based checks for different drift signals.
- Tag a drift event by source (data, model, traffic) so the diagnostic step precedes the remediation.
- Recognise the difference between a one-incident response (a Wed war-room outcome) and a standing policy (a Thu deliverable).

## 2. Introduction

Drift monitoring is the discipline of detecting when a production AI system's behaviour starts to differ from the behaviour the system was validated for. A model's accuracy can degrade silently. A retriever's k=0 rate can creep up after a corpus change. An agent's token cost can double when a foundation-model version is rotated. None of these announce themselves. Without drift monitoring, the team learns about them from user complaints — which is too late.

A drift-monitoring **policy** is the standing artifact: per endpoint, what counts as drift, what triggers an alert, who is paged, and what authority the alert grants for remediation. A drift-monitoring **event** is a single incident: this metric, on this endpoint, crossed this threshold, at this time. The policy decides every future event. The event tests the policy.

Most teams produce events before they produce a policy. The first time `k=0` rate spikes after a Friday deploy, the team responds to the event ad-hoc. The Thu-after-launch policy formalises the response: next time, the threshold is named, the routing is named, the authority for circuit-breaker-trip is named. The Wed war-room (in the daily overview) was the event-level response; this Thu reading is the policy-level capture.

## 3. Core Concepts

### 3.1 The four flavours of drift

Evidently AI's ML-in-production guide names four related but distinct phenomena [Evidently AI data drift guide, retrieved 2026-05-26]:

- **Data drift.** The distribution of the input data shifts. The model is unchanged; the data it sees in production has moved away from the data it was trained or validated on.
- **Concept drift.** The relationship between input and target shifts. Even if the inputs look the same, the right answer for them has changed. (Classic example: customer-purchase intent during an economic shock — same purchase history, different propensity.)
- **Prediction drift.** The distribution of the model's outputs shifts. This can be a consequence of data or concept drift; it is the easiest of the four to measure because outputs are usually logged in full.
- **Training-serving skew.** The data the model sees in production differs from the data the team uses to train or evaluate. This is not a temporal drift but a systematic mismatch (e.g., the eval set was clean, production data has typos).

For LLM applications, three more drift sources show up:

- **Retrieval drift.** The corpus changes (new documents, deleted documents), and retrieval quality moves with it.
- **Model-version drift.** The foundation-model vendor rotates a model behind the same identifier, or the team upgrades a model on a managed catalog [Bedrock model lifecycle, retrieved 2026-05-26].
- **Traffic drift.** New user cohorts onboard with different query patterns.

A policy that covers only "data drift" misses model-version drift entirely. The policy needs to name which flavours it covers.

### 3.2 Detection methods

Two families dominate [Evidently AI data drift guide, retrieved 2026-05-26]:

- **Statistical / distance-based.** Population Stability Index (PSI), Kolmogorov-Smirnov test, Jensen-Shannon divergence, Wasserstein distance. Pick a method based on whether the feature is continuous or categorical and whether the baseline is fixed or rolling. Useful when the input is high-cardinality and rule-based checks would be brittle.
- **Rule-based.** Per-feature thresholds: "k=0 rate above 5% over a 1-hour window," "p95 token cost above 2× baseline," "citation-mismatch rate above 3%." Easier to reason about; easier to debug; harder to apply to many features at once. Useful when the team can name the symptom precisely.

A pragmatic policy uses both: rule-based for the dimensions the team cares about most (the ones the rubric grades on), and statistical for the high-cardinality features (input embeddings, prompt-length distribution).

Amazon SageMaker Model Monitor formalises this pattern by computing statistical baselines and comparing live data against them [SageMaker Model Monitor, retrieved 2026-05-26]. Tooling exists; what tooling cannot decide for the team is the threshold.

### 3.3 The four-field policy row

A useful policy is a table. Each row covers one signal on one endpoint:

| Field | Meaning |
|-------|---------|
| Threshold | What deviation triggers the rule? (e.g., `>5%`, `>2× baseline`, `>3-sigma`) |
| Window | Over what observation window? (`5 min`, `1 hour`, `1 day`, `rolling 7-day`) |
| Routing | Who gets paged? Which dashboard updates? Which ticket queue receives? |
| Authority | What automated remediation is permitted? Tier (A/B/C — full/partial/no) and named action (e.g., circuit-breaker trip, traffic shed, retry-with-fallback) |

A policy that has rows but no authority field is incomplete: the alert fires but no one knows who can act. A policy that has authority but no routing is incomplete: the action is permitted but no one knows when. A policy that has neither is a wishlist.

### 3.4 Drift-source tagging precedes remediation

A drift event needs a diagnostic step before any remediation: *which source produced this drift?*

- **Data source.** The corpus or input distribution changed (a new agency tenant onboarded; the upstream provider's feed format changed; the user cohort shifted).
- **Model source.** The model itself changed (vendor rotated a version; the team upgraded; a fine-tune was deployed).
- **Traffic source.** Volume or shape changed (a marketing campaign drove a 10× spike; a holiday produced unusual queries).

The same symptom — token cost p95 doubled — has very different remediations for each source. A data-source cost spike calls for prompt or chunk-size review. A model-source cost spike calls for version-pin review. A traffic-source cost spike calls for capacity or rate-limit review. A policy that does not require source-tagging produces remediations that miss the source.

### 3.5 Wed event versus Thu policy

The pattern that makes the policy stick: the team produces a one-incident response on Wednesday (the war-room) and a standing policy on Thursday. The one-incident response is the data point; the standing policy is the rule that decides the next thousand events. Teams that skip the policy step end up re-deciding every event from scratch, which scales badly. Teams that skip the event step write policies divorced from real signal and end up with rules that fire on noise or miss the real drift.

## 4. Generic Implementation

A policy as a YAML file, vendor-neutral, that monitoring tooling can consume:

```yaml
# drift-monitoring-policy.yaml
# Owned by: <policy-owner-human>
# Reviewed: 2026-05-26

policy_id: drift-monitoring-v1
flavours_covered:
  - data
  - prediction
  - retrieval
  - model-version
  - traffic

endpoints:
  - name: /search/answer
    rules:
      - rule_id: retrieval-zero-k-rate
        signal: retrieval_k_zero_rate
        threshold: ">5%"
        window: 1h
        routing:
          page: on-call-llm
          dashboard: drift-overview
          ticket: llm-search-queue
        authority:
          tier: B  # partial — allowed actions named below
          actions:
            - retry-with-fallback-corpus
        source_tags_required:
          - data  # corpus may have shifted
          - traffic  # new query types may produce k=0

      - rule_id: token-cost-p95
        signal: token_cost_p95
        threshold: ">2x baseline"
        window: 1h
        routing:
          page: on-call-cost
          dashboard: cost-overview
        authority:
          tier: C  # no auto-remediation; human-only
        source_tags_required:
          - model-version
          - data
          - traffic

  - name: /agent/triage
    rules:
      - rule_id: confirmation-acceptance-rate
        signal: user_confirmation_accept_rate
        threshold: "<70%"
        window: rolling-7d
        routing:
          dashboard: agent-quality
          ticket: agent-quality-queue
        authority:
          tier: C
        source_tags_required:
          - model-version
          - data
```

A code reader can produce three immediate operational tests from the YAML:

```python
def evaluate_rule(rule, signal_value, baseline):
    """Return True if the rule fires."""
    threshold_str = rule["threshold"]
    if "x baseline" in threshold_str:
        multiplier = float(threshold_str.split("x")[0].strip(">< "))
        return signal_value > multiplier * baseline
    if threshold_str.startswith(">"):
        return signal_value > float(threshold_str.lstrip(">% "))
    if threshold_str.startswith("<"):
        return signal_value < float(threshold_str.lstrip("<% "))
    raise ValueError(f"Cannot parse threshold {threshold_str}")

def execute_authority(rule, fired: bool, source_tag: str):
    """Execute permitted remediation only if the source tag matches."""
    if not fired:
        return
    if rule["authority"]["tier"] == "C":
        return  # no auto-remediation
    if source_tag not in rule["source_tags_required"]:
        page_human(reason="source-tag mismatch — manual diagnosis")
        return
    for action in rule["authority"]["actions"]:
        perform(action)
        write_audit_event(action=action, source_tag=source_tag)
```

Three things to notice:

- The threshold parsing is intentionally limited to a few shapes; richer policies use a real expression language but the limit forces clarity.
- Authority tier C is the no-auto-remediation default. Auto-remediation requires both a non-C tier *and* a source-tag match — defense in depth against firing on the wrong source.
- Every auto-remediation writes an AuditEvent. The audit-of-the-action requirement is policy-level, not code-level.

## 5. Real-world Patterns

**Healthcare — drift on a sepsis-risk predictor.** A hospital network's early-warning model had a documented drift-monitoring policy: PSI on key clinical features daily, alert at PSI > 0.25, page the clinical-informatics on-call. After a hospital expanded to include a new ward (pediatrics), input distributions shifted and PSI tripped. The source tag identified "data" (new patient cohort), and the remediation was a model-retraining cycle rather than a configuration tweak [Evidently AI data drift guide, retrieved 2026-05-26].

**Fintech — model-version drift on a credit-scoring model.** A consumer lender ran a model-version-pin audit: the policy required any model-version change to produce a 24-hour shadow-evaluation before traffic-rotation. After a vendor silently rotated a managed model behind an unchanged identifier, the shadow-eval detected a 4-point drop in AUC. The policy's source-tag-required field caught the source quickly; the lender raised the issue with the vendor and rolled back [Bedrock model lifecycle, retrieved 2026-05-26].

**E-commerce — retrieval drift on a product-search assistant.** A retailer's LLM product-search had a policy alert on k=0 rate > 3% over 1 hour. After a category-tree refactor, k=0 spiked. The source tag identified "data" (the corpus had restructured); the remediation was re-indexing rather than model intervention. Without the source-tag step, the team would have spent hours debugging the model.

**Gaming — concept drift on a player-churn predictor.** A studio's churn model started missing players after a competitor's launch shifted player preferences. The concept drift was subtle — input features looked normal, but the predicted-vs-observed churn diverged. The policy's prediction-drift rule caught it via a comparison of predicted distribution vs. actual outcome over a 14-day window; the remediation was a retraining cycle with the post-launch period in the training data [Evidently AI data drift guide, retrieved 2026-05-26].

## 6. Best Practices

- Cover all relevant drift flavours in the policy, not just data drift; LLM applications add retrieval and model-version drift to the standard four.
- Use both rule-based thresholds and statistical methods; rule-based for known-signal symptoms, statistical for high-cardinality inputs.
- Require source-tagging before remediation; same symptom with different sources demands different responses.
- Pair each rule with an authority tier that names what auto-remediation is permitted; never let a rule fire without specifying who or what acts.
- Commit the policy as code (YAML, Terraform, Datadog-monitor-as-code); console-clicked policies drift from the policy doc and are hard to review.
- Re-tune thresholds quarterly; thresholds that fire on noise lose on-call trust; thresholds that never fire are not protecting anything.
- Capture a one-incident response (Wed war-room) before writing the policy (Thu); policies divorced from real events fire on noise.

## 7. Hands-on Exercise

Pick an AI feature you know (or invent a plausible one — e.g., an automated email-summariser for a help desk). In 15 minutes, draft a drift-monitoring policy with at least three rules:

1. For each rule, fill in the four required fields: threshold, window, routing, authority. Use realistic units.
2. For each rule, list the drift sources it might detect (data, model, traffic, retrieval, etc.).
3. For one of the three, walk through what the team would do if the rule fired with each source tag. The diagnostic step should be different per source.

**What good looks like.** Thresholds are concrete (`>5%`, `>2× baseline`), not vague (`high`). Each rule's routing names a specific channel or queue. Each rule's authority either names auto-remediation actions or explicitly says "Tier C — human only." The source-tag walk-through produces meaningfully different actions for at least two of the sources.

## 8. Key Takeaways

- What are the four canonical drift flavours, and which additional sources matter specifically for LLM applications?
- What four fields does each policy rule require, and why is "authority" the field most often missing?
- How do rule-based thresholds and statistical methods complement each other in a single policy?
- Why must drift-source tagging precede remediation, and what is the cost of skipping the tagging step?
- How does the Wed war-room event versus Thu standing policy distinction make policies survive contact with reality?

## Sources

1. [Amazon SageMaker Model Monitor — data and model quality monitoring](https://docs.aws.amazon.com/sagemaker/latest/dg/model-monitor.html) — retrieved 2026-05-26
2. [Evidently AI — What is data drift in ML, and how to detect and handle it](https://www.evidentlyai.com/ml-in-production/data-drift) — retrieved 2026-05-26
3. [Amazon Bedrock — model lifecycle and versioning](https://docs.aws.amazon.com/bedrock/latest/userguide/model-lifecycle.html) — retrieved 2026-05-26
4. [Datadog Watchdog documentation](https://docs.datadoghq.com/watchdog/) — retrieved 2026-05-26

Last verified: 2026-05-26
