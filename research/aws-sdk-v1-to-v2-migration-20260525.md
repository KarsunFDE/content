---
template: research-brief
tech: AWS SDK for Java — v1 to v2 migration
version_pinned: "v1: 1.12.x (END-OF-SUPPORT since 31 Dec 2025); v2: 2.44.12 (current, released 22 May 2026)"
last_verified: 2026-05-25
last_verified_via: web-research (Firecrawl self-hosted)
recency_window: foundation-stable 12mo
sources_count: 5
target_weeks: [W04, W05]
candidates_deferred: []
known_bad_patterns_flagged:
  - id: aws-sdk-v1-as-current
    note: "AWS SDK for Java 1.x reached end-of-support on 31 Dec 2025. Any tutorial showing `com.amazonaws.*` imports as the current way is now teaching unsupported software. Brief counter-teaches with `software.amazon.awssdk.*` (v2)."
  - id: aws-sdk-v1-mutable-builder
    note: "v1 used `AmazonDynamoDBClientBuilder.standard().withRegion(...).build()` with `.withX()` setters on mutable request/response POJOs. v2 is immutable: `.builder().region(...).build()` and result objects use `Response` suffix (not `Result`). Brief shows the v2 builder pattern."
  - id: aws-sdk-v1-blocking-sync-default
    note: "v1 clients are sync-blocking only. v2 ships `AsyncClient` variants returning `CompletableFuture` with non-blocking I/O on Netty. Brief teaches the sync/async split as a deliberate v2-only capability."
  - id: aws-sdk-v1-transfermanager-import
    note: "v1's `com.amazonaws.services.s3.transfer.TransferManager` is removed in v2. v2 equivalent is `software.amazon.awssdk.transfer.s3.S3TransferManager` (separate artifact, since v2.19). Brief flags the artifact name change."
known_bad_patterns_checked: true
author: research-subagent
---

# Tech research brief — AWS SDK for Java v1 → v2 migration

Last verified: 2026-05-25 · Recency window applied: foundation-stable 12mo · Pinned versions: v1 = 1.12.x EOS; v2 = 2.44.12 (current)

## 1. What it is (3–5 sentences, no jargon)

The AWS SDK for Java is the Java client library for every AWS service. Version 1.x (groupId `com.amazonaws`, package root `com.amazonaws.*`) was the original SDK released March 2010. Version 2.x (groupId `software.amazon.awssdk`, package root `software.amazon.awssdk.*`) is a **complete rewrite on Java 8+** released November 2018 — different package names, immutable POJOs with builder pattern, `Response` suffix on result types (not `Result`), async clients with `CompletableFuture` + non-blocking Netty I/O, pluggable HTTP transport, autopaginated responses, dramatically faster Lambda cold-start. For the Karsun FDE auditor question, the load-bearing fact is timing: **AWS SDK for Java 1.x is end-of-support since 31 Dec 2025** — acquire-gov is shipping with an unsupported AWS SDK, and the W4 modernization deliverable is to plan + execute the v1 → v2 migration.

## 2. Current stable state (as of `last_verified`)

- **v1 lifecycle (AWS Developer Tools Blog, originally posted Aug 2024):**
  - General Availability: 25 Mar 2010 → 30 Jul 2024
  - Maintenance mode: 31 Jul 2024 → 30 Dec 2025 (critical bug + security fixes only; no new services / regions / API updates)
  - **End-of-support: 31 Dec 2025 onward — no further releases.** Existing artifacts remain on Maven Central + GitHub but receive no patches.
