# infra-next-auth-postgres

Markdown-only skills library for bootstrapping Next.js + Auth.js + Neon Postgres apps with GitHub, Vercel, and Marketplace Neon.

## When the user asks to bootstrap

If the user says **"Bootstrap my app"**, **"bootstrap"**, or similar, read and follow:

`.cursor/skills/bootstrap/SKILL.md`

Do not run bootstrap steps unless the user explicitly requests it. This repo contains skills only — no application code.

## Runtime parameters

| Variable | Default | Description |
| --- | --- | --- |
| `APP_NAME` | _(ask)_ | GitHub repo name, Vercel project name, package name |
| `GITHUB_OWNER` | _(ask)_ | GitHub user or org for `create_repository` |
| `VERCEL_TEAM` | _(ask)_ | Vercel team slug for `--scope` |
| `LOCAL_PATH` | `/Users/rey/dev/{APP_NAME}` | Where the scaffolded app is created |
| `DB_RESOURCE_NAME` | `{APP_NAME}-db` | Neon resource name for Marketplace |

## Tooling

| Task | Tool |
| --- | --- |
| GitHub repo create/push | Cursor **GitHub** plugin (MCP) |
| Neon DDL / SQL | Cursor **Neon** plugin (MCP) |
| Vercel link, Marketplace Neon, env vars | **Vercel CLI** (shell) |

The Cursor Vercel MCP plugin is not used in this workflow.

## Next.js scaffolding (step 2)

When running `scaffold-app`, the agent should also use installed **Vercel plugin skills** (`nextjs`, `routing-middleware`) for Next.js 16 conventions — especially `proxy.ts` (not `middleware.ts`). The scaffold skill includes a fallback template if those skills are unavailable. If missing, run `npx skills add vercel-labs/next-skills` before step 2 ([skills.sh/topic/nextjs](https://www.skills.sh/topic/nextjs)).
