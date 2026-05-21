---
name: verify-deploy
description: >-
  Verify Vercel deployment and login page after bootstrap. Invoked by bootstrap
  orchestrator only.
disable-model-invocation: true
---

# Verify deploy

Confirm the bootstrapped app is deployed and reachable.

Parameters: `APP_NAME`, `VERCEL_TEAM`, `LOCAL_PATH`, `GITHUB_OWNER`.

## Trigger deployment

If no deployment exists after git push + Vercel link:

```bash
cd "${LOCAL_PATH}"
vercel deploy --prod --scope "${VERCEL_TEAM}"
```

Or push an empty commit to trigger GitHub → Vercel auto-deploy.

## Wait for deployment

```bash
vercel list "${APP_NAME}" --scope "${VERCEL_TEAM}"
```

Poll until status is `READY` (up to 10 minutes).

## Get production URL

```bash
vercel inspect --scope "${VERCEL_TEAM}" <deployment-url-or-id>
```

Or read from Vercel dashboard / project settings.

## Smoke checks

1. **Login page:** fetch production `/login` — expect 200 and sign-in UI
2. **Dashboard redirect:** fetch `/dashboard` unauthenticated — expect redirect to `/login`
3. **Neon (optional):** Neon MCP `run_sql` with `SELECT 1` on the provisioned project

Sign-in flow requires GitHub OAuth vars — if OAuth not configured yet, login page loading is sufficient for this gate.

## Completion report

Provide the user:

| Item | Value |
| --- | --- |
| GitHub | `https://github.com/{GITHUB_OWNER}/{APP_NAME}` |
| Vercel project | link from `vercel project inspect {APP_NAME}` |
| Production URL | deployment URL |
| Local path | `LOCAL_PATH` |
| Local dev | `cd LOCAL_PATH && npm run dev` |

List any remaining manual steps (OAuth callback URL update after custom domain, etc.).

## Gate

Production `/login` returns successfully. Report partial success if OAuth not yet configured but deployment is green.
