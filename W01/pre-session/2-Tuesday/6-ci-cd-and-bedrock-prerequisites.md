---
week: W01
day: Tue
topic_slug: ci-cd-and-bedrock-prerequisites
topic_title: "CI/CD + cloud-LLM prerequisites — GHA workflows, OIDC-to-cloud, model-access verification"
parent_overview: W01/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 8
sources:
  - url: https://github.com/aws-actions/configure-aws-credentials
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.github.com/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://aws.amazon.com/blogs/security/use-iam-roles-to-connect-github-actions-to-actions-in-aws/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://devopscube.com/github-actions-oidc-aws/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.freecodecamp.org/news/how-to-set-up-openid-connect-oidc-in-github-actions-for-aws/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# CI/CD + cloud-LLM prerequisites — GHA workflows, OIDC-to-cloud, model-access verification

## 1. Learning Objectives

By the end of this reading, the learner can:

- Explain why **long-lived cloud credentials stored as CI secrets** is the antipattern OIDC-to-cloud replaces, and articulate the security argument in one sentence.
- Read a `.github/workflows/deploy.yml` and identify the three load-bearing OIDC elements: `permissions: id-token: write`, the IAM role ARN, and the trust-policy claims that scope which repo/branch can assume the role.
- Name the default session-credential lifetime (1 hour) returned by AWS STS `AssumeRoleWithWebIdentity` and the configurable range (15 min–12 hr).
- Describe **per-model access entitlement** at the cloud-LLM-platform level (Bedrock model access, GCP Vertex AI model enablement) as a separate concern from code-level IAM permissions.

## 2. Introduction

CI/CD pipelines in 2024-2026 land on the same set of standard moves: GitHub Actions (GHA) is the dominant CI runner for new projects; deploys flow into AWS, GCP, or Azure; the credential model is **OIDC-to-cloud short-lived tokens**, not long-lived secrets. The AWS Security blog from 2023 captured the rationale that has only grown stronger since: "Use temporary credentials when possible. OIDC is recommended because it provides temporary credentials and it's easy to set up." This reading walks the OIDC pattern, the workflow-level configuration shape, and the cloud-LLM model-access concern that overlays it.

The GitHub official documentation goes one step further on the framing: "GitHub recommends using GitHub's OIDC provider to get short-lived AWS credentials needed for your actions." Not "consider"; not "for security-sensitive deploys"; *recommends*. That signal is worth taking seriously — every greenfield project at a federal-systems integrator should default to OIDC, and every brownfield CI pipeline storing long-lived `AWS_ACCESS_KEY_ID` in GitHub Secrets is on a remediation list.

The companion concern — **model access at the cloud-LLM platform** — is sometimes confused with code-level IAM permissions. They're distinct. Bedrock model access is an account-level entitlement (your AWS account is or is not allowed to call Anthropic Claude models in `us-east-1`); IAM permission is a code-level grant (your application's IAM role is or is not allowed to call `bedrock:InvokeModel`). Both must be in place. The first is a console / support-ticket concern; the second is a code / Terraform concern.

## 3. Core Concepts

### 3.1 The antipattern: long-lived cloud keys as CI secrets

The pre-OIDC pattern, still visible in many older CI pipelines, looks like this in `.github/workflows/deploy.yml`:

```yaml
- uses: aws-actions/configure-aws-credentials@v1
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1
```

The values stored in `secrets.AWS_ACCESS_KEY_ID` and `secrets.AWS_SECRET_ACCESS_KEY` are **long-lived IAM-user credentials**. The freecodecamp.org 2024 guide on OIDC enumerates the failure modes: secrets leak through CI logs, exfiltrated from a developer's machine, or get committed by mistake; the credentials remain valid until rotated manually; departed employees' access lingers because there's no expiry. The aws-actions/configure-aws-credentials README states the security position plainly: **"Use temporary credentials when possible."** OIDC is the mechanism for temporary credentials in GitHub Actions.

### 3.2 What OIDC-to-cloud actually does

OIDC (OpenID Connect) is the protocol layer on top of OAuth 2.0 that adds identity assertions. In the CI-to-cloud context, OIDC works like this:

1. GitHub runs your workflow. The workflow declares `permissions: id-token: write`, which allows the runner to request an OIDC token from GitHub's token service.
2. The runner requests a JWT from `https://token.actions.githubusercontent.com`. The JWT contains claims like `repo:KarsunFDE/acquire-gov:ref:refs/heads/main`, `actor:jestercharles`, plus standard JWT claims (`iss`, `aud`, `exp`).
3. The workflow calls AWS STS via `aws-actions/configure-aws-credentials`, passing the JWT and an IAM role ARN to assume.
4. AWS STS validates the JWT (signature against GitHub's JWKS, claims against the IAM role's trust policy) and, if it accepts, returns **short-lived AWS credentials** good for the configured duration (default 1 hour).
5. The workflow proceeds using those credentials.

The devopscube.com 2024 guide condenses this into one sentence: "Instead of storing long-lived credentials, GitHub Actions requests a short-lived token directly from AWS every time your workflow runs."

