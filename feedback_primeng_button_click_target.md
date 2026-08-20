---
name: feedback-primeng-button-click-target
description: "Cypress .click() on a <p-button>'s own host element doesn't reliably reach its nested native <button> for (onClick)/form-submit purposes — target the inner button explicitly"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 9490a6be-eca4-4020-a624-b91245b42f3c
---

`<p-button>` (and other PrimeNG components with the same "component wraps a real native element"
shape, e.g. `styleClass="p-datepicker-prev-button"` buttons inside a date picker) renders as its
own custom-element tag with a nested REAL `<button>` inside it — `<p-button><button
class="p-ripple p-button ...">...</button></p-button>`. `cy.get('[data-cy=...]').click()`, when
the `[data-cy]`/other selector resolves to the OUTER `<p-button>` host, does not reliably deliver a
click that Angular picks up for the component's `(onClick)` output or a native form-submit — it
silently does nothing (no error, no exception), even though the button visually looks clickable
and isn't disabled.

**Why:** Confirmed in [[project_carstone]] (2026-08-21) by literally dumping the post-click DOM to
a file and reading it — the button's own internal state (`ng-reflect-disabled="false"`, valid
form) looked completely normal, but the handler had plainly never run. Clicking the SAME button via
`cy.get('[data-cy=...] button').click()` (explicitly the nested native element) worked immediately.
This same root cause likely explains an earlier, never-fully-diagnosed mystery in the same project:
a PrimeNG DatePicker's `.p-datepicker-prev-button` (also a `p-button`-styled element) not
responding to a Cypress click when trying to navigate to an earlier decade in a year-picker.

**How to apply:** when a Cypress `.click()` on a PrimeNG button-family component (`p-button`, a
`pButton`-styled nav arrow, etc.) doesn't seem to do anything — no error, just silent — before
assuming it's an app bug, retarget the click at the nested native interactive element (`button`,
sometimes `a`) instead of the component's own host/wrapper selector. A quick way to confirm before
committing to either theory: `cy.writeFile()` a dump of `document.body.outerHTML` right after the
click and grep it for whatever the click was supposed to produce — direct evidence beats guessing
at framework internals.
