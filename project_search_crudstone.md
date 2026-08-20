---
name: project-search-crudstone
description: "gen/search-crudstone: new sibling Angular workspace to dynamic-crud, hosting the 'searchcrudstone' library — a read-only entity search/typeahead picker, same context-gen contract as crudstone's own CRUD table"
metadata: 
  node_type: memory
  type: project
  originSessionId: 234d18c6-f1ac-4b2e-bcc3-12e9a700a46e
---

# search-crudstone — entity search/pick library (created 2026-07-26)

New sibling project to [[project-gen-crudstone]] (`gen/dynamic-crud`/`crudstone`), at
`/home/devary/sandbox/gen/search-crudstone`. Same idea (metadata-driven, backed by the same
context-gen contract, GET /context/{name} + GET /{entity}), different purpose: a single search
bar + filter panel that finds and lets a user **pick one existing row** of a chosen entity. Never
creates/edits/deletes anything — a picker, not a CRUD screen.

## How it was built
Copied `dynamic-crud`'s whole Angular workspace (rsync, excluding node_modules/.git/dist/.angular),
renamed the library project `crudstone` → `searchcrudstone` (angular.json, tsconfig path, package
names, prefix `d` → `sc`), then **deleted** everything CRUD/table-only: dynamic-table, entity-form-
dialog, entity-detail-dialog, object-list, dialog-close-footer, export.service, theme-palettes,
MessageAction. Trimmed `EntityContext`/`TableField` to a narrower read/filter-only slice (no
`createEditStrategy`/`customCallParams`/`fieldObjectsFilter`/image-upload-constraint fields/toolbar
flags) — deliberately NOT matching crudstone's own full interface, since TS structural typing means
a narrower client-side interface doesn't require any backend change; context-gen's actual JSON
response is unchanged and unaware which library reads it.

