---
name: feedback-html-min-max-not-enforcement
description: "A native <input type=number>'s min/max HTML attribute is cosmetic, not enforcement — always pair it with a real clamp/validation handler, and test the actual value, not the attribute"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 9490a6be-eca4-4020-a624-b91245b42f3c
---

A native `<input type="number">`'s `min`/`max` HTML attribute does NOT block a typed keystroke,
paste, or autofill from exceeding it — it only disables the spinner arrows past that point and
marks the element `:invalid` via the constraint-validation API (which does nothing unless
something explicitly checks `checkValidity()`/`reportValidity()`, e.g. native `<form>` submission).
The same applies to `<input type="date">`'s `min`/`max`.

**Why:** Discovered in [[project_carstone]] (2026-08-21) — a year range filter's `[min]`/`[max]`
Angular bindings were set correctly from backend-declared bounds (`CrudstoneField#minValue/
maxValue/noFutureValue`), and a Cypress test asserted the attribute value was correct, so the fix
was believed complete and shipped (twice — once for a min>max cross-bound, once for a
no-future-value cap). The user could still type an out-of-bounds value the whole time; the tests
never caught it because they also only checked the (cosmetic) attribute, not the resulting stored
value.

**How to apply:**
- Whenever a numeric/date bound needs real enforcement in a template-driven form, wire an explicit
  clamp/validation handler to an actual interaction event — don't rely on the `[min]`/`[max]`
  binding alone.
- Clamp on `(blur)` (the settled, final value), not on every `(ngModelChange)`/keystroke — clamping
  a multi-digit number's incomplete intermediate typed states (e.g. "2", then "20", then "202" on
  the way to "2022") force-corrects them mid-keystroke and corrupts the rest of the user's typing.
  This is the same class of bug as a one-way binding fighting a live drag/interaction — see
  [[project_carstone]]'s slider-blocked fix for the drag-specific version of this same lesson.
- When writing a Cypress (or any E2E) test for a min/max constraint, assert the actual
  `.val()`/stored value after typing an out-of-bounds input, not just `.should('have.attr', 'max',
  ...)` — the attribute assertion passes even when enforcement is completely absent.
