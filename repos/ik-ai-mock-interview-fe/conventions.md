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

- Workflow file is **in-repo**: `.github/workflows/vercel-deploy.yml`. It is not org-shared. (An earlier note said there were no workflows in-repo; that was wrong, corrected 2026-11-09.)
- Triggers: auto on push to `qa` and `production`, plus workflow_dispatch. The `production` branch maps to `--prod`.
- Jobs: `deploy` (Vercel pull, build, deploy) and `e2e`. The Vercel deploy step runs only when `MIGRATION_CUTOVER` is `true`; otherwise it is shadow mode and nothing deploys.
- `e2e` fires the `automation-testing-gate` check through the test-automation repo's `post-deploy-e2e.yml`, using a GitHub App (`AUTOMATION_TESTING_TRIGGER_APP_ID`). It runs only after a real deploy. On `qa` it runs the full QA suite. On `production` it runs only the QA team's smoke check.
- "Expected — waiting for status" on an FE PR means the gate has not been reported yet. Usually no real deploy has run for that commit.
- **Answered 2026-11-09:** `automation-testing-gate` is produced by the test-automation repo. Its trigger is the `e2e` job in each repo's deploy workflow, and the plan is attributed to Dipen in the workflow comments. Still unconfirmed: whether a production PR's gate on a cherry-picked SHA clears from the QA run. Same open flag as the BE; see [`../../workflows/release-flow.md`](../../workflows/release-flow.md).

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
