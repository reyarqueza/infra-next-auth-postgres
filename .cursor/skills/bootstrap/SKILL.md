---
name: bootstrap
description: >-
  Master orchestrator for infra-next-auth-postgres. Collects APP_NAME and
  related parameters, then runs sub-skills in order to scaffold a Next.js app
  and provision GitHub, Vercel, Marketplace Neon, and auth DDL. Use only when
  the user explicitly asks to bootstrap an app.
disable-model-invocation: true
---

# Bootstrap orchestrator

Run the full bootstrap pipeline. Read each sub-skill file in order and execute its steps before proceeding.

## Collect parameters

Ask for any missing values:

| Variable | Default | Notes |
| --- | --- | --- |
| `APP_NAME` | required | kebab-case recommended |
| `GITHUB_OWNER` | required | GitHub username or org |
| `VERCEL_TEAM` | required | Vercel team slug (`vercel teams ls`) |
| `LOCAL_PATH` | `../{APP_NAME}` | Alongside this skills repo; resolve `../{APP_NAME}` relative to repo root |
| `DB_RESOURCE_NAME` | `{APP_NAME}-db` | Marketplace Neon resource name |
| `AUTH_GITHUB_ID` | required | GitHub OAuth App Client ID (created during preflight) |
| `AUTH_GITHUB_SECRET` | required | GitHub OAuth App Client Secret; never echo in chat |

Echo all resolved values and get user confirmation before step 1. Show `AUTH_GITHUB_ID`; mask or omit `AUTH_GITHUB_SECRET`.

**Safety:** Do not scaffold inside the skills repo directory (`infra-next-auth-postgres`). By default, generated apps live alongside this repo at `../{APP_NAME}`.

## Start timer

After the user confirms parameters and **before** step 1, record bootstrap start time:

```bash
date -u +%Y-%m-%dT%H:%M:%SZ > "/tmp/bootstrap-${APP_NAME}-started-at"
```

After `scaffold-app` creates `LOCAL_PATH`, copy the timestamp into the app directory (do not commit it):

```bash
cp "/tmp/bootstrap-${APP_NAME}-started-at" "${LOCAL_PATH}/.bootstrap-started-at"
```

## Sub-skills (execute in order)

Stop on gate failure. Do not skip gates.

| Step | Skill file | Gate |
| --- | --- | --- |
| 1 | [preflight-auth](../preflight-auth/SKILL.md) | GitHub MCP, Neon MCP, Vercel CLI auth OK |
| 2 | [scaffold-app](../scaffold-app/SKILL.md) | `npm run build` succeeds at `LOCAL_PATH` |
| 3 | [github-create-push](../github-create-push/SKILL.md) | Repo exists on GitHub |
| 4 | [vercel-git-link](../vercel-git-link/SKILL.md) | `.vercel/project.json` exists |
| 5 | [vercel-marketplace-neon](../vercel-marketplace-neon/SKILL.md) | `POSTGRES_URL` in Vercel env |
| 6 | [neon-ddl](../neon-ddl/SKILL.md) | Auth tables exist |
| 7 | [auth-env-setup](../auth-env-setup/SKILL.md) | All `.env.example` vars in Vercel |
| 8 | [verify-deploy](../verify-deploy/SKILL.md) | Deployment ready, login page loads |

**Step 2 note:** Before scaffolding, follow [scaffold-app](../scaffold-app/SKILL.md) **Next.js 16 prerequisites** — verify `next-best-practices` is installed (auto-install with `npx skills add vercel-labs/next-skills --agent cursor --skill next-best-practices --skill next-cache-components --copy -y -g` if missing), read `next-best-practices` and `next-cache-components`, or use the inline fallback `proxy.ts` template. See [vercel-labs/next-skills](https://www.skills.sh/vercel-labs/next-skills).

## Completion report

After step 8 gate passes, compute elapsed minutes from the start timestamp:

```bash
START=$(cat "${LOCAL_PATH}/.bootstrap-started-at" 2>/dev/null || cat "/tmp/bootstrap-${APP_NAME}-started-at")
END=$(date -u +%Y-%m-%dT%H:%M:%SZ)
# Round up to whole minutes; minimum 1 if elapsed > 0 seconds
```

Lead with:

> **Bootstrap complete** — `{APP_NAME}` is deployed to production. **Completed in X minutes** (infra-next-auth-postgres).

Then summarize:

- GitHub repo URL
- Vercel project URL
- Production URL
- Local path (`LOCAL_PATH`)
- Local dev: `cd LOCAL_PATH && npm run dev`
- Remaining manual steps (if any — e.g. OAuth callback URL after custom domain)

Delete `${LOCAL_PATH}/.bootstrap-started-at` and `/tmp/bootstrap-${APP_NAME}-started-at` after reporting.

Never echo secret values in the summary.
