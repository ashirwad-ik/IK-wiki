# 2026-11-09 — QA-first release flow, branch-base rule, INV-1156 / INV-1157 status

**Repos touched:** `ik-ai-mock-interview-fe` (INV-1156), `ik-ai-mock-interview-be` (INV-1157)
**Tickets:** INV-1156, INV-1157 — both now In Review (production PRs open).

## Key facts

- **Release flow (confirmed by manager, 2026-11-09):** feature branch → PR into `qa` → user merges → user runs QA action → test on https://qa-mi.interviewkickstart.com → final detailed PR into `production` → teammate review/approve → production deploy action → merge to `production` → Jira Done.
- **`production` is the live branch.** `main` is not used as a release target for these two repos.
- **`qa` and `production` have diverged** (not fast-forwardable). QA PR branch must be based on `origin/qa`; production PR branch on `origin/production`. A `main`- or `production`-based branch merged into `qa` drags unrelated commits and conflicts.
- **PR state:** INV-1156 FE QA PR #19 merged to `qa`; production PR #18 open. INV-1157 BE QA PR #28 merged to `qa`; production PR #27 open. User verified both on the QA URL.
- Earlier mis-targeted PRs (FE #17, BE #25, plus first attempts at the production PRs) were closed during cleanup.

## Decisions

- Document the QA-first flow and branch-base rule in a new workflow page (`workflows/release-flow.md`) rather than in each repo's conventions; both repos link to it.
- Jira mapping: In Progress before standup; In Review while a teammate/manager reviews the production PR; Done only after merge to production and live.

## Gotchas

1. **FE `src/constants/apiConstants.ts` cherry-pick conflict (recurring).** production/qa use `buildEndpoints(brand)` with a `get ENDPOINTS()` getter. Older feature commits carry a flat `const ENDPOINTS = {...}` block that references module-level `BASE_URL` / `RECRUITER_BASE_URL`, which no longer exist. Add new constants inside `buildEndpoints`, drop the flat block, then run `npx tsc --noEmit`.
2. **`automation-testing-gate` status check** shows "Expected — waiting for status to be reported". Not an error. It blocks the merge button until the automation-testing tool reports, around review time.
3. **FE deploy = `vercel-deploy`** (auto on push to `qa` / `production`, also workflow_dispatch). **BE deploy** = `build-qa` / `deploy-qa` and `build-prod` / `deploy-production` (workflow_dispatch, ECS). Both QA and prod actions are manually triggered after review.
4. **gh account:** per-call `GH_TOKEN="$(gh auth token --user ashirwad-ik)" gh ...`. Never `gh auth switch` or `GH_ACCOUNT`.

## Contradiction (NOT resolved, needs a human)

The Notion page "Interview Kickstart context" (https://app.notion.com/p/Interview-Kickstart-context-410b9157a2e78035a049c4cc6b1e845d), step 6, says "Merge the PR into `main`" and "ticket done". The manager verbally said `production` is the live branch and to ignore `main`. The wiki follows the manager. The Notion page is linked only, not copied. Someone should reconcile the Notion page with the manager's guidance.

## Flags for review

- PR numbering in the session notes is ambiguous: notes say "FE #18-first-attempt" and "BE #27-first" were closed, yet FE #18 and BE #27 are the open production PRs. Numbers are unique per repo, so one label is wrong. Confirm on GitHub before relying on the "first attempt" labels.
- Jira status was taken from the session notes, not fetched from Jira in this cycle (minimum-token policy).
- The FE route-shape mismatch and the local-login cookie limit from the earlier 2026-11-09 digest still stand.
