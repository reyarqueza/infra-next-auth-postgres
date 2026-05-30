# infra-next-auth-postgres

> **What this creates** — End to end: a production-deployed app **and** the infrastructure behind it.
>
> - **Application** — Next.js 16 app with Auth.js (GitHub sign-in), Neon Postgres adapter, shadcn/ui, a login page, and an empty dashboard page (for you to customize) for signed-in users only.
> - **GitHub** — New repository under your owner/org with the scaffolded code pushed.
> - **Vercel** — Project linked to that repo; production deploy on push (or explicit `vercel deploy --prod`).
> - **Custom domain** — `{APP_NAME}.{DOMAIN_NAME}` wired via Cloudflare DNS (A record or CNAME) to Vercel.
> - **Neon** — Postgres database via the Vercel Marketplace integration; `POSTGRES_URL` and related env vars in Vercel.
> - **Database schema** — Auth.js tables applied on Neon (`sql-ddl/auth-schema.sql`).
> - **OAuth & secrets** — GitHub OAuth App for sign-in; `AUTH_SECRET`, callback URLs, and other vars from `.env.example` set in Vercel.
> - **Verification** — Agent confirms the production deployment is ready and the login page loads.
>
> **What this is** — A **markdown-only skills library** for Cursor. Clone this repo, complete the one-time setup (GitHub, Neon, Vercel, Cloudflare), and ask the agent to **bootstrap** a new Next.js app with Auth.js, Neon Postgres, GitHub, and Vercel.
>
> This repo contains **skills only** — no application code. Generated apps are created alongside this repo in the parent directory (default: `../{APP_NAME}`).

## Prerequisites

Complete these **once** before your first bootstrap:

| Prerequisite | What to configure | How to verify |
| --- | --- | --- |
| **GitHub (CLI)** | Install [GitHub CLI](https://cli.github.com/); run `gh auth login` | `gh auth status` succeeds in preflight |
| **Neon (MCP)** | Enable Cursor **Neon plugin**; sign in when prompted | Agent runs Neon MCP `list_projects` in preflight |
| **Vercel (CLI)** | Install [Vercel CLI](https://vercel.com/docs/cli); run `vercel login` | `vercel whoami` succeeds |
| **Vercel ↔ GitHub** | Install [Vercel for GitHub](https://vercel.com/docs/git/vercel-for-github) on your account/org | Required for git-linked Vercel projects (one-time) |
| **Cloudflare DNS** | Scoped API token for your domain zone; export env vars (see below) | Agent runs Cloudflare zone check in preflight |
| **GitHub OAuth App** | Create during steps 2–7 (URLs known at start); provide creds before step 8 | See speed tips and OAuth section below |
| **Neon Marketplace terms** | Accept terms on first Vercel Neon integration (once per account) | Avoids browser popup mid-bootstrap |

> **Why CLI for GitHub and Vercel?** The Cursor **GitHub** and **Vercel** MCP plugins are not used in this workflow — GitHub MCP often lacks `repo` create scope (403 on `create_repository`), and the Vercel MCP lacks project linking, Marketplace integrations (Neon), and env var management. Use **`gh`** and the **Vercel CLI** instead.

### Cloudflare API token (one-time)

Create a scoped token at [Cloudflare API Tokens](https://dash.cloudflare.com/profile/api-tokens):

| Setting | Value |
| --- | --- |
| Template | Edit zone DNS (customize) |
| Permissions | `Zone` → `DNS` → `Edit`; `Zone` → `Zone` → `Read` |
| Zone resources | Include → Specific zone → your domain (e.g. `example.com`) |

Copy the zone ID from Cloudflare dashboard → your domain → Overview → right sidebar. `CLOUDFLARE_ZONE_ID` is optional — the agent looks it up from `DOMAIN_NAME` at preflight if unset.

Add to `~/.zshrc` (or equivalent — never commit these):

```bash
export CLOUDFLARE_API_TOKEN="your-token"
export CLOUDFLARE_ZONE_ID="your-zone-id"
```

Restart Cursor (or open a new terminal) so the agent inherits the variables.

Verify:

```bash
curl -sf -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  "https://api.cloudflare.com/client/v4/zones/${CLOUDFLARE_ZONE_ID}" \
  | jq -e '.success == true'
```

**GitHub OAuth App (create during steps 2–7 — before step 8):**

`PRODUCTION_URL` is `https://{APP_NAME}.{DOMAIN_NAME}` and is **known at bootstrap start**. Create the OAuth App **while the agent runs steps 2–7** so you are not blocked at step 8.

Create **one OAuth App per bootstrapped app** (OAuth Apps allow only one callback URL each).

1. GitHub → Settings → Developer settings → OAuth Apps → New OAuth App
2. **Homepage URL:** `{PRODUCTION_URL}` (e.g. `https://my-app.example.com`)
3. **Callback URL:** `{PRODUCTION_URL}/api/auth/callback/github` (optional: `http://localhost:3000/api/auth/callback/github` for local dev)
4. Copy **Client ID** and **Client Secret** — reply in chat anytime before step 8

If you have not created the app when step 7 finishes, the agent will pause and ask.

## Bootstrap flow

1. Clone this repo:

   ```bash
   git clone git@github.com:reyarqueza/infra-next-auth-postgres.git && cd infra-next-auth-postgres
   ```

2. Open the folder in **Cursor**.

3. Complete the prerequisites above (GitHub CLI, Neon MCP, Vercel CLI, Cloudflare token, and Vercel ↔ GitHub).

4. In chat, say:

   > **Bootstrap my app**

   **Optional (faster):** run custom domain and Neon Marketplace in parallel after step 4:

   > **Bootstrap my app.** After step 4, run vercel-custom-domain and vercel-marketplace-neon in parallel subagents.

   The agent asks for:

   | Parameter | Default | Description |
   | --- | --- | --- |
   | `APP_NAME` | _(required)_ | GitHub repo name and package name |
   | `GITHUB_OWNER` | _(required)_ | GitHub username or org |
   | `VERCEL_TEAM` | _(required)_ | Vercel team slug (`vercel teams ls`) |
   | `LOCAL_PATH` | `../{APP_NAME}` | In the parent directory, alongside this repo |
   | `DB_RESOURCE_NAME` | `{APP_NAME}-db` | Vercel Marketplace Neon resource name |
   | `DOMAIN_NAME` | _(required)_ | Root domain in Cloudflare (e.g. `example.com`) |
   | `VERCEL_PROJECT_NAME` | `{APP_NAME}` | Vercel project name; auto-suffixed if name exists in team |
   | `PRODUCTION_URL` | derived | `https://{APP_NAME}.{DOMAIN_NAME}` |

   OAuth credentials (`AUTH_GITHUB_ID`, `AUTH_GITHUB_SECRET`) can be supplied **during steps 2–7** or when the agent pauses before step 8.

   The agent resolves `VERCEL_PROJECT_NAME` and `PRODUCTION_URL` before starting. If a Vercel project named `APP_NAME` already exists in your team, the agent auto-suffixes the Vercel project only (e.g. `test-3-a7f2c1`); the GitHub repo stays `{GITHUB_OWNER}/{APP_NAME}`.

5. Confirm parameters. The agent records a start timestamp after confirmation, then runs sub-skills (unattended after confirmation if preflight is complete):

   ```
   preflight-auth → scaffold-app → github-create-push → vercel-git-link
   → vercel-custom-domain + vercel-marketplace-neon (parallel optional)
   → neon-ddl → auth-env-setup → verify-deploy
   ```

   Steps 5 and 6 can run in **parallel subagents** after `vercel-git-link` — use the optional prompt above or see [Speed tips](#speed-tips).

   During `scaffold-app`, the agent auto-installs [vercel-labs/next-skills](https://www.skills.sh/vercel-labs/next-skills) globally if needed (`next-best-practices`, `next-cache-components`), reads them before generating code, and uses either Auth.js' supported proxy export or the fallback `proxy.ts` template.

6. Work in the generated app at `LOCAL_PATH` (separate from this skills repo).

7. On success, the agent reports total bootstrap time, e.g. **Completed in X minutes** (infra-next-auth-postgres).

## What bootstrap creates

| Output | Location / notes |
| --- | --- |
| Next.js app (Auth.js + Neon + shadcn/ui) | `LOCAL_PATH` |
| GitHub repo | `github.com/{GITHUB_OWNER}/{APP_NAME}` |
| Vercel project | `{VERCEL_PROJECT_NAME}` linked to GitHub; production deploy on push |
| Production URL | `{PRODUCTION_URL}` (e.g. `https://{APP_NAME}.{DOMAIN_NAME}`) |
| Cloudflare DNS | A record or CNAME `{APP_NAME}` → Vercel |
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
├── vercel-custom-domain/SKILL.md   # Cloudflare DNS + Vercel domain
├── vercel-marketplace-neon/SKILL.md
├── neon-ddl/SKILL.md
├── auth-env-setup/SKILL.md
└── verify-deploy/SKILL.md
```

## Tooling summary

| Task | Tool |
| --- | --- |
| Create/push GitHub repo | **GitHub CLI** (`gh`) + git |
| Run auth DDL on Neon | Cursor **Neon** plugin (MCP) |
| Vercel link, Marketplace Neon, env vars, domains | **Vercel CLI** |
| Cloudflare DNS | **Cloudflare API** (`curl` + scoped token) |
| Next.js 16 scaffold (step 2) | [vercel-labs/next-skills](https://www.skills.sh/vercel-labs/next-skills) (`next-best-practices`, `next-cache-components`) — auto-installed during bootstrap; Auth.js proxy export or inline fallback template in `scaffold-app` |

## Speed tips

Typical bootstrap is **8–12 minutes** after these optimizations (down from ~15–25 min with blocking pauses and CLI hangs).

| Tip | Why it helps |
| --- | --- |
| **Create the GitHub OAuth App during steps 2–7** | `PRODUCTION_URL` is known at start — avoids a blocking pause before step 8 |
| **Accept Neon Marketplace terms once** | Prevents a browser popup during step 6 |
| **Use a reference app or template for scaffold** | Agent copies a prior bootstrap instead of codegen (`scaffold-app` fast path) |
| **Parallel subagents for steps 5 + 6** | Custom domain and Neon Marketplace overlap (~1–3 min saved). Prompt: *"After step 4, run vercel-custom-domain and vercel-marketplace-neon in parallel subagents."* |
| **Preflight complete before you say bootstrap** | GitHub CLI, Neon MCP, Vercel CLI, and Cloudflare token verified in one shot |

Hard floor (~5–7 min) is set by local scaffold build, one Vercel production deploy, and Neon first provision — those cannot be parallelized away.

## Known human pause points

| When | Action |
| --- | --- |
| Preflight (once) | Create Cloudflare API token; export `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ZONE_ID` |
| During steps 2–7 (recommended) | Create GitHub OAuth App for `{PRODUCTION_URL}`; provide Client ID + Secret before step 8 |
| Before step 8 (fallback) | If OAuth creds not yet sent, agent pauses and asks |
| Preflight (once per account) | Accept Vercel Neon Marketplace terms |
| Preflight fails | Run `gh auth login`, enable Neon plugin, run `vercel login`, or fix Cloudflare token |
| Neon provisioning | Agent waits for `POSTGRES_URL` — usually automatic, no human action |
| DNS propagation | Agent creates the Cloudflare record and continues; HTTP verified at deploy step |

After preflight is complete, bootstrap runs through step 7 without OAuth credentials in Vercel. Supply OAuth creds anytime before step 8.

## Limits

- Skills are **instructions for the agent**, not a guaranteed workflow engine
- Execution quality depends on the agent following skills in order
- Preflight requires one-time human setup (Cloudflare token, Marketplace terms); OAuth App creation can overlap steps 2–7

## License

MIT
