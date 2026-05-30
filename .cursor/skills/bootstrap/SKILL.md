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

Do **not** collect `AUTH_GITHUB_ID` or `AUTH_GITHUB_SECRET` at bootstrap start — user may supply them during steps 2–7 or before step 8 (see [Collect OAuth credentials](#collect-oauth-credentials)).

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

Note for the user: **`PRODUCTION_URL` is known now** (e.g. `https://my-app.example.com`). Create the GitHub OAuth App **while steps 2–7 run** (see [Collect OAuth credentials](#collect-oauth-credentials)) — reply with credentials anytime before step 8. DNS is wired during step 5.

## Collect OAuth credentials

**Required before step 8 (`auth-env-setup`).** `PRODUCTION_URL` is known at bootstrap start — the user can create the OAuth App in parallel with steps 2–7 to avoid a blocking pause.

At parameter confirmation, tell the user:

> Create the GitHub OAuth App **while the agent runs steps 2–7**. Reply with `AUTH_GITHUB_ID` and `AUTH_GITHUB_SECRET` anytime before step 8.

**If credentials are not yet provided when step 7 finishes**, pause and collect them:

1. Instruct the user to create a **GitHub OAuth App** — **one per bootstrapped app** (OAuth Apps support only one callback URL):
   - **Homepage URL:** `{PRODUCTION_URL}`
   - **Authorization callback URL:** `{PRODUCTION_URL}/api/auth/callback/github`
   - Optional for local dev: `http://localhost:3000/api/auth/callback/github`
2. Collect `AUTH_GITHUB_ID` and `AUTH_GITHUB_SECRET` (never echo the secret)
3. Optionally validate credentials (see [preflight-auth](../preflight-auth/SKILL.md) OAuth validation)
4. Continue with step 8

Steps 4–7 run without OAuth credentials in Vercel. Do not block step 8 if the user already supplied credentials during steps 2–7.

## Start timer

After the user confirms parameters and **before** step 1, record bootstrap start time (epoch seconds):

```bash
date +%s > "/tmp/bootstrap-${APP_NAME}-started-at"
```

`scaffold-app` copies this into `LOCAL_PATH` after its build gate passes (see [scaffold-app](../scaffold-app/SKILL.md)). Wall-clock time includes any pause while collecting OAuth credentials.

## Sub-skills (execute in order)

Stop on gate failure. Do not skip gates.

| Step | Skill file | Gate |
| --- | --- | --- |
| 1 | [preflight-auth](../preflight-auth/SKILL.md) | GitHub CLI, Neon MCP, Vercel CLI, Cloudflare DNS API OK |
| 2 | [scaffold-app](../scaffold-app/SKILL.md) | `npm run build` succeeds at `LOCAL_PATH` |
| 3 | [github-create-push](../github-create-push/SKILL.md) | Repo exists on GitHub |
| 4 | [vercel-git-link](../vercel-git-link/SKILL.md) | `.vercel/project.json` exists |
| 5 | [vercel-custom-domain](../vercel-custom-domain/SKILL.md) | Cloudflare DNS record exists for `{APP_NAME}` |
| 6 | [vercel-marketplace-neon](../vercel-marketplace-neon/SKILL.md) | `POSTGRES_URL` in Vercel env |
| 7 | [neon-ddl](../neon-ddl/SKILL.md) | Auth tables exist |
| — | **Collect OAuth credentials** (see above) | `AUTH_GITHUB_ID` and `AUTH_GITHUB_SECRET` from user (may arrive during steps 2–7) |
| 8 | [auth-env-setup](../auth-env-setup/SKILL.md) | All `.env.example` vars in Vercel |
| 9 | [verify-deploy](../verify-deploy/SKILL.md) | Deployment ready, login page loads |

## Parallel steps 5 and 6 (optional)

After step 4 (`vercel-git-link`), steps 5 and 6 are **independent** — both only require a linked Vercel project. To reduce wall-clock time, launch **two subagents in parallel** (Task tool, one message, two calls):

1. Subagent A: execute [vercel-custom-domain](../vercel-custom-domain/SKILL.md) with all bootstrap parameters
2. Subagent B: execute [vercel-marketplace-neon](../vercel-marketplace-neon/SKILL.md) with all bootstrap parameters

Pass `APP_NAME`, `DOMAIN_NAME`, `VERCEL_PROJECT_NAME`, `VERCEL_TEAM`, `LOCAL_PATH`, `PRODUCTION_URL`, `DB_RESOURCE_NAME`, and Cloudflare env vars in each task prompt. Subagents do not share shell state.

**Do not start step 7 until both gates pass.** If either subagent fails, stop and fix before continuing.

User prompt example:

> Bootstrap my app. After step 4, run vercel-custom-domain and vercel-marketplace-neon in parallel subagents.

Expected savings: ~1–3 minutes. All other steps remain sequential.

**Step 2 note:** Before scaffolding, follow [scaffold-app](../scaffold-app/SKILL.md) **Next.js 16 prerequisites** — verify `next-best-practices` is installed (auto-install with `npx skills add vercel-labs/next-skills --agent cursor --skill next-best-practices --skill next-cache-components --copy -y -g` if missing), read `next-best-practices` and `next-cache-components`, or use the inline fallback `proxy.ts` template. See [vercel-labs/next-skills](https://www.skills.sh/vercel-labs/next-skills).

## Completion report

After step 9 gate passes, [verify-deploy](../verify-deploy/SKILL.md) prints the timed success message, summary table, and deletes timer files. Do not duplicate that report here.

Never echo secret values in the summary.
