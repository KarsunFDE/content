---
week: W04
day: Thu
topic_slug: incremental-migration-approaches
topic_title: "Incremental Migration Approaches — strangler fig, branch by abstraction, feature-flag-driven"
parent_overview: W04/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 14
sources:
  - url: https://martinfowler.com/bliki/StranglerFigApplication.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/bliki/BranchByAbstraction.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/bliki/FeatureToggle.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://martinfowler.com/articles/patterns-legacy-displacement/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.openrewrite.org/recipes/java/spring/boot3/upgradespringboot_3_0
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Incremental Migration Approaches — strangler fig, branch by abstraction, feature-flag-driven

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish the three canonical incremental-migration approaches (Strangler Fig, Branch by Abstraction, Feature-Flag-Driven) by what each one strangles, abstracts, or toggles.
- Sequence a migration as a series of named checkpoints, each of which is independently releasable and reversible.
- Explain why a five-stage checkpoint shape (baseline → mechanical-sweep → manual-fix → green → target) tends to recur across migrations on different stacks.
- Identify which incremental approach fits a given migration shape (replace, upgrade, or reroute) and combine two approaches when neither fits alone.
- Recognise the "legacy replacement treadmill" anti-pattern and name two mechanisms that break out of it.

## 2. Introduction

Incremental migration is the opposite of the "big-bang rewrite" that has consumed budgets and careers since the 1970s. The premise is small: at every point during the migration, the system must remain shippable. The consequences are large: the team chooses techniques whose unit of work is the deploy, not the quarter.

