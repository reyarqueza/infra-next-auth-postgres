---
name: github-create-push
description: >-
  Create GitHub repository and push scaffolded app via GitHub MCP. Invoked by
  bootstrap orchestrator only.
disable-model-invocation: true
---

# GitHub create and push

Push the scaffolded app at `LOCAL_PATH` to a new GitHub repository.

Parameters: `APP_NAME`, `GITHUB_OWNER`, `LOCAL_PATH`.

## Create repository

GitHub MCP `create_repository`:

- `name`: `APP_NAME`
- `private`: ask user or default `false`
- `autoInit`: `false` (we push full tree)
- `organization`: `GITHUB_OWNER` if it's an org; omit for personal account

## Push files

Collect all project files from `LOCAL_PATH` (exclude `node_modules`, `.next`, `.env.local`, `.vercel`).

GitHub MCP `push_files`:

- `owner`: `GITHUB_OWNER`
- `repo`: `APP_NAME`
- `branch`: `main`
- `message`: `Initial commit: {APP_NAME} scaffold`
- `files`: array of `{ path, content }` for each file

Alternatively, if local git is initialized:

```bash
cd "${LOCAL_PATH}"
git init
git checkout -b main
git add .
git commit -m "Initial commit: ${APP_NAME} scaffold"
git remote add origin "https://github.com/${GITHUB_OWNER}/${APP_NAME}.git"
git push -u origin main
```

Prefer GitHub MCP if available; use git CLI as fallback.

## Gate

Verify repo exists: `https://github.com/{GITHUB_OWNER}/{APP_NAME}`

GitHub MCP `get_file_contents` or web check — default branch has `package.json`.
