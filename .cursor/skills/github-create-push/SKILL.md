---
name: github-create-push
description: >-
  Create GitHub repository and push scaffolded app via GitHub CLI. Invoked by
  bootstrap orchestrator only.
disable-model-invocation: true
---

# GitHub create and push

Push the scaffolded app at `LOCAL_PATH` to a new GitHub repository.

Parameters: `APP_NAME`, `GITHUB_OWNER`, `LOCAL_PATH`.

Ask the user whether the repo should be **public** or **private**; default **public**.

## Prepare local git

`create-next-app` usually initializes git. Ensure a `main` branch and at least one commit:

```bash
cd "${LOCAL_PATH}"

git rev-parse --git-dir >/dev/null 2>&1 || git init

git checkout -B main

if ! git rev-parse HEAD >/dev/null 2>&1; then
  git add .
  git commit -m "Initial commit: ${APP_NAME} scaffold"
fi
```

## Create repository and push

Use **GitHub CLI** (`gh`). Do not use the Cursor GitHub MCP plugin for repo create or push.

If `GITHUB_OWNER` is an **organization**, use `owner/repo` form. If it is the authenticated user's personal account, `APP_NAME` alone is fine (both work when owner matches):

```bash
cd "${LOCAL_PATH}"

VISIBILITY="--public"   # or --private per user choice

gh repo create "${GITHUB_OWNER}/${APP_NAME}" \
  ${VISIBILITY} \
  --source=. \
  --remote=origin \
  --push
```

If the repo already exists remotely but has no commits, add the remote and push instead:

```bash
git remote get-url origin >/dev/null 2>&1 \
  || git remote add origin "https://github.com/${GITHUB_OWNER}/${APP_NAME}.git"

git push -u origin main
```

## Gate

Verify the repo exists and `package.json` is on the default branch:

```bash
gh repo view "${GITHUB_OWNER}/${APP_NAME}" --json name,defaultBranchRef \
  --jq '.name'

gh api "repos/${GITHUB_OWNER}/${APP_NAME}/contents/package.json" --jq .name
```

Both commands must succeed before proceeding to `vercel-git-link`.
