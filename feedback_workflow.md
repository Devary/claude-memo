---
name: feedback-workflow
description: "User's preferred working style — push immediately, no confirmation prompts"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: f72047e5-3b36-4037-9590-0c35dcc016c4
---

Always push changes immediately after committing. Do not ask "should I push?" or "want me to push?" — just do it.

For every new project, always create a GitHub repo named after the project under the Devary org and push everything. Do not ask for confirmation.

**Why:** User explicitly requested both; asking wastes time and breaks flow.

**How to apply:**
1. After every `git commit`, follow immediately with `git push origin <branch>`.
2. For new projects: create the GitHub repo via API (`curl -H "Authorization: Bearer $GITHUB_PAT" https://api.github.com/orgs/Devary/repos -d '{"name":"<project>","private":true}'`), add remote, push. Requires `GITHUB_PAT` env var — if missing, prompt user once to set it with `echo "export GITHUB_PAT=..." >> ~/.profile`.
