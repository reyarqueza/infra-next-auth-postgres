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
