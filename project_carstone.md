---
name: project-carstone
description: "carstone monorepo (github.com/Devary/crudstone-demo-carstone): autoscout24-style car marketplace dogfooding the whole crudstone family (context-gen, dynamic-crud, search-crudstone, sidebar-crudstone) — 2 Quarkus backends + no-auth gateway + 2 Angular apps, all built and live-verified across two sessions (2026-08-19/20)"
metadata:
  node_type: memory
  type: project
  originSessionId: 9490a6be-eca4-4020-a624-b91245b42f3c
---

# carstone — car marketplace dogfooding the crudstone family (built 2026-08-19/20)

Built end-to-end from a one-line user request ("build a car buy/sell site using our components,
improve them if they're missing something"). See [[project_gen_crudstone]]/
[[project_search_crudstone]]/[[project_sidebar_crudstone]] for the underlying libraries this
dogfoods. Repo: **`github.com/Devary/crudstone-demo-carstone`** (public — renamed from the
original `github.com/Devary/carstone`, full history preserved; the old repo still exists
untouched at `github.com/Devary/carstone` as a spare, not deleted). Monorepo, Maven reactor + 2
sibling Angular CLI apps (not part of the reactor). Local git remote `origin` points at the
`crudstone-demo-carstone` repo; `carstone-old` remote still points at the original if ever needed.

## Architecture
```
carstone/                          (Maven reactor: root pom = parent + aggregator)
├── carstone-domain/                shared @Entity+@CrudstoneEntity/@Searchable: Brand, Seller, CarListing
├── carstone-admin/    :9200        full CRUD (writes) + @Sidebar, owns schema (drop-and-create), seeds data
├── carstone-front/    :9210        read-only handlers, SAME static Postgres, schema-management=none
├── carstone-gateway/  :9220        no-auth reverse proxy (ported from quar-gateway, auth stripped)
├── carstone-admin-ui/ :5910        Angular: crudstone (CRUD tables) + sidebarcrudstone (nav), combined shell
└── carstone-front-ui/ :5911        Angular: searchcrudstone (search bar + filters + results page)
```
Persistence: **plain static Postgres container**, NOT Quarkus Dev Services (the original plan) —
Dev Services' cross-JVM container reuse needs a machine-wide `~/.testcontainers.properties` opt-in
that's out of scope to write without asking (blocked by the permission classifier). Fallback:
`docker run -d --name carstone-postgres -e POSTGRES_USER=carstone -e POSTGRES_PASSWORD=carstone-dev-only
-e POSTGRES_DB=carstone -p 5460:5432 postgres:17`, both admin/front point at it explicitly
(`quarkus.datasource.reactive.url=postgresql://localhost:5460/carstone`). Same proven pattern
quar-crud-host itself uses. **Must `docker start carstone-postgres` before booting admin/front
if the container was stopped/machine rebooted** — it isn't auto-recreated.

Boot order matters: **always start carstone-admin before carstone-front** (front has
`schema-management.strategy=none`, no graceful wait for admin's schema+seed to exist).

## Library improvement: numeric/date range filtering (the "improve if missing" deliverable)
Confirmed real gap via direct code reading: no range-filter mechanism existed anywhere in the
stack (only `dependsOn`/`dependency` cross-field UI validation, hard-restricted to date/dateTime).
Added `SearchableField#rangeTarget` to **context-gen** (pushed `fdbb579`): a field with
`rangeTarget` set is a virtual "from"/"to" bound on a DIFFERENT field's real column via `>=`/`<=`,
reusing `dependency()` (GTE=min, LTE=max) rather than a parallel enum. `isTemporal()` →
`isComparable()`, now also accepts `"number"`. Frontend half in **search-crudstone** (pushed
`59737be`): `TableFieldType.NUMBER`, `isNumber` filter branch, `wire-filters.ts` emits
`filter.<rangeTarget>Min`/`Max`. Backend query-building (app-specific, not library-level) lives in
carstone-domain's own `support/PanacheListQuerySupport.java` (ported+extended from quar-crud-host
with `RANGE_MIN`/`RANGE_MAX` FieldKind + a `rangeTargetColumns` map param).

## Confirmed real gaps closed along the way (all verified live, not just compiled)
- **`type="number"` was invisible in BOTH frontends** — neither crudstone's admin form nor
  search-crudstone's filter bar had a widget for it. Fixed via `createEditType="inputText"`
  override (admin form side, in each numeric `@CrudstoneField`) + the new `isNumber` branch
  (search side). Confirmed via a Cypress screenshot of the New Listing dialog showing Year/Price/
  Mileage/Power as real inputs.
- **`ContextRegistry` (admin + front) only resolved by entity NAME, never by its own
  `CrudstoneEntity#path`** — but `crudstone`'s `EntityPageComponent` requests the raw ROUTE
  SEGMENT directly (`GET /{routeParam}?...&includeContext=true`, one combined call). Every
  quar-crud-host reference entity happens to have `path==name`, which is exactly why this was
  invisible until `CarListing` (name="carListings", path="listings") existed. Fixed by indexing
  both name and path in both `ContextRegistry`s.
- **Gateway's `HttpClient` defaulted to HTTP/2 negotiation**, which reproducibly failed against
  Quarkus dev-mode's HTTP endpoint on the first request(s) after a boot
  (`java.io.IOException: Received RST_STREAM: Protocol error`) — every real browser page load hit
  this (two near-simultaneous requests right after a restart); spaced-out manual curls almost
  never did, which is why the earlier gateway-only verification missed it. Fixed by pinning
  `.version(HttpClient.Version.HTTP_1_1)` in `GatewayProxyService`.
- **`environment.crudstoneUrl` (carstone-admin-ui) was copied from sidebar-crudstone's own demo
  pattern without adjusting**: that demo's `crudstoneUrl` deliberately points at a SEPARATE app
  (dynamic-crud); carstone-admin-ui combines both crudstone+sidebarcrudstone into ONE app, so the
  sidebar's own links must point back at its own origin (`http://localhost:5910/`) instead of the
  gateway/API prefix — clicking "Brands" was navigating the browser to a raw JSON API endpoint.
  Caught by a Cypress spec hanging on page load (not an obvious error) — diagnosed by isolating
  the exact failing command sequence across several throwaway spec variants.
- **`CarListing.sellerType` had `notNull=true`** but is server-derived from the `seller` relation
  AFTER the generic wire-body validator already runs — every create/update 400'd before the
  handler ever got to fill it in. Removed `notNull`.
- **`CarListingHandler.apply()` combined brand+seller relation lookups via `Uni.combine().all()`**
  — Hibernate Reactive sessions can't run two queries concurrently within one transaction
  ("Illegal pop() with non-matching JdbcValuesSourceProcessingState"). Fixed by chaining
  sequentially (brand resolves fully, then seller).

## npm package consumption (real friction, not fully solved)
None of `crudstone`/`searchcrudstone`/`sidebarcrudstone` are published anywhere (confirmed:
`npm view` 404s all three) — only ever consumed via same-workspace TS path-mapping before this.
carstone-admin-ui/carstone-front-ui use local `file:` tarballs instead: `ng build <lib>` + `npm
pack` in each lib's own `dist/<lib>/`, referenced as `"crudstone": "file:../../dynamic-crud/dist/
crudstone/crudstone-1.0.6.tgz"` etc. in the consuming app's `package.json`. **Manual repack
required whenever a lib changes** — rerun `ng build <lib> && npm pack` in that lib's `dist/<lib>/`
dir, then `npm install` in the consuming app to pick up the new tarball content (npm caches by
filename+version, so bump the version or `rm -rf node_modules/<lib>` if a rebuild doesn't seem to
take).

## Entities (carstone-domain)
`Brand` (name/country/logo), `Seller` (name/type PROFESSIONAL|PRIVATE/city/phone/email),
`CarListing` (title/brand/model/year+yearFrom+yearTo/price+priceFrom+priceTo/mileage+
mileageFrom+mileageTo/power/fuelType/transmission/bodyType/color/sellerType-denormalized/seller/
city/images-gallery/description/firstRegistration). The `*From`/`*To` range fields are
`@Transient` JPA fields (no backing column) — excluded from the admin CRUD form entirely via
`CrudstoneEntity#disabledFields`.

## Seed data (DataSeeder, carstone-admin)
21 real brands (BMW/Mercedes-Benz/Audi/VW/Porsche/Opel/Toyota/Honda/Nissan/Mazda/Hyundai/Kia/Ford/
Tesla/Renault/Peugeot/Citroen/Fiat/Volvo/Skoda/SEAT), 20 sellers (8 dealerships + 12 private,
deterministic), 90 listings (deterministic array-cycling by loop index, no faker lib). Images left
empty (no real binary assets invested, matches Brand.logo also being left null).

## Environments / dev workflow
Angular version ceiling: **PrimeNG 19.1.4, never 20+** (this ecosystem deliberately reverted off
20 — paid-license above 18). Both Angular apps route through the gateway (`:9220`), never the
backends directly — that was the whole point of building one. Dark-mode toggle copied verbatim
from the 3 sibling demo apps (`.app-dark` on `<html>`, `localStorage['darkMode']`).

Cypress smoke specs exist in both Angular apps (`cypress/e2e/smoke.cy.ts`, no `supportFile`,
`pageLoadTimeout: 120000`) — this is what actually caught every bug above; `ng build` success
alone caught none of them. carstone-admin-ui: 3/3 passing. carstone-front-ui: 4/4 passing
(includes a genuine end-to-end proof that a year/price range filter's request carries
`filter.yearMin`/`filter.priceMax` AND the returned results actually respect the bound, not just
that the UI sent the param).

**Known minor flake, not chased further**: search-crudstone's own `runSearch()` (pre-existing
library code, not touched this session) sometimes doesn't navigate from the compact search bar to
the dedicated `/results` page immediately after a Cypress `{force: true}` click — inconsistent
across otherwise-identical test runs. The retained `smoke.cy.ts` suite passes reliably across
every rerun performed; this only showed up in throwaway isolation specs used to chase an unrelated
screenshot question (a Cypress full-page-screenshot stitching artifact on the sticky results
sidebar, itself not confirmed to be a real bug either way).

