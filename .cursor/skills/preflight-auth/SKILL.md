---
name: preflight-auth
description: >-
  Verify GitHub MCP, Neon MCP, and Vercel CLI authentication before bootstrap.
  Invoked by the bootstrap orchestrator only.
disable-model-invocation: true
---

# Preflight auth

Verify tooling before any scaffold or infra steps.

Parameters: `APP_NAME`, `GITHUB_OWNER`, `AUTH_GITHUB_ID`, `AUTH_GITHUB_SECRET` (when validating OAuth creds).

## Checks

### 1. GitHub MCP

Call GitHub MCP `get_me`.

- **Pass:** returns authenticated user
- **Fail:** instruct user to enable the Cursor **GitHub plugin** and sign in; link to README prerequisites

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

### 4. Vercel GitHub App (informational)

Remind user that [Vercel for GitHub](https://vercel.com/docs/git/vercel-for-github) must be installed on `{GITHUB_OWNER}` before git linking (step 4). If unsure, note they may need to install it during that step.

### 5. GitHub OAuth App (human — before bootstrap)

The human must create a **GitHub OAuth App** for app sign-in (not the same as GitHub MCP plugin) **before** starting bootstrap:

1. GitHub → Settings → Developer settings → OAuth Apps → New OAuth App (or reuse an org-level app)
2. **Homepage URL:** `https://{APP_NAME}.vercel.app`
3. **Authorization callback URL:** `https://{APP_NAME}.vercel.app/api/auth/callback/github`
   - Optional for local dev: `http://localhost:3000/api/auth/callback/github`
4. Copy **Client ID** and generate **Client Secret**
5. Provide both as `AUTH_GITHUB_ID` and `AUTH_GITHUB_SECRET` when the agent collects bootstrap parameters

**Optional validation** (if creds are already supplied):

```bash
curl -s -o /dev/null -w "%{http_code}" -u "${AUTH_GITHUB_ID}:${AUTH_GITHUB_SECRET}" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/applications/${AUTH_GITHUB_ID}/token"
```

- **401** — invalid credentials; stop and ask the user to fix the OAuth App
- **Other** — basic auth accepted; proceed

### 6. Neon Marketplace terms (human — once per account)

Recommend accepting Vercel Neon Marketplace terms **before** first bootstrap so step 5 does not open a browser mid-run. One-time: run `vercel integration add neon` on any linked project, or accept terms when prompted during the first bootstrap.

## Gate

All three automated checks (GitHub MCP, Neon MCP, Vercel CLI) must pass before proceeding to `scaffold-app`.

If the user is already authenticated, do not ask them to log in again — only stop when a check fails.

Bootstrap parameters must include `AUTH_GITHUB_ID` and `AUTH_GITHUB_SECRET` before step 7 (`auth-env-setup`); collect them at bootstrap start, not mid-pipeline.