## New core component: `EntitySearchComponent` (`sc-entity-search`)
PrimeNG AutoComplete typeahead + a `p-popover` filter panel (funnel icon button, badge shows active
filter count). Rows fetched ONCE per context via `EntityService.getAll()`, matched client-side
against both free-text query and active filters (same "load once, filter client-side" approach
crudstone's own global search already used) — no new backend search endpoint needed.
- Free-text matching: every `showsInTable` field's own `displayValue()` (new helper, added to this
  library's own `TableField.ts` copy) — so a relation's related-row name, or an enum's humanized
  label, is searchable, not just literal raw stored strings.
- Filter panel: one control per filterable field type — text (contains), date (exact-day match via
  `p-datepicker`), enum (multiselect of humanized values), select/multiSelect (multiselect of the
  related entity's own rows, options fetched generically via `TableField.listType` +
  `EntityService.getAllByEntityName`, same mechanism crudstone's column filters use).
- `[field]="primaryFieldName()"` on `p-autoComplete` (not a custom template) is what controls what
  text shows in the input box after picking a suggestion — the FIRST displayable field's raw value,
  not the nicer computed `resultPrimary()` label. Known simplification, not fixed.
- **Real bug found+fixed**: updating the `suggestions` signal from outside PrimeNG's own
  `completeMethod` flow (i.e. from `setFilter`/`clearFilters`) does NOT reopen an already-closed
  suggestion panel — PrimeNG only auto-shows the panel in direct response to its own internal
  typing-triggered flow. Confirmed via `ng.getComponent()` that the `suggestions()` signal held the
  CORRECT filtered rows the whole time — the UI just wasn't re-displaying them. Fixed with
  `private readonly autoComplete = viewChild(AutoComplete);` + calling `.autoComplete()?.show()`
  explicitly after a filter-driven re-search (not after a normal typed search, which already shows
  the panel itself). Lesson: a signal update alone doesn't reopen a closed PrimeNG overlay that
  isn't itself watching that signal for open/close state — same class of gotcha as the
  [[project-gen-crudstone]] menu-array-stability bug, different mechanism (overlay visibility vs.
  DOM rebuild-on-new-reference).

## Search button + persistent results panel (added 2026-07-26, same session)
A "Search" button next to "Filters" shows the FULL filtered result set as a persistent list of
rows below the search bar (`showResults`/`searchResults` signals), distinct from the autocomplete's
own transient suggestion dropdown: not capped at `maxSuggestions`, not gated by `minQueryLength`
(clicking it with an empty query + active filters is a valid "browse all" action), and critically
**not a PrimeNG overlay** — so unlike the dropdown, it stays open and correctly updated across
popover interactions (opening the filter panel, picking an option) that would otherwise count as
"click outside, close the overlay" for the dropdown.
- **Real bug found+fixed**: `lastQuery` (what the Search button searches with) was only updated by
  `search()`, itself only called from PrimeNG's own debounced/gated `completeMethod` — so clicking
  Search immediately after typing (before that debounce elapsed) ran against a stale/empty query.
  Fixed by tracking the native `(input)` event directly (`onRawInput`, reads
  `event.target.value`), synchronous on every keystroke, independent of the dropdown's own
  debounce — confirmed via `ng.getComponent()` tracing that `lastQuery` now matches actual typed
  text at every step, where before it lagged or missed short/fast edits entirely.
- **Cypress gotcha this exposed**: the old filter-narrowing test drove the *dropdown* through a
  filter change and pressed Escape to close the filter popover — Escape (or ANY click outside)
  closes every open PrimeNG overlay it can reach, including the dropdown that had *just* reopened
  from the filter change, not only the popover being explicitly dismissed. This isn't a bug, it's
  correct overlay semantics — but it makes testing the dropdown through a filter-panel interaction
  inherently fragile. Rewrote the test against the new persistent results panel instead (immune to
  this since it's plain markup, not an overlay) — 9/9 Cypress tests green, no flakiness after the
  rewrite (confirmed via 3 repeated full-suite runs).

## Backend: zero new code, one CORS line
Reused `quar-crud-host` (`:9100`) as-is — no new entities, no new endpoints. Only change:
`application.properties`'s `quarkus.http.cors.origins` gained `http://localhost:5901` (this app's
dev port) alongside the existing `5900` (dynamic-crud's). Both dev servers can run simultaneously
against the same backend. Demo app tested live against BOTH `conventions` (many field types: text,
links, dates, enums, relations, progress bar) and `sharacters` (a totally different, simpler
entity) to confirm genuine per-entity-code-free genericness.

## Repo state
Fresh git repo (not a copy of dynamic-crud's history — deliberate, since it's conceptually a new
project even though scaffolded from a copy), branch `main`, pushed to
`https://github.com/Devary/search-crudstone` (2026-07-26) — the remote had a GitHub-created Apache
2.0 LICENSE-only initial commit, merged in via `--allow-unrelated-histories` (clean, no conflicts)
rather than force-pushed over. Initial commit covers the whole scaffold + library + demo wiring +
Cypress suite (5/5 green, new specs written from scratch for search/filter/select/cross-entity
behavior — the OLD crud-flow.cy.ts/entity-table.cy.ts specs were deleted, not adapted, since they
tested features that no longer exist here).

## Naming/ports (mirrors dynamic-crud's own scoped naming)
- npm package: unscoped `searchcrudstone` (workspace dir `search-crudstone` has a dash, package
  name doesn't — same convention `crudstone`/`dynamic-crud` already established).
- Demo app dev port 5901 (crudstone's dynamic-crud uses 5900) — `npm start`, Dockerfile/nginx.conf
  both updated to 5901 too.
- Component prefix `sc` (crudstone uses `d`) — `sc-entity-search`, `sc-entity-search-page`.

## SearchContext wired end-to-end + athome.lu-style redesign (2026-07-26, same session)

Convention (quar-crud-host) is now `@Searchable`: `name`/`startDate`/`hostStudio`/`country` are
`external=true` with Bootstrap-style grid `style` classes summing to 12 (4+3+3+2), 16 other
filterable-typed fields are dropdown (`external=false`, own `order`), `contactEmail` demonstrates
`showInResult=false` (searchable, never returned). Studio also made `@Searchable` (name external
col-md-6, website dropdown) specifically so there'd be a SECOND real `@Searchable` entity to prove
cross-entity genericness — `sharacters`/`animes` (YAML-only contexts) can't carry `@Searchable` at
all since it's a Java annotation, so they no longer work with search-crudstone now that matching
is scoped to `SearchContext` fields rather than "every context field" like before.
- **Backend**: `SearchContextRegistry` (mirrors `ContextRegistry`, scans `@Searchable` via
  `EntityScanner.scan(Searchable.class, packages)`) + `GET /context/{name}/search` (added as a
  sub-path on the existing `ContextResource`, synchronous — same as the CRUD context endpoint,
  NOT reactive, since serving already-built JSON metadata needs no query; only the eventual
  actual-search-execution endpoint would need `SearchHandler`'s reactive contract). 28/28
  quar-crud-host tests green (was 23; +5 new for the search-context endpoint).
- **Frontend**: `SearchField extends TableField` (so every existing type helper — isDate, isEnum,
  displayValue, enumOptions, ... — works unchanged, zero duplication) + `SearchContext` models,
  `ContextService.getSearchContext(name)`. `EntitySearchComponent` now splits fields into
  `externalFields()`/`dropdownFields()` from the fetched SearchContext (not a client-side
  "every filterable field" heuristic anymore) and projects every result row down to its
  `showInResult=true` fields before ever displaying/emitting it (matching still uses the full row).
- **Redesign to match a user-supplied reference (athome.lu, a real-estate portal)**: fetched the
  actual site via Playwright screenshot (WebFetch alone — HTML-to-markdown — loses too much visual
  info for a styling task) to see the real pattern: one continuous card, thin dividers between
  fields instead of individually-boxed inputs, a prominent solid CTA button, a secondary row of
  compact always-visible fields, minimal ghost-link secondary actions, no per-field labels (each
  control's own placeholder doubles as one). Rebuilt `entity-search.component.html/scss` to match:
  `.entity-search-card` (bordered/shadowed) → `.entity-search-main-row` (query + solid `Search`
  button, `var(--p-primary-color)` not a hardcoded brand color) → `.entity-search-external-row`
  (the 4 external fields + a plain `Filters` text+icon trailing link, `col-md-N` re-implemented as
  flex-basis percentages, no Bootstrap dependency). Search/Filters became plain `<button>`s instead
  of `<p-button>` for full styling control — Cypress's `clickButton` helper (assumes a PrimeNG
  `p-button` wrapper) had to be swapped for direct `cy.get(...).click()` on these two specifically.
- 9/9 Cypress green (stable across repeated runs), both builds clean, committed+pushed
  (`5bd44bf`). context-gen itself unchanged this round (already had the SearchContext contract
  from the prior message); quar-crud-host has no git repo, so its Convention/Studio/registry/
  resource changes exist only on disk, not in any commit.

## Autocomplete removed entirely, single athome-style row, layout/UX fixes (2026-07-26, later same day)

User clarified the athome reference further: no free-text "search anything" box at all — the bar
IS made of exactly the `@SearchableField(external=true)` fields, nothing else. Removed
`AutoComplete`/`AutoCompleteModule` and all its supporting code (`matchesQuery`, `applySearch`,
`search()`, `onRawInput`, `onAutoCompleteSelect`, `clearSelection`, `lastQuery`, the `autoComplete`
viewChild, `SEARCH_CRUDSTONE_CONFIG`/`minQueryLength`/`debounceMs`/`maxSuggestions`). Collapsed the
two-row card (main query row + external-fields row) into ONE row: external fields (styled by their
own `col-md-N`, ordered, no separate `<label>`, placeholder doubles as one) → loading spinner →
`Filters` link (dropdown fields, only if any exist) → `Search` button, in that order (Search must
be the LAST element — user-specified). `placeholder` input on the component was removed as
meaningless now. Rewrote `entity-search.cy.ts` from scratch around the persistent results panel
only (no dropdown-driven tests at all) — 7/7 green.

Bugs found+fixed in this round:
- **`flex-wrap` layout bug**: the 4 external fields' `col-md-N` values sum to exactly 12/12 by
  design, so with `flex-wrap: wrap` the browser computed each field's un-shrunk hypothetical size,
  saw them alone already fill 100%, and pushed Search+Filters to a SECOND flex line without ever
  shrinking the fields. Confirmed via computed-style/offsetTop inspection. Fixed: `.entity-search-
  row` → `flex-wrap: nowrap` (forces one line so shrink math applies across every item together)
  + `.col-md-N` → `flex: 0 1 X%` (was `0 0`, no-shrink) + `flex-shrink: 0` on the button/link/spinner
  so only the fields absorb the shrink.
- **`clearFilters()`/`activeFilterCount` scoping bug**: both operated on the WHOLE `filterValues`
  map, so clicking "Clear filters" inside the Filters popover (which only ever shows dropdown
  fields) also wiped external field values the user could still see filled in on the main row —
  caught via a Cypress assertion failure. Fixed: both now scope to `dropdownFields()` names only.
- **Click-target width bug** (user-reported): `::ng-deep .entity-search-segment` only set
  `width: 100%` on `.p-multiselect`/`.p-datepicker`, NOT plain `input` — so a text field's actual
  clickable/focusable box was PrimeNG's own narrower default width, while the wider segment div
  around it looked like part of the same field but did nothing when clicked. Fixed by adding
  `input` to that selector. Verified live via Playwright: clicks at 25/50/75/90% across the segment
  width all now focus the input (was previously only the left ~50%).
- **startDate wrong widget type** (user-reported): `Convention.startDate` was `String` with
  `@CrudstoneField(type="inputText")` despite holding real ISO date values — changed to
  `LocalDate` with `type="date"` (`DataSeeder` seeding call updated to `LocalDate.of(...)`
  accordingly). Quarkus dev mode's own hot-reload picked up this schema-affecting entity change
  (drop-and-create) on the next request with NO manual restart needed — confirmed via curl
  returning a proper date string post-change, contradicting the earlier assumption that a full
  process restart would be required for schema changes (that requirement was only ever confirmed
  for *external snapshot jar* changes, e.g. context-gen, not the host app's own source).
- **Button order**: Search must be the last button in the row — swapped Filters/Search order in
  the template (Filters now comes before Search).

**Sizing/chrome polish** (user-specified, no bug): `.entity-search-card` centered via
`margin: 25vh auto 0` (25% of window height as top margin) and `width: 60%` with a
`min-width: min(900px, 95vw)` floor added afterward — pure 60% got uncomfortably narrow on
~1280px-wide windows and started truncating labels like "Any Host Studio" to "Any Ho…"; the floor
trades strict 60%-on-every-viewport for "60% on wide screens, never narrower than ~900px" so field
placeholders stay legible. Row height increased via `min-height: 4.5rem` on segments/button/link +
taller input/multiselect/datepicker padding (`1.35rem 0`) and `font-size: 1.05rem`. Border/shadow
made more prominent: `2px solid var(--p-primary-color)` (was a subtle 1px neutral border) +
stronger shadow (`0 12px 36px`, 18% opacity, was `0 4px 20px` 10%).

Verified live via Playwright (headless chromium from `~/.cache/ms-playwright`, executablePath
pointed at the cached `chromium-1228` binary since a freshly-installed playwright npm package
defaults to expecting a newer bundled revision not present in cache). All fixes + the full Cypress
suite (7/7) confirmed green; committed + pushed as `6b44d90`.

## @SearchResult annotation + card-grid results (2026-07-26, later same day, pushed `79bbca6`/`c7b17d6`)

User asked for a field-level `@SearchResult` annotation ("appears in the resulted json of the
search"), applied to `image`/`name`/`startDate`/`endDate`, plus results shown as CARDS (5/row on a
24"/1920px screen, responsive down to 1 column) with the image at 75% of the card's height and
other info below. Mid-implementation, user added: "search order must be unique and mandatory, for
all" — applied to both `@SearchableField.order` and the new `@SearchResult.order`.

**Design decision**: rather than bolt this onto the existing `SearchableField.showInResult`
boolean, introduced `@SearchResult` as a fully independent annotation/list — `SearchContext` now
carries `resultFields` alongside `fields`, and `showInResult` was REMOVED entirely (from
`SearchableField`, `SearchField`, both backend and frontend). A field can now carry
`@SearchableField`, `@SearchResult`, both, or neither — searchable-without-being-a-result and
result-without-being-searchable (the `image` field: no filter makes sense for it, but it belongs in
a result card) are both first-class, not edge cases of one flag. This is why `contactEmail` (still
searchable, never a result) and `hostStudio`/`country` (external search fields, but NOT results —
only `image`/`name`/`startDate`/`endDate` got `@SearchResult`) needed no annotation changes at all:
simply never marking them `@SearchResult` achieves what `showInResult=false` used to.

- **context-gen**: `SearchResult.java` (new, `order()` only, mandatory+unique like
  `SearchableField.order()` now is — no default on either anymore).
  `AnnotationSearchContextLoader` scans for `@SearchableField` OR `@SearchResult` (previously only
  `@SearchableField`), builds `fields`/`resultFields` as two separate lists, and validates order
  uniqueness in 3 independent groups: external fields, dropdown fields, result fields (a 4th/5th
  field silently sharing an existing field's order is almost always copy-paste — fails fast with
  `IllegalArgumentException: Foo: duplicate <group> order <n> on fields [a, b]`). 13/13 tests (was
  8), covering both-annotations-on-one-field, result-only fields, and both duplicate-order cases.
- **quar-crud-host** (disk-only, no git repo): `Convention.image`/`name`/`startDate`/`endDate` get
  `@SearchResult(order = 1..4)`; `contactEmail`'s old `showInResult = false` arg simply deleted (no
  replacement needed — absence of `@SearchResult` already achieves it). `Studio.name` also got
  `@SearchResult(order = 1)` — without it, Studio's search results would come back completely empty
  (no fields at all), breaking the existing "works generically for a different entity" Cypress test
  which expects "Wit Studio" to render. Quarkus dev-mode hot-reload picked up the schema/annotation
  changes fine on next request, no manual restart needed this time (only external-jar changes, e.g.
  the context-gen snapshot itself, need a manual kill+relaunch — confirmed again this round: had to
  restart for the context-gen jar bump, but NOT for the Convention/Studio annotation edits
  afterward). 29/29 tests passing (was 28).
- **Frontend**: `SearchField`/`SearchContext` models updated (removed `showInResult`, added
  `resultFields: SearchField[]`). `EntitySearchComponent` gained `resultFields`/`resultImageField`/
  `resultInfoFields` computeds (image field, if any, separated out from the text-info fields);
  `projectRow()` rewritten from a blacklist (strip `showInResult=false` fields from the full row) to
  a whitelist (build a fresh object containing only `resultFields` values, plus `id` — needed
  structurally for selection/`track`, never itself a `@SearchResult` field since `id` isn't even a
  declared field on the entity class, inherited from `PanacheEntity`).
- **Card grid CSS**: `.entity-search-results-grid` is a CSS `grid` (`repeat(N, 1fr)`), N stepping
  1→2→3→4→5 at 0/768/1200/1600/1920px — chosen so 1920px (a common 24" panel's native resolution)
  lands exactly on 5 columns, per the user's own reference point. Each `.entity-search-result-card`
  gets a fixed aspect ratio (`aspect-ratio: 3/4`) specifically so the 75%/25% image/info flex-basis
  split has a real height to resolve against — flex percentages on `flex-direction: column` need a
  definite container height, which `aspect-ratio` provides without a hardcoded pixel value (stays
  responsive as the grid column width itself changes). An entity with no image `@SearchResult`
  field (there isn't one in the fixture data — Convention's `image` is never actually populated by
  `DataSeeder`) falls back to a plain icon placeholder (`pi-image`), confirmed both by a dedicated
  Cypress test and by Playwright screenshots at 1920/900/400px showing 5/2/1 columns respectively.
- Removed dead config (`minQueryLength`/`debounceMs`/`maxSuggestions` in `SearchCrudstoneConfig`) —
  leftover from the autocomplete removed earlier this session, never actually read anywhere,
  caught while updating the README that still documented them.
- Both READMEs (`search-crudstone`'s own + context-gen's `DOCUMENTATION.md` Search contexts section)
  rewritten to match: `@SearchResult`'s attribute table, the mandatory+unique `order` rule and its
  fail-fast message shape, and the card-grid behavior description.
- Cypress 8/8 (was 7 — added one new test for the image-placeholder rendering), backend 29/29,
  frontend `tsc --noEmit` clean. Verified live via Playwright (headless chromium,
  `chromium-1228` pinned executablePath — see prior round's note on why) at three viewport widths.

**Follow-up bug, same session**: user pointed at `/studios` specifically — Studio has only ONE
external field (`name`, `col-md-6`), and since `.entity-search-row`'s children had no `flex-grow`,
the Filters/Search buttons ended right after that one field (~50-60% across) with a dead strip of
plain white space between them and the card's actual right edge, instead of sitting flush against
it. Fixed by wrapping the loading-spinner/Filters/Search trio in a new `.entity-search-actions` div
with `margin-left: auto` — pins them to the right edge regardless of how much/little space the
external fields need, without touching the fields' own flex-basis math (still `flex: 0 1 X%` per
`col-md-N`, still `nowrap` for the shrink-together behavior from the earlier round). Verified via
Playwright bounding-box math (button's right edge now within 2px — the border width — of the
card's own right edge) on both `/studios` (1 field) and `/conventions` (4 fields, confirmed
unaffected). Cypress 8/8 still green. Pushed `489a2cc`.

## Reset filters button + datepicker icon border fix (2026-07-26, later same day, pushed `2d40d3d`/`72f4ee4`)

Added a red "Reset filters" button between Filters and Search (in that DOM order, so it's the
last-but-one element in `.entity-search-actions`) — unlike the existing "Clear filters" (scoped to
dropdown fields only, lives inside the Filters popover), this one sits on the main bar itself and
wipes EVERY active filter, external fields included (`resetFilters()`: `filterValues.set({})`, then
re-runs the search if results are already showing — same pattern as `clearFilters`, just unscoped).
Disabled (`anyFilterActive` computed: `Object.keys(filterValues()).length > 0`) when nothing is
set — reuses `filterValues` directly since it already holds both external and dropdown entries
together in one map, no new per-field-group bookkeeping needed. Styled `var(--p-red-500)` (the
established danger-color convention already used elsewhere in this project family, e.g.
crudstone's own delete-confirmation styling) — a disabled state falls back to a neutral gray, not
a dimmed red, so it doesn't read as "a red button that's currently broken."

New translation key `resetFiltersButton` added (`SearchCrudstoneTranslations`/`defaultTranslations`
— no code changes needed elsewhere, `TranslationService.t()` is already fully generic over the key
union). Also noticed and removed genuinely dead config while touching the README:
`minQueryLength`/`debounceMs`/`maxSuggestions` on `SearchCrudstoneConfig` were leftover from the
autocomplete removed earlier this session and were never read anywhere anymore — confirmed via
grep before deleting.

**Test note**: a Cypress test opening the Filters popover then immediately clicking Reset failed —
not a bug in the new button, but a genuine PRE-EXISTING visual overlap: PrimeNG's Filters popover
(tall, since Convention has ~16 dropdown fields) can visually cover the main bar's own buttons
(Search included) at narrower viewport widths (confirmed via Playwright bounding-box math at
Cypress's default 1000×660 — the popover's box fully overlapped the Reset button's position, and
even partially covered Search too, pre-existing this change). Fixed the TEST by closing the popover
(`{esc}`) before clicking Reset, matching what a real user would naturally do — this is a
known/accepted rough edge in the popover's own positioning, not something this task asked to fix.

**Separate small fix, same round**: user pointed out the Start Date field's calendar-icon trigger
button still had its own default PrimeNG box (background + border) even though every other
control's box had been stripped via `::ng-deep .entity-search-segment` earlier — looked like a
small floating boxed button glued to the input's right edge rather than part of the same flat,
borderless field. Fixed by adding `.p-datepicker-dropdown { border: none; background: transparent;
color: var(--p-text-muted-color); }` to that same `::ng-deep` block. Verified via Playwright
element-screenshot of just the Start Date segment, before/after.

Cypress 11/11 (was 8 — 3 new Reset-button tests), `tsc --noEmit` clean, `.entity-search-actions`
right-alignment (previous round's fix) confirmed unaffected by the new button's insertion.

## Faded placeholders + vertical layout alternative (2026-07-26, later same day, pushed `3c174b2`/`d6e9a28`)

Placeholder text opacity made adjustable per user iteration: tried 0.05 → 0.1 → settled at 0.5
(`opacity: 0.5` on `input::placeholder` and PrimeNG's own `.p-multiselect-label.p-placeholder`,
shared across both `.entity-search-segment` and `.filter-field` via one combined `::ng-deep`
selector, since both use the same `filterControl` ng-template). If asked to change this again,
it's the two `opacity:` lines right after the `@use 'sass:math';` header comment block in
`entity-search.component.scss`.

**Vertical layout, added for comparison against the original horizontal bar**: `EntitySearchComponent`
gained a `layout` input (`'horizontal' | 'vertical'`, default `'horizontal'`) — 100% shared TS logic
(filtering, `runSearch`/`resetFilters`/`clearFilters`, `resultFields` projection, everything);
only the template branches (`@if (layout() === 'horizontal') {...} @else {...}`) and a small
amount of CSS differ. Vertical stacks every searchable field — `verticalFields()` computed:
`[...externalFields(), ...dropdownFields()]`, i.e. external fields first in their own order, then
dropdown fields in theirs — in ONE labeled column (reuses the existing `.filter-field` class
verbatim, so a field looks identical in either layout), Filters-popover-free since nothing's
hidden, with Reset/Search as two full-width buttons at the bottom (same `.entity-search-reset-
button`/`.entity-search-run-button` classes, just resized via a `.entity-search-vertical-actions`
wrapper — same colors/behavior, not a separate implementation). Data-cy suffixes are distinct per
layout to avoid collisions when both render on one page: `-vertical-filter` (field wrappers),
`-run-button-vertical`/`-reset-button-vertical`/`-loading-vertical`.

**Comparison page, demo-app-only** (not part of the published library): `ComparePageComponent`
(`src/app/compare-page/`) at route `/compare/:entity` (new, added alongside — NOT replacing — the
existing `:entity` route the Cypress suite already targets, specifically so the comparison
wouldn't create duplicate-data-cy ambiguity on `/conventions`/`/studios`) renders BOTH layouts for
the same entity stacked vertically on the page, each in its own `data-cy="compare-horizontal"`/
`"compare-vertical"` `<section>`, each with fully independent selection state (two separate
`sc-entity-search` instances = two separate component instances = two separate signal graphs,
zero shared state beyond both loading the same `EntityContext`). Verified live via Playwright:
typing/searching in the vertical instance doesn't touch the horizontal one's state at all.

Cypress 13/13 (was 11 — 2 new tests scoped to `/compare/conventions`, both using `.within()`-style
`[data-cy=compare-vertical] [data-cy=...]` scoping to avoid ambiguous matches against the
horizontal instance's own same-named-field elements on the same page).

**Correction, same session**: user clarified the vertical layout must ALSO split external vs.
dropdown fields exactly like horizontal — not flatten everything into one column. Reworked:
vertical now shows only `externalFields()` inline (was `verticalFields()` = external+dropdown
combined, since removed); dropdown fields moved behind the SAME `<p-popover>` mechanism, now a
single top-level element per component instance (was duplicated inside the horizontal-only `@if`
branch) triggered by a `layout`-appropriate button (`entity-search-filters-button` horizontal /
`entity-search-filters-button-vertical` vertical) via a new `toggleFilters(event)` method backed by
`viewChild(Popover)` — replaced the old `#filterPanel` template-reference-variable approach, since
a template ref declared inside one `@if`/`@else` branch can't be referenced from a sibling branch
cleanly once the popover itself needed to be shared across both. Verified live: dropdown field
count inside the popover matched exactly (15 for Convention), each layout's own popover instance
opens independently (two separate `sc-entity-search` instances on the compare page = two separate
popovers, confirmed no cross-instance leakage). Cypress test rewritten to match (still 13/13, one
test's assertions inverted to check the field IS behind Filters rather than inline). Pushed
`1e82356`.

**Date field border, two attempts**: user reported the boxed date fields (`.filter-field`, used by
both the vertical layout and the Filters popover) looked wrong — first fix (`bc3d278`) just
stripped the calendar-icon button's own border/background, mirroring the earlier horizontal-bar
fix, but that broke it differently: PrimeNG's default puts the border ONLY on the input (rounded
left corners, no right border) and relies on the icon button's own border to close the box's right
edge — stripping it left the box open, icon floating past where the border should end (confirmed
via `getBoundingClientRect`/computed-style inspection: input and button were flush with zero gap,
but the button carried zero border after the first fix, so nothing visually closed the shape).
Real fix (`471dc38`): move the border to the shared `.p-datepicker` wrapper instead (`border: 1px
solid var(--p-inputtext-border-color)` — confirmed via inspection this is the exact variable/value
a plain boxed `input` already resolves to) and strip borders from BOTH inner elements (input +
button), `overflow: hidden` on the wrapper to clip corners cleanly. Verified via zoomed
element-screenshot comparison against a plain text field (Name) — now pixel-consistent. Applies
to all three boxed-date contexts (vertical layout, horizontal's own popover) since all share
`.filter-field`; horizontal's own flat/borderless segment date field (a different, unboxed design)
confirmed untouched/unaffected by either fix.

## @Searchable.theme() — backend-only, no frontend consumption yet (2026-07-26, later same day, pushed context-gen `3d9cf18`)

User asked to let a search form's initial accent color be configured via the `@Searchable`
annotation itself, mirroring `@CrudstoneEntity.theme()` (same fixed preset name list — "primary"/
"emerald"/"blue"/"purple"/etc.). Mid-implementation, before any frontend work started, user
clarified: **backend only** — no UI/frontend changes at all, not even updating `search-crudstone`'s
`SearchContext`/`SearchField` TS models to include the new field. Scope stayed strictly:

- `Searchable.theme()` (context-gen): `default "primary"`, independent of the entity's own CRUD
  table theme (can match or differ freely) — same semantics as `CrudstoneEntity#theme()`'s own doc.
- `SearchContext.theme` (context-gen model) + `AnnotationSearchContextLoader` reads
  `searchable.theme()` into it.
- `Convention` gets `@Searchable(theme = "blue")` (matches its own `@CrudstoneEntity(theme="blue")`
  for consistency); `Studio` gets `@Searchable(theme = "purple")` (deliberately DIFFERENT from its
  own CRUD table's default "primary" theme, to prove the two are independent) — quar-crud-host,
  disk-only, no git repo. Verified live via curl post-restart:
  `GET /context/conventions/search` → `"theme":"blue"`, `GET /context/studios/search` →
  `"theme":"purple"`.
- context-gen tests 14/14 (was 13, +1: default-is-primary + override-works), quar-crud-host tests
  30/30 (was 29, +1: per-entity theme independence). `DOCUMENTATION.md`'s Search contexts section
  got a `@Searchable` attribute table (previously undocumented as a bare marker) plus an inline
  `context.getTheme()` example line.
- **search-crudstone (frontend) deliberately untouched this round** — `SearchField.ts`/
  `SearchContext.ts` do NOT yet have a `theme` property, `EntitySearchComponent` does not read or
  apply it. If a future request asks to actually wire this into the UI, that's a distinct,
  not-yet-started task — see [[project_gen_crudstone]]'s and dynamic-crud's own `theme-palettes.ts`
  (`THEME_PALETTES`/`themeVars()`) for the established pattern this would likely reuse/mirror,
  including the "overlay panels use `appendTo=\"body\"`, so host-scoped CSS vars won't reach them"
  caveat that would need solving (dynamic-crud's own table handles this via a document-root mirror
  + per-row baseline-reset mechanism — search-crudstone would need its own version, complicated by
  the fact that TWO differently-themed `sc-entity-search` instances can coexist on one page, as the
  `/compare/:entity` demo page already does).

## Frontend theme consumption, same day, pushed `a06ca9d`

Immediately followed by: "do the frontend changes" — wired `SearchContext.theme` into the actual
UI, resolving the "not yet done" note above.

- **`theme-palettes.ts`** (new, `projects/searchcrudstone/src/lib/theme/`): a leaned-down port of
  dynamic-crud's own module — same `THEME_PALETTES` Tailwind color values (for ecosystem
  consistency), same `themeVars(theme)` shape, but WITHOUT the row-baseline-capture/reset
  machinery: that exists in dynamic-crud purely to stop a CRUD table's own row content (p-tag
  badges with independent enum-driven colors) from being repainted by the table's theme override,
  and search-crudstone's result cards never render severity-colored tags at all (`displayValue()`
  always returns plain humanized text) — nothing to protect, so nothing to build.
- **Host-scoped, not global**: `themeStyle = computed(() => themeVars(searchContext()?.theme))`,
  bound via the `@Component`'s `host: {'[style]': 'themeStyle()'}` metadata. Deliberately NOT a
  `document.documentElement` override (dynamic-crud's own table can get away with that since
  there's always exactly one CRUD table visible at a time) — this library can have MULTIPLE
  differently-themed `sc-entity-search` instances on one page simultaneously (literally
  demonstrated by the `/compare/:entity` page built earlier this session), so a global override
  would make the second instance's theme bleed into the first the moment both are mounted.
- **The `appendTo="body"` problem, solved via panel-level inputs, not a document-root mirror**:
  PrimeNG's datepicker/multiselect/popover overlay panels render into `document.body`, physically
  outside this component's own DOM subtree — a host-scoped CSS var override can't reach them via
  normal cascade. Solved by binding the SAME `themeStyle()` object directly onto each control's own
  `panelStyle` input (`p-datepicker`, `p-multiselect`) or `style` input (`p-popover` — its whole
  rendered content IS the panel). Confirmed all three PrimeNG APIs expose exactly this (checked
  `node_modules/primeng/*/*.d.ts` first rather than guessing) before implementing.
- Verified live via Playwright: `getComputedStyle` on `sc-entity-search` shows
  `--p-primary-color: #3b82f6` for Convention (`theme="blue"`) and `#a855f7` for Studio
  (`theme="purple"`) — both the host AND the (separately opened) Filters-popover panel and
  datepicker-calendar panel. Confirmed the `/compare/:entity` page keeps both instances
  independently blue (both point at `conventions` in that demo, so same theme — but architecturally
  proven independent by construction, not just coincidence).
- **Known, accepted gap** (matches dynamic-crud's own precedent exactly, not a regression):
  PrimeNG's Aura theme defines some tokens (e.g. a datepicker's *selected*-day background) as
  build-time-compiled derived values that don't necessarily chain through `--p-primary-color` as a
  live `var()` reference at the point of use — confirmed via inspecting Aura's own compiled
  `datepicker/index.mjs` (`selectedBackground: "{primary.color}"` is a BUILD-TIME token reference,
  not a guarantee the compiled CSS reads `var(--p-primary-color)` live). A selected calendar date
  and the calendar's own prev/next nav arrows still render in the app's default emerald, not the
  entity's own theme color. Not chased further since dynamic-crud's own `themeVars()` has this
  exact same gap (verified by reading its full source) — matching an established, accepted
  precedent rather than a new shortfall unique to this component.
- Cypress 16/16 (was 13 — 3 new tests: per-entity host color, per-entity independence, popover
  panel reachability). `theme-palettes.ts` exported from `public-api.ts` for API completeness
  (advanced hosts can inspect `THEME_PALETTES` directly), though nothing in this library exposes a
  picker UI that would need it. README got a new "Theming" section.

## Angular/PrimeNG 19→22 upgrade (2026-07-27, by Claude — 16/16 Cypress green, both builds clean)

Third of a 3-project sweep this session ([[project_sidebar_crudstone]] first, [[project_gen_crudstone]]/dynamic-crud second) — same sequential 19→20→21→22 Angular bump with PrimeNG in lockstep at each step, `@primeng/themes`→`@primeuix/themes` at the end, Node v24.18.0 via nvm. Unlike dynamic-crud, **this project hit zero breaking changes beyond the theme import path and library peerDependencies bump** — no `p-table`, no camelCase PrimeNG selectors, no `pTemplate="x"` usage anywhere in this codebase, so none of the Table-specific breaking changes documented in [[project_gen_crudstone]]'s own upgrade entry applied here. Both `ng build searchcrudstone` and `ng build` succeeded on the very first try after the version bumps + one import fix (`@primeng/themes/aura` → `@primeuix/themes/aura` in `app.config.ts`).

Also produces PrimeNG 22's cosmetic "Invalid PrimeUI License" nag banner (see [[project_gen_crudstone]]'s upgrade entry for the full investigation — confirmed harmless, doesn't gate any functionality) unless a license key is configured; not addressed, same as the other two sibling projects.

`.npmrc` (leaked-PAT file, see [[project_leaked_npm_token]]) deliberately excluded from the commit, untouched this session.

## Single-pick + recursive drill-down relation filters (2026-07-27/28, context-gen `a476e22`, search-crudstone `02759f8`) — SUPERSEDED, see "Live-faceted checkbox tree" section below

**The UI design in this section and the next one (`NestedFilterFieldsComponent`, a flat "type into
the related entity's own scalar fields" popover) was ENTIRELY REPLACED on 2026-07-28** by a
live-faceted checkbox tree (`FacetFieldTreeComponent`) once the user showed a screenshot of what
they actually wanted (a car marketplace's own "Marque et modèle" filter). `NestedFilterFieldsComponent`
no longer exists in the codebase. Kept below for historical trail only — do not treat file paths or
component names in these two sections as current; jump to that later section for the real state.

Backend annotation contract (`SearchableField#single`/`#maxDepth`, `context-gen a476e22`) is
UNCHANGED and still accurate — only the FRONTEND rendering of a multi-pick relation field changed.

User asked for two additions to `@SearchableField`: (1) a flag marking a relation/enum filter as
single-pick rather than multiselect, and (2) an option to drill into a relation's own sub-entity
fields (and THEIR relations recursively), with max depth declared on the annotation. Clarified via
AskUserQuestion: `single` governs the FILTER WIDGET only (p-select vs p-multiselect, independent of
whether the underlying relation is to-one/to-many); `maxDepth` means "recursive filter drill-down" —
picking into a related row's own fields, filterable themselves, up to N levels.

- **context-gen**: `SearchableField.single()`/`maxDepth()` (both default false/0), propagated by
  `AnnotationSearchContextLoader` into `SearchField.single`/`maxDepth` — `maxDepth` is force-zeroed
  for any non-relation field (`fieldMeta.listType().isBlank()`) regardless of what's declared, per
  its own javadoc contract. 17/17 tests (was 14).
- **quar-crud-host** (disk-only): `Convention.hostStudio` is the live demo —
  `@SearchableField(..., single = true, maxDepth = 1)`. Studio has no relations of its own, so this
  demoes exactly one level of drill-down (Studio's own `name`/`website`).
- **Frontend key design decision — depth budget, not per-field maxDepth, governs how far a chain
  goes**: only the ORIGINAL relation field (hostStudio) has a real declared `maxDepth`; every field
  reached BY drilling into it (Studio.name, or a further relation if Studio had one) does NOT need
  its own `maxDepth` set — the recursion simply decrements the ORIGINAL budget by 1 per hop
  (`NestedFilterFieldsComponent`'s own `depth` input, `canDrillInto(field) = !!field.listType &&
  depth() > 1`). This was a deliberate reading of the annotation javadoc's "and so on" phrasing —
  re-check this interpretation if a future request implies otherwise.
- **New component**: `sc-nested-filter-fields` (`entity-search/nested-filter-fields/`) — a
  standalone Angular component that **imports itself** in its own `imports: []` array for the
  recursive template (`<sc-nested-filter-fields>` inside its own template). This works in Angular
  despite looking like a TDZ problem, because decorator metadata evaluation is emitted to run AFTER
  the class binding exists — confirmed via a real `ng build` (Ivy partial compile), not just
  reasoned about. Renders one flat list of the target entity's own SearchContext fields
  (text/date/enum/select/multiselect, same single-vs-multi branching as the top level), with an
  inline (not popover-in-popover) expand toggle for any of ITS OWN relation fields when depth
  budget remains — deliberately inline rather than nested overlays, to dodge PrimeNG
  popover-in-popover z-index/positioning issues.
- **Dotted-path filter keys**: a drilled-into field's filter is stored under
  `"<rootField>.<subField>"` (e.g. `"hostStudio.name"`) in the SAME flat `filterValues` map
  top-level fields use — no separate nested-state structure. `setFilterByPath`/`setFilter` (now a
  thin wrapper) and a new recursive `matchesPath`/`matchesField` pair (replacing the old flat
  `matchesFilters` body) walk the dotted segments, resolving each hop's own `SearchField` metadata
  from that hop's own `SearchContext` (`ContextService.getSearchContextSnapshot`, a new synchronous
  cache-read counterpart to the existing async `getSearchContext`) so a nested date/enum/relation
  field matches with its own real semantics, not a blind text comparison. `clearFilters`/
  `activeFilterCount` (both dropdown-scoped) now key off a filter's ROOT segment
  (`rootFieldName(path) = path.split('.')[0]`) so a drilled-into sub-filter is correctly grouped
  under its owning top-level field.
- **UI**: relation filter control now wrapped in `.relation-filter-control` (flex row: the
  select/multiselect + an optional `.drill-down-toggle` chevron button, `pi-sitemap` icon) — the
  toggle opens a `p-popover` containing the first `sc-nested-filter-fields` level. `.p-select` added
  alongside the pre-existing `.p-multiselect` in every `::ng-deep .entity-search-segment` stripped-
  chrome rule (border/background/width/label-padding) so the new single-pick control gets the same
  flat, borderless athome-style treatment the multiselect already had.
- Verified live against the real quar-crud-host backend (`localhost:9100`, dev server already
  running) via a throwaway debug Cypress spec that dumped `document.body.innerHTML` — caught one
  real test-authoring bug this way (the nested field's `data-cy` sits directly on the `<input>`
  itself, no separate wrapper div like the top-level fields have — `[data-cy="hostStudio.name-
  filter"] input` matched nothing, needed to drop the trailing ` input`). 5 new Cypress tests (2
  single-pick, 3 drill-down) — 21/21 in `entity-search.cy.ts` (was 16), 22/22 full suite including
  `dark-mode.cy.ts`. Both `ng build searchcrudstone` and the full library build clean.

## Multi-hop relation chain added to test drill-down past one level (2026-07-28, search-crudstone `beeb1cf`, quar-crud-host disk-only) — PARTLY SUPERSEDED

The `Country`/`City`/`Studio.headquarters` ENTITIES and the id-stringification fixes described
here are all still real and current. What's superseded: `hostStudio`'s own `single = true` got
flipped to `single = false` (the default) in the very next round below, since the WHOLE UI this
section describes testing (flat drill-down popover) no longer exists — see the "Live-faceted
checkbox tree" section for the current state of both the UI and the `hostStudio` field's own flag.

The maxDepth=1 demo above only ever showed ONE level of drill-down (Studio had no relations of
its own) — never actually exercised the recursion RECURSING. User asked to add a sub-entity that
itself has subs. Added a real 3-hop chain, all new Panache entities in quar-crud-host (disk-only,
no git repo — this section is the only record of them):

- **`Country`** (leaf: `name`, `code`, no relations) and **`City`** (`name`, `country` ->
  `Country`, `single=true`, no `maxDepth` of its own) — new `@CrudstoneEntity`+`@Searchable`
  classes in `entities/`. Neither needs a custom `CrudHandler` beyond list/get/create/update/
  delete — `query()` is left at `CrudHandler`'s own in-memory default (`CityHandler`/
  `CountryHandler`, new), since these are small reference tables, not worth a real HQL query for.
- **`Studio`** gained `headquarters -> City` (`single=true`, no `maxDepth` — per this project's own
  established design, only the field that STARTS a drill-down chain needs a real declared
  `maxDepth`; everything reached by drilling into it is governed by that field's own remaining
  depth budget, not by anything declared on the reached field itself. See this file's own
  "Single-pick + recursive drill-down" section above for why).
- **`Convention.hostStudio`**'s own `maxDepth` bumped `1` -> `3` — the whole chain (Studio's own
  fields, then City's own fields via `headquarters`, then Country's own fields via `country`) is
  exactly 3 hops.
- **New shared helper, `RelationIds.stringifyNestedIds(Map)`** (`org.devary.crudhost`): Jackson
  serializes a relation's nested `id` as a raw `Long`, but every id in this API needs to be a
  String (PrimeNG's dataKey comparison uses `===`). The PRE-EXISTING fix for this
  (`ConventionHandler.toMap`'s manual `hostStudio.put("id", ...)`) only patched ONE level and
  would NOT have reached `hostStudio.headquarters.id`/`hostStudio.headquarters.country.id` several
  relations deep. Replaced with a generic recursive walker (any nested Map's `id` key, at any
  depth, any list-of-maps too) — now used by `ConventionHandler`, `StudioHandler`, and
  `CityHandler`'s own `toMap()`. Verified live via curl: `GET /conventions` row ->
  `hostStudio.headquarters.country.id` is a real JSON string, three levels deep.
- **`DataSeeder`**: seeds `Country`/`City` rows FROM the same `CITY_COUNTRY` name pairs already
  used for Convention's own plain city/country text fields (coherent demo data, not an unrelated
  second place-name set), persisted in FK order (Country -> City -> Studio -> Convention, all one
  `Panache.withTransaction`), each Studio's `headquarters` cycling through the seeded cities.
- Backend: 50/50 tests green (unchanged count — no new Java tests added for this round, it's demo
  fixture data, not library behavior). Frontend: extended the existing drill-down Cypress specs to
  open and filter through all 3 levels (`hostStudio.headquarters.country.name` narrows results,
  verified against real seeded data e.g. "Japan") — 23/23 in `entity-search.cy.ts` (was 21), 24/24
  full suite. Confirmed live: `GET /context/studios/search`, `/cities/search`, `/countries/search`
  each return the expected field metadata; `.nested-drill-down-toggle` count is exactly 2 once all
  3 levels are expanded (Studio's own `headquarters` toggle + City's own `country` toggle; Country
  itself has none, both because it has no relations AND because depth budget is spent by then).

## Live-faceted checkbox tree replaces the flat drill-down panel (2026-07-28, search-crudstone `1e21df5`, quar-crud-host disk-only) — CURRENT DESIGN

User said the earlier flat drill-down popover (`NestedFilterFieldsComponent`, listed further above)
was the wrong shape entirely, then showed a screenshot of a real car marketplace's own "Marque et
modèle" (Brand/Model) filter drawer: a checkbox list of brands, each with a live result count:
checking one reveals, indented right below it, that brand's own MODELS as a further checkable
group with their OWN counts (e.g. checking Chevrolet reveals both Corvette AND Tahoe). That's a
fundamentally different UI (checkable ROWS with live counts, nested inline) from the old design
(a popover of flat scalar-field text inputs for the related entity). Full rewrite:

- **`FacetFieldTreeComponent`** (`entity-search/facet-field-tree/`, replaces the deleted
  `NestedFilterFieldsComponent`) — self-recursive standalone component, one instance per relation
  field's own facet group. Root instance gets `catalogOptions` (the full fetched row list, e.g.
  every Studio — so every option is always browsable, 0-count ones included, like a real facet).
  Every RECURSED instance gets `catalogOptions=null` and instead computes `occurringValues()` —
  only values that actually occur among rows on that specific branch, since showing an unrelated
  City with a 0 count under a checked Studio would be noise a real facet list doesn't have.
  A `single`-flagged field (top-level or reached mid-tree) renders radios instead of checkboxes,
  same live counts either way.
- **Live per-option counts**: computed client-side, `optionCount(id)` = rows matching every OTHER
  active filter with THIS field's own value temporarily overridden to `[id]` (or `id` for single)
  — so checking more options in the same field doesn't shrink its own siblings' shown counts. This
  needed `EntitySearchComponent` to eagerly prefetch `allRows` right after the search context
  loads (`allRows` converted from a plain field to a `signal`) instead of waiting for the user to
  press Search, since counts must be visible in the popover before that.
- **Filter keys are now id-scoped per branch**, not just field-name-based:
  `"hostStudio[3].headquarters"`, not `"hostStudio.headquarters"` — the `[3]` is the SPECIFIC
  checked studio's own id, embedded by `FacetFieldTreeComponent.childPath()`. This was a real bug
  fix mid-implementation, not just a design choice up front — see the two-part bug below.
- **Two real bugs found via user-reported repro, both in the matching logic, not the UI**:
  1. *Cross-branch constraint leak*: checking a child under Studio A was, before the id-scoping
     fix, incorrectly zeroing out every OTHER studio's own top-level count too — because the old
     flat `"hostStudio.headquarters"` key applied as a blanket AND across the WHOLE row set,
     regardless of which studio it was checked under.
  2. *Cross-branch leak into option lists* (**user's own exact repro**: "check mad house studio
     then check mappa and see sub trees, there's a redundance in countries"): fixing bug 1 by
     making a scope mismatch "trivially pass" (so Studio B's rows aren't blocked by Studio A's own
     drill-down) had the mirror-image effect of also making Studio B's rows trivially pass Studio
     A's own ANCESTOR checks when computing Studio A's own option list — so Studio B's country
     leaked INTO Studio A's own "Country" facet options. Fixed by `ownScopedRows()`: before
     computing a node's own options/counts, first strictly filter to rows that ACTUALLY belong to
     that exact branch (every ancestor segment's scope id must genuinely match, not just "not
     conflict") — verified against a hand-built two-studio mock scenario matching the user's exact
     repro (`Mad House Studio` / `MAPPA`, distinct countries) before even touching the live app,
     then confirmed live via `ng.getComponent()` state inspection.
- **`search-matching.ts`** (new shared module, used by both `EntitySearchComponent` and
  `FacetFieldTreeComponent`) — `matchesFilters`/`matchesPath`/`matchesField`/`resolveValue`/
  `rootFieldName`/`parseSegment`/`matchByScope`, extracted so both components use identical
  semantics instead of duplicating or one reaching into the other's internals. `parseSegment`
  splits a path segment into `{name, scopeId}` (e.g. `"headquarters[10]"` → `{name:"headquarters",
  scopeId:"10"}`). Made **array-aware** throughout for the to-many demo chain below: a to-many hop
  (e.g. `Studio.titles`) resolves to an ARRAY at that point in the path, not a single embedded
  object the way a to-one hop (`Studio.headquarters`) does — `matchByScope` treats a single object
  as a 1-element array so both shapes share one lookup path.
- **Second demo chain added, genuinely to-many at every hop** (user: "add another example of
  multiselection now of studio + it's sub entities"), since the existing `hostStudio -> headquarters
  -> country` chain is all to-ONE (every checked parent only ever reveals exactly one child,
  never the "Corvette AND Tahoe" multi-child shape the reference actually shows):
  `Convention.hostStudio` (now `single=false`, was `true` — see the superseded section above) →
  `Studio.titles` → `StudioTitle.genres` → `Genre` (leaf). New entities `Genre`/`StudioTitle` in
  quar-crud-host (disk-only). **Deliberately NOT a real `@ManyToMany`**: Hibernate Reactive
  requires an explicit `Mutiny.fetch()` per collection level to avoid
  `LazyInitializationException`, and this chain is 2 collection-levels deep serialized across 3
  DIFFERENT handlers (Convention embeds Studio embeds StudioTitle embeds Genre) — same complexity
  `Convention.featuredAnimes` already opted out of. Used that exact same precedent instead: raw
  JSON-text columns (`Studio.titles`, `StudioTitle.genres`), hand (de)serialized by each field's
  own handler (`StudioHandler`/`StudioTitleHandler`), embedding the FULLY resolved snapshot
  (including nested genres) so a reader 3 hops up never needs a further lookup.
- **Real gap found+fixed while wiring this**: the pre-existing `RelationIds.stringifyNestedIds`
  (id-string-fixing walker) only fixed a JSON-text field when the OWNING handler's own `toMap()`
  ran it directly — but `ConventionHandler.toMap()` never calls `StudioHandler`'s own logic when
  embedding `hostStudio`; Jackson just serializes `Studio.titles` as whatever raw Java type it
  is (`String`, i.e. the UNPARSED JSON text) when reached via a DIFFERENT handler's own generic
  `mapper.convertValue()`. Fixed by extending `RelationIds` to ALSO parse known JSON-text-list
  field names (`"titles"`, `"genres"`) by NAME, recursively, at ANY depth, regardless of which
  handler triggered the walk — not just fix ids. Confirmed via `ng.getComponent()` inspection that
  `hostStudio.titles` was arriving as a raw string (Python's bare `print()` masked this in an
  earlier curl check — looked identical to a real parsed array in the terminal output, a real
  "verify the actual type, not just how it prints" lesson).
- Seed data: `Genre`×10, `StudioTitle`×20 (each 2–3 genres), each of the 10 studios gets 3
  overlapping titles (`titles.get(i%20)`, `(i+5)%20`, `(i+10)%20)` — co-productions, not a clean
  partition, so the shape matches a real catalog.
- Cypress rewritten: the old single-pick/flat-drill-down describe blocks replaced with
  `multi-pick relation facet tree`, `recursive facet tree, to-one chain`, and
  `recursive facet tree, to-many chain` (includes a direct regression test for bug 2 above: two
  studios checked, asserts their own title-name lists have zero overlap). 26/26 in
  `entity-search.cy.ts` (was 23), 27/27 full suite. Both builds clean, backend 50/50.
- **Known limitation, not chased**: `childRelationFields` (which relation fields become nested
  facet groups under a checked option) has no cap on how MANY relation fields render as siblings —
  fine for Studio's 2 (headquarters, titles), would need actual UI treatment (tabs? accordion?) if
  an entity had many more.

## Not yet done / natural next steps (not requested yet)
- No "Full view" detail dialog equivalent (crudstone has one) — could reuse the same paginated
  7-fields-per-tab pattern to show a highlighted suggestion's full record before picking.
- No theming/toolbar-visibility config at all (deliberately dropped, this is a compact single
  control, not a full-page screen) — would need to be added if a host wants to reskin it.
- Publishing to npm not set up/discussed (Jenkinsfile mirrors dynamic-crud's version-gated publish
  stage, mechanically renamed, but never actually run).

## Cross-field dependencies: SearchableField#dependsOn/#dependency (2026-08-16, by Claude)

User's two examples ("end date must be >= start date", "car name only fillable once car model is
set") mapped onto a new `dependsOn`(field name)/`dependency`(enum) pair on `@SearchableField`.
- **context-gen**: new `FieldDependency` enum — `NONE`, `REQUIRES_VALUE` (this field's control
  disabled until `dependsOn` has a value, any field type), `GREATER_THAN_OR_EQUAL`/
  `LESS_THAN_OR_EQUAL` (this field's value must be >=/<= dependsOn's — only valid between two
  date/dateTime fields). `AnnotationSearchContextLoader.validateDependencies` fails fast at
  startup on: unknown `dependsOn` name, self-reference, a comparison declared on non-date fields,
  or a dependency CYCLE (walks each chain with a visited-set). 24/24 in
  `AnnotationSearchContextLoaderTest` (was 17, +7 covering all of the above incl. both valid
  cases). `SearchField` gained `dependsOn`/`dependency` (String, null when unset).
- **search-crudstone frontend**: `SearchField.dependsOn?`/`dependency?` (raw JSON passthrough, no
  normalization needed). `EntitySearchComponent`: `isFieldBlocked(field)` (REQUIRES_VALUE unmet),
  `minDateFor`/`maxDateFor` (bind straight to `p-datepicker`'s own `[minDate]`/`[maxDate]`),
  `dependencyFieldLabel` (for the "Select X first" placeholder swap, translation key
  `fieldDependsOnPlaceholder`). **Cascade-clear, the real behavioral piece**:
  `clearDependentsViolatedBy(changedPath, filters)` runs inside `setFilterByPath`'s own update —
  when a field changes, any sibling field that `dependsOn` it gets its own now-invalid value
  wiped (REQUIRES_VALUE target went empty, or a GTE/LTE bound the current value no longer
  satisfies). Only applies to a top-level field changing (skipped if `changedPath` has a `.`/`[` —
  dependsOn is a same-entity sibling concept, not reachable through a facet tree's own drilled-into
  sub-fields). Template: every filter control branch (text/date/enum-select/enum-multiselect/
  relation-select) gets `[disabled]="isFieldBlocked(field)"` + placeholder swap; date fields
  additionally get `[minDate]`/`[maxDate]`; the facet-tree trigger button (multi-pick relation)
  gets the same disabled+label treatment by hand since it's a plain `<button>`, not a PrimeNG
  component (`.facet-trigger:disabled` added to match PrimeNG's own `.p-disabled` look elsewhere).
- **Demo data, quar-crud-host Convention.java** (disk-only): `city` (dropdown) now
  `dependsOn="country", REQUIRES_VALUE` — country is external/always visible, city sits in the
  Filters popover disabled until country's filled. `endDate` now `dependsOn="startDate",
  GREATER_THAN_OR_EQUAL`. **Real bug fixed in passing**: `endDate` was `String`/`inputText` despite
  `DataSeeder` always giving it a real ISO date (startDate + 2 days) — same class of mistyped-date
  bug already fixed for `startDate` itself in this project's own earlier history, just never caught
  for `endDate` until this session. Changed to `LocalDate`/`type="date"`.
  `ConventionHandler.apply()`'s existing ISO-datetime-to-plain-date truncation (previously only
  applied to `eventDate`, a known Angular-`Date.toJSON()`-vs-`LocalDate.parse` mismatch) generalized
  to also cover `startDate`/`endDate` — needed because both are now real editable date fields
  reachable through the CRUD form (`Convention.dedicatedCreateFormView`, see
  [[project_gen_crudstone]]'s own dependsOn-split entry), not just seed-time construction.
- **Cypress** (`entity-search.cy.ts`, new `field dependencies` describe block, +6 tests, 30/30 in
  the spec / 31/31 full suite): REQUIRES_VALUE (disabled→enabled→re-disabled+cleared as country is
  set/cleared) and GTE (clears an already-picked endDate once startDate moves past it; keeps a
  still-valid one when startDate stays before it). **Two real Cypress gotchas hit while writing
  these, both fixed, worth remembering for any future date-field test in this suite**:
  1. A plain `data-cy` on an `isInputText` field sits directly on the `<input>` itself, but on a
     date field it sits on the OUTER `<p-datepicker>` (PrimeNG wraps its own inner `<input>`) — so
     `[data-cy=x-filter] input` is correct for a date field but WRONG for a plain text field
     (needs `[data-cy=x-filter]` alone, no nested `input`).
  2. **Manual keyboard date entry into a `p-datepicker` (e.g. `.type('06/15/24{enter}')`) proved
     unreliable to assert against in this environment** — typed+Enter did not reliably commit/
     parse, confirmed via screenshot showing the calendar overlay still open on today's month
     rather than the typed one. Switched to clicking an actual day cell in the currently-open
     panel instead (`.p-datepicker-panel:visible .p-datepicker-calendar
     td:not(.p-datepicker-other-month) span.p-datepicker-day`, matched by exact day-of-month text)
     — reliable, and since both tests only care about relative day order (5 vs 20), no need to
     navigate to a specific month/year. **Also needed `:visible` scoping specifically**: an
     already-closed datepicker's own panel can linger in the DOM (just hidden), and an unscoped
     query silently matched the stale one instead of the newly-opened field's panel — false
     failures until scoped.
  3. Also reconfirmed the established "typing into a field OUTSIDE the Filters popover auto-closes
     it" behavior (already documented elsewhere in this file) applies here too — every dependency
     test that types into an external field (country, startDate) needs to re-click
     `entity-search-filters-button` before asserting anything inside the popover again.
- Not extended to the facet tree's own nested/drilled-into fields (out of scope — the two examples
  given were both same-entity sibling fields on the flat search bar, not facet-tree relations).

## Searchable#externalResult + dedicated results page + shared pagination (2026-08-16, by Claude)

Two related features in one session: (1) `@Searchable(externalResult = true)` routes Search to a
dedicated results page (sidebar filters + main-area grid) instead of showing results inline; (2)
centered, search-bar-styled pagination on EVERY results grid (inline panel AND the new page).

- **context-gen**: `Searchable.externalResult()` (default false) → `SearchContext.externalResult`
  → `AnnotationSearchContextLoader`. 2 new loader tests (default-false + override-true).
- **New `EntitySearchResultsComponent`** (`sc-entity-search-results`,
  `entity-search/entity-search-results/`): the results grid + empty-state markup EXTRACTED out of
  `EntitySearchComponent`'s own template (was inline), now takes `results`/`resultFields`/
  `resultImageField`/`entityLabel`/`themeStyle`/`pageSize` (default 12) as inputs. Adds pagination:
  `pageMarkers` computed does first/last/current±1 windowing with `'ellipsis'` gaps (real PrimeNG-
  paginator-style truncation, not a wall of buttons for large sets); `page` resets to 0 via an
  `effect()` tracking `results()` whenever a fresh array arrives. Styled to match the search bar's
  own tokens exactly — pill buttons (`border-radius:999px`, same as the filters-count badge),
  solid `--p-primary-color` for the active page, no separate "pagination" look invented.
- **`EntitySearchComponent` gained**: `showResultsPanel` input (default true; false = manage
  state but render no panel of its own — how the results page embeds this as a pure sidebar),
  `initialFilters` input (seeds `filterValues` + auto-runs a search once the context loads — how
  the carried-over compact-bar filters reach the sidebar), `resultsChange` output (fires on every
  search regardless of `showResultsPanel`, what the results page listens to for its own grid).
  `runSearch()`: when `searchContext()?.externalResult && showResultsPanel()`, navigates
  (`router.navigate(['/', entityName, 'results'], {state: {filterValues}})`) instead of searching
  in place — router navigation `state`, not query params (per explicit user choice): Date objects/
  id arrays survive as-is (structured-cloned by the History API), no serialization, at the cost of
  the results page not being independently bookmarkable — an accepted trade-off matching this
  library's other ones (e.g. dedicated form view isn't route-based either).
- **New `EntitySearchResultsPageComponent`** (`sc-entity-search-results-page`,
  `entity-search-results-page/`): `:entity/results` route (new, added before `:entity` in the
  demo's `app.routes.ts` — order matters for route matching precedence, though Angular's own
  segment-count matching means it wouldn't actually collide either way). Reads
  `history.state?.filterValues` (not `Router.getCurrentNavigation()` — only reliable during the
  navigation itself, not by component-construction time) as the sidebar's `initialFilters`.
  Layout: `<sc-entity-search layout="vertical" [showResultsPanel]="false">` as a STICKY left
  sidebar (reuses the vertical layout's own self-contained card look, destyled from its normal
  centered/max-width-420px presentation via `::ng-deep` to stretch full-width in the sidebar slot)
  + `<sc-entity-search-results>` fed by the sidebar's `resultsChange` in the main area. Own "Back
  to search" link (RouterLink to `/{entity}`).
- **Demo data**: `City` (`quar-crud-host`, disk-only) is the `externalResult=true` entity —
  deliberately NOT conventions/studios, both already covered by ~30 inline-results Cypress tests
  that assume Search stays in place; flipping either would have broken all of them. `City.country`
  also got `@SearchResult(order=2)` (was search-filterable only) so results page cards show more
  than just the name.
- **A REAL, pre-existing architectural bug found and fixed, not caused by this feature but first
  EXPOSED by it**: `EntitySearchComponent`'s constructor `effect()` (resets state + calls
  `loadSearchContext` on `context()` change) wasn't wrapped in `untracked()`. `ContextService
  .getSearchContext()` returns a SYNCHRONOUS `of(cached)` once an entity's search context is
  already cached — true for `city` specifically because the compact bar page always loads it
  first, before the sidebar reuses the SAME component for the SAME entity on `/cities/results`.
  Because `of()` emits synchronously, `loadSearchContext`'s entire downstream chain — including
  `runSearch()`'s own signal reads — executed INSIDE the outer effect's own synchronous execution
  window, so Angular tracked them as the effect's dependencies too. Since the effect also WRITES
  those same signals, this created a self-retriggering loop: **confirmed via a debug counter mid-
  investigation — 20 firings off a single click, all against the IDENTICAL `context` reference**
  (proving `context` itself was never the actual trigger — proof technique: log every
  reference+timestamp to `window.__log`, inspect via `cy.window().then()` since `cy.log()` doesn't
  surface in `cypress run`'s terminal output, only `expect(...).to.eq('EXPECTED')`-style forced
  failures do). Symptom: pagination clicks appeared to do literally nothing (page always snapped
  back to 0) ONLY on the results page's sidebar, never on the compact-bar's own inline panel —
  because only the sidebar path ever hits the synchronous-cache case. Fixed with one `untracked()`
  wrapping the effect's whole body below the `context()` read. Cypress `{force: true}` clicks did
  NOT fix it (ruled out a click-targeting/layout theory before finding the real cause) —
  worth remembering: a click that lands correctly (confirmed via `elementFromPoint`) but has zero
  effect on state is a signal-loop smell, not a DOM-targeting one.
- Also fixed the OTHER pre-existing part of that same investigation: a full-page Cypress
  `cy.screenshot()` on a page taller than the viewport, containing a `position: sticky` element,
  visually shows that element MULTIPLE times (once per scroll-and-stitch segment) — a screenshot-
  capture artifact, not a real DOM duplication; don't mistake it for one (confirmed by directly
  counting `document.querySelectorAll(...)` — always 1 — while the screenshot showed 3).
- **4 pre-existing tests in `entity-search.cy.ts` updated** for the new pagination reality (were
  asserting the full unpaginated card count — 100 conventions, 20 filtered rows — now correctly
  capped at 12/page): `shows every row...`, both reset-button tests, and the facet-tree "checking
  several studios" test (now checks page 1 = 12 + page 2 = 8).
- 11 new tests in `external-results-page.cy.ts` (externalResult navigation/sidebar/pagination) +
  4 fixed + rest unchanged = 42/42 full suite green. context-gen: +2 tests (26/26 in
  AnnotationSearchContextLoaderTest).

## Server-side lazy pagination (3 load modes) + sidebar redesign (2026-08-16, later same day)

Two follow-up requests on the same day, same session: (1) the results-page sidebar had to drop
the dropdown/Filters-popover + Search button entirely (all fields flat, live filtering, Reset
pinned to the TOP), (2) results loading had to become genuinely server-side lazy — "same system
as the CRUD part" — with a new annotation flag choosing PAGINATED / LOAD_MORE / INFINITE_SCROLL.

- **Backend**: new `ResultsLoadMode` enum + `Searchable.resultsLoadMode()` (default `PAGINATED`)
  + `resultsPageSize()` (default -1 = "frontend's own 12"), both propagated to `SearchContext`.
  **No new endpoint** — reused the CRUD table's own existing `GET /{entity}?from=&to=
  &filter.<field>=v1,v2` (`EntityResource.all()`, `PanacheListQuerySupport`/`InMemoryQueryEngine`)
  verbatim; confirmed via research it's fully generic, not CRUD-specific. Backend `FieldKind` has
  NO date-comparison kind — date fields are silently ignored server-side if filtered (matches the
  CRUD table's own pre-existing gap, not a new regression); `search-crudstone`'s own wire-filter
  builder deliberately skips date fields for this reason (documented in `wire-filters.ts`'s own
  header comment). +2 loader tests (default + override for both new attributes).
- **Frontend — the fetch itself**: `SearchPage.ts` (new: `SearchPageParams`/`SearchResultsPage`,
  mirrors crudstone's own `EntityPage.ts`), `EntityService.getPage()` (mirrors crudstone's
  `getPage` almost line-for-line), `wire-filters.ts`'s `buildWireFilters()` (translates
  `filterValues` → the server's `filter.field=v1,v2` shape; select/multiSelect values are
  ALREADY plain id strings here unlike crudstone's own `{id,...}` option objects, so no
  `.map(o=>o.id)` step; skips dotted/scoped facet-tree drill-down paths too — only a facet
  field's own ROOT-level pick, e.g. `hostStudio=3,5`, reaches the server, never the nested
  refinement). `EntitySearchComponent.runSearch()`/new `fetchResultsPage`/`loadMoreResults`/
  `goToResultsPage` replace the old "fetch everything, filter client-side" path for the RESULTS
  GRID specifically — **the facet tree's own full-catalog fetch (`withRows`/`allRows`) is
  UNTOUCHED**, deliberately kept as its own separate full fetch, since live per-option counts
  genuinely need whole-table visibility and that's a distinct concern from paginating displayed
  results.
- **`EntitySearchResultsComponent` rewritten**: no more client-side slicing (`pagedResults`
  removed) — renders `results()` exactly as given by the parent. New inputs `total`/`mode`/
  `pageSize`/`currentPage`/`loading`, outputs `pageChange`/`loadMore`. Three render branches:
  PAGINATED (existing numbered nav, unchanged look), LOAD_MORE (a centered pill button matching
  the search bar's own primary-solid style), INFINITE_SCROLL (an invisible sentinel `<div>` +
  `IntersectionObserver`, `ngOnDestroy` disconnects it). **Same class of effect-retrigger bug
  almost reintroduced, caught and avoided this time**: the observer callback reads `loading()`/
  `hasMore()` INSIDE the callback, not in the `effect()`'s own tracked scope — reading them in
  the effect would reconnect the observer on every loading toggle (the exact bug fixed earlier
  the same day in `EntitySearchComponent`'s own constructor effect, see that entry above).
- **New layout `'sidebar'`** on `EntitySearchComponent` (was `'horizontal' | 'vertical'`, now a
  third option, `'vertical' `kept as-is for the demo app's own `/compare/:entity` page — NOT
  removed/repurposed): ALL searchable fields flat (`[...externalFields(), ...dropdownFields()]`,
  no popover at all), Reset button alone in its own header row ABOVE the fields, no Search button
  anywhere. **"Live" already came almost for free**: `setFilterByPath` already re-ran the search
  on every change whenever `showResults()` was true — the only real gap was a `showResultsPanel
  =false` instance with NO `initialFilters` ever getting `showResults` set true in the first
  place (e.g. a direct/refreshed navigation straight to `/cities/results`, no filter snapshot
  carried over) — since a sidebar has no Search button to manually trigger one. Fixed by widening
  `loadSearchContext`'s auto-run condition from `if (initial)` to `if (initial ||
  !showResultsPanel())` — a headless instance always gets an initial run regardless.
  `resultsTotal`/`resultsLoadMode`/`resultsPageSize`/`resultsPage`/`resultsLoading`/
  `loadMoreResults`/`goToResultsPage` all made PUBLIC (were protected) specifically so
  `EntitySearchResultsPageComponent`'s own template can reach them via a template reference
  variable (`#search` on `<sc-entity-search>`) — the standard Angular way to touch a child's
  public API from a parent template when the child's OWN output events aren't rich enough (needed
  5 pieces of state + 2 methods, not just one event).
- **Demo entities**: `conventions` stays `PAGINATED` (default, untouched — 30 existing inline-
  results tests already assume it) but its FETCH is now genuinely server-driven underneath, not
  just client-sliced. `cities` → `LOAD_MORE` (15 rows / 12 per page = one real "Load more" click).
  `studios` → `INFINITE_SCROLL` with `resultsPageSize=4` (10 rows, 3 real auto-loads: 4+4+2) — the
  small custom page size exists SPECIFICALLY to make infinite-scroll demoable without needing
  hundreds of seeded rows.
- **A real Cypress flake found+fixed while testing infinite-scroll**: calling
  `cy.scrollIntoView()` twice in a row on the SAME sentinel is NOT two separate scroll events if
  the element never actually left the viewport between calls — `IntersectionObserver` only fires
  on a threshold CROSSING, so the second call can be a same-state no-op that never retriggers
  `loadMore`. Fixed by `cy.scrollTo('top')` between each `scrollIntoView()` call, forcing a
  genuine out-of-view → in-view transition every time. Confirmed stable across 3 repeated runs
  before trusting it.
- Cypress: `external-results-page.cy.ts` grew to 16 (was 11: +4 sidebar-structure tests, +2
  LOAD_MORE, +1 INFINITE_SCROLL, 2 old vertical-layout selectors renamed to `-sidebar-filter`,
  1 stale PAGINATED-cities test replaced). `entity-search.cy.ts` untouched, still 30/30 (the
  `'vertical'` layout tests there — `/compare/:entity` — never touched `'sidebar'` at all).
  47/47 full suite. context-gen 26/26. Verified via curl that `GET /cities?filter.name=Tokyo` and
  `GET /studios?filter.name=Mad` both genuinely narrow server-side (not just believed to).

## sidebarPosition (LEFT/RIGHT) + overridable --sc-* CSS custom-property system (2026-08-17, code complete, NOT YET LIVE-VERIFIED)

**IMPORTANT: this entry's code changes are believed correct (compiles clean, context-gen tests
green, dev-server hot-rebuilds clean) but were NEVER exercised against a running app** — the
Postgres DB backing quar-crud-host (`192.168.178.41:31432`, a separate LAN machine) went
unreachable ("No route to host") mid-session and stayed down. User chose to go check the machine
rather than have the session continue without verification. If picking this up later: confirm the
DB is reachable (`timeout 3 bash -c "cat < /dev/tcp/192.168.178.41/31432"`), restart
quar-crud-host (context-gen jar changed, needs a full restart per established pattern), then run
`cypress run` in search-crudstone before trusting anything below as actually working.

- **Backend**: new `SidebarPosition` enum (LEFT default, RIGHT) + `Searchable.sidebarPosition()`
  → `SearchContext.sidebarPosition`, propagated through `AnnotationSearchContextLoader`. +2 loader
  tests (26/26 total, confirmed green — this part IS verified, no DB dependency).
- **Frontend**: `SearchContext.sidebarPosition: 'LEFT'|'RIGHT'`.
  `EntitySearchResultsPageComponent` gets a `sidebarPosition` computed off `searchContext()`,
  applied on the results-page root as `[class.results-page--sidebar-right]` +
  `[attr.data-cy]="'results-page-' + (RIGHT?'right':'left')"`. SCSS: `.results-page--sidebar-right`
  uses `flex-direction: row-reverse` (and `column-reverse` on the mobile single-column breakpoint,
  so the sidebar still ends up AFTER the results grid there too) — the sidebar `<aside>` stays
  FIRST in DOM/template order regardless (correct a11y/reading order), only the visual position
  flips.
- **New overridable `--sc-*` CSS custom-property system** (15 tokens): every "brand chrome" value
  (card radius/border/shadow, button/pill radius, field height, grid gap, sidebar width, page
  gap/max-width, a `--sc-danger-color` for the Reset button since `--p-red-500` has no per-entity
  override path) converted from a hardcoded SCSS value to `var(--sc-token, <default>)` across
  `entity-search.component.scss`, `entity-search-results.component.scss`,
  `entity-search-results-page.component.scss`. Deliberately separate from the EXISTING per-entity
  color theme (`Searchable#theme`/`themeVars()`, a host-scoped inline `[style]`) — this new system
  is plain CSS custom properties with no Angular plumbing at all: a host app sets
  `--sc-card-radius: 4px` (etc.) on any ancestor in its OWN global stylesheet and it just cascades
  in, since Angular's emulated view encapsulation does NOT scope custom-property inheritance (only
  scopes selector matching). Full token table + explanation added to
  `projects/searchcrudstone/README.md`'s new "Styling (CSS custom properties)" section (also fixed
  the stale `layout` prop-table row there, which hadn't been updated for `'sidebar'` in the earlier
  same-day session, and added a "Results pages" section explaining `externalResult`/
  `sidebarPosition`/`resultsLoadMode` together — the README had drifted since those features were
  first built).
- **Demo entity — `Country`** (quar-crud-host, disk-only, no git repo): now
  `@Searchable(theme="purple", externalResult=true, sidebarPosition=RIGHT)` (was: `@Searchable
  (theme="purple")` only, no search results page at all — Country was previously only reachable
  through the `hostStudio` facet-tree drill-down chain from Conventions, never visited directly as
  its own `/countries` page by any test). Also added `@SearchResult(order=2)` to `Country.code`
  (was search-filterable but not shown in a result card) so the demo card shows more than just the
  bare name. Chosen deliberately over reusing/flipping `studios` (already the INFINITE_SCROLL demo,
  heavily covered by existing stable tests that assume it stays INLINE, not a results page) —
  `Country` was the one entity in the whole demo app that had zero direct-route Cypress coverage,
  making it a zero-collateral-risk choice. `City` (existing `externalResult=true` entity, LOAD_MORE
  demo) left with `sidebarPosition` UNSET — its results page is the implicit "LEFT, the default"
  demo, `Country`'s is the explicit "RIGHT" one; `/cities/results` vs `/countries/results` are
  meant to be compared side by side.
- **CSS-override DEMO, in the demo app itself** (`search-crudstone/src/styles.scss`, appended after
  the existing global-style imports): overrides all 15 `--sc-*` tokens to a sharp/flat/dense
  alternate look (4px radii, no shadow, `#111827` instead of red for danger, tighter spacing),
  scoped via a plain attribute selector `[data-cy='results-page-right']` — since `Country` is the
  ONLY entity that ever renders that data-cy value, this cleanly isolates the override demo to
  `/countries/results` without touching the look of `/cities/results`, `/conventions`, `/studios`,
  or the compact search bar (confirmed by design, not yet by screenshot — see verification TODO
  above). This is presented as "exactly what a real parent/host project would do" — zero library
  source touched, zero Angular config, plain global CSS.
- **New Cypress coverage written** (`external-results-page.cy.ts`, NOT yet run — DB was down):
  a `sidebarPosition` describe block (cities sidebar left-of-main via bounding-rect comparison,
  countries sidebar right-of-main) and a `CSS custom-property override system` describe block
  (asserts `getComputedStyle(...).borderRadius` is `10px` on cities' own result cards vs `4px` on
  countries', PLUS a scoping check that countries' own compact-bar-before-Search card
  (`.entity-search-card`) still reads the library DEFAULT `16px`, proving the override is scoped to
  the results page only, not the whole entity).
- Not yet done: actually running any of this — see the warning at the top of this entry.

## Bug fix: "Back to search" lost the filters (2026-08-17, same session, still NOT live-verified)

User-reported bug, found and fixed while DB was still down (fix itself needed no DB — pure
Angular/routing logic, confirmed via clean dev-server rebuilds only).

- **Root cause**: `EntitySearchResultsPageComponent`'s "Back to search" link was a plain
  `[routerLink]="['/', ctx.name]"` — no navigation `state`, so `EntitySearchPageComponent` (the
  compact-bar route) always landed with a blank `filterValues`, regardless of what was set on the
  results-page sidebar. Nothing carried the filters BACKWARD — only the FORWARD hand-off (compact
  bar → results page, via `EntitySearchComponent.runSearch()`'s own `router.navigate(...,
  {state:{filterValues}})`) existed.
- **Fix, `EntitySearchComponent`**: added `autoSearchOnInit` input (default `true`). Auto-run
  condition changed from `if (initial || !showResultsPanel())` to `if ((initial &&
  autoSearchOnInit()) || !showResultsPanel())` — a seeded `initialFilters` no longer FORCES an
  immediate `runSearch()` when the caller opts out. This distinction matters because `runSearch()`
  on an `externalResult` entity NAVIGATES — if the compact-bar page auto-searched immediately
  after restoring filters, it would just bounce straight back to the results page, turning "Back"
  into a no-op. `filterValues` also made public (was `protected`) — same reasoning as the earlier
  `resultsTotal`/`resultsPage`/etc. exposure, reached via `#search` template-ref from the results
  page's own back-link handler.
- **Fix, `EntitySearchResultsPageComponent`**: `goBack(event, entityName, filterValues)` replaces
  the plain `routerLink` — `event.preventDefault()` + `router.navigate(['/', entityName],
  {state: {filterValues}})`, sending the SIDEBAR's CURRENT filter snapshot (read live off
  `search.filterValues()` via the template-ref, not whatever was originally carried forward — so
  if the user changes a filter ON the results page before clicking Back, that latest value is
  what returns, not the stale original). `[attr.href]` kept on the `<a>` alongside the click
  handler (NOT `routerLink` — the two would double-navigate/race if combined on one element) so
  ctrl/middle-click still opens a real (if state-less) URL in a new tab, same as a fresh direct
  visit already looks like.
- **Fix, `EntitySearchPageComponent`** (the compact-bar page, was previously the one page in this
  whole feature with ZERO `initialFilters` support): now reads `history.state.filterValues` and
  passes `[initialFilters]` + `[autoSearchOnInit]="false"` to its own `sc-entity-search`.
- **Secondary hardening applied to BOTH page components while in this code**: `initialFilters` was
  a ONE-SHOT constructor-time field read (`= (history.state?.filterValues ?? null) as ...`) in
  both `EntitySearchResultsPageComponent` (pre-existing, from earlier the same day) and the new
  `EntitySearchPageComponent` code. Since Angular's default route-reuse strategy REUSES a routed
  component instance across an in-app `:entity` PARAM change (same route config, different param
  — confirmed by the pre-existing `route.paramMap.subscribe(...)` pattern both components already
  needed for `context`/`searchContext` reloading, for the exact same reason), a one-shot field
  would leak a STALE filter snapshot from an earlier entity into a newly-switched-to one. Converted
  both to a `signal<Record<string,FilterValue>|null>(null)`, SET INSIDE the existing
  `paramMap.subscribe` callback (re-reading `history.state` fresh on every entity switch) instead
  of the constructor body. Not independently Cypress-tested (this demo app has no in-app link
  between two different entities' compact-bar pages to trigger the reuse path — `cy.visit()` is
  always a full fresh page load, doesn't exercise Angular route-instance reuse) — a correctness
  improvement made defensively while already touching this exact code, not something reported as
  observably broken.
- **4 new Cypress tests** added to the `external results page (Searchable#externalResult, cities)`
  describe block in `external-results-page.cy.ts`: filters restored (not blank) on Back, inline
  results panel stays hidden after landing back (no bounce), the SIDEBAR's latest value wins over
  the originally-carried-forward one, and a full round trip (Back → edit nothing → Search again →
  correct filtered results). NOT YET RUN — same DB-down blocker as the rest of this session; dev
  server hot-rebuild is clean (only compile-time confidence so far).
- Still blocked on `192.168.178.41:31432` being unreachable — nothing in THIS fix required DB
  access to write/reason about, but running the actual Cypress suite (including these new tests)
  still needs quar-crud-host serving real data. Check reachability before trusting anything in
  this session as verified.
