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

> **Why CLI, not the Vercel plugin?** The Cursor Vercel MCP plugin is not supported in this workflow yet — it lacks project linking, Marketplace integrations (Neon), and env var management. Use the **Vercel CLI** for those steps instead.

**Not in preflight:**

- **GitHub OAuth App** (for app sign-in) — configured during bootstrap step `auth-env-setup`
- **Vercel Marketplace terms** — may require browser acceptance on first `vercel integration add neon`

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
   | `APP_NAME` | _(required)_ | GitHub repo name, Vercel project, package name |
   | `GITHUB_OWNER` | _(required)_ | GitHub username or org |
   | `VERCEL_TEAM` | _(required)_ | Vercel team slug (`vercel teams ls`) |
   | `LOCAL_PATH` | `../{APP_NAME}` | In the parent directory, alongside this repo |
   | `DB_RESOURCE_NAME` | `{APP_NAME}-db` | Vercel Marketplace Neon resource name |

5. Confirm parameters. The agent runs sub-skills in order:

   ```
   preflight-auth → scaffold-app → github-create-push → vercel-git-link
   → vercel-marketplace-neon → neon-ddl → auth-env-setup → verify-deploy
   ```

   During `scaffold-app`, the agent auto-installs [vercel-labs/next-skills](https://www.skills.sh/vercel-labs/next-skills) globally if needed (`next-best-practices`, `next-cache-components`), reads them before generating code, and uses either Auth.js' supported proxy export or the fallback `proxy.ts` template.

6. Work in the generated app at `LOCAL_PATH` (separate from this skills repo).

## What bootstrap creates

| Output | Location / notes |
| --- | --- |
| Next.js app (Auth.js + Neon + shadcn/ui) | `LOCAL_PATH` |
| GitHub repo | `github.com/{GITHUB_OWNER}/{APP_NAME}` |
| Vercel project | Linked to GitHub; production deploy on push |
| Neon database | Vercel Marketplace; `POSTGRES_URL` in Vercel env |
| Auth schema | Applied via Neon MCP (`sql-ddl/auth-schema.sql`) |
| GitHub OAuth App | Sign-in for the app; callback URL points at Vercel production |
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
| Preflight fails | Log in to GitHub/Neon plugins or run `vercel login` |
| First Neon Marketplace install | Accept terms in browser (CLI waits) |
| Neon provisioning | Wait 1–3 minutes for `POSTGRES_URL` |
| Auth env setup | Create GitHub OAuth App; add callback URL |
| OAuth callback | Update callback when custom domain is added |

## Limits

- Skills are **instructions for the agent**, not a guaranteed workflow engine
- Execution quality depends on the agent following skills in order
- Not fully hands-off — expect browser steps on first run per account

## License

MIT