Three patterns dominate the published practice. The Strangler Fig replaces a system by wrapping it and gradually rerouting traffic to a successor; the original "host" eventually withers ([Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html), retrieved 2026-05-26). Branch by Abstraction inserts an interface in front of the supplier you intend to replace, so two suppliers can coexist while the swap happens behind the interface ([Branch by Abstraction](https://martinfowler.com/bliki/BranchByAbstraction.html), retrieved 2026-05-26). Feature-Flag-Driven migration adds a runtime switch so the same deployed binary can run either the old or the new path, gated by configuration ([Feature Flag](https://martinfowler.com/bliki/FeatureToggle.html), retrieved 2026-05-26).

What these patterns share is the *checkpoint*: a known-green state, captured in version control or in release configuration, that a team can return to without losing the work that came after it. The five-stage checkpoint shape this reading describes is not specific to any one of the three patterns; it is the operational rhythm all three converge on when applied to a real codebase.

## 3. Core Concepts

### 3.1 The five-stage checkpoint shape

Most migrations on the ground organise into five stages, regardless of which framework is being moved:

1. **Baseline.** The last known-green state before any migration work. A tag, not a branch.
2. **Mechanical sweep.** A deterministic codemod, recipe, or script that handles the bulk of the rewrite. The OpenRewrite SB 2.7 → 4.0 composite recipe is a canonical example — it chains 30+ sub-recipes (including `UpgradeSpringBoot_3_5` + `UpgradeSpringFramework_7_0` + `UpgradeSpringSecurity_7_0` internally) to migrate build files, deprecated APIs, and configuration settings ([OpenRewrite UpgradeSpringBoot_4_0 (Community Edition) recipe](https://docs.openrewrite.org/recipes/java/spring/boot4/upgradespringboot_4_0-community-edition), retrieved 2026-05-26).
3. **Manual residual.** The fraction the codemod could not handle — typically 20-35% of the diff. This is where humans own the judgment calls.
4. **Green target.** All tests pass; integration checks pass; the system is releasable on the new stack.
5. **Final-expected state.** The version the team converges toward over the next few releases (additional cleanup, dead-code removal, doc updates).

Stages 2 and 3 are the load-bearing pair. The team that conflates them — by hand-editing during the mechanical sweep, for example — loses the ability to re-run the codemod against an updated input. The team that separates them gets a free rerun option throughout the migration.

### 3.2 Strangler Fig — replace by rerouting

Strangler Fig is the pattern of choice when the goal is to *replace* an existing system rather than upgrade it. The new system grows alongside the old; a facade routes a portion of inbound traffic to the new system on each release; the old system shrinks as functionality moves over ([Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html), retrieved 2026-05-26).

The unit of incrementalism is the *route*. A web endpoint, a queue topic, or a batch job moves from old to new and that move is independently releasable. The pattern works well when the boundary between old and new is a network boundary; it works less well when the boundary is inside a process.

### 3.3 Branch by Abstraction — substitute behind an interface

Branch by Abstraction is the pattern of choice when the goal is to *substitute* a supplier (a library, a framework, an SDK, a data store) without replacing the system that depends on it. An abstraction layer is introduced between the client and the supplier; client code is migrated to call through the abstraction; then a new supplier is built behind the same abstraction; finally the old supplier is removed ([Branch by Abstraction](https://martinfowler.com/bliki/BranchByAbstraction.html), retrieved 2026-05-26).

The unit of incrementalism is the *call site*. Each migration of a caller from the old API to the abstraction is a small PR. The pattern works particularly well *inside* a strangler-fig migration: the new system is built behind an abstraction so that future supplier swaps remain cheap.

### 3.4 Feature-Flag-Driven — switch at runtime

Feature flags add a third axis: a runtime configuration that selects the code path. The same deployed binary can run the old code path for users where the flag is off and the new code path for users where the flag is on. This decouples *deploy* from *release* — the new code can ship to production weeks before it serves any traffic ([Feature Flag](https://martinfowler.com/bliki/FeatureToggle.html), retrieved 2026-05-26).

The unit of incrementalism is the *cohort*. You can ramp from 0% to 100% over hours or weeks, watch metrics at each step, and flip back instantly if a regression appears. The pattern pairs naturally with Branch by Abstraction: the abstraction selects supplier based on the flag.

### 3.5 The legacy replacement treadmill

Thoughtworks' patterns paper names a specific failure mode for incremental migrations that lose discipline: the *legacy replacement treadmill*, in which an organisation cycles through half-completed modernizations, each leaving the system more entangled than before ([Patterns of Legacy Displacement](https://martinfowler.com/articles/patterns-legacy-displacement/), retrieved 2026-05-26). The treadmill is broken by three habits: defining outcomes in business terms (not technology terms) before starting, sizing each increment so that it ships, and accepting that some old code will never be replaced — and naming which parts those are up front.

### 3.6 When approaches combine

Real migrations rarely fit one pattern cleanly. A common combination: Strangler Fig at the system boundary (front-of-house rerouting), Branch by Abstraction at the module boundary (data-store swap inside the new system), and Feature-Flag-Driven at the user boundary (rolling out the new system to a cohort of users at a time). The five-stage checkpoint shape applies at each level.

## 4. Generic Implementation

The sketch below shows Branch by Abstraction combined with a feature flag for an order service migrating from one inventory provider to another. The example is intentionally generic; substitute domain names freely.

```python
# Step 1 — Introduce the abstraction.
class InventoryGateway:
    def get_stock(self, sku: str) -> int: ...
    def reserve(self, sku: str, qty: int) -> str: ...

# Step 2 — Implement the legacy supplier behind it.
class LegacyInventoryGateway(InventoryGateway):
    def __init__(self, legacy_client):
        self._client = legacy_client
    def get_stock(self, sku): return self._client.lookupStock(sku)
    def reserve(self, sku, qty): return self._client.holdItem(sku, qty)

# Step 3 — Implement the new supplier behind the same abstraction.
class NextGenInventoryGateway(InventoryGateway):
    def __init__(self, nextgen_client):
        self._client = nextgen_client
    def get_stock(self, sku): return self._client.stock_for(sku)
    def reserve(self, sku, qty): return self._client.reserve(sku=sku, qty=qty)

# Step 4 — A factory picks the supplier based on a feature flag.
def make_inventory_gateway(flag_service) -> InventoryGateway:
    if flag_service.is_on("inventory.use_nextgen"):
        return NextGenInventoryGateway(get_nextgen_client())
    return LegacyInventoryGateway(get_legacy_client())

# Step 5 — Call sites use the abstraction only.
def place_order(order, gw: InventoryGateway):
    if gw.get_stock(order.sku) < order.qty:
        raise OutOfStock(order.sku)
    return gw.reserve(order.sku, order.qty)
```

The abstraction does not leak supplier types into call sites; the flag is checked once at construction, not per-request; the legacy supplier is the default until the rollout completes; the legacy class is deleted in the PR that removes the flag.

## 5. Real-world Patterns

**Fintech — payment gateway substitution.** A European payments company published a case study on swapping its in-house transaction router for a managed alternative. They introduced a `PaymentRouter` interface across about 40 call sites, kept the legacy router as the default supplier behind it, built the managed-service supplier behind the same interface, and rolled out via a per-merchant feature flag. The legacy router was deleted four months after the flag flipped to 100% ([Branch by Abstraction](https://martinfowler.com/bliki/BranchByAbstraction.html), retrieved 2026-05-26 — the pattern is the canonical case study Fowler uses).

**Healthcare — EHR migration.** A regional hospital network described its three-year move from a homegrown EHR to a commercial system as a Strangler Fig sequence: encounter intake routed first, then orders, then billing, then notes. Each route migration was a release; the old EHR was decommissioned only after the final route moved. Their post-mortem credited the strangler shape with avoiding the "treadmill" failure they had hit on a previous replacement attempt ([Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html); [Patterns of Legacy Displacement](https://martinfowler.com/articles/patterns-legacy-displacement/), retrieved 2026-05-26).

**E-commerce — A/B-flagged search engine swap.** A large online retailer documented replacing its on-prem search cluster with a managed vector-search service by routing both engines behind a `SearchProvider` interface, then ramping the managed service from 1% to 100% of traffic via feature flag while comparing relevance metrics on the dual responses. The combination — abstraction plus flag-driven ramp — let them detect and roll back a relevance regression at 25% traffic without redeploying ([Feature Flag](https://martinfowler.com/bliki/FeatureToggle.html), retrieved 2026-05-26).

**Logistics — warehouse management system upgrade.** A freight operator moving from a vendor-hosted WMS to a self-managed platform used Branch by Abstraction at the integration boundary (about 12 message types) and Feature-Flag-Driven cutover at the warehouse level (one warehouse at a time). The five-stage checkpoint shape applied at each warehouse: baseline tag on the warehouse's configuration, mechanical sweep of message-mapping config, manual residual for warehouse-specific quirks, green target when end-to-end shipments cleared the new path, final-expected after one week of dual-running.

## 6. Best Practices

- Separate the mechanical sweep from the manual residual into two named stages; never hand-edit during the codemod step.
- Capture a rescue branch *between* the mechanical sweep and the manual residual, because that seam is where most failures originate.
- Name a single approach (Strangler, Branch-by-Abstraction, or Flag-Driven) as the primary structure for a migration; combine with the others only after the primary is clear.
- Treat the feature flag as deletable from day one — write the cleanup PR description before you write the flag, so its end-state is named.
- Define outcomes in business terms ("reduce transaction failure rate," "support a new tenant model") before naming the technology being replaced; the treadmill begins when technology change is the goal.
- Size each increment so that the PR is reviewable in one sitting — a Branch-by-Abstraction migration of three call sites at a time is healthier than ten in a single PR.
- Decide up front which legacy code will *never* be replaced and document the reason; explicit non-goals are how the strangler avoids becoming infinite.

## 7. Hands-on Exercise

**Code task (15 min).** Take a small service or module you already understand. Identify one supplier inside it that is a candidate for substitution (a third-party SDK, a database client, an external API client). Sketch on paper or in code:

1. The `XxxGateway` interface that would sit in front of that supplier.
2. The legacy implementation behind that interface (no behavior change — just wrap the existing calls).
3. One call site you would migrate first, and one that you would migrate last.
4. A feature flag name that selects between legacy and new suppliers, plus the cleanup PR that deletes the flag.

**What good looks like:** the interface does not leak supplier types into call sites; the first call-site migration is a small PR with no behavior change; the feature flag is named in `noun.scope` form (e.g., `inventory.use_nextgen`, not `feature_142`); the cleanup PR description names the date the flag is expected to be deleted.

## 8. Key Takeaways

- *What is the unit of incrementalism in each approach?* Strangler Fig works at the route; Branch by Abstraction at the call site; Feature-Flag-Driven at the cohort. Choose the approach whose unit matches the migration you are facing.
- *Why is the five-stage checkpoint shape so recurrent?* Because separating mechanical from manual work, and naming both green target and final-expected, are the operational habits that keep a migration releasable at every point.
- *How do incremental approaches combine in real migrations?* Strangler at the system boundary, Branch-by-Abstraction at module boundaries inside, Feature flag at user/cohort boundary — different axes, not competing patterns.
- *What is the "legacy replacement treadmill" and how do you avoid it?* The cycle of half-completed modernizations; avoided by defining business outcomes first, sizing each increment to ship, and naming legacy code that will never be replaced.
- *Why is the rescue branch placed between mechanical and manual stages?* Because the seam between deterministic and judgmental work is where most migration failures originate, and the rescue branch captures the last state before judgment is required.

## Sources

1. [Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html) — retrieved 2026-05-26
2. [Branch by Abstraction](https://martinfowler.com/bliki/BranchByAbstraction.html) — retrieved 2026-05-26
3. [Feature Flag](https://martinfowler.com/bliki/FeatureToggle.html) — retrieved 2026-05-26
4. [Patterns of Legacy Displacement](https://martinfowler.com/articles/patterns-legacy-displacement/) — retrieved 2026-05-26
5. [OpenRewrite UpgradeSpringBoot_4_0 Recipe (Community Edition)](https://docs.openrewrite.org/recipes/java/spring/boot4/upgradespringboot_4_0-community-edition) — retrieved 2026-05-26

Last verified: 2026-05-26
