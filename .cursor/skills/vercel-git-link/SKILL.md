---
name: vercel-git-link
description: >-
  Create Vercel project linked to GitHub repo and run vercel link locally.
  Uses Vercel CLI only. Invoked by bootstrap orchestrator only.
disable-model-invocation: true
---

# Vercel git link

Create a Vercel project connected to the GitHub repo and link the local app.

Parameters: `APP_NAME`, `GITHUB_OWNER`, `VERCEL_TEAM`, `LOCAL_PATH`.

Requires [Vercel for GitHub](https://vercel.com/docs/git/vercel-for-github) installed on `{GITHUB_OWNER}`.

## Create project with GitHub connection

```bash
vercel api /v11/projects -X POST \
  -F "name=${APP_NAME}" \
  -F "gitRepository[type]=github" \
  -F "gitRepository[repo]=${GITHUB_OWNER}/${APP_NAME}" \
  --scope "${VERCEL_TEAM}"
```

If the API returns an error about git access, stop and instruct user to install/configure the Vercel GitHub App.

## Link locally

```bash
cd "${LOCAL_PATH}"
vercel link --yes --scope "${VERCEL_TEAM}" --project "${APP_NAME}"
```

This creates `.vercel/project.json`.

## Gate

Verify `LOCAL_PATH/.vercel/project.json` exists and contains `projectId`.

Do not commit `.vercel/` to git — add to `.gitignore` if missing:

```
.vercel
.env.local
```