- **v2 latest stable: 2.44.12 (2026-05-22).** Cadence is daily-to-weekly auto-released minor bumps tracking AWS service launches. BOM-based dependency management: import `software.amazon.awssdk:bom:2.44.12` in `<dependencyManagement>` and depend on per-service artifacts (`s3`, `dynamodb`, `bedrockruntime`, etc.) without version pins.
- **Headline v2 architectural changes** (AWS SDK for Java 2.x developer guide):
  - **Package name change.** `com.amazonaws.*` → `software.amazon.awssdk.*`. This is the mechanical scope of most migration work.
  - **Immutable POJOs + builder pattern.** v1's `new DescribeAlarmsRequest()` + mutable setters → v2's `DescribeAlarmsRequest.builder()...build()`. Request/response objects cannot be mutated after creation.
  - **Result → Response naming.** v1: `CreateApiKeyResult`. v2: `CreateApiKeyResponse`.
  - **Setter/getter rename.** v1: `.withRegion(...)`, `.getNextToken()`. v2: `.region(...)`, `.nextToken()` (no `with`/`get` prefixes).
  - **Async client family.** Every service has both `XxxClient` (sync) and `XxxAsyncClient` (non-blocking, returns `CompletableFuture`, Netty I/O under the hood).
  - **Pluggable HTTP transport.** Default is Apache HttpClient for sync, Netty for async; pluggable per client via `httpClient(...)` / `httpClientBuilder(...)`.
  - **Autopaginated responses.** v2 paginator clients (`xxxPaginator()` methods) handle continuation tokens automatically.
  - **Faster Lambda cold-start** — SDK init optimized; documented in `lambda-optimize-starttime`.
  - **Shorthand request lambdas.** `dynamoDbClient.putItem(req -> req.tableName(TABLE))` is a v2-only idiom.
- **Library/utility migration map (selected, from `migration-whats-different`):**
  - `DynamoDBMapper` (v1) → `DynamoDbEnhancedClient` (v2, since 2.12.0)
  - `Waiters` → `Waiters` (v2, since 2.15.0)
  - `CloudFrontUrlSigner` / `CloudFrontCookieSigner` → `CloudFrontUtilities` (since 2.18.33)
  - `TransferManager` → `S3TransferManager` (since 2.19.0; separate artifact)
  - EC2 IMDS metadata client → `Ec2MetadataClient` (since 2.19.29)
- **Migration tooling.** AWS publishes an **automated migration tool** (`aws-migration-tool`, an OpenRewrite-style runner) that mechanically replaces ~70-80% of v1 imports/calls with v2 equivalents; remaining patterns require manual edits per `migration-steps`. Side-by-side v1 + v2 in the same project is supported as a transitional state via the `migration-side-by-side` guide (different groupId, so no Maven conflict).
- **Credential provider chain difference.** v1's `DefaultAWSCredentialsProviderChain` → v2's `DefaultCredentialsProvider`. The chain order is similar (env vars → system properties → profile → ECS → EC2 IMDS) but the v2 version is faster and respects new IDC SSO env vars.
- **Sigv4a (multi-region signing) is v2-only** — needed for S3 Multi-Region Access Points and other cross-region features. v1 cannot produce sigv4a signatures.
- Known-bad-pattern check: **flagged** — four new patterns surfaced (v1-as-current, mutable-builder + `.withX()`, sync-only assumption, `TransferManager` import). All four common in pre-2024 AWS Java tutorials. Brief counter-teaches.

## 3. What we teach (and what we deliberately don't)

- **In scope (W4 brownfield modernization — Bedrock + S3 + DynamoDB cohort touchpoints):**
  - The **strangler-fig migration playbook**: (1) audit v1 usage with `aws-migration-tool find-apps-using-v1` (lists all `com.amazonaws` imports); (2) add the v2 BOM + v2 per-service artifacts alongside v1 (no conflict — different groupId); (3) migrate one service client at a time, leaf-first (DynamoDB → S3 → Bedrock), running tool + manual cleanup per service; (4) remove the v1 dependency only when the last `com.amazonaws` import is gone; (5) verify with a v1-import linter rule.
  - The **builder/immutable pattern** as the v2 idiom; the `Response` suffix; the `.region()` / `.nextToken()` setter shape.
  - The **sync vs async client decision tree**: cohort uses `BedrockRuntimeAsyncClient` for the FastAPI ai-orchestrator path (because FastAPI is event-loop-based); `S3Client` (sync) for the audit-log ship-to-S3 batch job (cron-driven, not latency-sensitive).
  - **Default credentials provider chain** — pinning to `DefaultCredentialsProvider.create()` and letting it find IRSA / IMDSv2 / profile credentials.
  - **`S3TransferManager`** as the v2 replacement for the v1 `TransferManager` — separate artifact (`software.amazon.awssdk:s3-transfer-manager`), CRT-backed.
