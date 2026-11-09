# Release flow (FE + BE)

Applies to `ik-ai-mock-interview-fe` and `ik-ai-mock-interview-be`. Confirmed by the manager on 2026-11-09. This page supersedes the branch notes in each repo's `conventions.md`.

## Branches

| Branch | Role |
|---|---|
| `production` | **Live.** The only branch that defines what users run. |
| `qa` | Integration and QA environment. Feature PRs merge here first. |
| `main` | **Not used as a release target for these two repos.** Ignore it. See the open contradiction below. |

`qa` and `production` have **diverged** and are not fast-forwardable.

## Flow

1. Create a feature branch named with the ticket key, based on `origin/qa`.
2. Raise a PR into `qa`. The user merges it.
3. The user runs the QA GitHub Action. BE: `build-qa` then `deploy-qa` (workflow_dispatch on `qa`). FE: `vercel-deploy` runs automatically on push to `qa`.
4. Test on https://qa-mi.interviewkickstart.com.
5. Once verified, raise the **final, detailed PR into `production`**. Its branch is based on `origin/production`.
6. A teammate reviews and approves.
7. Run the production deploy action. BE: `build-prod` then `deploy-production` (workflow_dispatch on `production`). FE: `vercel-deploy` on push to `production`.
8. Merge to `production`. Move the Jira ticket to Done.

## Branch-base rule

- QA PR branch: base on `origin/qa`.
- Production PR branch: base on `origin/production`.
- Never merge a `main`-based or `production`-based branch into `qa`. It drags unrelated commits and conflicts.
- To get a change into production, cherry-pick or re-apply it onto `origin/production`. Expect conflicts. For the FE, see the `apiConstants.ts` gotcha in [`../repos/ik-ai-mock-interview-fe/gotchas.md`](../repos/ik-ai-mock-interview-fe/gotchas.md).

## Checks and deploy behaviour

- **`automation-testing-gate`** is a required status check. "Expected — waiting for status to be reported" is normal. It clears when the automation-testing tool reports, usually around review time.
- **FE deploy:** `vercel-deploy`, auto on push to `qa` and `production`, also workflow_dispatch.
- **BE deploy:** ECS via the workflows in [`../repos/ik-ai-mock-interview-be/conventions.md`](../repos/ik-ai-mock-interview-be/conventions.md). Check the `MIGRATION_CUTOVER` variable before assuming a deploy is real.
- QA and production actions are **manually triggered** after review. Nothing deploys on merge alone for BE.

## Jira status mapping

| Status | When |
|---|---|
| In Progress | Work is active, before standup. |
| In Review | A teammate or manager is reviewing the production PR. |
| Done | Merged to `production` **and** live. |

Jira min-token policy still applies. See [`../CLAUDE.md`](../CLAUDE.md).

## Team and sprint (summary)

Full detail is in the Notion page below. Summary only:

- 2-week sprints, starting every other Tuesday.
- Story points: 1 SP is roughly 1 day, counted at quarter end toward rating.
- Frontend: Prithvi, Ratul. Backend: Ratul.

## Reference

- Notion: "Interview Kickstart context" — https://app.notion.com/p/Interview-Kickstart-context-410b9157a2e78035a049c4cc6b1e845d (sprint cadence, story points, team, Jira lifecycle, branching rules). Link only, not copied here.

## Open contradiction (not resolved)

- The Notion page's step 6 says "Merge the PR into `main`" and "ticket done". The manager said `production` is the live branch and to ignore `main`.
- This page follows the manager. A human should reconcile the Notion page.
- Source: [`../inbox/2026-11-09-release-flow-and-inv-1156-1157.md`](../inbox/2026-11-09-release-flow-and-inv-1156-1157.md).

## Related

- [`../repos/ik-ai-mock-interview-fe/conventions.md`](../repos/ik-ai-mock-interview-fe/conventions.md)
- [`../repos/ik-ai-mock-interview-be/conventions.md`](../repos/ik-ai-mock-interview-be/conventions.md)
- [`../tickets/INV-1156.md`](../tickets/INV-1156.md), [`../tickets/INV-1157.md`](../tickets/INV-1157.md)