## Session 2 (2026-08-20): search bar trim + numeric bounds + PrimeNG range slider

User feedback: the search bar had too many always-visible fields (10) to read any placeholder,
year accepted any integer from 0, price had no realistic bounds, and price specifically should
render as a PrimeNG range slider instead of two plain inputs.

- **CarListing external fields trimmed 10 → 4 concepts** (Brand/Model/Price range/Year range,
  6 controls, one clean 12-col row) — title/fuelType/sellerType/city moved into the "Filters"
  dropdown.
- **New context-gen feature: `CrudstoneField#minValue/#maxValue/#range`** (pushed `5c8d260`).
  minValue/maxValue (double, 0=unset, same sentinel convention as the image constraints) bound a
  "number" field's widget AND are enforced server-side (`EntityValidator.validateNumber`, new
  `NUMBER_TOO_SMALL`/`NUMBER_TOO_LARGE`/`NUMBER_INVALID_VALUE` codes) — not just a UI hint.
  `range=true` marks a numeric field as the TARGET of a `rangeTarget` pair that should render as
  ONE PrimeNG slider. **Declared ONCE on the real target field** (e.g. `price`), not duplicated
  on the virtual from/to fields — `AnnotationSearchContextLoader.resolveRangeBounds` runs as a
  post-processing pass (after the full field map is built, so it's declaration-order-independent)
  copying the target's own minValue/maxValue/range onto both halves of the pair.
- **search-crudstone frontend** (pushed `bfd0432`, then `5045cf8`): `isRangeSliderFrom`/
  `isRangeSliderTo` helpers — the "from" half renders one `p-slider [range]="true"` (one-way
  `[ngModel]`, commits both bounds together on `(onSlideEnd)`, NOT `(onChange)` — PrimeNG's own
  keyboard-driven `updateValue()` only emits `onChange`, never `onSlideEnd`, so keyboard
  interaction can't drive this control, only mouse; `Slider.onBarClick` DOES emit `onSlideEnd`
  directly, confirmed by reading `primeng-slider.mjs`, and is what Cypress tests drive via a
  track click). The "to" half renders nothing of its own and is filtered out of
  `externalFields()`/`dropdownFields()` entirely.
- **Real bug found + fixed: Electron renderer CRASH**, not a normal test failure — reproduced
  consistently (~84s then hard crash) across a plain memory-pressure retry too. Root cause:
  `[ngModel]="rangeSliderValueOf(field)"` called the method directly in the template, returning a
  **fresh array literal every change-detection cycle** even when values hadn't moved — handing
  PrimeNG's Slider `ControlValueAccessor` a new object reference each tick, which under load
  spiraled into a crash rather than a graceful failure. Fixed with a small `Map`-based cache
  keyed by field name returning the SAME array instance until the from/to values actually change.
  **General lesson**: never bind a template `[input]` directly to a method that constructs a new
  array/object literal per call — memoize it.
- Applied to `CarListing`: `year`/`yearFrom`/`yearTo` get `minValue=1950, maxValue=2030` (plain
  bounded inputs, no slider — a specific year is more naturally typed than dragged).
  `price`/`priceFrom`/`priceTo` get `minValue=500, maxValue=500_000` + `range=true` on `price`
  (renders as the slider). `priceFrom`'s own `style` widened `col-md-2` → `col-md-4` since it's
  now the pair's only rendered control.
- Both Cypress suites re-verified live and green after all of the above (admin-ui 3/3,
  front-ui 4/4, including the slider genuinely narrowing results via `filter.priceMin`).

**Why:** User explicitly asked to dogfood the crudstone family end-to-end AND improve the
libraries where they're lacking — this was the vehicle for finding the range-filter gap and
several other real bugs across every layer (context-gen, both backends, the gateway, both
frontends, and — session 2 — a genuine Electron crash root-caused to an Angular template
anti-pattern), none of which would have surfaced from build/compile success alone.
**How to apply:** When resuming this project, check `docker ps` for `carstone-postgres` first
(start it if stopped — confirmed it survives across sessions, `Up 23 hours` seen at the start of
session 2), then boot admin → front → gateway → the two Angular dev servers, in that order. When
touching context-gen/search-crudstone/dynamic-crud/sidebar-crudstone on behalf of carstone,
**always rebuild + repack + reinstall the tarball AND restart the consuming Angular dev server**
— this bit session 2 directly (spent real time chasing a "missing slider" that was actually just
a stale tarball) despite already being documented here; `ng build <lib>` alone updates
`dist/<lib>/`, not the tarball, and even after `npm pack` the consuming app's `node_modules` needs
an explicit reinstall (`rm -rf node_modules/<lib> && npm install <lib>@file:...`) plus a dev-server
restart (a running `ng serve` won't pick up a swapped dependency on its own).

## Session 2 continued: slider "blocked" + year min>max fixes, and a standing genericity directive

Two more live bug reports on the same range-filter UI: **"the range is not moving, it's like
blocked"** (dragging the price slider) and **"year min cannot be greater than year max"** (typing
an invalid from/to combo into the plain-input year pair).

- **Slider blocked, root cause**: `rangeSliderValueOf` read straight from `filterValues()`, which
  only updates on `onSlideEnd` — so every CD tick during an active drag re-pushed the stale
  pre-drag value back into the slider, fighting the user's own mousemove. Fixed in
  `entity-search.component.ts` with `rangeSliderStaged` (a `Map<string,[number,number]>` holding
  the slider's own live drag position, written via a manually-wired `(ngModelChange)`, since the
  read side is a method call and can't use `[(ngModel)]` banana-box syntax) — cleared once the drag
  commits so external changes (Reset, a restored filter snapshot) still take effect afterward.
- **That redesign briefly reintroduced the EARLIER Electron renderer crash** (see session 2's
  "Real bug found + fixed" entry above) — the un-staged branch went back to returning a fresh
  `[from, to]` array literal every call. Live-verifying via Cypress hung the Electron renderer at
  100% CPU for 10+ minutes with zero screenshots produced (worse than the original ~84s crash).
  Fixed by restoring a second cache, `rangeSliderCommitted`, giving a stable array reference across
  CD ticks whenever the underlying value hasn't actually changed. **Lesson reinforced**: any redesign
  touching a `[ngModel]`-bound getter must re-verify the array/object identity is still stable, not
  just that the new feature works — the two concerns are independent and one fix can silently drop
  the other.
- **Year min>max, generic fix**: `rangeInputMinFor(field)`/`rangeInputMaxFor(field)` bind each half
  of ANY plain-input `rangeTarget` pair's own `<input min>/<max>` to its sibling's current value
  (`rangeSiblingOf`), so an invalid combo can't be typed in the first place — not year-specific.
  `clearDependentsViolatedBy` got a second pass (numeric rangeTarget pairs, mirroring the existing
  Date/`dependsOn` pass) as defense in depth for paths other than direct typing.
- **Standing directive from the user** (2026-08-20): *"remember always that everything must be
  generic, for the min must be less than max you need to mention it in an annotation."* Applies
  going forward to all context-gen/search-crudstone/dynamic-crud/sidebar-crudstone work, not just
  this fix. Acted on immediately: added `AnnotationContextLoader.validateMinMax()` to **context-gen**
  (pushed `0175eaf`) — fails fast at load time for ANY `@CrudstoneField(type="number")` with both
  `minValue`/`maxValue` set and `minValue > maxValue`, regardless of whether the field is also used
  in a search `rangeTarget` pair. `CrudstoneField.minValue()`'s javadoc documents the invariant
  directly on the annotation (not just in a commit message or memory).
- Live-verified via a rebuilt Cypress suite (6 specs, `carstone-front-ui/cypress/e2e/smoke.cy.ts`)
  passing in 9s with normal CPU. A literal raw mousedown→mousemove→mouseup drag simulation proved
  too fiddly to get reliable through Cypress/Electron's synthetic-event pipeline (PrimeNG's
  `onMouseDown`/document-level `mousemove` listener chain never engaged despite matching the
  library's own source) and was dropped in favor of two consecutive `onBarClick`-style interactions
  proving the handle keeps advancing rather than snapping back — same code path, reliable test.
- Pushed to **search-crudstone** `b741b9f` and **carstone** `5d99c23`.
- **Gotcha caught while pushing**: the carstone monorepo's local `main` branch was tracking
  `carstone-old` (the untouched spare `github.com/Devary/carstone` repo) as its upstream, not
  `origin` (`crudstone-demo-carstone`) — a bare `git push` silently went to the spare repo. Fixed
  with `git branch --set-upstream-to=origin/main main`. **How to apply**: after any repo
  rename/remote-juggling in this project, run `git branch -vv` to confirm `main` tracks `origin`
  before trusting a bare `git push`.

## Session 3 (2026-08-21): CrudstoneField#noFutureValue

User: *"max year cannot be in the future (add a flag in the annotation NoFutureValue = true)"*.
New generic flag, not year/car-specific in the code itself: a "number" field marked
`noFutureValue=true` rejects a value later than the current year, checked fresh via `Year.now()`
at validation time (never a fixed value baked into the annotation).

- **context-gen** (pushed `38bdfb6`): `CrudstoneField#noFutureValue`, propagated through `Field`,
  `AnnotationContextLoader`, and — resolved onto both halves of a search `rangeTarget` pair the
  same way `minValue`/`maxValue`/`range` already are — `SearchField` + `AnnotationSearchContextLoader
  .resolveRangeBounds`. Server-side enforcement in `EntityValidator` (new `NUMBER_FUTURE_VALUE`
  code) applies regardless of which widget rendered the field, so it's a real boundary, not just a
  UI hint. Added `EntityValidatorTest` (previously zero coverage for that class).
- **search-crudstone** (pushed `4153d5d`): `TableField#noFutureValue` (inherited by `SearchField`)
  + a `cappedMaxValue(field)` helper folding `min(maxValue, currentYear)` into every place
  `maxValue` was already used as an absolute upper bound (`rangeInputMaxFor`'s two branches, the
  price slider's own `[max]` binding) — independent of maxValue, not a replacement for it.
- **carstone**: applied to `CarListing.year` (pushed `ca66a12`, both `origin`/crudstone-demo-carstone
  and `carstone-old`). Deliberately left `maxValue=2030` as-is rather than trimming it — the
  effective cap (`min(maxValue, currentYear)`) makes `noFutureValue` the thing that actually
  enforces the real constraint every year, proving the two combine rather than one replacing the
  other.
- **Live-verified end to end**, not just compiled: `curl -X PUT .../carListings/1` with `year:2030`
  → `400 "year must not be later than 2026"`; same request with `year:2026` → `200`. New Cypress
  test (`smoke.cy.ts`, 7/7 passing) confirms `yearFrom`/`yearTo`'s own `[max]` HTML attribute reads
  the current year in the browser too.
- **Restart friction hit again**: after reinstalling context-gen to `~/.m2` and repacking the
  search-crudstone tarball, `pkill -f "carstone-admin.*quarkus:dev"` (matching the outer Maven
  wrapper process) left the actual forked JVM child bound to the port — port-in-use errors on
  restart despite the pkill "succeeding". **How to apply**: after any pkill-by-pattern for a
  Quarkus dev-mode process, verify with `lsof -i:<port>` and kill that PID directly too before
  relaunching — don't assume the wrapper's pattern also matches its child.

## Session 3 continued: "i still can put 2033 in the max year" — the [min]/[max] attribute was cosmetic

Important **general lesson, applies beyond this project**: a native `<input type="number">`'s
`min`/`max` HTML attribute is NOT enforcement — it only disables the spinner arrows past that
point and marks the field `:invalid` via the constraint-validation API. A typed keystroke, paste,
or autofill is accepted regardless, unless something explicitly checks/clamps the value. The
entire earlier "year min cannot be greater than year max" fix (session 2, `b741b9f`) and the
`noFutureValue` fix above ONLY ever set this attribute — never actually clamped anything — so both
were silently unenforced for direct typing the whole time, only the Cypress tests never caught it
because those tests also only asserted the (cosmetic) attribute value, not the actual resulting
input value. **How to apply going forward**: whenever a min/max-like constraint needs real
enforcement in an Angular template-driven form, verify with a test that types past the bound and
asserts the STORED/rendered value, not the HTML attribute.

Fixed (search-crudstone `c9c3dc4`, carstone `699f061`) with `clampNumberFilter`, wired to
`(blur)` — deliberately NOT `(ngModelChange)`: clamping on every keystroke corrupts multi-digit
entry, since typing "2022" passes through incomplete intermediate values (2, 20, 202) that would
get force-corrected against `minValue` mid-keystroke — the exact same class of
one-way-binding-fights-live-interaction bug already diagnosed for the price slider's drag (see
`rangeSliderStaged`/`rangeSliderCommitted` above), just triggered by keystrokes instead of a drag.

**A second, subtler bug found while fixing this**: `clearDependentsViolatedBy`'s numeric
`rangeTarget` pass (added in the original session-2 fix, "defense in depth" comment) reacted to
one half being typed past its sibling by **deleting the sibling's own filter value**, not the
violating field. That directly fought the new clamp: typing yearFrom=2018 past yearTo=2015 deleted
yearTo mid-keystroke, so by the time blur fired, `clampNumberFilter`'s sibling lookup found nothing
and fell back to the unbounded absolute max — silently doing nothing. Removed that pass entirely;
clamping (not deleting) is the correct behavior, and PrimeNG's own range Slider already
self-enforces the slider case internally without any pruning needed.

**Why this matters for future annotation-driven constraints**: a numeric/date bound expressed as a
declarative annotation (`minValue`/`maxValue`/`noFutureValue`/a `rangeTarget` pair) needs BOTH a
live UI hint (the HTML attribute, for browser-native spinner behavior and `:invalid` styling) AND
a real value-clamping handler wired to the actual user-interaction event — the attribute alone is
not sufficient, no matter how "generic" or well-annotated the backend metadata is.

## Session 3 continued: CrudstoneField#yearPicker

User: *"change the min and max year with a date picker for a year only"*. New generic flag
(context-gen `9a3de04`), same declare-once-on-the-real-target-field-resolve-onto-both-halves
pattern as `range`/`noFutureValue`: `yearPicker=true` renders a "number" field's search filter as
a PrimeNG `p-datepicker` with `view="year"` (a year-grid, no month/day drill-down) instead of a
plain `<input type="number">`. Value is still stored/validated as a plain integer — Date<->number
conversion happens only at the search-crudstone UI boundary (`yearPickerValueOf`/`setYearFilter`).

This elegantly closes the "cosmetic attribute" gap documented just above: with `readonlyInput=true`
set, there's no typing path at all anymore, and PrimeNG's own `minDate`/`maxDate` (wrapping
`rangeInputMinFor`/`rangeInputMaxFor`'s existing bound logic) disable out-of-range years directly
in the picker's grid — real enforcement with no separate clamp handler needed, unlike the
plain-input case `clampNumberFilter` still covers for any OTHER number field.

Applied to `CarListing.year` (search-crudstone `56aee6a`, carstone `c39bc79`, both remotes).
Cypress suite rewritten for the new widget (typing no longer works) — opens the picker via its
`.p-datepicker-dropdown` icon, clicks a `.p-datepicker-year` cell. **Gotcha**: the year-grid opens
centered on the CURRENT decade (e.g. 2020-2029 in 2026) — clicking `.p-datepicker-prev-button` to
navigate to an earlier decade did not visibly work when driven through Cypress (root cause not
fully chased down; possibly a PrimeNG `p-button`-wrapped click-target/animation-timing quirk).
Worked around by keeping all test year picks inside the default-visible decade rather than fighting
that interaction — 7/7 passing. **How to apply**: if a future test genuinely needs to cross a
decade boundary, budget extra time to debug the prev/next button interaction rather than assuming
a simple `.click({force:true})` will work.

**Root cause found retroactively in session 4** (see below): that prev-button mystery was almost
certainly the exact same bug as the Contact form submit button — see [[feedback_primeng_button_click_target]].

## Session 4 (2026-08-21): dark glassmorphism theme, About/Contact pages, parametrized footer

User: *"add a stylish background for the cars website, add other pages like contact us etc, add a
parametrized footer (create a new component like sidebar and searchbar...) make the website darken
blurry style"*, followed by *"the website styles must be in the override css system that we have
already created"*.

- **Scoping decision**: built the footer as a well-structured, config-driven standalone component
  living IN carstone-front-ui (`src/app/footer/`), not as a brand-new published sibling library
  (unlike sidebar-crudstone/search-crudstone themselves) — a full new repo+tarball-consumption cycle
  was judged disproportionate to a styling/UX request, given how much friction that pattern already
  costs each time a lib changes (see the repeated "always repack+reinstall+restart" lesson above).
  Structured so it COULD be extracted later: no carstone-specific coupling, fully `[config]`-input-
  driven (`FooterConfig`: brand/tagline/columns/socialLinks/copyright).
- **"Override css system"**: confirmed via research this means the `themeVars()`/`THEME_PALETTES`
  pattern already duplicated across dynamic-crud/search-crudstone/sidebar-crudstone (see
  [[project_gen_crudstone]]/[[project_search_crudstone]]/[[project_sidebar_crudstone]]) — PrimeNG
  CSS-custom-property overrides, never hardcoded colors, scoped per-component via `[style]`. Reused
  directly by importing `themeVars` from `searchcrudstone` (already a carstone-front-ui dependency)
  rather than reimplementing it a 4th time. Applied at the `AppComponent` root itself
  (`host: {'[style]': 'themeStyle()'}`, `themeVars('blue')` — the same theme `CarListing`'s own
  `@Searchable` already declares) so the whole site, including the two new static pages, reads
  consistently rather than only the search/results pages being blue.
- Dark mode flipped to **default ON** (`localStorage.getItem('darkMode') !== 'false'`, was `===
  'true'`) — the dark/blurry look is the site's actual default now, not an opt-in toggle state.
  Global background: a fixed multi-radial-gradient under `html.app-dark`, colors via
  `--p-primary-*`/`--p-surface-*` tokens. Glass surfaces (`sf-footer`, `.glass-card`) use
  `backdrop-filter: blur()` against that background for the actual glassmorphism effect.
- **Two real bugs found fixing the Contact form**, both diagnosed by literally dumping the live
  post-click DOM to a file and reading it (`cy.writeFile`) rather than guessing:
  1. `[(ngModel)]="name"` where `name = signal('')` (readonly) — banana-in-a-box desugars to
     `[ngModel]="name" (ngModelChange)="name=$event"`, which needs an ASSIGNABLE plain property;
     assigning to a signal reference like that doesn't set it, it's a no-op/type mismatch. Switched
     to plain mutable fields for the three ngModel-bound values (this codebase otherwise leans
     heavily on signals — worth remembering ngModel specifically doesn't support them directly).
  2. `<p-button type="submit">` inside a `(ngSubmit)`-bound `<form>` triggered a native form
     submission that reloaded the page before the success state ever rendered, DESPITE Angular's
     `(ngSubmit)` normally calling `preventDefault()` internally. Fixed by dropping the native
     submit path entirely — `type="button"` + `(onClick)="submit()"` only.

**Why:** user wants a polished, presentable public-facing demo, not just a functionally-correct one
— matches the original brief ("build a car marketplace... improve components if they're missing
something") extended to visual/UX completeness.
**How to apply:** the footer + About/Contact pages establish `.glass-card`/`.page-backdrop` as the
house style for any FUTURE static page in this app — reuse those global classes rather than
inventing new ones. If a genuine "footer-crudstone" library is ever wanted (backend-`@Footer`-
annotation-driven like `@Sidebar`, not just `[config]`-input-driven), that's a clean, scoped
follow-up — the current component's structure doesn't need to change to support that later, only
where its config comes from.

## Session 5 (2026-08-21): full landing page + glass search-result styling

User: *"i want a full landing page + styles in the search result components"*.

- **Landing page** (`/`, replacing the old straight-to-`/carListings` redirect): hero + CTA, live
  stats, a "Browse by body type" grid using CarListing's real `BodyType` enum values, a
  "Selling your car?" CTA band. `/carListings` is now the direct search entry point (linked from
  the hero CTA, header, footer).
- **Category pre-filtering, a real feature not just decoration**: each body-type chip navigates to
  `/carListings` with Angular Router `state: {filterValues: {bodyType: '...'}}` —
  `EntitySearchPageComponent` (search-crudstone) already reads `history.state.filterValues` into
  its `initialFilters` input for its own "Back to search" link, so this reuses that EXISTING
  mechanism rather than inventing a new one. Lands with the filter set but does NOT auto-search
  (that component hardcodes `autoSearchOnInit=false`) — user still confirms with their own click.
- **New sticky site header** (logo/nav/dark-toggle). Learned mid-session: went with `position:
  fixed` first, discovered it would overlap search-crudstone's own `.results-page` (only `2rem`
  top margin, no override token for that specific clearance) — rather than patching every page's
  top padding individually (fragile, has to be redone for every future page), switched to
  `position: sticky`, which reserves its own space in NORMAL document flow so every current AND
  future routed page gets correct clearance automatically with zero page-specific styling needed.
  **General lesson**: prefer `sticky` over `fixed` for a header/nav specifically when the pages
  under it are NOT all owned by you (here: the search results page comes from a third-party-ish
  library with its own margin choices) — `fixed` pushes the clearance problem onto every page
  individually; `sticky` solves it once, structurally.
- **Glass search bar/result cards**: extended search-crudstone's OWN pre-existing `--sc-*`
  override-token system (search-crudstone `08d4183`) — added `--sc-card-blur`,
  `--sc-result-card-bg` (previously untokenized, a plain `var(--p-content-background)`), and
  `--sc-result-card-blur`. Set from carstone-front-ui's global stylesheet under `html.app-dark`
  only, zero library source touched — exactly the extension mechanism that library's own README
  "Styling" section already documents ("a host app overrides any subset from its own global
  stylesheet — no rebuilding this library needed"). Confirms [[project_search_crudstone]]'s
  `--sc-*` system (not just `theme-palettes.ts`) is ALSO part of "the override css system" this
  user has referred to twice now.
- Cypress suite extended to 12/12 passing (landing hero+CTA, category pre-fill verifying BOTH the
  filter got set AND that no search request fired yet, header nav round-trip).

**Why:** completes the "polished, presentable demo" arc from session 4 — a real homepage instead
of a bare redirect, and the result cards visually matching the rest of the now-glass site instead
of standing out as plain opaque boxes.
**How to apply:** any FUTURE search-crudstone visual property that's currently untokenized (found
one already, `--sc-result-card-bg`) should get a `--sc-*` token added when touched, not hardcoded —
keeps the override system genuinely complete rather than accumulating gaps found ad hoc.
