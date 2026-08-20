---
name: project-leaked-npm-token
description: "A live GitHub Personal Access Token is committed in the git history of two public repos (dynamic-crud, search-crudstone) via .npmrc — found 2026-07-27, user is handling rotation/cleanup themselves"
metadata:
  node_type: memory
  type: project
  originSessionId: 234d18c6-f1ac-4b2e-bcc3-12e9a700a46e
---

# Leaked GitHub PAT in .npmrc (found 2026-07-27)

While scaffolding [[project_sidebar_crudstone]] (copying search-crudstone's workspace as a
template), GitHub's push protection blocked the very first push of the new repo: `.npmrc`
(copied verbatim in the scaffold) contains a real token,
`ghp_k5Se... (full value deliberately redacted from this memory file)` (configured for a `@devary` GitHub-Packages registry
none of these projects actually need — all three sibling libraries, crudstone/search-crudstone/
sidebar-crudstone, publish unscoped to public npm).

Checked further: **the same token is already committed AND already pushed** in the git history of
two existing public repos:
- `github.com/Devary/dynamic-crud` — since its very first commit
- `github.com/Devary/search-crudstone` — since its very first commit

Push protection didn't catch either at the time (org-level protection may not have been active
yet, or was bypassed). Both repos are public, so the token is currently extractable by anyone who
clones them.

**Why:** GitHub Personal Access Tokens grant scoped access (at minimum, package-publish rights to
whatever registry it was issued for; possibly broader repo access depending on how it was scoped
when created) — leaving one in public history is a standing credential-exposure risk until
revoked, independent of whether it's ever actually misused.

**How to apply:** Flagged directly to the user via `AskUserQuestion` rather than silently fixing
it — this is exactly the kind of already-shared, high-blast-radius situation (public exposure,
plus any history rewrite would need a destructive force-push affecting other clones) that calls
for the user driving the decision, not an unprompted fix. **User's explicit choice: they will
handle both revocation/rotation AND deciding whether/how to rewrite the affected history
themselves** — don't assume either has happened without checking again (`git log --oneline --
.npmrc` in each repo, or checking whether the token still authenticates) before referencing this
token or these two repos' history in a future session. For sidebar-crudstone itself (caught
before any successful push): `.npmrc` was deleted from the repo, added to `.gitignore`, the
not-yet-pushed commit was amended (safe, since nothing external depended on that commit hash yet)
before the first successful push — that repo's history is clean.

If asked to scaffold another new sibling project the same way (copying an existing workspace as a
template) in the future: check whether `.npmrc` (or any other credentials file) is being copied
along, and whether it's actually needed, before it ends up committed again.
