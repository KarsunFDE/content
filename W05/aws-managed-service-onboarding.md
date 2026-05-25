---
week: W05
day: Mon afternoon (gate open) + Thu afternoon (migration work)
title: "AWS managed-service onboarding checklist — D-050 carve-out lift + D-060 reconciliation"
audience: instructor + cohort
last_verified: 2026-05-23
decisions: [D-050, D-060]
account_model: Single Karsun-paid cohort #1 AWS account (per D-050)
budget_cap: $50/day cohort-wide alarm (per Mon onboarding; configurable)
spin_down_policy: OpenSearch t3.small.search dev-tier spin-down at 18:00 via EventBridge
last_verified: 2026-05-23
---

# W5 AWS Managed-Service Onboarding Checklist

> AWS managed services were deferred to W5 per D-050 (cost-blast-radius decision); Bedrock InvokeModel was carved out for W2-onward per D-060. **W5 Mon afternoon = gate open.** Thu afternoon = migration work. This file is the cohort's checklist + instructor's verification surface.

## D-050 + D-060 reconciliation (cohort understanding by EOD Mon)

| Service | Status W2–W4 | Status W5 onward | Rationale |
|---------|--------------|-------------------|-----------|
| Bedrock `InvokeModel` (Claude) | Live | Live | D-060 carve-out — LLM provider, not a managed service |
| Bedrock token/cost metering | Live | Live (fans into Datadog Tue) | D-060 carve-out |
| **Bedrock Knowledge Bases** | Deferred | **Live (Mon afternoon)** | D-050 cost-blast-radius — managed-service stack |
| **Agents-for-Bedrock** | Deferred | **Live (Mon afternoon)** | D-050 |
| **OpenSearch Managed** | Deferred | **Live (Mon afternoon)** | D-050 |
| MongoDB Atlas Vector Search | Live (carve-out N/A; Atlas not AWS-managed) | Live (migration question Thu) | D-050 carve-out — not AWS-managed |
| AWS S3 (object storage) | Live (already in use for cohort artifacts) | Live | Not a Bedrock managed service per D-050 |
| AWS Lambda | Deferred (per D-050 conservative read) | **Live if needed** — not a primary W5 target | Mentioned in D-050 § AWS deferred-to-W5 |

## Mon afternoon onboarding sequence (instructor + cohort, ~45 min total)

### 1. Service quota raises (instructor; AWS Support tickets, pre-cohort)

| Service | Quota raised | Default | Cohort #1 target |
|---------|--------------|---------|------------------|
| Bedrock | Claude 3.5 Sonnet TPM | 200,000 | 2,000,000 |
| Bedrock Agents | Agents per account | 50 | 50 (default OK) |
| Bedrock Knowledge Bases | KBs per account | 100 | 100 (default OK) |
| OpenSearch | Domains per account | 20 | 20 (default OK) |

Raises filed by instructor 1 week before W5 Mon (per `pipeline/PIPELINE.md` §17 cohort instantiation checklist).

### 2. Budget + alarm setup (instructor; AWS Console)

```
aws budgets create-budget --account-id <cohort1-account> --budget '{
  "BudgetName": "cohort1-w5-aiops",
  "BudgetLimit": {"Amount": "350", "Unit": "USD"},
  "TimeUnit": "DAILY",
  "BudgetType": "COST",
  "CostFilters": {},
  "CostTypes": {"IncludeRefund": true, "IncludeSubscription": true}
}' --notifications-with-subscribers '[{
  "Notification": {"NotificationType": "ACTUAL", "ComparisonOperator": "GREATER_THAN", "Threshold": 50},
  "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "instructor@karsun..."}]
}]'
```

Alarm fires at $50/day actual; instructor reviews + decides whether to spin down ephemeral services.

### 3. OpenSearch dev-tier spin-down policy (instructor; EventBridge)

```
aws events put-rule --name cohort1-opensearch-evening-spindown \
  --schedule-expression 'cron(0 22 ? * MON-FRI *)'  # 18:00 ET = 22:00 UTC
aws events put-targets --rule cohort1-opensearch-evening-spindown \
  --targets 'Id=1,Arn=<lambda-arn>'
```

Lambda stops the OpenSearch domain; Tuesday morning instructor confirms restart at 08:00 ET via the same pattern.

### 4. Per-pair SSO + verification (cohort, 15 min)

Each pair runs in their cohort SSO session:

