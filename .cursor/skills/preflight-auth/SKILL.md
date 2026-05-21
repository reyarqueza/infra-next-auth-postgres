---
name: preflight-auth
description: >-
  Verify GitHub MCP, Neon MCP, and Vercel CLI authentication before bootstrap.
  Invoked by the bootstrap orchestrator only.
disable-model-invocation: true
---

# Preflight auth

Verify tooling before any scaffold or infra steps.

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

## Gate

All three automated checks (GitHub MCP, Neon MCP, Vercel CLI) must pass before proceeding to `scaffold-app`.

If the user is already authenticated, do not ask them to log in again — only stop when a check fails.

## Not in preflight

- **GitHub OAuth App** for app sign-in → handled in `auth-env-setup`
- **Vercel Marketplace terms** → may appear during `vercel-marketplace-neon`
