# infra-next-auth-postgres

> **What this creates** — End to end: a production-deployed app **and** the infrastructure behind it.
>
> - **Application** — Next.js 16 app with Auth.js (GitHub sign-in), Neon Postgres adapter, shadcn/ui, a login page, and an empty dashboard page (for you to customize) for signed-in users only.
> - **GitHub** — New repository under your owner/org with the scaffolded code pushed.
> - **Vercel** — Project linked to that repo; production deploy on push (or explicit `vercel deploy --prod`).
> - **Neon** — Postgres database via the Vercel Marketplace integration; `POSTGRES_URL` and related env vars in Vercel.
> - **Database schema** — Auth.js tables applied on Neon (`sql-ddl/auth-schema.sql`).
> - **OAuth & secrets** — GitHub OAuth App for sign-in; `AUTH_SECRET`, callback URLs, and other vars from `.env.example` set in Vercel.
> - **Verification** — Agent confirms the production deployment is ready and the login page loads.
>
> **What this is** — A **markdown-only skills library** for Cursor. Clone this repo, complete the one-time setup (GitHub, Neon, Vercel), and ask the agent to **bootstrap** a new Next.js app with Auth.js, Neon Postgres, GitHub, and Vercel.
>
> This repo contains **skills only** — no application code. Generated apps are created alongside this repo in the parent directory (default: `../{APP_NAME}`).

## Prerequisites

Complete these **once** before your first bootstrap:

| Prerequisite | What to configure | How to verify |
| --- | --- | --- |
| **GitHub (MCP)** | Enable Cursor **GitHub plugin**; sign in when prompted | Agent runs GitHub MCP `get_me` in preflight |
| **Neon (MCP)** | Enable Cursor **Neon plugin**; sign in when prompted | Agent runs Neon MCP `list_projects` in preflight |
| **Vercel (CLI)** | Install [Vercel CLI](https://vercel.com/docs/cli); run `vercel login` | `vercel whoami` succeeds |
| **Vercel ↔ GitHub** | Install [Vercel for GitHub](https://vercel.com/docs/git/vercel-for-github) on your account/org | Required for git-linked Vercel projects (one-time) |
| **GitHub OAuth App** | Create OAuth App for app sign-in; note Client ID + Secret for bootstrap | See preflight checklist below |
| **Neon Marketplace terms** | Accept terms on first Vercel Neon integration (once per account) | Avoids browser popup mid-bootstrap |

> **Why CLI, not the Vercel plugin?** The Cursor Vercel MCP plugin is not supported in this workflow yet — it lacks project linking, Marketplace integrations (Neon), and env var management. Use the **Vercel CLI** for those steps instead.

**GitHub OAuth App (preflight — after the agent resolves `PRODUCTION_URL`):**

1. GitHub → Settings → Developer settings → OAuth Apps → New OAuth App
2. **Homepage URL:** `{PRODUCTION_URL}` (e.g. `https://test-3-rey-arquezas-projects.vercel.app` — **not** `https://{APP_NAME}.vercel.app`)
3. **Callback URL:** `{PRODUCTION_URL}/api/auth/callback/github` (optional: `http://localhost:3000/api/auth/callback/github` for local dev)
4. Copy **Client ID** and **Client Secret** — provide as `AUTH_GITHUB_ID` and `AUTH_GITHUB_SECRET` when the agent collects bootstrap parameters

Team production URLs follow `{project-name}-{scope-slug}.vercel.app` ([Vercel generated URLs](https://vercel.com/docs/deployments/generated-urls)). The agent resolves `PRODUCTION_URL` before asking for OAuth credentials.

## Bootstrap flow

1. Clone this repo:

   ```bash
   git clone git@github.com:reyarqueza/infra-next-auth-postgres.git && cd infra-next-auth-postgres
   ```

2. Open the folder in **Cursor**.

3. Complete the prerequisites above (GitHub MCP, Neon MCP, Vercel CLI, and Vercel ↔ GitHub).

4. In chat, say:

   > **Bootstrap my app**

   The agent asks for:

   | Parameter | Default | Description |
   | --- | --- | --- |
   | `APP_NAME` | _(required)_ | GitHub repo name and package name |
   | `GITHUB_OWNER` | _(required)_ | GitHub username or org |
   | `VERCEL_TEAM` | _(required)_ | Vercel team slug (`vercel teams ls`) |
   | `LOCAL_PATH` | `../{APP_NAME}` | In the parent directory, alongside this repo |
   | `DB_RESOURCE_NAME` | `{APP_NAME}-db` | Vercel Marketplace Neon resource name |
   | `VERCEL_PROJECT_NAME` | `{APP_NAME}` | Vercel project name; auto-suffixed if name exists in team |
   | `PRODUCTION_URL` | derived | `https://{VERCEL_PROJECT_NAME}-{VERCEL_TEAM}.vercel.app` |
   | `AUTH_GITHUB_ID` | _(required)_ | GitHub OAuth App Client ID (use `PRODUCTION_URL` in OAuth App) |
   | `AUTH_GITHUB_SECRET` | _(required)_ | GitHub OAuth App Client Secret (from preflight) |

   The agent resolves `VERCEL_PROJECT_NAME` and `PRODUCTION_URL` before collecting OAuth credentials. If a Vercel project named `APP_NAME` already exists in your team, the agent auto-suffixes the Vercel project only (e.g. `test-3-a7f2c1`); the GitHub repo stays `{GITHUB_OWNER}/{APP_NAME}`.

5. Confirm parameters. The agent runs sub-skills in order (unattended after confirmation if preflight is complete):

   ```
   preflight-auth → scaffold-app → github-create-push → vercel-git-link
   → vercel-marketplace-neon → neon-ddl → auth-env-setup → verify-deploy
   ```

   During `scaffold-app`, the agent auto-installs [vercel-labs/next-skills](https://www.skills.sh/vercel-labs/next-skills) globally if needed (`next-best-practices`, `next-cache-components`), reads them before generating code, and uses either Auth.js' supported proxy export or the fallback `proxy.ts` template.

6. Work in the generated app at `LOCAL_PATH` (separate from this skills repo).

7. On success, the agent reports total bootstrap time, e.g. **Completed in X minutes** (infra-next-auth-postgres).

## What bootstrap creates

| Output | Location / notes |
| --- | --- |
| Next.js app (Auth.js + Neon + shadcn/ui) | `LOCAL_PATH` |
| GitHub repo | `github.com/{GITHUB_OWNER}/{APP_NAME}` |
| Vercel project | `{VERCEL_PROJECT_NAME}` linked to GitHub; production deploy on push |
| Production URL | `{PRODUCTION_URL}` (e.g. `https://{VERCEL_PROJECT_NAME}-{VERCEL_TEAM}.vercel.app`) |
| Neon database | Vercel Marketplace; `POSTGRES_URL` in Vercel env |
| Auth schema | Applied via Neon MCP (`sql-ddl/auth-schema.sql`) |
| GitHub OAuth App | Sign-in for the app; callback URL points at `PRODUCTION_URL` |
| Auth & DB secrets | From `.env.example` synced to Vercel (never echoed in chat) |
| Production deployment | Verified — deployment `READY`, login page reachable |

## Skills layout

```
.cursor/skills/
├── bootstrap/SKILL.md              # Master orchestrator — start here
├── preflight-auth/SKILL.md
├── scaffold-app/SKILL.md           # Next.js 16 + Auth.js scaffold (includes proxy.ts template)
├── github-create-push/SKILL.md
├── vercel-git-link/SKILL.md
├── vercel-marketplace-neon/SKILL.md
├── neon-ddl/SKILL.md
├── auth-env-setup/SKILL.md
└── verify-deploy/SKILL.md
```

## Tooling summary

| Task | Tool |
| --- | --- |
| Create/push GitHub repo | Cursor **GitHub** plugin (MCP) |
| Run auth DDL on Neon | Cursor **Neon** plugin (MCP) |
| Vercel link, Marketplace Neon, env vars | **Vercel CLI** |
| Next.js 16 scaffold (step 2) | [vercel-labs/next-skills](https://www.skills.sh/vercel-labs/next-skills) (`next-best-practices`, `next-cache-components`) — auto-installed during bootstrap; Auth.js proxy export or inline fallback template in `scaffold-app` |

## Known human pause points

| When | Action |
| --- | --- |
| Preflight (once) | Create GitHub OAuth App using agent-resolved `PRODUCTION_URL`; have Client ID + Secret ready |
| Preflight (once per account) | Accept Vercel Neon Marketplace terms |
| Preflight fails | Log in to GitHub/Neon plugins or run `vercel login` |
| Neon provisioning | Agent polls 1–3 minutes for `POSTGRES_URL` — no human action |
| OAuth callback (later) | Update callback when custom domain is added |

After preflight is complete, bootstrap runs end-to-end without mid-run browser steps for OAuth.

## Limits

- Skills are **instructions for the agent**, not a guaranteed workflow engine
- Execution quality depends on the agent following skills in order
- Preflight requires one-time human setup (OAuth App, Marketplace terms); bootstrap itself is designed to run unattended after parameter confirmation

## License

MIT
