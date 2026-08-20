---
name: project-carstone
description: "carstone monorepo (github.com/Devary/carstone): autoscout24-style car marketplace dogfooding the whole crudstone family (context-gen, dynamic-crud, search-crudstone, sidebar-crudstone) — 2 Quarkus backends + no-auth gateway + 2 Angular apps, all built and live-verified in one session (2026-08-19/20)"
metadata:
  node_type: memory
  type: project
  originSessionId: 9490a6be-eca4-4020-a624-b91245b42f3c
---

# carstone — car marketplace dogfooding the crudstone family (built 2026-08-19/20)

Built end-to-end in one very long session, from a one-line user request ("build a car buy/sell
site using our components, improve them if they're missing something"). See [[project_gen_crudstone]]/
[[project_search_crudstone]]/[[project_sidebar_crudstone]] for the underlying libraries this
dogfoods. Repo: `github.com/Devary/carstone` (public), monorepo, Maven reactor + 2 sibling Angular
CLI apps (not part of the reactor).

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

**Why:** User explicitly asked to dogfood the crudstone family end-to-end AND improve the
libraries where they're lacking — this was the vehicle for finding the range-filter gap and 6
other real bugs across every layer (context-gen, both backends, the gateway, both frontends),
none of which would have surfaced from build/compile success alone.
**How to apply:** When resuming this project, check `docker ps` for `carstone-postgres` first
(start it if stopped), then boot admin → front → gateway → the two Angular dev servers, in that
order. When touching context-gen/search-crudstone/dynamic-crud/sidebar-crudstone on behalf of
carstone, remember the tarball-repack step or the Angular apps will silently keep using stale
lib code.
