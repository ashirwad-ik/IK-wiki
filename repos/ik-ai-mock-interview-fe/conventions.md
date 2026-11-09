# ik-ai-mock-interview-fe — conventions

## Commits

**Enforced by `.husky/commit-msg`** with the regex `^[A-Z]+-[0-9]+: .{1,50}`.

- Format: **`TICKET-ID: <imperative subject>`**, max 50 chars.
- Seen in history: `INV-...`, `CP-...`.
- `commitlint.config.js` extends `@commitlint/config-conventional`, **but no hook invokes commitlint** — the ticket-ID regex is the real rule. Don't be misled into writing `feat:` / `fix:` — it'll be accepted locally but isn't the house style.

**Safe default:** `INV-123: cache launch response in react-query`.

## Pre-commit

`.husky/pre-commit` runs `lint-staged`:
- `eslint --fix` on staged JS/TS files.
- `prettier --write` on staged files.

Don't disable husky. Install before first commit if needed (`npm install` wires it up).

## Branches

Release flow is in [`../../workflows/release-flow.md`](../../workflows/release-flow.md). Summary (confirmed 2026-11-09):

- `production` — **live**. The target of the final release PR.
- `qa` — integration and QA environment. Feature PRs merge here first.
- `main` — **not used as a release target** for this repo. Ignore it. (Open contradiction with the Notion context page; see the workflows page.)
- Feature branches: one per ticket, named with the ticket key. Base the QA branch on `origin/qa` and the production branch on `origin/production`. The two have diverged.
- Merge history is GitLab-style (migrated).

## CI and deploy

- Deploy is **Vercel** (`vercel-deploy`): auto on push to `qa` and `production`, also workflow_dispatch.
- `automation-testing-gate` is a required check. "Expected — waiting for status" is normal until the tool reports.
- No `.github/workflows/` in-repo for FE. Gating is in GitHub settings and Vercel.
- **Open question for manager:** does `vercel-deploy` live in the org's shared workflows, and who owns `automation-testing-gate`?

## Tooling

| Tool | Purpose |
|---|---|
| ESLint | `eslint.config.mjs` (flat config, Next-aware). `npm run lint` / `npm run lint:fix`. |
| Prettier | `.prettierrc.json` + `.prettierrc.mjs`, `.prettierignore`. `npm run format`. |
| TypeScript | Strict. `npx tsc --noEmit` — no script shortcut. |
| Husky | Commit-msg regex + pre-commit lint-staged. |

## Scripts (from `package.json`)

- `dev` — Next dev server (port 3000).
- `build` — production build.
- `start` — production server.
- `lint` / `lint:fix` — ESLint.
- `format` — Prettier.

## PR preparation checklist

- [ ] Commit matches `TICKET-ID: subject` (husky will reject otherwise).
- [ ] `npm run lint` clean (warnings OK, errors not).
- [ ] `npx tsc --noEmit` clean.
- [ ] `npm run build` succeeds locally.
- [ ] No new secret pasted into `.env.*` (those files are tracked — see [`gotchas.md`](gotchas.md)).
- [ ] If backend contract changed, update [`../../integrations/fe-be-contract.md`](../../integrations/fe-be-contract.md).

## GitHub CLI account (applies to all IK repos)

Use the matching account per call; do not switch the global `gh` account.

- `GH_ACCOUNT=...` is silently ignored by the installed `gh`. Do not rely on it.
- `gh auth switch` changes global state. Do not use it.
- Working form: `GH_TOKEN="$(gh auth token --user ashirwad-ik)" gh pr create ...` (single call, no persistence).