- **Out of scope (deliberately):** Writing custom HTTP transport (use the default Apache/Netty); Bedrock Agents SDK surface (cohort uses LangChain `create_agent` over `BedrockRuntimeAsyncClient` directly); cross-account assume-role chains (acquire-gov is single-account for the cohort); sigv4a multi-region signing (mentioned as "v2 enables this if you ever need it" — not built).
- **Misconceptions to pre-empt:** (a) "v1 still works, so we can wait." — false; v1 is end-of-support since 31 Dec 2025; any newly-released AWS service (incl. recent Bedrock models) does not get v1 SDK support. (b) "v2 is just a rename — I can sed `com.amazonaws` → `software.amazon.awssdk` and be done." — false; the immutable-builder + Response-suffix + setter-rename changes are mechanical but invasive. (c) "Async clients are always better." — false; async adds complexity (CompletableFuture orchestration, Netty event-loop tuning); use only where the call site is async (FastAPI event loop, Kotlin coroutines, Reactor pipeline).

## 4. Recommended primary sources

- [The AWS SDK for Java 1.x is in maintenance mode, effective July 31, 2024 (AWS Developer Tools Blog)](https://aws.amazon.com/blogs/developer/the-aws-sdk-for-java-1-x-is-in-maintenance-mode-effective-july-31-2024/) — accessed 2026-05-25. Authoritative GA → maintenance → end-of-support timeline.
- [Migrate from version 1.x to 2.x of the AWS SDK for Java](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/migration.html) — accessed 2026-05-25. Canonical migration entry point + "what's new in v2".
- [What's different between the AWS SDK for Java 1.x and 2.x](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/migration-whats-different.html) — accessed 2026-05-25. Authoritative on package rename, immutable POJOs, `Response`-suffix, setter rename, library migration table.
- [How to migrate your code from AWS SDK for Java 1.x to 2.x](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/migration-howto.html) — accessed 2026-05-25. Branches into automated tool vs manual approach.
- [AWS SDK for Java v2 releases (GitHub)](https://github.com/aws/aws-sdk-java-v2/releases) — accessed 2026-05-25. Source for current 2.44.12 version + daily cadence evidence.

## 5. Code reference snippets (idiomatic, current API)

**v2 idiomatic — sync Bedrock InvokeModel using builder + immutable POJOs:**

```java
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.core.SdkBytes;
import software.amazon.awssdk.services.bedrockruntime.BedrockRuntimeClient;
import software.amazon.awssdk.services.bedrockruntime.model.InvokeModelRequest;
import software.amazon.awssdk.services.bedrockruntime.model.InvokeModelResponse;

BedrockRuntimeClient bedrock = BedrockRuntimeClient.builder()
    .region(Region.US_EAST_1)
    .build();

InvokeModelRequest req = InvokeModelRequest.builder()
    .modelId("anthropic.claude-sonnet-4-5-20250929-v1:0")
    .body(SdkBytes.fromUtf8String(jsonBody))
    .contentType("application/json")
    .build();

InvokeModelResponse resp = bedrock.invokeModel(req);
String responseBody = resp.body().asUtf8String();
```

**v2 idiomatic — async DynamoDB query returning `CompletableFuture`:**

```java
import software.amazon.awssdk.services.dynamodb.DynamoDbAsyncClient;
import software.amazon.awssdk.services.dynamodb.model.QueryRequest;
import java.util.concurrent.CompletableFuture;

DynamoDbAsyncClient ddb = DynamoDbAsyncClient.builder()
    .region(Region.US_EAST_1)
    .build();

CompletableFuture<QueryResponse> future = ddb.query(req -> req
    .tableName("vendors")
    .keyConditionExpression("uei = :u")
    .expressionAttributeValues(Map.of(":u", AttributeValue.fromS(uei))));

future.thenAccept(resp -> resp.items().forEach(item -> log.info("hit: {}", item)));
```

**BOM-based Maven dependency management (no per-artifact version pins):**

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>software.amazon.awssdk</groupId>
            <artifactId>bom</artifactId>
            <version>2.44.12</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>bedrockruntime</artifactId>
    </dependency>
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>dynamodb-enhanced</artifactId>
    </dependency>
</dependencies>
```

Anti-pattern (do NOT teach as current — appears in pre-2024 AWS Java tutorials):

```java
// v1 — END-OF-SUPPORT since 31 Dec 2025
import com.amazonaws.regions.Regions;
import com.amazonaws.services.dynamodbv2.AmazonDynamoDB;
import com.amazonaws.services.dynamodbv2.AmazonDynamoDBClientBuilder;
import com.amazonaws.services.dynamodbv2.model.QueryRequest;
import com.amazonaws.services.dynamodbv2.model.QueryResult;     // v1: "Result"

AmazonDynamoDB ddb = AmazonDynamoDBClientBuilder.standard()
    .withRegion(Regions.US_EAST_1)        // v1: withX() setter
    .build();

QueryRequest req = new QueryRequest()      // v1: mutable POJO
    .withTableName("vendors")              // v1: withX() setter
    .withKeyConditionExpression("uei = :u");
QueryResult resp = ddb.query(req);         // v1: "Result" suffix
```

## 6. Risks and watch-items

- **End-of-support cliff already passed.** Any acquire-gov dependency that still pulls v1 transitively is on unsupported software TODAY. W4 modernization deliverable should treat this as a P0-severity finding.
- **Transitive v1 dependencies.** Other libraries (e.g., older versions of `spring-cloud-aws`, `aws-encryption-sdk-java` 1.x line, third-party connectors) may still pull `com.amazonaws` transitively. `aws-migration-tool find-apps-using-v1` catches direct usage but not always transitive — run `mvn dependency:tree | grep com.amazonaws` as a belt-and-braces check.
- **Side-by-side period is bounded.** AWS recommends side-by-side as a transitional state, not a long-term posture. Plan for full v1 removal at the end of W4.
- **`CompletableFuture` orchestration** is a common cohort stumbling block when first adopting async clients — `.thenCompose()` vs `.thenApply()` vs `.thenCombine()` taxonomy lives in a W5 war-room scenario.
- **CRT (Common Runtime) dependency for `S3TransferManager`** can be heavy on Lambda cold-start; profile before adopting.

## 7. Alternatives the cohort should be aware of

- **AWS CDK for infrastructure** — separate concern (IaC vs runtime SDK), but worth flagging that CDK is also Java-supported (groupId `software.amazon.awscdk`).
- **Spring Cloud AWS 3.x** — Spring-Boot-native wrapper around v2 SDK; sometimes a better fit than raw v2 for Spring Boot 3.x services. Surfaced as W4 scenario-alternative for the `services/api-gateway` migration.
- **Direct HTTP** — for the one or two services where the cohort uses an AWS endpoint that's not in the SDK yet (rare; sigv4 signing manually).

## 8. Brief sign-off

- Drafted by: research-subagent (2026-05-25)
- Reviewed against `known-bad-patterns.yml`: **flagged** — four new patterns surfaced (v1-as-current, mutable-builder, sync-only, TransferManager-import); brief counter-teaches.
- 1-month-release check: triggered (2.44.12 dropped 22 May 2026 = 3 days ago). Resolved by pinning to 2.44.12 and noting daily-cadence releases; load-bearing facts (immutable builder, Response suffix, package rename, async client family) have been stable since v2 GA in Nov 2018.
- Approved for downstream artifact authoring: 2026-05-25

## Sources

- The AWS SDK for Java 1.x is in maintenance mode, effective July 31, 2024, aws.amazon.com/blogs/developer, retrieved 2026-05-25 via web-research (Firecrawl). <https://aws.amazon.com/blogs/developer/the-aws-sdk-for-java-1-x-is-in-maintenance-mode-effective-july-31-2024/>
- Migrate from version 1.x to 2.x of the AWS SDK for Java, docs.aws.amazon.com, retrieved 2026-05-25 via web-research (Firecrawl). <https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/migration.html>
- What's different between the AWS SDK for Java 1.x and 2.x, docs.aws.amazon.com, retrieved 2026-05-25 via web-research (Firecrawl). <https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/migration-whats-different.html>
- How to migrate your code from AWS SDK for Java 1.x to 2.x, docs.aws.amazon.com, retrieved 2026-05-25 via web-research (Firecrawl). <https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/migration-howto.html>
- AWS SDK for Java v2 releases, github.com/aws/aws-sdk-java-v2, retrieved 2026-05-25 via web-research (Firecrawl). <https://github.com/aws/aws-sdk-java-v2/releases>
