# infra-next-auth-postgres

Markdown-only skills library for bootstrapping Next.js + Auth.js + Neon Postgres apps with GitHub, Vercel, and Marketplace Neon.

## When the user asks to bootstrap

If the user says **"Bootstrap my app"**, **"bootstrap"**, or similar, read and follow:

`.cursor/skills/bootstrap/SKILL.md`

Do not run bootstrap steps unless the user explicitly requests it. This repo contains skills only — no application code.

## Runtime parameters

| Variable | Default | Description |
| --- | --- | --- |
| `APP_NAME` | _(ask)_ | GitHub repo name and package name |
| `GITHUB_OWNER` | _(ask)_ | GitHub user or org for `gh repo create` |
| `VERCEL_TEAM` | _(ask)_ | Vercel team slug for `--scope` |
| `LOCAL_PATH` | `../{APP_NAME}` | Alongside this skills repo in the parent directory |
| `DB_RESOURCE_NAME` | `{APP_NAME}-db` | Neon resource name for Marketplace |
| `DOMAIN_NAME` | _(ask)_ | Root domain in Cloudflare (e.g. `example.com`) |
| `VERCEL_PROJECT_NAME` | `{APP_NAME}` | Vercel project name; auto-suffixed if name exists in team |
| `PRODUCTION_URL` | derived | `https://{APP_NAME}.{DOMAIN_NAME}` — use for OAuth App and `AUTH_URL` |
| `AUTH_GITHUB_ID` | _(after step 7)_ | GitHub OAuth App Client ID — collected before step 8, not at bootstrap start |
| `AUTH_GITHUB_SECRET` | _(after step 7)_ | GitHub OAuth App Client Secret (never echo) |

Environment (shell — not asked in chat):

| Variable | Description |
| --- | --- |
| `CLOUDFLARE_API_TOKEN` | Zone-scoped API token (DNS Edit + Zone Read for the bootstrap `DOMAIN_NAME`) |
| `CLOUDFLARE_ZONE_ID` | Zone ID for `DOMAIN_NAME`; optional (looked up at preflight) |

Resolve `VERCEL_PROJECT_NAME` and `PRODUCTION_URL` at bootstrap start. Collect OAuth credentials after step 7. See [bootstrap/SKILL.md](.cursor/skills/bootstrap/SKILL.md).

## Tooling

| Task | Tool |
| --- | --- |
| GitHub repo create/push | **GitHub CLI** (`gh`) + git |
| Neon DDL / SQL | Cursor **Neon** plugin (MCP) |
| Vercel link, Marketplace Neon, env vars, domains | **Vercel CLI** (shell) |
| Cloudflare DNS (CNAME for custom domain) | **Cloudflare API** via `curl` (shell) |

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
