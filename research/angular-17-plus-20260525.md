---
template: research-brief
tech: Angular (frontend framework) — 17 → 21 trajectory
version_pinned: "21.2.14 (current GA, v21 line); v20 is the current LTS-equivalent prior major"
last_verified: 2026-05-25
last_verified_via: web-research (Firecrawl self-hosted)
recency_window: foundation-stable 12mo
sources_count: 4
target_weeks: [W04, W05]
candidates_deferred: []
known_bad_patterns_flagged:
  - id: angular-ngmodule-first
    note: "Pre-v17 Angular tutorials lead with `NgModule` + `declarations: [...]` arrays. Modern Angular is standalone-component-first (default since v17 GA, Nov 2023). Brief teaches standalone-by-default; NgModule mentioned only as legacy interop."
  - id: angular-ngif-ngfor-structural
    note: "`*ngIf` / `*ngFor` / `*ngSwitch` structural directives still work but are no longer idiomatic. v17+ uses `@if` / `@for` / `@switch` control-flow syntax (stable since v17). Brief teaches the new control flow."
  - id: angular-zonejs-default
    note: "New Angular apps in v21 do NOT include zone.js by default — zoneless change detection is stable since v20.2 and the v21 default. Tutorials assuming zone.js + `NgZone` for change detection are stale for greenfield. Brief frames zone.js as legacy-only."
  - id: angular-karma-jasmine-default
    note: "Karma was deprecated in 2023; Vitest is the v21 default test runner (stable). Karma/Jasmine still supported but not idiomatic for new code. Brief points to Vitest."
known_bad_patterns_checked: true
author: research-subagent
---

# Tech research brief — Angular 17 → 21

Last verified: 2026-05-25 · Recency window applied: foundation-stable 12mo · Pinned version: 21.2.14 (v21 GA line)

## 1. What it is (3–5 sentences, no jargon)

Angular is Google's TypeScript-first single-page-app framework, currently on the **v21 release line** (GA 19 Nov 2025, latest patch 21.2.14 on 20 May 2026). Between Angular 17 (the acquire-gov starting baseline, Nov 2023) and Angular 21, the framework underwent its biggest paradigm shift since AngularJS-to-Angular: **standalone components are now the default** (NgModule is legacy), **Signals replace RxJS for most state** (no more `async` pipe + Subject ceremony for simple cases), **new control-flow syntax `@if`/`@for`/`@switch` replaces structural directives**, **zoneless change detection is the default for new apps**, and **Vitest replaces Karma+Jasmine as the default test runner**. For the Karsun FDE auditor question, the load-bearing facts are: (a) acquire-gov's Angular 17 baseline is OSS-unsupported (active support ended May 2024, security support ended 15 May 2025), and (b) the **FDE W4 modernization scope does NOT include an Angular major hop** — the cohort stays on Angular 17 to keep W4 focused on the Java/Spring Boot/AWS SDK modernization arc. This brief documents the post-17 ecosystem evolution for context; the v21 target is a hypothetical future-cohort scope, not Cohort #1's W4 work.

## 2. Current stable state (as of `last_verified`)

- **Latest stable releases (endoflife.date, 2026-05-22):**
  - 21.2.14 — released 20 May 2026 (GA line opened 19 Nov 2025); active support ended 19 May 2026; security support until 19 May 2027
  - 20.3.21 — released 12 May 2026 (GA line opened 28 May 2025); active support ended 19 Nov 2025; security support until 28 Nov 2026
  - 19.2.22 — security support ended 19 May 2026 (i.e., **6 days before `last_verified`**); now fully OSS-unsupported
  - 17.x (acquire-gov baseline) — active support ended 8 May 2024; security support ended 15 May 2025; fully OSS-unsupported
- **Angular release cadence:** major every ~6 months, 6 months active support + 12 months LTS security-only.
- **v21 GA headline features (Angular blog, 19 Nov 2025):**
  - **Signal Forms** (experimental) — new signal-based forms library, `form(signal)` API, schema-based validation, no `ControlValueAccessor` for custom components.
  - **Angular Aria** (developer preview) — `@angular/aria` headless accessible component library (8 patterns / 13 components: accordion, combobox, grid, listbox, menu, tabs, toolbar, tree).
  - **Angular MCP server stable** — 7 tools incl. `get_best_practices`, `search_documentation`, `find_examples`, `onpush_zoneless_migration`, `ai_tutor`.
  - **Vitest stable as default test runner** for new apps (`ng test` runs Vitest); Karma/Jasmine still supported. Web Test Runner + Jest experimental support deprecated, slated for removal in v22. Migration schematic: `ng g @schematics/angular:refactor-jasmine-vitest`.
  - **Zoneless is now the default for new apps** — `zone.js` is not included by default. Zoneless went experimental in v18, dev-preview in v20, stable in v20.2.
  - Experimental Navigation API support; type-checking of host bindings on by default; `ngModuleFactory` input removed from `NgComponentOutlet`.