```
aws sso login --profile cohort1
aws bedrock-agent list-knowledge-bases --region us-east-1
# expect: empty Knowledge-bases list
aws bedrock-agent list-agents --region us-east-1
# expect: empty Agents list
aws opensearch list-domains --region us-east-1
# expect: cohort1-acquire-gov-opensearch (state: Active or Stopped)
```

If any pair's SSO fails — instructor walks IAM trust policy live (same pattern as W1 Tue Bedrock model-access verification).

## Thu afternoon migration work (per pair)

Per `war-room/Thu.md` Acts 3 + 4 — pair owns the migration choice. Below is the *operational* checklist, not the decision logic (decision logic lives in `scenarios/W05-SA-3.md`).

### Migration 1 — Bedrock Knowledge Base wraps `/rag/clause-search`

```bash
# 1. Create the KB
aws bedrock-agent create-knowledge-base \
  --name acquire-gov-far-dfars-pair-N \
  --role-arn arn:aws:iam::<cohort1>:role/BedrockKnowledgeBaseRole \
  --knowledge-base-configuration 'type=VECTOR,vectorKnowledgeBaseConfiguration={embeddingModelArn=arn:aws:bedrock:us-east-1::foundation-model/amazon.titan-embed-text-v2:0}' \
  --storage-configuration '...'  # OpenSearch Managed-backed

# 2. Upload corpus
aws s3 sync ./data/far-dfars/ s3://cohort1-acquire-gov-far-pair-N/

# 3. Trigger ingestion
aws bedrock-agent start-ingestion-job --knowledge-base-id <kb-id> --data-source-id <ds-id>

# 4. Wrap retrieval in ai-orchestrator
# services/ai-orchestrator/app/rag/kb_clause_search.py uses
# bedrock-agent-runtime:Retrieve with filter={agency_id: <tenant>}
```

**Multi-tenant filter (Item 10) MUST be preserved.** The Bedrock KB `Retrieve` call's filter on `agency_id` metadata is non-negotiable.

### Migration 2 — Agents-for-Bedrock wraps `POST /agent/intake-triage`

```bash
# 1. Define action groups (OpenAPI schemas exported from existing services)
# Stored at: ./data/agents/solicitation-actions.yaml, evaluation-actions.yaml

# 2. Create the agent
aws bedrock-agent create-agent \
  --agent-name acquire-gov-intake-triage-pair-N \
  --foundation-model anthropic.claude-3-5-sonnet-20240620-v1:0 \
  --instruction "You are an intake triage agent for federal acquisition proposals. ..." \
  --idle-session-ttl-in-seconds 1800

# 3. Attach action groups
aws bedrock-agent create-agent-action-group \
  --agent-id <agent-id> \
  --action-group-name solicitation-fetch \
  --api-schema 'file://./data/agents/solicitation-actions.yaml' \
  --action-group-executor 'lambda=arn:aws:lambda:us-east-1:<cohort1>:function:solicitation-action-group-handler'

# 4. Prepare + test
aws bedrock-agent prepare-agent --agent-id <agent-id>
aws bedrock-agent-runtime invoke-agent --agent-id <agent-id> --session-id <pair-session> --input-text '<test proposal>'
```

**HITL fidelity check.** Bedrock Agents' user-confirmation API surfaces at action-group invocation. Pair's HITL #5 LangGraph ADR (W3) + HITL #7 ADR (W5 Wed) must align with this surface — or the pair commits to no-migrate for this endpoint.

## Verification dashboard (instructor, EOD Thu)

Per pair × per migration:

| Pair | Migration 1 (KB) | Migration 2 (Agents) | OpenSearch stretch |
|------|------------------|----------------------|---------------------|
| pair-1 | migrated / hybrid / no-migrate + ADR | ... | ... |
| pair-2 | ... | ... | ... |
| pair-3 | ... | ... | ... |

Each cell links to:
- The pair's worth-it ADR.
- The migration code commit (or no-migrate hardening commit).
- Measured latency / cost / debuggability data.

## Cost reconciliation (Fri EOD, instructor)

Instructor runs:

```bash
aws ce get-cost-and-usage \
  --time-period Start=2026-06-22,End=2026-06-26 \
  --granularity DAILY \
  --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE
```

Verifies actual spend vs $50/day alarm; surfaces to cohort retro. Outliers (e.g., one pair's Bedrock spend 5× the cohort median) feed pair-level debugging.

## Cohort #2 carry-forward

If cohort #1's actual W5 spend stays under $250 total — D-050's conservative deferral was correct + the W5 timing held. If spend overshot — surface to skill lead for re-evaluation of which services deserve earlier exposure (precedent for narrowing D-050 toward a W3 or W4 exposure for some services).
