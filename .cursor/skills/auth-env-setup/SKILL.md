---
name: auth-env-setup
description: >-
  Configure AUTH_SECRET, GitHub OAuth app vars, and AUTH_URL in Vercel. Invoked
  by bootstrap orchestrator only.
disable-model-invocation: true
---

# Auth env setup

Configure remaining environment variables for Auth.js GitHub sign-in.

Parameters: `APP_NAME`, `VERCEL_TEAM`, `LOCAL_PATH`.

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

## GitHub OAuth App (user action)

Guide the user to create a **GitHub OAuth App** (not the same as GitHub MCP plugin):

1. GitHub → Settings → Developer settings → OAuth Apps → New OAuth App
2. **Authorization callback URL:** `https://<vercel-domain>/api/auth/callback/github`
   - Local dev: `http://localhost:3000/api/auth/callback/github`
3. Copy Client ID and Client Secret

Add to Vercel (do not echo secrets in chat):

```bash
vercel env add AUTH_GITHUB_ID production preview development --scope "${VERCEL_TEAM}"
vercel env add AUTH_GITHUB_SECRET production preview development --scope "${VERCEL_TEAM}"
```

For local dev, user may use separate OAuth app or same app with localhost callback.

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

Pause until user completes OAuth app setup if credentials are missing.
