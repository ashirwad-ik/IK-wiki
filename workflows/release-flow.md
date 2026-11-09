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

## FE v2 variant (`ai-mock-interview-fe-v2`)

The live FE is `ikickstart/ai-mock-interview-fe-v2`, which has only `main` and `qa`. Same QA-first flow, but the final PR targets **`main`** (Vercel deploys `main` to production). QA URL: https://qa-mi-v2.interviewkickstart.com. The "ignore `main`" rule applies to the BE and the V1 FE only. See [`../repos/ai-mock-interview-fe-v2/overview.md`](../repos/ai-mock-interview-fe-v2/overview.md).

## Branch-base rule

- QA PR branch: base on `origin/qa`.
- Production PR branch: base on `origin/production`.
- Never merge a `main`-based or `production`-based branch into `qa`. It drags unrelated commits and conflicts.
- To get a change into production, cherry-pick or re-apply it onto `origin/production`. Expect conflicts. For the FE, see the `apiConstants.ts` gotcha in [`../repos/ik-ai-mock-interview-fe/gotchas.md`](../repos/ik-ai-mock-interview-fe/gotchas.md).

## Checks and deploy behaviour

- **`automation-testing-gate`** is a required status check, reported by the QA team's post-deploy tests (test-automation repo, `post-deploy-e2e.yml`). An `e2e` job in each repo's deploy workflow fires it, and only after a deploy that really ran (see `MIGRATION_CUTOVER` below). "Expected — waiting for status to be reported" means no real deploy has fired it yet. It is not an error. See "Backend CI workflows & the gate" below.
- **FE deploy:** `vercel-deploy` (in-repo, `.github/workflows/vercel-deploy.yml`), auto on push to `qa` and `production`, also workflow_dispatch. Its `e2e` job fires the gate on both branches.
- **BE deploy:** ECS via the workflows below and in [`../repos/ik-ai-mock-interview-be/conventions.md`](../repos/ik-ai-mock-interview-be/conventions.md).
- **`MIGRATION_CUTOVER`** (repo variable, both repos): when not `true`, deploy runs in shadow mode. No deploy happens, and the gate does not fire. A green dispatch is not proof of a real deploy.
- QA and production actions are **manually triggered** after review for BE. Nothing deploys on merge alone for BE.

### Backend CI workflows & the gate

Verified from `.github/workflows/` in `ik-ai-mock-interview-be` (2026-11-09).

| Workflow | Ref | Does | Fires the gate? |
|---|---|---|---|
| `build-qa` | `qa` | Build and push the image to ECR. | No |
| `deploy-qa` | `qa` | Update QA ECS. Its `e2e` job then starts the QA tests. | **Yes**, only if the deploy really ran |
| `build-prod` | `production` | Build and push the production image. | No |
| `deploy-production` | `production` | Update production ECS. | No |

Order for a backend release: **build-qa → deploy-qa → test on QA → review and approval → build-prod → deploy-production**.

Consequences:
- Running `build-qa` alone leaves the gate "Expected" forever. Run `deploy-qa` too.
- If `MIGRATION_CUTOVER` is not `true`, `deploy-qa` runs in shadow mode and the gate does not fire.
- No production workflow fires the gate. The production PR's gate depends on the QA run (see the flag below).

**Observed behaviour (INV-1156/1157, 2026-11-09):** `automation-testing-gate` is a required status posted by an external QA post-deploy E2E automation ("Gate Plan"), scoped to a **commit SHA** and only in the QA flow. On a **production PR it is never reported**, so it shows "Expected, waiting for status" forever. It is not a job and will not "run". A cherry-pick to production makes a new SHA, so a QA-green status does not carry over. A dev cannot satisfy it from the PR: needs admin merge/override or the gate owner.

**Open flag (partly answered above; not verified, confirm with the gate owner, Dipen):** the gate reports on the deployed commit SHA. The production PR branch is a separately cherry-picked commit with a different SHA. It is unconfirmed whether the production PR's required gate clears from the QA e2e result or needs a separate run.

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
