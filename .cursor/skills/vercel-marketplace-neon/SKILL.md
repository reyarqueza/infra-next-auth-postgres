---
name: vercel-marketplace-neon
description: >-
  Provision Neon Postgres via Vercel Marketplace using Vercel CLI. Do not use
  Neon MCP create_project. Invoked by bootstrap orchestrator only.
disable-model-invocation: true
---

# Vercel Marketplace Neon

Provision Neon through Vercel Marketplace so env vars are injected into the Vercel project.

Parameters: `APP_NAME`, `VERCEL_TEAM`, `LOCAL_PATH`, `DB_RESOURCE_NAME`.

**Do not** use Neon MCP `create_project` — that bypasses Marketplace billing and auto env injection.

## Prerequisites

- `vercel link` completed in `LOCAL_PATH`
- Vercel CLI authenticated (`vercel whoami`)

## Discover setup

```bash
cd "${LOCAL_PATH}"
vercel integration guide neon --framework nextjs
```

## Install Neon integration

```bash
vercel integration add neon \
  --name "${DB_RESOURCE_NAME}" \
  --format=json \
  --scope "${VERCEL_TEAM}"
```

Notes:

- First-time install may open a browser for Marketplace/provider terms — CLI polls and resumes; user must accept
- In non-TTY, specify product slug if required (e.g. `neon/postgres` per guide output)
- Provisioning may take **1–3 minutes**

## Poll for env vars

```bash
vercel env ls --scope "${VERCEL_TEAM}"
```

Wait until `POSTGRES_URL` appears (retry every 15s, up to 5 minutes).

## Pull locally

```bash
vercel env pull .env.local --yes --scope "${VERCEL_TEAM}"
```

## Map DATABASE_URL

If the scaffolded app expects `DATABASE_URL` but Vercel injects `POSTGRES_URL`:

**Do not** run bare `vercel env add` in the agent shell — it prompts interactively and hangs in non-TTY. Always pipe the value:

```bash
cd "${LOCAL_PATH}"
vercel env pull .env.local --yes --scope "${VERCEL_TEAM}"

POSTGRES_URL="$(grep '^POSTGRES_URL=' .env.local | cut -d= -f2- | tr -d '"')"

printf "%s" "${POSTGRES_URL}" | vercel env add DATABASE_URL production preview development --scope "${VERCEL_TEAM}"
unset POSTGRES_URL
```

Do not echo `POSTGRES_URL` or `DATABASE_URL` in chat.

## Gate

`vercel env ls` shows `POSTGRES_URL` (and `DATABASE_URL` if app requires it) for the linked project.
