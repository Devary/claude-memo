---
name: feedback-crud-modal-pattern
description: All CRUD UIs must use modals — confirmed by user as a universal pattern for all projects
metadata: 
  node_type: memory
  type: feedback
  originSessionId: f72047e5-3b36-4037-9590-0c35dcc016c4
---

Every CRUD operation in any management UI must follow this modal pattern:

1. **Form opens in a modal** — never inline/embedded in the page
2. **Confirmation before submit** — a second modal asking "Are you sure?" before saving creates or updates
3. **Confirmation before discard** — if the form is dirty (user made changes), show a confirmation modal before cancelling/closing
4. **Delete always confirms** — a confirmation modal before any delete (single, bulk, or delete-all)

**Why:** User explicitly stated this as a required UX pattern for all projects — prevents accidental data loss and accidental submissions.

**How to apply:** On every new management component or CRUD screen:
- Use a `CrudModalComponent` shell (backdrop + centered panel) for create/edit forms
- Use a `ConfirmModalComponent` for confirmations (save, delete, discard)
- Track `isDirty` signal — set true on any form field change, used to gate the discard confirm
- Never use browser `confirm()` — always use the modal
