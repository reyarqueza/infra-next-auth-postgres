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

## Next.js 16 prerequisites

Before generating app code (especially `proxy.ts`), read the installed **Vercel** skills if available:

| Skill | When to read |
| --- | --- |
| `nextjs` | App Router file conventions; **middleware → proxy** rename |
| `routing-middleware` | Proxy redirects, matcher patterns, auth at the network layer |

If those skills are not installed, follow the **`proxy.ts` fallback template below exactly** for the generated app's lightweight redirect behavior. Do not use legacy `middleware.ts`.

**Next.js 16 proxy rules (required):**

- File: `proxy.ts` at the project root (not `middleware.ts`)
- Export for the fallback template: `export function proxy(request: NextRequest)` — a named function or a plain default function
- Auth.js' supported proxy integration is also valid when you want Auth.js session semantics at the proxy layer: `export { auth as proxy } from "@/auth"`
- Do not mix the Auth.js proxy export with the fallback cookie-check template in the same file
- The fallback template uses an **optimistic cookie check** in `proxy.ts`; enforce auth authoritatively in Server Components (e.g. `dashboard/page.tsx` calls `auth()` and `redirect("/login")`)

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
| `app/(dashboard)/dashboard/page.tsx` | Protected; `auth()` + `redirect("/login")` if no session; welcome card |
| `app/actions/auth.ts` | Server actions: `loginWithGitHub`, `logout` |
| `proxy.ts` | Next.js 16 proxy — optimistic fallback redirect (see template below) |
| `components/login-form.tsx` | Button calling `loginWithGitHub` |
| `components/welcome-card.tsx` | Card with avatar and welcome message |
| `components/app-header.tsx` | Header with logout |
| `components/theme-provider.tsx` | next-themes provider |
| `sql-ddl/auth-schema.sql` | Auth.js Neon adapter schema (see below) |
| `.env.example` | Document all required env vars |

## `proxy.ts` fallback template

Use at project root for lightweight optimistic redirects. If the generated app needs Auth.js session handling in proxy instead, use Auth.js' supported `export { auth as proxy } from "@/auth"` pattern and keep the dashboard's authoritative Server Component guard.

```ts
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function proxy(request: NextRequest) {
  const sessionCookie =
    request.cookies.get("authjs.session-token") ??
    request.cookies.get("__Secure-authjs.session-token");

  if (
    !sessionCookie &&
    request.nextUrl.pathname.startsWith("/dashboard")
  ) {
    return NextResponse.redirect(new URL("/login", request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/dashboard/:path*"],
};
```

## Dashboard protection (authoritative)

Proxy-layer redirects are UX only. The fallback `proxy.ts` checks cookie presence, while Auth.js' proxy export can apply Auth.js session semantics; either way, the dashboard page must also guard:

```ts
const session = await auth();
if (!session?.user) redirect("/login");
```

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
