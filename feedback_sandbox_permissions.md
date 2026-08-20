---
name: feedback-sandbox-permissions
description: Never ask for confirmation when working inside the sandbox directory
metadata: 
  node_type: memory
  type: feedback
  originSessionId: f72047e5-3b36-4037-9590-0c35dcc016c4
---

Never ask the user for permission when performing actions inside `/home/devary/sandbox/`. Just proceed with creating files, running commands, making changes.

**Why:** User explicitly said so — the sandbox is a safe workspace for development.

**How to apply:** In `/home/devary/sandbox/*`, do not pause to confirm before writing files, running builds, initializing git repos, or making other changes. Only ask outside sandbox or for risky actions (force push, deleting non-sandbox data, etc.).
