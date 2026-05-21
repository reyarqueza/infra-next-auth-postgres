---
name: scaffold-app
description: >-
  Scaffold a Next.js App Router app with Auth.js, Neon adapter, shadcn/ui,
  login/dashboard routes, and auth DDL at LOCAL_PATH. Invoked by bootstrap
  orchestrator only.
disable-model-invocation: true
---

# Scaffold app

Create the application at `LOCAL_PATH`. Use parameters from the bootstrap orchestrator: `APP_NAME`, `LOCAL_PATH`.

## Create Next.js app

```bash
npx create-next-app@latest "${LOCAL_PATH}" \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir=false \
  --import-alias "@/*" \
  --turbopack \
  --yes
```

If the directory must be created by create-next-app, use the directory name from `LOCAL_PATH`.

Set `package.json` `"name"` to `APP_NAME`.

## Dependencies

```bash
cd "${LOCAL_PATH}"
npm install next-auth@beta @auth/neon-adapter @neondatabase/serverless
npx shadcn@latest init -y -d
npx shadcn@latest add button card avatar separator dropdown-menu -y
npm install next-themes lucide-react class-variance-authority clsx tailwind-merge
```

## Auth (`auth.ts` at repo root)

- Lazy `NextAuth(() => { ... })` with `Pool` from `@neondatabase/serverless`
- `NeonAdapter(pool)`, GitHub provider, `session: { strategy: "database" }`
- `pages: { signIn: "/login" }`, `trustHost: true`
- Export `handlers`, `auth`, `signIn`, `signOut`

## Routes and files

| Path | Purpose |
| --- | --- |
| `app/api/auth/[...nextauth]/route.ts` | Export `GET`, `POST` from `@/auth` handlers |
| `app/page.tsx` | Redirect to `/dashboard` or `/login` by session |
| `app/(auth)/login/page.tsx` | Hero with `{APP_NAME}` title + GitHub sign-in |
| `app/(dashboard)/dashboard/page.tsx` | Protected; show welcome card with user from session |
| `app/actions/auth.ts` | Server actions: `loginWithGitHub`, `logout` |
| `proxy.ts` | Next.js 16 proxy: redirect unauthenticated `/dashboard` → `/login` |
| `components/login-form.tsx` | Button calling `loginWithGitHub` |
| `components/welcome-card.tsx` | Card with avatar and welcome message |
| `components/app-header.tsx` | Header with logout |
| `components/theme-provider.tsx` | next-themes provider |
| `sql-ddl/auth-schema.sql` | Auth.js Neon adapter schema (see below) |
| `.env.example` | Document all required env vars |

## Auth schema (`sql-ddl/auth-schema.sql`)

Include tables: `users`, `accounts`, `sessions`, `verification_token` with indexes per Auth.js Neon adapter requirements.

## Environment template (`.env.example`)

```env
DATABASE_URL=
AUTH_SECRET=
AUTH_URL=http://localhost:3000
AUTH_GITHUB_ID=
AUTH_GITHUB_SECRET=
```

## Gate

From `LOCAL_PATH`:

```bash
npm install
npm run build
```

Build may use stub env vars locally if needed (`DATABASE_URL=postgresql://stub`, `AUTH_SECRET=build-stub-secret-min-32-chars-long`, etc.) — do not commit stubs to git.

- **Pass:** build succeeds, `package.json` name is `APP_NAME`
- **Fail:** fix errors before `github-create-push`
