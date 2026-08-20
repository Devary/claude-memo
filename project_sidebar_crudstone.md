---
name: project-sidebar-crudstone
description: "gen/sidebar-crudstone: metadata-driven navigation sidebar, back on free MIT Angular/PrimeNG 19 after the PrimeUI license revert — sb-sidebar is now a HAND-WRITTEN compound sidebar (own markup/SCSS, same data-attribute contract and variants as the paid PrimeNG 20+ compound API), backend-only layout config"
metadata:
  node_type: memory
  type: project
  originSessionId: 234d18c6-f1ac-4b2e-bcc3-12e9a700a46e
---

# sidebar-crudstone — navigation sidebar library (created 2026-07-27, rebuilt on PrimeNG 22 same session)

Third sibling to [[project-gen-crudstone]] (`crudstone`, full CRUD table) and
[[project_search_crudstone]] (`searchcrudstone`, search-and-pick) — same context-gen backend,
same ecosystem, a third independent UI concern: a **navigation sidebar** that links *out* to other
entities' own CRUD pages. It never renders any entity's own fields/rows itself.

**This memory supersedes the original v19/`p-sidebar`-simple-overlay version of this project** —
mid-session the user required exact structural fidelity to PrimeNG's own "Variants" sidebar demo
(https://primeng.dev/sidebar), which only exists on PrimeNG 20+ (compound multi-directive Sidebar,
totally different API from v19's single-component drawer). Required upgrading Angular+PrimeNG
19→22 first (sequentially, one major at a time — `ng update` refuses multi-major jumps), then a
full rewrite of the frontend component on the real compound API.

## User's original design vs. what actually shipped

User's own pseudocode (`@Sidebar(theme=..., children=true)` with a `TreeMap`-shaped field) implied
expressing the whole nav tree as ANNOTATION ATTRIBUTES. Java forbids a self-referential annotation
type, so this became a **plain Java object graph** instead: `SidebarNode` (`org.devary.table.sidebar`)
is a `@Getter` class with a **private constructor** and four static factories —
`group(title, SidebarNode...)` / `group(icon, title, SidebarNode...)` / `link(title, Class<?>)` /
`link(icon, title, Class<?>)` — making a node's role (group vs. link) an unrepresentable-otherwise
state. `@Sidebar` stays a real annotation marking a plain class whose `private final
List<SidebarNode> nodes` field the loader reads via reflection. Multiple named sidebars per app are
supported.

## Backend (context-gen, pushed `e7d998a`)

- `Sidebar.java`: `name` (default `"default"`), `theme` (default `"primary"`), plus **six
  backend-only layout attributes added mid-session**: `side` (`SidebarSide` enum, LEFT/RIGHT),
  `variant` (`SidebarVariant`, SIDEBAR/FLOATING/INSET), `collapsible` (`SidebarCollapsible`,
  OFFCANVAS/ICON/NONE), `overlay`/`openOnHover`/`dismissable` (booleans) — every one mirrors a real
  PrimeNG `Sidebar` component input **name-for-name and wire-value-for-wire-value**
  (`.name().toLowerCase()` on each enum matches PrimeNG's own literal unions exactly, e.g.
  `"sidebar"`/`"floating"`/`"inset"`), so the frontend binds them straight off `SidebarContext` with
  zero translation and **no client-facing toggle** — this was the user's explicit requirement
  ("variants bar is activated disabled from the backend only").
- `SidebarNode.java` gained an optional `icon` (PrimeIcons class name, e.g. `"pi pi-calendar"`) via
  overloaded factories, backward-compatible with the no-icon originals.
- `SidebarContext`/`SidebarNodeContext` carry all of the above; `AnnotationSidebarLoader` populates
  them from the annotation/node graph. A node's `type` ("group"|"link") is still INFERRED from
  which factory built it, never a separate flag.
- 15/15 loader tests (was 11 — added coverage for layout-config defaults, overrides, and icon
  propagation/null-when-absent). `DOCUMENTATION.md`'s "Sidebars" section updated with the new
  attributes and a refreshed worked example.

## quar-crud-host (disk-only, no git repo)

`MainSidebar.java`: `@Sidebar(name="main", theme="blue", variant=SIDEBAR, collapsible=ICON)`, one
group ("Conventions Management", icon `pi pi-calendar` → Convention link, icon `pi pi-list`) + one
bare top-level link (Studios, icon `pi pi-building`) — two levels deep, no nested group in the real
data (context-gen's OWN unit tests separately cover 3+ level nesting via a dedicated fixture, not
this reference app). `SidebarResourceTest` (5/5, was 4) asserts every new field via
`GET /sidebar/main`. Verified live: `curl localhost:9100/sidebar/main` echoes
`side/variant/collapsible/overlay/openOnHover/dismissable` + per-node `icon` correctly after
`mvn install`-ing context-gen's bump and restarting the dev server (same port-9100-vs-8080
disambiguation gotcha as before — always check `ss -ltnp` before killing).

