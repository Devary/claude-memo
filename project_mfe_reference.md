---
name: project-mfe-reference
description: ui-service-template is the canonical reference for creating new Angular MFE remotes in the Redzone platform
metadata: 
  node_type: memory
  type: project
  originSessionId: f72047e5-3b36-4037-9590-0c35dcc016c4
---

# MFE Reference Template

`ui-service-template` is the official reference/blueprint for all new Angular MFE remote services.

## REQUIRED: Always ask before creating a new MFE

Before scaffolding any new MFE, **always ask**:

> "Should this MFE require shell authentication, or should it be accessible standalone (directly without the shell)?"

- **Auth required (default)**: embedded in shell only, button/UI gated behind shell postMessage token
- **Standalone**: `window.self === window.top` auto-detected, `tokenReady` set immediately, no shell token needed. Also requires adding the relevant gateway routes to the public bypass list in `GatewayAuthorizationService.isPublicRoute()`.

Both modes include the theme toggle button (visible only when not in iframe).

## What to copy from it
- `src/app/theme.store.ts` — local BehaviorSubject ThemeStore; handles `data-theme` attribute; has `toggle()`, `setTheme()`, `isDark`, `isLight`
- `src/app/app.ts` — postMessage bridge + standalone detection; `isStandalone = window.self === window.top`
- `src/styles.scss` — full `--rz-*` CSS token definitions for `[data-theme="light"]` and `[data-theme="dark"]`
- `src/app/app.scss` — all component styles use `var(--rz-*)` tokens; includes `.theme-toggle` button styles
- `src/app/app.html` — theme toggle button shown only when `isStandalone`
- `angular.json` — base href pattern: `/{service-name}/`
- `Jenkinsfile` — CI/CD: build → Docker → Harbor → Rundeck deploy

## Shell integration contract (postMessage protocol)
- MFE sends on init: `{ type: '{app-name}:request-token' }` and `{ type: '{app-name}:request-theme' }`
- MFE also retries token requests at 300ms and 1000ms (race with shell readiness)
- Shell sends: `{ type: '{app-name}:session', accessToken, tokenType, username, expiresIn }`
- Shell sends: `{ type: '{app-name}:theme', theme: 'light' | 'dark' }`
- MFE stores token in `sessionStorage` under key `{app-name}.accessToken`
- All postMessages use `window.location.origin` as targetOrigin (same-origin iframe via Traefik)

## Gateway public bypass (for standalone MFEs)
Add to `isPublicRoute()` in `quar-gateway/src/main/java/com/devary/gateway/service/GatewayAuthorizationService.java`:
```java
if ("{service-name}".equals(service)) {
    return "GET".equalsIgnoreCase(method) && normalizedPath.startsWith("hello/");
}
```
Only expose the specific routes that need to be public. All other routes for that service still require auth.

## Standalone mode behavior
- `window.self === window.top` → standalone: `tokenReady.set(true)` immediately, no postMessage setup
- In iframe (shell): normal flow, wait for shell token via postMessage
- Theme toggle button: `@if (isStandalone)` — visible only outside shell
- API calls in standalone: no Bearer token sent → gateway must have the route in public bypass

## Why it works (confirmed 2026-06-21)
- Theme toggle propagates via `theme$` Observable subscription in `MfeHostComponent` (not `effect()` — unreliable in Angular 19 when signal set inside RxJS)
- CSS tokens cascade once `data-theme` is set on `document.documentElement` in each document independently

**How to apply:** When the user asks to create a new MFE, FIRST ask auth vs standalone. Then clone `ui-service-template`, rename `{app-name}` in postMessage types, register in ui-shell routes and `MfeHostComponent`, and if standalone add gateway bypass.
