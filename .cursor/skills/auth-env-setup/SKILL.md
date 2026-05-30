---
name: auth-env-setup
description: >-
  Configure AUTH_SECRET, GitHub OAuth app vars, and AUTH_URL in Vercel. Invoked
  by bootstrap orchestrator only.
disable-model-invocation: true
---

# Auth env setup

Configure remaining environment variables for Auth.js GitHub sign-in.

Parameters: `PRODUCTION_URL`, `VERCEL_TEAM`, `LOCAL_PATH`, `AUTH_GITHUB_ID`, `AUTH_GITHUB_SECRET`.

Read required vars from `LOCAL_PATH/.env.example`.

## Vercel CLI notes

Use **production** target only for bootstrap (preview/development are optional and require extra flags for branch scope).

Always append `</dev/null` to each `vercel env add` — without it the CLI may hang waiting for stdin in non-TTY sessions.

Run all four adds in one shell block; do not loop over preview/development unless the user explicitly requests them.

## Add auth env vars (production)

From `LOCAL_PATH`:

```bash
cd "${LOCAL_PATH}"

AUTH_SECRET="$(node -e "console.log(require('node:crypto').randomBytes(32).toString('base64url'))")"

vercel env add AUTH_SECRET production --value "$AUTH_SECRET" --yes --force --scope "${VERCEL_TEAM}" --non-interactive </dev/null
vercel env add AUTH_URL production --value "${PRODUCTION_URL}" --yes --force --scope "${VERCEL_TEAM}" --non-interactive </dev/null
vercel env add AUTH_GITHUB_ID production --value "${AUTH_GITHUB_ID}" --yes --force --scope "${VERCEL_TEAM}" --non-interactive </dev/null
vercel env add AUTH_GITHUB_SECRET production --value "${AUTH_GITHUB_SECRET}" --yes --force --scope "${VERCEL_TEAM}" --non-interactive </dev/null

unset AUTH_SECRET AUTH_GITHUB_SECRET
```

Do not echo secrets in chat.

Requires [vercel-custom-domain](../vercel-custom-domain/SKILL.md) to have completed before this step (DNS record exists; HTTP verification happens in [verify-deploy](../verify-deploy/SKILL.md)).

## GitHub OAuth credentials

Collect `AUTH_GITHUB_ID` and `AUTH_GITHUB_SECRET` from the user if not already provided (see [bootstrap](../bootstrap/SKILL.md)). Do not proceed without them.

## Refresh local env

```bash
cd "${LOCAL_PATH}"
vercel env pull .env.local --yes --scope "${VERCEL_TEAM}" </dev/null
```

## Gate

```bash
vercel env ls --scope "${VERCEL_TEAM}"
```

Must include in **Production**: `DATABASE_URL` (or `POSTGRES_URL` from Marketplace), `AUTH_SECRET`, `AUTH_URL`, `AUTH_GITHUB_ID`, `AUTH_GITHUB_SECRET`.

If OAuth credentials are missing, stop and instruct the user to create the GitHub OAuth App for `{PRODUCTION_URL}` — see [bootstrap](../bootstrap/SKILL.md).