## Frontend: sb-sidebar, rebuilt on PrimeNG 22's real compound Sidebar API

**Not** `p-sidebar-layout` + `p-sidebar-main` (that pairing is for apps that want PrimeNG to lay out
their OWN page content, which this library doesn't own) — `p-sidebar-layout` is still required even
for a single standalone sidebar, though: `.p-sidebar-aside`'s `position:absolute; height:100%` only
resolves because `.p-sidebar` is a flex item of `.p-sidebar-layout` (`min-height:100svh`); skip the
Layout and the whole panel silently collapses to zero height (first real bug hit, caught by Cypress
`overflow:hidden`/`223x0px` failure, not by `ng build` — template compilation only validates
directive/input names, never cross-component CSS/layout assumptions).

Final template shape (`sidebar.component.html`):
```
p-sidebar-layout
  p-sidebar [id] [(open)] [side] [variant] [collapsible] [overlay] [dismissable] [openOnHover]  ← all bound off sidebarContext(), no user toggle
    p-sidebar-spacer
    p-sidebar-aside
      p-sidebar-panel
        p-sidebar-header > button[pSidebarTrigger]        ← icon-mode collapse toggle
        p-sidebar-content
          @for node: p-sidebar-group > [p-sidebar-group-label]? > p-sidebar-group-content > p-sidebar-menu
            @for child: p-sidebar-menu-item [collapsible]=child.type==='group' [open]/(openChange)
              child is link: a[pSidebarMenuButton]
              child is group (one extra nesting level only): button[pSidebarMenuButton] + p-sidebar-menu-sub > p-sidebar-menu-sub-item > a[pSidebarMenuSubButton]
      button[pSidebarRail]
    @if (sidebarContext()?.overlay) { p-sidebar-backdrop }   ← see gotcha below
```

Renders only two levels of the arbitrarily-deep `SidebarNode` tree in the UI (top-level
group/link, and one nested level via `p-sidebar-menu-sub`) — matches both the real data
(`MainSidebar.java` never nests deeper) and PrimeNG's own menu structure, which doesn't support a
third level cleanly either (`SidebarMenuSubItem`'s content is just a button, no further sub-menu).

**Every PrimeNG Sidebar sub-component's template is literally `<ng-content/>` and nothing else** —
all behavior comes from host-bound CSS classes/data-attributes (`cx('...')`, `data-side`,
`data-variant`, `data-collapsible-mode`, `data-state`) styled entirely by the theme preset. This is
a fully consumer-assembled compound API (same headless philosophy as shadcn/ui's own Sidebar, which
this visibly ports) — nothing is auto-wrapped internally, so the exact nesting in the template above
had to be reverse-engineered from the compiled `.mjs`/base CSS (`@primeuix/styles/dist/sidebar`),
not just the `.d.ts` input list.

**`SidebarTrigger`/`SidebarBackdrop` resolve their target Sidebar via ancestor DI** (`inject(SIDEBAR_INSTANCE, {optional:true})`)
when no explicit `target`/`id` is given — meaning a trigger button placed inside `<p-sidebar>`
works standalone with zero `p-sidebar-layout`-level wiring (`registerSidebar`/`target` id is only
needed for cross-sidebar coordination, which this library doesn't need — one `sb-sidebar` instance
per named sidebar).

**Real bug #2 (not just a test issue): `SidebarBackdrop`'s own `visible` signal is `pcSidebar.open()`
unconditionally — NOT gated by `overlay` at all.** Rendering `<p-sidebar-backdrop/>` unconditionally
made it cover (and block clicks on) the whole sidebar including its own trigger button, even in
plain non-overlay "sidebar" mode, since `open()` defaults `true`. Fix: only render
`<p-sidebar-backdrop/>` at all when `sidebarContext()?.overlay` is true — presence in the template
IS the overlay-mode opt-in, by design of this compound API (nothing hides it automatically).

`SidebarContext.ts`/`SidebarNode.ts` gained matching `side`/`variant`/`collapsible`/`overlay`/
`openOnHover`/`dismissable`/`icon` fields + defaults in their `normalize*` functions.
`SidebarComponent`'s `visible` model renamed to `open` (matches PrimeNG's own new terminology; no
external consumers referenced the old name, confirmed via grep before renaming).

Theming (`theme-palettes.ts`, host-scoped `[style]` binding) carried over unchanged from the v19
version — PrimeNG's own aura preset now styles virtually everything (`p-sidebar-group-label`,
`p-sidebar-menu-button`, etc. all `dt('sidebar.*')` tokens), so the component's own SCSS shrank to
just the loading row and the icon-mode trigger button.

