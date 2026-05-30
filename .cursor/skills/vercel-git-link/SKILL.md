---
name: vercel-git-link
description: >-
  Create Vercel project linked to GitHub repo and run vercel link locally.
  Uses Vercel CLI only. Invoked by bootstrap orchestrator only.
disable-model-invocation: true
---

# Vercel git link

Create a Vercel project connected to the GitHub repo and link the local app.

Parameters: `APP_NAME`, `VERCEL_PROJECT_NAME`, `GITHUB_OWNER`, `VERCEL_TEAM`, `LOCAL_PATH`.

Requires [Vercel for GitHub](https://vercel.com/docs/git/vercel-for-github) installed on `{GITHUB_OWNER}`.

**Do not** use `vercel project add` — it creates a project with framework preset **Other** and no GitHub connection, which causes 404s on App Router routes until fixed.

The scaffolded app must include [`vercel.json`](../scaffold-app/SKILL.md) with `"framework": "nextjs"` (belt-and-suspenders for every deploy).

## Create project with GitHub connection

Use a JSON body — form-encoded `gitRepository[type]` is rejected by the API:

```bash
vercel api /v11/projects -X POST \
  --scope "${VERCEL_TEAM}" \
  --json "{\"name\":\"${VERCEL_PROJECT_NAME}\",\"framework\":\"nextjs\",\"gitRepository\":{\"type\":\"github\",\"repo\":\"${GITHUB_OWNER}/${APP_NAME}\"}}"
```

Use `VERCEL_PROJECT_NAME` for the Vercel project (may differ from `APP_NAME` if auto-suffixed). Git repo remains `{GITHUB_OWNER}/{APP_NAME}`.

If the API returns an error about git access, stop and instruct user to install/configure the Vercel GitHub App.

If the API returns **400** (e.g. form-encoding was used) or git linking fails on create, retry **without** `gitRepository` — still set `"framework": "nextjs"`:

```bash
vercel api /v11/projects -X POST \
  --scope "${VERCEL_TEAM}" \
  --json "{\"name\":\"${VERCEL_PROJECT_NAME}\",\"framework\":\"nextjs\"}"
```

Then complete git connection in the step below. **Never** fall back to `vercel project add`.

If the API returns that the project name already exists, skip create and proceed to link + git connect below.

## Validate project create

Confirm framework and repo link before continuing:

```bash
vercel project inspect "${VERCEL_PROJECT_NAME}" --scope "${VERCEL_TEAM}"
```

- **Pass:** framework is `nextjs` (not `null` / Other)
- **Git linked:** output mentions the GitHub repo, or the git connect step below succeeds
- **Fail:** do not proceed — fix create/link; never use `vercel project add`

## Link locally

```bash
cd "${LOCAL_PATH}"
vercel link --yes --scope "${VERCEL_TEAM}" --project "${VERCEL_PROJECT_NAME}"
```

This creates `.vercel/project.json`.

## Connect GitHub (if not linked by API create)

If the project was created without `gitRepository`, or git connect failed during create:

```bash
cd "${LOCAL_PATH}"
vercel git connect "https://github.com/${GITHUB_OWNER}/${APP_NAME}" --scope "${VERCEL_TEAM}" --yes
```

## Gate

Verify:

- `LOCAL_PATH/.vercel/project.json` exists and contains `projectId`
- `LOCAL_PATH/vercel.json` exists and sets `"framework": "nextjs"`

Do not commit `.vercel/` to git — add to `.gitignore` if missing:

```
.vercel
.env.local
```
