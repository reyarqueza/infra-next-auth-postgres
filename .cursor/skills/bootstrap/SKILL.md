---
name: bootstrap
description: >-
  Master orchestrator for infra-next-auth-postgres. Collects APP_NAME and
  related parameters, then runs sub-skills in order to scaffold a Next.js app
  and provision GitHub, Vercel, Marketplace Neon, and auth DDL. Use only when
  the user explicitly asks to bootstrap an app.
disable-model-invocation: true
---

# Bootstrap orchestrator

Run the full bootstrap pipeline. Read each sub-skill file in order and execute its steps before proceeding.

## Collect parameters

Ask for any missing values:

| Variable | Default | Notes |
| --- | --- | --- |
| `APP_NAME` | required | kebab-case recommended |
| `GITHUB_OWNER` | required | GitHub username or org |
| `VERCEL_TEAM` | required | Vercel team slug (`vercel teams ls`) |
| `LOCAL_PATH` | `/Users/rey/dev/{APP_NAME}` | Must not be this skills repo path |
| `DB_RESOURCE_NAME` | `{APP_NAME}-db` | Marketplace Neon resource name |

Echo all resolved values and get user confirmation before step 1.

**Safety:** Do not scaffold inside the skills repo directory (`infra-next-auth-postgres`). Generated apps live elsewhere.

## Sub-skills (execute in order)

Stop on gate failure. Do not skip gates.

| Step | Skill file | Gate |
| --- | --- | --- |
| 1 | [preflight-auth](../preflight-auth/SKILL.md) | GitHub MCP, Neon MCP, Vercel CLI auth OK |
| 2 | [scaffold-app](../scaffold-app/SKILL.md) | `npm run build` succeeds at `LOCAL_PATH` |

**Step 2 note:** Before scaffolding, the agent must follow [scaffold-app](../scaffold-app/SKILL.md) **Next.js 16 prerequisites** (read Vercel `nextjs` / `routing-middleware` skills when installed, or use the inline `proxy.ts` template).
| 3 | [github-create-push](../github-create-push/SKILL.md) | Repo exists on GitHub |
| 4 | [vercel-git-link](../vercel-git-link/SKILL.md) | `.vercel/project.json` exists |
| 5 | [vercel-marketplace-neon](../vercel-marketplace-neon/SKILL.md) | `POSTGRES_URL` in Vercel env |
| 6 | [neon-ddl](../neon-ddl/SKILL.md) | Auth tables exist |
| 7 | [auth-env-setup](../auth-env-setup/SKILL.md) | All `.env.example` vars in Vercel |
| 8 | [verify-deploy](../verify-deploy/SKILL.md) | Deployment ready, login page loads |

## Completion report

After step 8, summarize for the user:

- GitHub repo URL
- Vercel project URL
- Production URL
- Local path (`LOCAL_PATH`)
- Remaining manual steps (if any)

Never echo secret values in the summary.