## Testing

Rewrote `cypress/e2e/sidebar.cy.ts` for the new DOM/interaction model — the old "click group to
collapse/expand" test no longer applies (top-level `p-sidebar-group` isn't itself collapsible in
the real demo architecture; only nested `p-sidebar-menu-item`s are). New coverage: group+nested
link render, icons render, **whole-sidebar icon-mode collapse via the trigger** (asserts
`data-state`/`data-collapsible-mode` host attributes, not visibility, since the panel doesn't
disappear in icon mode — it just shrinks), layout config attributes exactly as served, hrefs,
theme CSS var, unknown-sidebar error toast. **7/7 passing**, run against quar-crud-host's real dev
server (default `ng serve` config is `environment.ts` = dev = real backend at `:9100`, not the
local JSON fixture) — this is genuine integration coverage, not mocked.

No Playwright available in this environment/session (unlike earlier in this same project's history
where it was apparently used) — Cypress via `npx cypress run` (Electron headless) was the only real
browser available and is what actually caught both real bugs above; template-compile success alone
(`ng build`) did NOT catch either one.

## Security incident (from original session, still unresolved as of this writing)

`.npmrc` with a leaked GitHub PAT (`ghp_k5Se...`) is still committed in the git history of
`dynamic-crud` and `search-crudstone` (both since their first commit) — pre-existing exposure, user
explicitly chose to handle rotation/history-rewrite themselves ("I'll handle all of it myself").
Don't assume it's been fixed without checking again. sidebar-crudstone itself was cleaned before
its first successful push and has no exposure.

## Repo/environment state

`https://github.com/Devary/sidebar-crudstone`, branch `main`, latest `d947821`. Demo app `:5902`
(`:5900` dynamic-crud, `:5901` search-crudstone). Angular/PrimeNG **22.x** (was 19.x) — same bump
still needs doing for [[project_gen_crudstone]] (dynamic-crud) and
[[project_search_crudstone]] (search-crudstone), both still on 19.x as of this writing, tracked as
separate pending tasks. context-gen `e7d998a`. quar-crud-host backend 39/39 (was 34). Node
`v24.18.0` via nvm (`nvm use 24.18.0` needed explicitly per Bash call — shell state doesn't persist
between tool invocations).

## Not yet done / natural next steps (not requested yet)
- No unit tests for `SidebarComponent`/`SidebarService` themselves (only Cypress e2e + backend
  loader tests) — matches the established testing balance in this ecosystem so far.
- `offcanvas`/`floating`/`inset`/`overlay=true`/`openOnHover=true` variants are all supported by the
  component (bound generically off `sidebarContext()`) but not exercised by any *real* `@Sidebar`
  class yet — `MainSidebar.java` only ever sets `variant=SIDEBAR, collapsible=ICON` (the defaults).
  Worth a second demo `@Sidebar` fixture if visual verification of the other variants is ever wanted.