The architectural payoff: **no long-lived secret ever exists** that can be exfiltrated to do damage. The GitHub OIDC token is short-lived; the AWS STS-returned credentials are short-lived; the workflow's lifetime is measured in minutes. A leaked token from an old workflow run is not usable an hour later.

### 3.3 The IAM trust policy — where scoping actually happens

The IAM role's trust policy is the load-bearing security boundary. AWS will not return STS credentials for the role unless the OIDC token's claims match the trust policy's conditions. A generic trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub":
            "repo:my-org/my-repo:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

The two conditions are what stop a different repo, or even a feature branch of the same repo, from assuming this role. `:sub` is the GitHub OIDC token's subject claim, which encodes the exact `org/repo:ref:branch` running the workflow. If the workflow runs on `refs/heads/feature/x`, the `StringLike` match fails and STS returns Access Denied. AWS Security's 2023 blog post: "configure an IAM role with appropriate claim limits and permission scope."

The freecodecamp 2024 guide flags the most common mistake explicitly: leaving the `StringLike` condition as `repo:my-org/my-repo:*` (any branch, any environment), defeating the point of OIDC. Always scope to the branch (or branch pattern) that should be allowed to deploy.

### 3.4 The workflow side — three load-bearing lines

In a `.github/workflows/deploy.yml`, three pieces of configuration must be present together:

```yaml
permissions:
  id-token: write          # 1. allow this workflow to mint OIDC tokens
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Assume AWS role via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-deploy-role  # 2
          aws-region: us-east-1
          role-duration-seconds: 3600                                      # 3

      - name: Deploy
        run: ./scripts/deploy.sh
```

Annotations: (1) without this, the runner cannot mint OIDC tokens — the workflow will fail at `configure-aws-credentials` with a "no id-token" error. (2) the IAM role to assume; its trust policy must match the workflow's `org/repo:ref` subject. (3) session duration in seconds. AWS STS `AssumeRoleWithWebIdentity` defaults to 1 hour (3600 s); valid range is 15 min (900 s) to 12 hr (43200 s) per the configure-aws-credentials README.

### 3.5 Cloud-LLM model access — a separate concern

Bedrock-style cloud LLM services have a two-layer access model that catches teams new to them:

| Layer | What it controls | Where it's configured | Who manages it |
|-------|------------------|------------------------|----------------|
| **Account-level model entitlement** | Is *this AWS account* allowed to call Anthropic Claude / Meta Llama / etc. | Bedrock console → "Model access" page | Account-admin requests; AWS approves (usually instant for Anthropic models) |
| **IAM permission on the application's role** | Is *this application* allowed to call `bedrock:InvokeModel` for a specific model | IAM policy attached to the application's role | Application engineer / IaC |

Both must be true. If the account is entitled but the IAM role lacks the permission, calls fail with `AccessDeniedException`. If the IAM role has the permission but the account isn't entitled, calls fail with a model-access error in the response body. Reading the error code and message tells you which layer to fix.

The smoke test command for AWS Bedrock — `aws bedrock list-foundation-models --region us-east-1` — returns the catalog *the account has access to*. If Anthropic models do not appear in the output, that's a model-access ticket. If they appear but `aws bedrock-runtime invoke-model …` fails, that's an IAM-policy issue.

GCP Vertex AI has an analogous structure (project-level model enablement + IAM bindings); Azure OpenAI has it (resource-level model deployment + RBAC). The pattern is universal across the three cloud LLM platforms.

## 4. Generic Implementation

A canonical generic OIDC-to-cloud setup looks like the below (no Karsun-specific terms; the same shape works for any non-federal SaaS deploy):

**One-time IAM setup (run once via Terraform, CloudFormation, or CDK):**

```hcl
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]  # GitHub OIDC thumbprint
}

resource "aws_iam_role" "gha_deploy" {
  name = "gha-deploy-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Federated = aws_iam_openid_connect_provider.github.arn }
      Action    = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
        }
        StringLike = {
          "token.actions.githubusercontent.com:sub" =
            "repo:my-org/my-repo:ref:refs/heads/main"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy" "gha_deploy_perms" {
  role = aws_iam_role.gha_deploy.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["ecr:GetAuthorizationToken", "ecr:BatchCheckLayerAvailability", ...]
      Resource = "*"
    }]
  })
}
```

**Per-workflow consumption** in `.github/workflows/deploy.yml` (the three-line pattern from §3.4 above).

That's the entire dance. The IAM resource setup is permanent infrastructure (you create it once per repo or environment); the workflow consumption is per-deploy. No long-lived secret ever lives anywhere outside AWS.

## 5. Real-world Patterns

**Fintech (SaaS-startup ECS deploys).** A typical Series-A fintech with a Rails monolith on ECS Fargate uses GHA + OIDC to deploy. The trust policy scopes to `repo:fintech/api:ref:refs/heads/main` for prod and `repo:fintech/api:ref:refs/heads/staging` for staging — two separate IAM roles, each with environment-scoped permissions. The pattern: **one trust-policy condition per environment, one IAM role per environment.**

