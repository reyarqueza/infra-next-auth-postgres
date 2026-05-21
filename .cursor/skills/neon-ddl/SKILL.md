---
name: neon-ddl
description: >-
  Run auth schema DDL on Marketplace-provisioned Neon via Neon MCP. Invoked by
  bootstrap orchestrator only.
disable-model-invocation: true
---

# Neon DDL

Apply auth schema to the Neon database provisioned by Vercel Marketplace.

Parameters: `DB_RESOURCE_NAME`, `LOCAL_PATH`.

## Find Neon project

Neon MCP `list_projects` with `search: "{DB_RESOURCE_NAME}"`.

If not found:

- Wait 1–2 minutes (Marketplace provisioning delay) and retry
- Confirm Neon MCP is authenticated to the same Neon org as Marketplace

Record `projectId` from the matching project.

## Read DDL

Read `LOCAL_PATH/sql-ddl/auth-schema.sql`.

## Execute

Neon MCP `run_sql_transaction`:

- `projectId`: resolved ID
- `sqlStatements`: split DDL file into individual statements, or pass as one transaction

Expected tables: `users`, `accounts`, `sessions`, `verification_token`.

## Verify

Neon MCP `run_sql`:

```sql
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'public'
AND table_name IN ('users', 'accounts', 'sessions', 'verification_token');
```

## Gate

All four tables exist. If any missing, report error and do not proceed to `auth-env-setup`.