- **Foundational features (v17+ baseline still load-bearing in v21):**
  - **Standalone components default** (v17 GA Nov 2023) — `bootstrapApplication()` replaces `platformBrowserDynamic().bootstrapModule()`; components declare their own imports.
  - **Built-in control flow** (`@if` / `@for` / `@switch`) — stable since v17, idiomatic since v18.
  - **Deferrable views** (`@defer`) — lazy-load template chunks with triggers (`on viewport`, `on idle`, `on interaction`, `on hover`, `on timer`, `when <expr>`).
  - **Signals** — fine-grained reactivity primitive (`signal()`, `computed()`, `effect()`); stable since v17.
  - **esbuild build system default** — Application Builder (esbuild + Vite dev server) since v17; Webpack-based Angular CLI builder is legacy.
  - **SSR + hydration** non-destructive hydration stable since v17.
  - **Material 3** — Angular Material on Material 3 since v17.
- Known-bad-pattern check: **flagged** — four new patterns surfaced (NgModule-first, `*ngIf`/`*ngFor`-as-idiomatic, zone.js-by-default, Karma+Jasmine-default). All four are common in Stack Overflow / older Angular University content. Brief counter-teaches.

## 3. What we teach (and what we deliberately don't)

- **In scope (W4 brownfield modernization frontend track):** Standalone components as the default (no NgModules in greenfield code); built-in control flow `@if`/`@for`/`@switch`; Signals for component state (replacing most RxJS BehaviorSubject patterns); `@defer` for lazy template loading on a federal-portal-relevant pattern (e.g., heavy data-grid only on hover); the v17 → v21 upgrade path using `ng update @angular/cli @angular/core` (Angular's migration schematics handle most code transforms automatically including standalone-component migration and control-flow migration); zoneless change detection for new feature modules.
- **Out of scope (deliberately):** Signal Forms (experimental in v21 — too unstable for cohort production code; flagged as "what's coming next"); Angular Aria (developer preview — same reason); writing custom MCP server tools (cohort uses Angular's MCP server but doesn't extend it); RxJS deep dive (cohort uses RxJS where Signals can't — HTTP responses, multi-emit streams — but doesn't teach `flatMap`/`switchMap`/`mergeMap` taxonomy).
- **Misconceptions to pre-empt:** (a) "Angular 17 is modern enough — it's only 18 months old." — false; v17 has been fully OSS-unsupported (security included) since 15 May 2025. (b) "Standalone components are optional." — technically true but the v21 default and all new docs/examples are standalone-first; NgModule is "legacy interop only". (c) "We need RxJS for everything." — false in v17+; Signals cover the bulk of component state. (d) "Zoneless means no change detection." — false; it means signal-triggered change detection rather than zone.js patching browser APIs.

## 4. Recommended primary sources

- [Angular.dev homepage](https://angular.dev/) — accessed 2026-05-25. Pins v21 as the live release line; shows current idiomatic Signals + `@for` example as the canonical pitch.
- [Announcing Angular v21 (Angular blog, Nov 19 2025)](https://blog.angular.dev/angular-v21-is-now-available-57946c34f14b) — accessed 2026-05-25. Authoritative v21 feature list + zoneless-default + Vitest-default + Signal Forms + Angular Aria + MCP server stable.
- [Angular v21 release event page](https://angular.dev/events/v21) — accessed 2026-05-25. Marketing-tier overview; confirms 11-20-2025 release date.
- [Angular endoflife.date](https://endoflife.date/angular) — updated 22 May 2026, accessed 2026-05-25. Canonical active + security support windows for every Angular major; documents that v17 went fully unsupported May 2025 and v19 went unsupported 19 May 2026.

## 5. Code reference snippets (idiomatic, current API)

**Standalone component with Signals + new control flow (v17+ idiomatic):**

```typescript
import { Component, signal, computed } from '@angular/core';

@Component({
  selector: 'app-vendor-search',
  standalone: true,    // default in v17+, optional to write
  template: `
    <input [value]="query()" (input)="query.set($any($event.target).value)" />
    @if (filtered().length > 0) {
      <ul>
        @for (vendor of filtered(); track vendor.uei) {
          <li>{{ vendor.name }} — {{ vendor.uei }}</li>
        }
      </ul>
    } @else {
      <p>No matching vendors.</p>
    }
  `,
})
export class VendorSearchComponent {
  vendors = signal<Vendor[]>([]);
  query = signal('');
  filtered = computed(() =>
    this.vendors().filter(v => v.name.toLowerCase().includes(this.query().toLowerCase()))
  );
}
```

**Bootstrap (v17+ idiomatic — no AppModule):**

```typescript
import { bootstrapApplication } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';
import { AppComponent } from './app/app.component';
import { routes } from './app/app.routes';

bootstrapApplication(AppComponent, {
  providers: [provideRouter(routes)],
});
```

**Deferred loading (v17+):**

```html
@defer (on viewport; prefetch on idle) {
  <app-heavy-grant-data-table [grants]="grants()" />
} @placeholder {
  <p>Loading grant data...</p>
} @loading (minimum 200ms) {
  <app-skeleton />
}
```

Anti-pattern (do NOT teach as current — appears in pre-v17 Angular tutorials):

```typescript
// Pre-v17 — NgModule-first; legacy-only in v21
@NgModule({
  declarations: [VendorSearchComponent],
  imports: [CommonModule, FormsModule],
  exports: [VendorSearchComponent],
})
export class VendorSearchModule {}

// Pre-v17 — structural-directive control flow; still works but no longer idiomatic
<ul *ngIf="filtered.length > 0; else empty">
  <li *ngFor="let vendor of filtered; trackBy: trackByUei">{{ vendor.name }}</li>
</ul>
<ng-template #empty><p>No matching vendors.</p></ng-template>
```

## 6. Risks and watch-items

- **Major version every 6 months.** v22 likely Nov 2026 (post-programme). Brief pins v21 today; cohort handoff should warn the federal client that another major is ~6 months out.
- **Signal Forms is experimental in v21.** Tempting because it's better, but unstable API — defer to v22 GA for production.
- **OpenRewrite-equivalent for Angular is `ng update`.** Angular's migration schematics are usually high-quality but the v17 → v21 jump is four majors; expect to land on v18 first, run schematics, then re-up to v19 → v20 → v21. Do not skip majors.
- **Zone.js removal in greenfield** changes how third-party libs that monkey-patch (e.g., older RxJS interop, some test utilities) behave. Audit before adopting zoneless on a brownfield migration.

## 7. Alternatives the cohort should be aware of

- **React (with TanStack Router / Next.js)** — surfaced in W4 scenario-alternatives ("why not React?"); answer leans on Angular's stronger conventions for federal multi-team codebases.
- **Vue 3 + Pinia + Vite** — niche federal relevance, mostly out-of-scope but mentioned.
- **HTMX + server-rendered Spring** — surfaced as a "do you even need an SPA" framing question for the AIOps dashboards in W5.

## 8. Brief sign-off

- Drafted by: research-subagent (2026-05-25)
- Reviewed against `known-bad-patterns.yml`: **flagged** — four new patterns surfaced (NgModule-first, ngIf-as-idiomatic, zone.js-default, Karma-default); brief counter-teaches.
- 1-month-release check: triggered (21.2.14 dropped 20 May 2026 = 5 days ago). Resolved by anchoring brief to v21 line as canonical; patch-level moves are additive.
- Approved for downstream artifact authoring: 2026-05-25

## Sources

- Angular.dev homepage, angular.dev, retrieved 2026-05-25 via web-research (Firecrawl). <https://angular.dev/>
- Announcing Angular v21, blog.angular.dev, published 2025-11-19, retrieved 2026-05-25 via web-research (Firecrawl). <https://blog.angular.dev/angular-v21-is-now-available-57946c34f14b>
- Angular v21 release event, angular.dev, retrieved 2026-05-25 via web-research (Firecrawl). <https://angular.dev/events/v21>
- Angular endoflife.date, endoflife.date, updated 2026-05-22, retrieved 2026-05-25 via web-research (Firecrawl). <https://endoflife.date/angular>