- Angular/PrimeNG 22 upgrade still pending for dynamic-crud and search-crudstone (much larger
  surface area than sidebar-crudstone — dynamic-crud's CRUD table uses p-table/p-dialog/filters
  extensively; search-crudstone's layout/theming/card-grid too).

## PrimeUI license revert + hand-written compound sidebar (2026-07-27, later same day)

PrimeNG went commercial ("PrimeUI" dual license) at **v18.0.0** — v22 actively shows an "Invalid
PrimeUI License" banner from every component (`config.verified()` check in BaseComponent). User
wants free-only: all three gen projects' "Upgrade to Angular/PrimeNG 22" commits were `git revert`ed
(dynamic-crud `c4f7131`, search-crudstone `b205f88`, sidebar-crudstone `37ee471`), all back on
Angular 19 + PrimeNG 19.1.4, whose LICENSE.md is genuinely MIT for community versions. GOTCHA: the
banner persisted after revert until the long-running `ng serve` dev servers were killed and
restarted — a running dev server keeps serving the stale pre-revert bundle.

Since the compound Sidebar API only exists in paid PrimeNG 20+, **sb-sidebar was then rebuilt as a
fully hand-written compound sidebar on free v19** (commit `f6a964c`): own markup + SCSS reproducing
the Variants-demo look/behavior with zero PrimeNG sidebar imports. Key facts:
- Same attribute contract as the compound API: `data-state`/`data-side`/`data-variant`/
  `data-collapsible-mode` (+ own `data-overlay`, `data-hover-open`) on `[data-cy="sidebar-root"]` —
  the pre-revert v22 Cypress spec passes UNMODIFIED against it (7/7).
- All variants implemented in `sidebar.component.scss` keyed off those attributes: side left/right
  (right pins via `margin-left: auto` — was a real bug caught by screenshot, panel stayed left),
  variant sidebar (edge border) / floating (padded+border+radius+shadow) / inset (padded+radius),
  collapsible icon (3rem rail, labels hidden) / offcanvas (width 0 + translateX) / none (no
  trigger/rail), overlay (position:fixed z-1101 + `.sb-backdrop` z-1100, rendered only while open;
  backdrop click closes only if `dismissable`), openOnHover (hoverOpen signal, state stays
  "collapsed", stylesheet widens on `[data-hover-open]`).
- Uses only v19 Aura tokens: `--p-content-background/-border-color/-border-radius/-hover-background`,
  `--p-text-color/-muted-color`. Widths as host vars `--sb-width: 16rem`/`--sb-width-icon: 3rem`.
- `cypress/e2e/variants-screenshots.cy.ts` (committed): drives every variant via `cy.intercept` on
  `**/sidebar/main` overriding layout fields (the real backend fixture only serves
  left/sidebar/icon), takes a screenshot per variant; 8/8, suite total 15/15. `cypress/screenshots/`
  gitignored.
- Models (`SidebarContext.ts` layout fields, `SidebarNode.icon`), `main.json` fixture, and the v22
  spec were restored from the reverted commit `d947821` via `git show` — backend (context-gen
  `e7d998a` + quar-crud-host) was never reverted and still serves the full layout config.

## Proper styling + nested sub-group demo (2026-07-27, pushed `bf0e8df`)

User pushed back hard on the first hand-written version ("where are the colors, the styles, the
sub nodes and children?") — the initial port was structurally correct but visually bare, AND the
demo data had no nested group so the sub-menu level never rendered. Both fixed:
- **Styling**: brand header (accent-bg rounded mark `pi pi-th-large` + capitalized `context.name`
  as title + trigger, bottom divider), small-caps letter-spaced group labels with accent icons,
  rounded menu rows with `--p-primary-50` hover bg / `--p-primary-700` text / accent icons,
  single rotating chevron (`.sb-chevron-open` → rotate 90deg) instead of icon-swapping, sub-menu
  behind a 2px `--p-primary-200` tree rail, panel bg `--p-surface-50` (runtime Aura token, NOT in
  built styles.css — grep there is misleading, tokens are injected at runtime by the theme JS; use
  fallbacks `var(--p-x, #hex)`). Accent vars centralized as host-scoped `--sb-accent*` custom props
  derived from the themeVars() `--p-primary-*` override.
- **Nested data**: `MainSidebar.java` (quar-crud-host, disk-only) now nests group "Administration"
  (pi pi-cog: links "All Conventions" pi pi-database / "All Studios" pi pi-warehouse — distinct
  titles from the top-level links deliberately, duplicate data-cy attrs would break Cypress) inside
  "Conventions Management"; Quarkus dev-mode hot-reloaded it, no restart. `main.json` fixture +
  variants spec base data mirror it. New Cypress test: sub-group default-expanded, click collapses,
  click re-expands. Suite 16/16.
- Demo URL the user views: **http://localhost:5902/main** (route is `/:sidebarName`).

## Dark-mode interactive playground (2026-07-27, pushed `162829f`)

User supplied a screenshot of the (paid) PrimeNG sidebar "interactive playground" demo in dark
mode and asked for an exact replica. Built `/playground` (now the demo app's ROOT route; `/main`
etc. still work via the `:name` route placed after it):
- Controls: Variant/Collapsible `p-select`s (label/value objects — pTemplate selectedItem/item on
  p-select v19 silently did NOT apply, use optionLabel/optionValue instead), Left/Right segmented
  buttons, Overlay / Open on Hover / Backdrop `p-toggleswitch`es. Control changes rewrite a local
  Acme Inc context object.
- Library API added for this (all in sb-sidebar): **`context` input** (full locally-supplied
  SidebarContext → skips fetch entirely; how playgrounds/tests feed content), `activePaths`
  (marks link rows active), `showTrigger` (hide in-header hamburger; playground's toggle lives in
  the mock content header bound to `[(open)]`), `[sb-footer]` ng-content slot (user card),
  frontend-only model fields `SidebarNode.badge` (pill, e.g. Inbox 12), `SidebarNode.defaultOpen`
  (nested group starts collapsed), `SidebarContext.backdrop` (defaults to `overlay`; backdrop div
  renders only when overlay && backdrop && expanded). Brand mark is now the name's first LETTER
  (accent bg), title = capitalized context name ("Acme Inc" → "A").
- Dark mode: `.app-dark` on `<html>` (matches app.config darkModeSelector), body bg
  `--p-surface-950`. Component restyled NEUTRAL (muted icons, `--p-content-hover-background`
  hover/active rows, quiet group labels — accent only on brand mark + rail hover) to match the
  demo's look; `:host-context(.app-dark)` overrides `--sb-surface` to surface-950 so the sidebar
  sits darker than the surface-900 content panels. `--sb-height` var lets a host embed the
  sidebar in a sized container (playground stage sets 100%).
- 18/18 Cypress (playground smoke spec + untouched /main suite + variants). README not yet
  updated for the new inputs — open follow-up.

## Branch feature/sidebar-settings-modal (2026-07-27, pushed `1635e1a`) — NOT merged to main

Brand-as-dropdown + Settings modal + pinned sidebar, per user request, on its own branch:
- Library: `settingsMenu` input turns the brand row into a dropdown button (chevron rotates,
  anchored menu panel with one i18n'd "Settings" item → `settingsSelected` output; document:click
  host listener closes on outside click; clicking the brand while icon-collapsed EXPANDS the
  sidebar instead of opening the menu — the panel's overflow:hidden would clip an anchored menu at
  3.25rem width). New translation key `settings`.
- Playground: controls row + caption REMOVED from the page; params live in a `p-dialog` Settings
  modal (Variant/Collapsible/Side selects + Overlay/Open on Hover/Backdrop switches, all
  `appendTo="body"`), opened via the brand dropdown. CRUD-modal pattern applied (staged copy,
  confirm-before-save "Yes, apply", confirm-before-discard when dirty, [closable]=false, silent
  close when clean). Blur: global `.p-dialog-mask { backdrop-filter: blur(6px) }` in styles.scss
  (all PrimeNG dialog types share that mask class). GOTCHA hit: `pTemplate="footer"` silently
  renders NOTHING unless `PrimeTemplate` (from primeng/api) is in the standalone component's
  imports — dialog opened fine but had no footer buttons.
- Sidebar pinned: `.pg-sidebar-pin` position:fixed full-height left (or right per settings);
  content offset via computed margin (16rem open / 3.25rem icon-collapsed / +1rem for
  floating/inset padding / none in overlay mode), transitioned.
- 22/22 Cypress green (6 playground tests incl. staged-not-applied-until-confirm assertions).
  main branch still has the pre-modal playground (controls row on page).

## Footer menu + theme/light-mode settings (2026-07-27, branch feature/sidebar-settings-modal, `9b91f11`)

Three follow-ups on the branch, all pushed:
- **Menu moved brand → footer** (`1f6003b`): brand is a plain row again (data-cy="sidebar-brand" on
  the title span); the projected [sb-footer] user card is now the dropdown trigger
  (data-cy="sidebar-footer", chevron-up rotates, menu panel anchored ABOVE the row, item
  data-cy="sidebar-footer-settings"). Footer-empty detection is `:not(:has([sb-footer]))` — the
  projected element's own sb-footer attribute, since the wrapper button always exists now.
- **Theme + Dark Mode in Settings** (`9b91f11`): Theme p-select over all THEME_PALETTES entries
  with color-dot item/selectedItem templates (works because PrimeTemplate is imported — same
  gotcha as the dialog footer), Dark Mode toggleswitch flipping `.app-dark` on
  `document.documentElement` — both staged + confirm-applied like the other params. Light mode
  needed one fix: mock .pg-block was hardcoded surface-900 → light default surface-100 with
  `:host-context(.app-dark)` dark override. index.html still boots dark.
- Removed the `.pg-block-top` placeholder div (user request); only the fill block remains.
- 23/23 Cypress green (new test: pick Blue + toggle dark off → html loses app-dark, sidebar
  --p-primary-color #3b82f6).

## Light-by-default + MainSidebar-based playground + ecosystem dark mode (2026-07-27)

- Branch `feature/sidebar-settings-modal` (latest): boots LIGHT like dynamic-crud (`dc2b821` —
  no .app-dark in index.html, DEFAULT_SETTINGS theme 'primary'/darkMode false); playground menu
  no longer fakes Acme — it fetches quar-crud-host's real "main" sidebar via SidebarService,
  seeds settings from the served @Sidebar config (blue/sidebar/icon/left), feeds sb-sidebar the
  merged context. Brand shows "Main".
- **Dark mode added to ALL THREE UI projects** (user: "add a dark mode in the three UI
  projects"): shared pattern = floating moon/sun button in AppComponent (fixed top-right,
  data-cy="dark-toggle") toggling `.app-dark` on `<html>` via an effect, persisted in
  localStorage key `darkMode`. dynamic-crud `1c7b472`+`771ae71` (master), search-crudstone
  `24e92ac` (main), sidebar-crudstone on the feature branch (also re-baselines the Settings
  modal's Dark Mode switch from the live DOM class on open so the two controls don't fight).
- **Shared gotcha fixed in all three**: the PrimeSakai-scaffold `_core.scss` body used v17-era
  `--surface-ground`/`--text-color` vars that DON'T EXIST in v19 Aura (background silently fell
  back to white even in dark mode) → replaced with `--p-text-color`/`--p-surface-50` + an
  `html.app-dark body { --p-surface-950 }` override.
- cypress/screenshots/ now gitignored in dynamic-crud + search-crudstone too (one screenshot had
  slipped into dynamic-crud's history, removed in `771ae71`).

## Per-node custom link target: host+port[+path] override (2026-08-17, code complete, NOT YET LIVE-VERIFIED)

**Same DB-outage caveat as search-crudstone's own same-day entries**: `192.168.178.41:31432`
(quar-crud-host's Postgres) was still unreachable when this was written. context-gen's own tests
(DB-independent) confirm the annotation/loader logic; the actual `/main` sidebar page and its new
Cypress tests were NEVER run. Also: **quar-crud-host was deliberately NOT restarted** to pick up
this session's context-gen jar bump — the already-running process (up since before the outage) was
left alone rather than risk killing it for a restart that couldn't be verified anyway (DB-backed
endpoints would still fail regardless of the context-gen version). Restart it before trusting any
of this live.

User asked, in `SidebarNode`'s own link-declaring API, for a way to point one link node at a
custom host+port+path, OR host+port only (keeping the entity's own default path), OR neither
(unchanged default behavior — same server/port as today, i.e. the consumer's own
`SidebarCrudstoneConfig#crudstoneUrl`).

- **`SidebarNode.java`** (context-gen): added `linkHost`/`linkPort`/`linkPath` fields (all null
  unless set) + 4 new `link(...)` factory overloads (icon/no-icon × host+port /
  host+port+path — mirrors the existing icon/no-icon overload convention already used for
  `group`/`link`): `link(title, entity, host, port)`, `link(icon, title, entity, host, port)`,
  `link(title, entity, host, port, path)`, `link(icon, title, entity, host, port, path)`. `entity`
  is STILL required in every overload (still must be `@CrudstoneEntity`) even for a fully custom
  host+port+path target — its name is still used as the node's own `entityName`.
- **`SidebarNodeContext.java`**: same 3 fields added, propagated by `AnnotationSidebarLoader
  .toLinkNode` — `path` (the entity-derived default) is ALWAYS still populated regardless of which
  overload built the node; `linkPath` is a separate, independent field that a consumer prefers
  over `path` only when set (never blanks `path` itself out).
- **Frontend (`SidebarNode.ts`, `sidebar.component.ts`)**: `linkHref(node)` — `if (node.linkHost
  != null && node.linkPort != null) return http://${host}:${port}/${linkPath ?? path ?? ''}`,
  else falls back to the existing `crudstoneUrl + path` behavior, UNCHANGED for every node that
  doesn't opt in. Scheme is always assumed `http` (documented assumption, matches every other URL
  this whole ecosystem's configs already use for local/dev — no per-node https override).
- **Demo, `MainSidebar.java`** (quar-crud-host, disk-only): new top-level group "Search App" (icon
  `pi-search`) with two links, BOTH pointing at `search-crudstone` (`:5901`) — a genuinely
  different, real running app, not just a synthetic host string:
  - "Search Conventions" → `link(icon, title, Convention.class, "localhost", 5901)` (host+port
    only) → `http://localhost:5901/conventions` (Convention's own resolved path, "conventions",
    still applies — and happens to be a REAL, working route in search-crudstone too, since both
    apps share the same `:entity` route-segment convention).
  - "Cities (results page)" → `link(icon, title, City.class, "localhost", 5901, "cities/results")`
    (host+port+path) → `http://localhost:5901/cities/results` (City's OWN `Searchable
    #externalResult` results page — see search-crudstone's own same-day memory entry — not its
    plain CRUD/search route at all).
- **`main.json`** (sidebar-crudstone's own local-mode fixture) updated to match, for consistency —
  in practice the RUNNING demo app (`ng serve`, default `environment.ts` = "dev") talks to the
  LIVE backend, not this fixture; it's only exercised under an explicit `local` config/Cypress
  override, confirmed via reading `environment.ts`/`environment.local.ts`/`cypress.config.ts`
  (baseUrl `:5902`, no environment override — Cypress runs against the dev/live-backend build).
- **README** (`sidebar-crudstone/projects/sidebarcrudstone/README.md`): new "Custom link targets
  (per-node host/port override)" section (worked Java examples, all 3 cases) inserted between
  `provideSidebarCrudstone(config)` and Theming; the `SidebarNode` TS interface table in "The
  SidebarContext/SidebarNode model" section updated with the 3 new fields.
- **Tests**: context-gen +5 (`AnnotationSidebarLoaderTest`: default-null case, host+port-only case,
  host+port+path case, icon+host+port combo, `path` always still populated regardless) — all
  confirmed green, DB-independent. `sidebar.cy.ts` +2 new tests (host+port-only href, host+port+path
  href) asserting exact `href` attribute values on `[data-cy="sidebar-link-{title}"]` — mirrors the
  existing href-assertion test's own style exactly. **Gotcha caught by re-reading the existing
  suite before writing new tests, not by trial and error**: groups in this hand-written compound
  sidebar default to EXPANDED, and clicking an already-expanded group COLLAPSES it (confirmed by
  the pre-existing "Administration" test in the same file) — the new "Search App" group tests
  deliberately do NOT click it, only assert `.should('be.visible')` on the group header, since a
  click would have hidden the very links being asserted on. NOT YET RUN.
