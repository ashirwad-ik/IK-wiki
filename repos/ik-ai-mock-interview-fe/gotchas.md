# ik-ai-mock-interview-fe — gotchas

## 🚨 `.env.development`, `.env.local`, `.env.production` are TRACKED

The three env files are committed into the repo and contain **real-looking secret values** (LiveKit secret, API key, analytics key). Confirmed at first-onboarding analysis (2026-11-05).

- **Do not** paste new secrets in and commit — you're exposing them to everyone with repo read.
- **Open question for manager:** intentional (rotated regularly?), or should these be rotated + untracked?
- When editing, prefer leaving `NEXT_PUBLIC_API_KEY` empty in `.env.development` / `.env.production` and only set in `.env.local` (observed pattern).

## `NEXT_PUBLIC_LIVEKIT_API_KEY` / `NEXT_PUBLIC_LIVEKIT_API_SECRET` on the client

Any `NEXT_PUBLIC_*` value is inlined into the browser bundle. Shipping a LiveKit **secret** to the client is almost certainly vestigial — the real token is minted server-side by the BE `launch` endpoint. **Open question for manager:** can these be removed from the FE env?

## Commitlint config vs reality

`commitlint.config.js` extends `@commitlint/config-conventional`, but **no hook actually invokes commitlint**. The real enforcement is the husky regex `^[A-Z]+-[0-9]+: .{1,50}`. Do not write `feat:` / `fix:` thinking it'll be accepted — it may pass locally but isn't the house style.

## No Node version pinned

No `.nvmrc`, no `engines` in `package.json`. Node 26 works locally for install/build/dev. CI may pin Node 20 LTS. **Open question for manager.**

## Mobile is gated out

`DesktopOnlyNotice` blocks viewports <1024px. Not a bug — **open question for manager:** intentional UX decision, or temporary?

## `npm install` rewrites `package-lock.json`

Even without changing deps, `npm install` can shuffle the lockfile. If you weren't adding deps, `git checkout package-lock.json` before committing.

## 36 npm audit findings (as of 2026-11-05)

3 critical / 11 high / 20 moderate / 2 low. Build and dev run anyway. **Open question for manager:** upgrade cadence or accepted risk?

## 5 unapproved install scripts

`@sentry/cli`, `@tailwindcss/oxide`, `fsevents`, `sharp`, `unrs-resolver`. npm warns; install still succeeds and the app runs.

## Real login does not work on localhost (cookie domain)

The Sentinel session cookie is scoped to `.interviewkickstart.com`, so browsers will not store or send it on `localhost`. Local FE therefore cannot log in for real, whichever BE it points at. For local UI checks only, the "logged-in" state can be seeded in browser storage (`auth_state`, `auth_user`, `auth_api_called`) so the provider treats the session as authenticated with no backend call. Verify full flows on a deployed `*.interviewkickstart.com` environment. Seeded values are not secrets; never put a real token in them.

## FE uses no-user-segment API paths

FE calls `/interview/{id}/...` (rating, report, feedback-status) without the `/user/{user_id}` segment that the BE code routes use. QA accepts the FE shape. A raw local BE does not, unless it has the short-path alias added in INV-1157. See [`../../integrations/fe-be-contract.md`](../../integrations/fe-be-contract.md).

## Fully client-rendered

Every page is `'use client'`. No server fetching, no server actions, no `app/api`. SEO + first-byte perf implications. Don't try to add a server component expecting data fetching to "just work" there without wiring.

---

## Open questions for manager (unresolved)

- Tracked `.env.*` with secrets — intentional? Rotation plan?
- Node version for CI/prod.
- LiveKit secrets on the client — remove?
- `NEXT_PUBLIC_API_KEY` — what it is, where issued, why empty in dev/prod but set in `.env.local`.
- CI location + merge gate.
- Testing policy (Vitest expected?).
- Mobile support plans.
- npm audit vulnerabilities — fix or accept?
