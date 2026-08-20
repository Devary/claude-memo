---
name: project-theme-store
description: "ThemeStore RxJS refactor across ui-common, ui-shell, ui-service-template — what was done, what is deployed, what is pending"
metadata: 
  node_type: memory
  type: project
  originSessionId: f72047e5-3b36-4037-9590-0c35dcc016c4
---

# ThemeStore Refactor (session 2026-06-20)

## What was done

### @devary/ui-common@0.2.0 (published to npm, pushed to main)
- Replaced `ThemeService` (signal-only) with `ThemeStore` (RxJS BehaviorSubject + Angular signals)
- `ThemeStore` exposes:
  - `theme$: Observable<Theme>` — BehaviorSubject-backed, `distinctUntilChanged`
  - `theme: Signal<Theme>` — readonly signal
  - `isDark`, `isLight` — computed signals
  - `toggle()`, `setTheme(theme)`, `snapshot()`
- `ThemeToggleComponent` updated to inject `ThemeStore`
- 8 Vitest unit tests in `theme.store.spec.ts`
- `public-api.ts` exports `ThemeStore` and `type Theme` from `theme.store.ts`
- Old `theme.service.ts` and its spec deleted
- Version bumped from `0.1.0` → `0.2.0`

### ui-shell (pushed to develop, NOT yet merged to main)
- Installed `@devary/ui-common@0.2.0`
- `src/app/core/services/theme.service.ts` re-exports `ThemeStore` (was `ThemeService`)
- `shell-layout.component.ts` injects `ThemeStore`
- `mfe-host.component.ts` key fix:
  - Removed `effect()` for theme sync (unreliable — Angular effect scheduling doesn't fire reliably when signals are set inside RxJS subscription callbacks)
  - **Now subscribes directly to `themeService.theme$` with `takeUntilDestroyed()`**
  - This fires synchronously on every toggle and guarantees postMessage to iframe

### ui-service-template (pushed to develop, NOT yet merged to main)
- New `src/app/theme.store.ts` — local BehaviorSubject-backed ThemeStore (NOT from npm)
- `handleShellMessage` now calls `themeStore.setTheme(msg.theme)` instead of direct `setAttribute`
- ThemeStore subscription sets `document.documentElement.setAttribute('data-theme', theme)`

## Pending
- Jenkins builds for ui-shell and ui-service-template need to complete (both on develop)
- Merge develop → main for both, then full redeploy

## Key insight
**Why:** `effect()` in Angular 19 schedules asynchronously. When a signal is set from inside a RxJS `.subscribe()` callback (outside Angular's reactive graph), the scheduled effect may never fire. Subscribing to `theme$` Observable directly avoids this — BehaviorSubject fires synchronously.
**How to apply:** For cross-frame communication that must be immediate, always use RxJS Observables, not Angular signals/effects.

## Theme propagation flow (current design)
```
User clicks toggle (ui-shell sidebar)
  → ThemeStore.toggle()
  → BehaviorSubject.next('dark')
  → theme$.subscribe fires (synchronous)
  → MfeHostComponent.postThemeToIframe('dark')
  → iframe postMessage({ type: 'ui-service-template:theme', theme: 'dark' })
  → ui-service-template handleShellMessage
  → themeStore.setTheme('dark')
  → document.documentElement.setAttribute('data-theme', 'dark')
  → [data-theme="dark"] CSS tokens apply
```

## CSS token architecture
- Both ui-shell and ui-service-template define `--rz-*` tokens locally in their `styles.scss`
- Both use `[data-theme="light"]` (default) and `[data-theme="dark"]` selectors on `html`
- The MFE is a same-origin iframe (same domain, different path via Traefik routing)
- `data-theme` must be set independently on each document's `documentElement`