**E-commerce (Shopify-app developers).** Independent Shopify-app developers deploying to GCP Cloud Run use GHA + GCP Workload Identity Federation (GCP's OIDC equivalent). The pattern is identical in shape: federate the GitHub OIDC token, mint short-lived GCP credentials, deploy. The cross-cloud commonality of the pattern is itself the architectural insight.

**Healthcare IT (HIPAA-bounded deploys).** HIPAA auditors specifically flag long-lived cloud keys in CI as a finding. Teams in regulated industries have moved aggressively to OIDC since 2023 because it's the cleanest answer to "show me your key-rotation evidence" — the answer becomes "we don't have keys to rotate; tokens are minted per workflow run." The pattern: **OIDC as compliance-positive control**.

**Gaming (Riot Games / Epic Games — large-monorepo deploys).** Large-monorepo CI/CD often runs multiple deploy jobs in parallel; each job assumes a different role with different scopes (one for game-client artifact upload, one for player-database migrations, one for analytics-pipeline). OIDC's scalability (no shared long-lived secret to coordinate across jobs) makes this pattern operationally tractable. The pattern: **role-per-job, each scoped via OIDC subject claims**.

## 6. Best Practices

- **Default every new repo to OIDC-to-cloud from Day 1.** Greenfield projects have no excuse to use long-lived keys.
- **Scope trust-policy `:sub` to the exact branch or branch-pattern allowed to deploy.** `:*` defeats the security argument.
- **One IAM role per environment** (prod / staging / dev). Don't share roles across environments.
- **Default `role-duration-seconds` to the lowest you can tolerate** (15–60 min for most deploys). 12 hr is rarely justified.
- **Never paste long-lived cloud keys into GitHub Secrets to "just unblock the deploy."** That's how technical debt accrues. Take the day to set up OIDC properly.
- **Treat the OIDC trust-policy as code.** Manage it in IaC; review it like any other security boundary.
- **Verify cloud-LLM model access at the account level before debugging code-level IAM.** `list-foundation-models` (Bedrock) or the equivalent first; then check IAM.

## 7. Hands-on Exercise

**Read-the-workflow exercise (10 min, no code execution):**

You are reviewing a `.github/workflows/deploy.yml` for a hypothetical **e-commerce SaaS** that deploys to AWS Lambda. The workflow looks like this:

```yaml
name: deploy-prod
on:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_KEY }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET }}
          aws-region: us-east-1
      - run: ./deploy-lambda.sh
```

**Identify**:

1. What security antipattern is present?
2. What are the three changes required to convert this to OIDC?
3. What additional resource needs to be created in AWS (one-time) to support the converted workflow?
4. The trust policy on that resource needs which two `Condition` clauses to scope securely to this repo on main?

**What good looks like:** (1) long-lived AWS keys in GitHub Secrets; (2) add `id-token: write` to permissions, replace `aws-access-key-id`/`aws-secret-access-key` with `role-to-assume: arn:aws:iam::...:role/...`, remove the `secrets.AWS_KEY`/`AWS_SECRET` references; (3) an IAM role with a trust policy allowing GitHub OIDC + an IAM OIDC identity provider (if not already created); (4) `StringEquals` on `token.actions.githubusercontent.com:aud = sts.amazonaws.com` AND `StringLike` on `token.actions.githubusercontent.com:sub = repo:org/repo:ref:refs/heads/main`. If you can do this exercise on a sample repo, you can audit a real federal-client repo's `deploy.yml` Day 1 of engagement.

## 8. Key Takeaways

- *Can I explain in one sentence why long-lived cloud keys in CI are an antipattern OIDC replaces?* (Maps to LO 1.)
- *Can I read a `deploy.yml` and identify the three OIDC load-bearing elements?* (Maps to LO 2.)
- *Can I name the default STS session lifetime and the configurable range?* (Maps to LO 3.)
- *Can I distinguish account-level model entitlement from code-level IAM permission for a cloud LLM platform?* (Maps to LO 4.)

## Sources

1. [aws-actions/configure-aws-credentials — GitHub README](https://github.com/aws-actions/configure-aws-credentials) — retrieved 2026-05-26
2. [GitHub Docs — Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) — retrieved 2026-05-26
3. [AWS Security Blog — Use IAM roles to connect GitHub Actions to actions in AWS](https://aws.amazon.com/blogs/security/use-iam-roles-to-connect-github-actions-to-actions-in-aws/) — retrieved 2026-05-26
4. [DevOpsCube — How to Configure GitHub Actions OIDC with AWS](https://devopscube.com/github-actions-oidc-aws/) — retrieved 2026-05-26
5. [freeCodeCamp — How to Set Up OpenID Connect (OIDC) in GitHub Actions for AWS](https://www.freecodecamp.org/news/how-to-set-up-openid-connect-oidc-in-github-actions-for-aws/) — retrieved 2026-05-26

Last verified: 2026-05-26
