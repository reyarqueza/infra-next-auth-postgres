---
name: verify-deploy
description: >-
  Verify Vercel deployment and login page after bootstrap. Invoked by bootstrap
  orchestrator only.
disable-model-invocation: true
---

# Verify deploy

Confirm the bootstrapped app is deployed and reachable.

Parameters: `APP_NAME`, `VERCEL_PROJECT_NAME`, `PRODUCTION_URL`, `VERCEL_TEAM`, `LOCAL_PATH`, `GITHUB_OWNER`.

## Trigger deployment

If no deployment exists after git push + Vercel link:

```bash
cd "${LOCAL_PATH}"
vercel deploy --prod --scope "${VERCEL_TEAM}"
```

Or push an empty commit to trigger GitHub → Vercel auto-deploy.

## Wait for deployment

```bash
vercel list "${VERCEL_PROJECT_NAME}" --scope "${VERCEL_TEAM}"
```

Poll until status is `READY` (up to 10 minutes).

## Get production URL

```bash
vercel inspect --scope "${VERCEL_TEAM}" <deployment-url-or-id>
```

Or read from Vercel dashboard / project settings. Prefer the deployment alias as source of truth if it differs from `PRODUCTION_URL`.

## Smoke checks

Use `{PRODUCTION_URL}` (or the deployment alias if different):

1. **Login page:** fetch `{PRODUCTION_URL}/login` — expect 200 and sign-in UI (GitHub sign-in button present)
2. **Dashboard redirect:** fetch `{PRODUCTION_URL}/dashboard` unauthenticated — expect redirect to `/login`
3. **Neon (optional):** Neon MCP `run_sql` with `SELECT 1` on the provisioned project

OAuth credentials are supplied at bootstrap start; do not accept partial success for missing OAuth env vars.

## Completion report

Compute elapsed minutes (same logic as [bootstrap](../bootstrap/SKILL.md) completion report):

```bash
START=$(cat "${LOCAL_PATH}/.bootstrap-started-at" 2>/dev/null || cat "/tmp/bootstrap-${APP_NAME}-started-at")
END=$(date -u +%Y-%m-%dT%H:%M:%SZ)
# Round up to whole minutes; minimum 1 if elapsed > 0 seconds
```

Lead with:

> **Bootstrap complete** — `{APP_NAME}` is deployed to production. **Completed in X minutes** (infra-next-auth-postgres).

Then provide:

| Item | Value |
| --- | --- |
| GitHub | `https://github.com/{GITHUB_OWNER}/{APP_NAME}` |
| Vercel project | link from `vercel project inspect {VERCEL_PROJECT_NAME}` |
| Production URL | `{PRODUCTION_URL}` (or deployment alias) |
| Local path | `LOCAL_PATH` |
| Local dev | `cd LOCAL_PATH && npm run dev` |

List any remaining manual steps (OAuth callback URL update after custom domain, etc.).

## Gate

Production `/login` returns 200 with sign-in UI. All smoke checks must pass — bootstrap is not complete until login page is verified with OAuth configured in Vercel.
