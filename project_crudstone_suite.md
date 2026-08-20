---
name: project-crudstone-suite
description: "gen/crudstone-suite: composite shell app consuming crudstone (dynamic-crud table) + sidebarcrudstone as file: library deps — real 'main' sidebar left, CRUD entity pages as content, port 5903, github.com/Devary/crudstone-suite"
metadata: 
  node_type: memory
  type: project
  originSessionId: 6e8faf99-060a-41ae-9fdc-dda341f2832e
---

# crudstone-suite — composite shell app (created 2026-07-27)

`/home/devary/sandbox/gen/crudstone-suite`, pushed to `github.com/Devary/crudstone-suite`
(`0f9d1da`, branch master). Composes the ecosystem's libraries into one app against
quar-crud-host (:9100): [[project-sidebar-crudstone]]'s `sb-sidebar name="main"` pinned left +
[[project-gen-crudstone]]'s crudstone `EntityPageComponent` as routed content
(`/:entity` → full CRUD table; default redirect `/conventions`). Port **5903** (CORS origin
added to quar-crud-host's application.properties, hot-reloaded). Persistent dark-mode toggle,
same pattern as siblings. Scaffolded by copying sidebar-crudstone's workspace minus
projects/cypress.

## How the libraries are consumed — the load-bearing details

- `"crudstone": "file:../dynamic-crud/dist/crudstone"` and
  `"sidebarcrudstone": "file:../sidebar-crudstone/dist/sidebarcrudstone"` in package.json.
  Rebuild a lib in its own workspace and the suite dev server picks it up.
- **npm installs `file:` directory deps as SYMLINKS** → without `"preserveSymlinks": true` in
  angular.json build options, each lib resolves `@angular/core` from ITS OWN workspace's
  node_modules (module resolution walks up from the symlink's REALPATH) → two Angular instances
  → runtime `NG0203: inject() must be called from an injection context` on the first library
  service. preserveSymlinks:true fixes it.
- Consequence of preserveSymlinks: the libs' own deps must exist in the SUITE's node_modules —
  `exceljs` (crudstone's export system) is a direct dependency here for that reason.
- crudstone's lib build needed `"allowedNonPeerDependencies": ["exceljs"]` in its
  ng-package.json (dynamic-crud `c1ad716`) — ng-packagr hard-fails otherwise.
- Sidebar links stay in-app: `provideSidebarCrudstone({crudstoneUrl: '/'})` — linkHref becomes
  `/conventions` etc., which IS this app's own CRUD route.
- sidebarcrudstone dist is built from the `feature/sidebar-settings-modal` branch working tree
  (context input, settingsMenu, footer slot, etc. included).

## Environment / gotchas
- No cypress suite of its own yet — verified via sidebar-crudstone's cypress with
  `--config baseUrl=http://localhost:5903` (throwaway spec, deleted after).
- Dev server gotcha (recurring): killing `ng serve` from a chained Bash call can orphan/crash
  the esbuild service; start `nohup npm start & disown` in its OWN tool call.
- angular.json serve buildTarget names needed the `sidebar-crudstone:` → `crudstone-suite:`
  rename (copied workspace), and the copied `local` configuration (fileReplacements to a
  deleted environment.local.ts) was removed.
