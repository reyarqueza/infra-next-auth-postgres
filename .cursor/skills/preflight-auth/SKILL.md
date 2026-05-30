---
name: preflight-auth
description: >-
  Verify GitHub CLI, Neon MCP, Vercel CLI, and Cloudflare DNS API authentication
  before bootstrap. Invoked by the bootstrap orchestrator only.
disable-model-invocation: true
---

# Preflight auth

Verify tooling before any scaffold or infra steps.

Parameters: `APP_NAME`, `GITHUB_OWNER`, `PRODUCTION_URL`, `DOMAIN_NAME`.

Environment (not collected in chat — must be set in the shell):

- `CLOUDFLARE_API_TOKEN` — zone-scoped DNS token for `DOMAIN_NAME`
- `CLOUDFLARE_ZONE_ID` — zone ID for `DOMAIN_NAME` (optional; looked up if missing)

## Checks

### 1. GitHub CLI

Run in shell:

```bash
gh auth status
```

- **Pass:** prints authenticated account with `repo` scope (or sufficient scopes for create/push)
- **Fail:** instruct user to install [GitHub CLI](https://cli.github.com/) and run `gh auth login`; link to README prerequisites

Optional — confirm `GITHUB_OWNER` is reachable (personal account or org the user can create repos in):

```bash
gh api "users/${GITHUB_OWNER}" --jq .login
```

If `GITHUB_OWNER` is an org, also verify membership or create permission:

```bash
gh api "orgs/${GITHUB_OWNER}" --jq .login 2>/dev/null \
  || gh api "users/${GITHUB_OWNER}" --jq .login
```

### 2. Neon MCP

Call Neon MCP `list_projects` with `limit: 1`.

- **Pass:** returns without auth error
- **Fail:** instruct user to enable the Cursor **Neon plugin** and sign in

### 3. Vercel CLI

Run in shell:

```bash
vercel whoami
```

- **Pass:** prints authenticated user/team
- **Fail:** instruct user to run `vercel login` (browser flow)

### 4. Cloudflare DNS API

Verify the token is set and can read the zone:

```bash
test -n "${CLOUDFLARE_API_TOKEN}"
```

If `CLOUDFLARE_ZONE_ID` is unset, resolve it:

```bash
CLOUDFLARE_ZONE_ID="$(curl -sf -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  "https://api.cloudflare.com/client/v4/zones?name=${DOMAIN_NAME}" \
  | jq -r '.result[0].id // empty')"
test -n "${CLOUDFLARE_ZONE_ID}"
```

Verify zone access:

```bash
curl -sf -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  "https://api.cloudflare.com/client/v4/zones/${CLOUDFLARE_ZONE_ID}" \
  | jq -e '.success == true'
```

- **Pass:** returns `"success": true`
- **Fail:** instruct user to create a scoped Cloudflare API token (Zone → DNS → Edit; Zone → Zone → Read; zone = `{DOMAIN_NAME}`) and export `CLOUDFLARE_API_TOKEN` and optionally `CLOUDFLARE_ZONE_ID` in `~/.zshrc` (or equivalent), then restart Cursor

### 5. Vercel GitHub App (informational)

Remind user that [Vercel for GitHub](https://vercel.com/docs/git/vercel-for-github) must be installed on `{GITHUB_OWNER}` before git linking (step 4). If unsure, note they may need to install it during that step.

### 6. GitHub OAuth App (informational — not required at preflight)

OAuth credentials are **not** collected at bootstrap start. After the GitHub repo is created (step 3) and infra through step 7 is provisioned, the user creates a **GitHub OAuth App** for app sign-in before step 8:

1. GitHub → Settings → Developer settings → OAuth Apps → New OAuth App (or reuse an org-level app)
2. **Homepage URL:** `{PRODUCTION_URL}` (e.g. `https://my-app.example.com` — from `{APP_NAME}.{DOMAIN_NAME}`)
3. **Authorization callback URL:** `{PRODUCTION_URL}/api/auth/callback/github`
   - Optional for local dev: `http://localhost:3000/api/auth/callback/github`
4. Copy **Client ID** and generate **Client Secret**
5. Provide both as `AUTH_GITHUB_ID` and `AUTH_GITHUB_SECRET` when the agent pauses before step 8

**Optional validation** (when creds are supplied before step 8):

```bash
curl -s -o /dev/null -w "%{http_code}" -u "${AUTH_GITHUB_ID}:${AUTH_GITHUB_SECRET}" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/applications/${AUTH_GITHUB_ID}/token"
```

- **401** — invalid credentials; stop and ask the user to fix the OAuth App
- **Other** — basic auth accepted; proceed

### 7. Neon Marketplace terms (human — once per account)

Recommend accepting Vercel Neon Marketplace terms **before** first bootstrap so step 6 does not open a browser mid-run. One-time: run `vercel integration add neon` on any linked project, or accept terms when prompted during the first bootstrap.

## Gate

All four automated checks (GitHub CLI, Neon MCP, Vercel CLI, Cloudflare DNS API) must pass before proceeding to `scaffold-app`.

If the user is already authenticated, do not ask them to log in again — only stop when a check fails.

Bootstrap parameters must include `AUTH_GITHUB_ID` and `AUTH_GITHUB_SECRET` before step 8 (`auth-env-setup`); collect them after step 7, not at bootstrap start.
