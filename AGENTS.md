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
| `LOCAL_PATH` | `../{APP_NAME}` | Alongside this skills repo in the parent directory |
| `DB_RESOURCE_NAME` | `{APP_NAME}-db` | Neon resource name for Marketplace |
| `AUTH_GITHUB_ID` | _(ask)_ | GitHub OAuth App Client ID (from preflight) |
| `AUTH_GITHUB_SECRET` | _(ask)_ | GitHub OAuth App Client Secret (from preflight; never echo) |

## Tooling

| Task | Tool |
| --- | --- |
| GitHub repo create/push | Cursor **GitHub** plugin (MCP) |
| Neon DDL / SQL | Cursor **Neon** plugin (MCP) |
| Vercel link, Marketplace Neon, env vars | **Vercel CLI** (shell) |

The Cursor Vercel MCP plugin is not used in this workflow.

## Next.js scaffolding (step 2)

When running `scaffold-app`, verify [vercel-labs/next-skills](https://www.skills.sh/vercel-labs/next-skills) is installed globally (`next-best-practices`, `next-cache-components`) and read them before generating code — especially `proxy.ts` (not `middleware.ts`), RSC boundaries, and async APIs. If missing, run:

```bash
npx skills add vercel-labs/next-skills \
  --agent cursor \
  --skill next-best-practices \
  --skill next-cache-components \
  --copy -y -g
```

The scaffold skill includes a fallback `proxy.ts` template if install fails.
