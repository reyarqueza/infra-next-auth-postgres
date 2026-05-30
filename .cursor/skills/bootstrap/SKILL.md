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
| `APP_NAME` | required | kebab-case recommended; GitHub repo name and package name |
| `GITHUB_OWNER` | required | GitHub username or org |
| `VERCEL_TEAM` | required | Vercel team slug (`vercel teams ls`) |
| `LOCAL_PATH` | `../{APP_NAME}` | Alongside this skills repo; resolve `../{APP_NAME}` relative to repo root |
| `DB_RESOURCE_NAME` | `{APP_NAME}-db` | Marketplace Neon resource name |
| `DOMAIN_NAME` | required | Root domain in Cloudflare (e.g. `example.com`) |
| `VERCEL_PROJECT_NAME` | `{APP_NAME}` | Resolved in name-resolution step below; may be auto-suffixed |
| `PRODUCTION_URL` | derived | `https://{APP_NAME}.{DOMAIN_NAME}` |

Do **not** collect `AUTH_GITHUB_ID` or `AUTH_GITHUB_SECRET` at bootstrap start — see [Collect OAuth credentials](#collect-oauth-credentials) after the GitHub repo exists.

Environment (not collected in chat):

| Variable | Notes |
| --- | --- |
| `CLOUDFLARE_API_TOKEN` | Zone-scoped token (DNS Edit + Zone Read for `DOMAIN_NAME`) |
| `CLOUDFLARE_ZONE_ID` | Zone ID for `DOMAIN_NAME`; looked up at preflight if unset |

**Safety:** Do not scaffold inside the skills repo directory (`infra-next-auth-postgres`). By default, generated apps live alongside this repo at `../{APP_NAME}`.

## Resolve Vercel project name and production URL

After `APP_NAME`, `VERCEL_TEAM`, and `DOMAIN_NAME` are known, resolve names **before** starting the pipeline:

```bash
VERCEL_PROJECT_NAME="${APP_NAME}"
if vercel project inspect "${VERCEL_PROJECT_NAME}" --scope "${VERCEL_TEAM}" >/dev/null 2>&1; then
  VERCEL_PROJECT_NAME="${APP_NAME}-$(node -e "console.log(require('crypto').randomBytes(3).toString('hex'))")"
fi
CUSTOM_HOSTNAME="${APP_NAME}.${DOMAIN_NAME}"
PRODUCTION_URL="https://${CUSTOM_HOSTNAME}"
```

- If the suffixed Vercel project name also exists, retry up to 3 times with a new random suffix
- GitHub repo stays `{GITHUB_OWNER}/{APP_NAME}` — only the Vercel project name is suffixed on conflict

Echo all resolved values and get user confirmation before step 1. Show:

- `APP_NAME`, `VERCEL_PROJECT_NAME`, `DOMAIN_NAME`, `CUSTOM_HOSTNAME`, `PRODUCTION_URL`, `GITHUB_OWNER`, `VERCEL_TEAM`, `LOCAL_PATH`, `DB_RESOURCE_NAME`
- GitHub repo: `https://github.com/{GITHUB_OWNER}/{APP_NAME}`

Note for the user: **`PRODUCTION_URL` is known now** (e.g. `https://my-app.example.com`). You will create the GitHub OAuth App and provide credentials **after the repo is pushed** (before step 8). DNS is wired during step 5.

## Collect OAuth credentials

**After step 7 (`neon-ddl`) and before step 8 (`auth-env-setup`):**

1. Pause the pipeline once `https://github.com/{GITHUB_OWNER}/{APP_NAME}` exists
2. Instruct the user to create a **GitHub OAuth App** — **one per bootstrapped app** (OAuth Apps support only one callback URL):
   - **Homepage URL:** `{PRODUCTION_URL}`
   - **Authorization callback URL:** `{PRODUCTION_URL}/api/auth/callback/github`
   - Optional for local dev: `http://localhost:3000/api/auth/callback/github`
3. Collect `AUTH_GITHUB_ID` and `AUTH_GITHUB_SECRET` (never echo the secret)
4. Optionally validate credentials (see [preflight-auth](../preflight-auth/SKILL.md) OAuth validation)
5. Continue with step 8

Steps 4–7 run without OAuth credentials. Do not block on OAuth until step 8.

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
| 1 | [preflight-auth](../preflight-auth/SKILL.md) | GitHub CLI, Neon MCP, Vercel CLI, Cloudflare DNS API OK |
| 2 | [scaffold-app](../scaffold-app/SKILL.md) | `npm run build` succeeds at `LOCAL_PATH` |
| 3 | [github-create-push](../github-create-push/SKILL.md) | Repo exists on GitHub |
| 4 | [vercel-git-link](../vercel-git-link/SKILL.md) | `.vercel/project.json` exists |
| 5 | [vercel-custom-domain](../vercel-custom-domain/SKILL.md) | CNAME in Cloudflare; Vercel domain verified |
| 6 | [vercel-marketplace-neon](../vercel-marketplace-neon/SKILL.md) | `POSTGRES_URL` in Vercel env |
| 7 | [neon-ddl](../neon-ddl/SKILL.md) | Auth tables exist |
| — | **Collect OAuth credentials** (see above) | `AUTH_GITHUB_ID` and `AUTH_GITHUB_SECRET` from user |
| 8 | [auth-env-setup](../auth-env-setup/SKILL.md) | All `.env.example` vars in Vercel |
| 9 | [verify-deploy](../verify-deploy/SKILL.md) | Deployment ready, login page loads |

**Step 2 note:** Before scaffolding, follow [scaffold-app](../scaffold-app/SKILL.md) **Next.js 16 prerequisites** — verify `next-best-practices` is installed (auto-install with `npx skills add vercel-labs/next-skills --agent cursor --skill next-best-practices --skill next-cache-components --copy -y -g` if missing), read `next-best-practices` and `next-cache-components`, or use the inline fallback `proxy.ts` template. See [vercel-labs/next-skills](https://www.skills.sh/vercel-labs/next-skills).

## Completion report

After step 9 gate passes, compute elapsed minutes from the start timestamp:

```bash
START=$(cat "${LOCAL_PATH}/.bootstrap-started-at" 2>/dev/null || cat "/tmp/bootstrap-${APP_NAME}-started-at")
END=$(date -u +%Y-%m-%dT%H:%M:%SZ)
# Round up to whole minutes; minimum 1 if elapsed > 0 seconds
```

Lead with:

> **Bootstrap complete** — `{APP_NAME}` is deployed to production. **Completed in X minutes** (infra-next-auth-postgres).

Then summarize:

- GitHub repo URL (`{GITHUB_OWNER}/{APP_NAME}`)
- Vercel project name (`VERCEL_PROJECT_NAME`) and project URL
- Production URL (`PRODUCTION_URL`)
- Local path (`LOCAL_PATH`)
- Local dev: `cd LOCAL_PATH && npm run dev`

Delete `${LOCAL_PATH}/.bootstrap-started-at` and `/tmp/bootstrap-${APP_NAME}-started-at` after reporting.

Never echo secret values in the summary.
