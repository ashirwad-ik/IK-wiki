# ik-ai-mock-interview-fe — overview

AI Mock Interview **frontend**. Next.js 15 App Router application, but effectively a **client-rendered SPA** — every page is `'use client'`, no server components, no server actions, no `app/api` route handlers. Data is fetched in-browser via React Query against the backend.

- **Remote:** `git@github-ik:ikickstart/ik-ai-mock-interview-fe.git`
- **Workspace:** `~/Desktop/IK/code/ik-ai-mock-interview-fe/`
- **Default branch:** `main` (+ `qa`).

## Stack

| | |
|---|---|
| Framework | Next.js **15.3.8** (App Router) |
| UI | React **19.0.1**, TypeScript 5 (strict) |
| Pkg mgr | **npm** (`package-lock.json`) |
| Styling | Tailwind CSS v4, `@codebards/up-components` (IK design system) |
| Data | `@tanstack/react-query` v5 over hand-rolled `fetch` wrapper (no axios) |
| Realtime | `livekit-client`, `@livekit/components-react` |
| Error reporting | `@sentry/nextjs` v10 |
| Analytics | Microsoft Clarity, custom Execute-API events, FingerprintJS for guest IDs |
| Node | Runs on Node 26 locally; repo targets `@types/node@^20`. **No `.nvmrc` / `engines`.** |

## Entry points

- `src/app/layout.tsx` — provider stack (Snackbar → QueryProvider → AuthProvider → InterviewProvider → gates/header/main).
- `src/app/page.tsx` — role-based route pick: RECRUITER→`/hr-dashboard`, CANDIDATE→`/invite/{id}`, else `/dashboard`.
- `src/utils/apiUtils.ts` — the API client (`apiClient()`, `sentinelFetch()`, `formatUrl()`).
- `src/constants/apiConstants.ts` — single source of endpoint URLs.
- `src/apis/*` — ~26 React Query hooks, one per endpoint.
- `src/providers/AuthProvider.tsx`, `src/utils/auth.ts` — auth state + guards.

## Local run

```bash
cd ~/Desktop/IK/code/ik-ai-mock-interview-fe
npm install
npm run dev     # http://localhost:3000
npm run build   # production build
npm run lint    # eslint, next-config aware
npx tsc --noEmit  # typecheck (no script shortcut exists)
```

## Routes (from build output)

| Kind | Route |
|---|---|
| Static | `/`, `/login`, `/signup`, `/configurations`, `/dashboard`, `/hr-dashboard`, `/forgot-password`, `/login/callback`, `/unauthorized` |
| Dynamic | `/invite/[id]`, `/report/[id]`, `/session/[roomId]`, `/hr-dashboard/template/[id](/report/[reportId])` |

## Related

- [`architecture.md`](architecture.md) — providers, API client layering, auth flow.
- [`integrations.md`](integrations.md) — backend hosts + third-party SDKs.
- [`gotchas.md`](gotchas.md) — committed `.env.*`, mobile gate, Node pin unknown.
- [`../../integrations/fe-be-contract.md`](../../integrations/fe-be-contract.md) — the FE↔BE contract.
