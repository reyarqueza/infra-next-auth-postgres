---
name: auth-env-setup
description: >-
  Configure AUTH_SECRET, GitHub OAuth app vars, and AUTH_URL in Vercel. Invoked
  by bootstrap orchestrator only.
disable-model-invocation: true
---

# Auth env setup

Configure remaining environment variables for Auth.js GitHub sign-in.

Parameters: `APP_NAME`, `VERCEL_TEAM`, `LOCAL_PATH`, `AUTH_GITHUB_ID`, `AUTH_GITHUB_SECRET`.

Read required vars from `LOCAL_PATH/.env.example`.

## AUTH_SECRET

Generate without printing:

```bash
AUTH_SECRET="$(node -e "console.log(require('node:crypto').randomBytes(32).toString('base64url'))")"
printf "%s" "$AUTH_SECRET" | vercel env add AUTH_SECRET production preview development --scope "${VERCEL_TEAM}"
unset AUTH_SECRET
```

## AUTH_URL

After first deployment URL is known (or use `https://${APP_NAME}.vercel.app` as initial value):

```bash
vercel env add AUTH_URL production preview development --scope "${VERCEL_TEAM}"
```

Use production URL for production env; preview can use Vercel preview URL pattern if needed.

## GitHub OAuth credentials

Use `AUTH_GITHUB_ID` and `AUTH_GITHUB_SECRET` from bootstrap parameters (created during preflight — see [preflight-auth](../preflight-auth/SKILL.md)). Do not pause for manual OAuth App creation.

Add to Vercel (do not echo secrets in chat):

```bash
printf "%s" "${AUTH_GITHUB_ID}" | vercel env add AUTH_GITHUB_ID production preview development --scope "${VERCEL_TEAM}"
printf "%s" "${AUTH_GITHUB_SECRET}" | vercel env add AUTH_GITHUB_SECRET production preview development --scope "${VERCEL_TEAM}"
unset AUTH_GITHUB_SECRET
```

## Refresh local env

```bash
cd "${LOCAL_PATH}"
vercel env pull .env.local --yes --scope "${VERCEL_TEAM}"
```

## Gate

```bash
vercel env ls --scope "${VERCEL_TEAM}"
```

Must include: `DATABASE_URL` (or `POSTGRES_URL` + mapped `DATABASE_URL`), `AUTH_SECRET`, `AUTH_URL`, `AUTH_GITHUB_ID`, `AUTH_GITHUB_SECRET`.

If OAuth credentials are missing from bootstrap parameters, stop and instruct the user to complete preflight OAuth App setup — do not guide mid-run browser creation.
