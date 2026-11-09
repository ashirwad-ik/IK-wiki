# 2026-11-09 — Backend QA deploy fires automation-testing-gate (build-qa vs deploy-qa)

**Repos touched:** `ik-ai-mock-interview-be` (workflows), `ik-ai-mock-interview-fe` (vercel-deploy workflow). Verified from the workflow YAML in both repos and the GitHub run history, not inferred.
**Tickets:** INV-1157 (BE QA flow), INV-1156 (FE, for the gate note).

## Key facts

- **BE has four manual (`workflow_dispatch`) workflows, split by job:**
  - `build-qa` (ref `qa`): builds and pushes the image to ECR only. Deploys nothing, tests nothing.
  - `deploy-qa` (ref `qa`): updates QA ECS. Its second job `e2e` triggers the QA team's post-deploy tests through a GitHub App.
  - `build-prod` (ref `production`): builds and pushes the production image.
  - `deploy-production` (ref `production`): updates production ECS. **No production workflow triggers the gate.**
- **`automation-testing-gate` is fired only by a real `deploy-qa` run**, via its `e2e` job. `build-qa` alone never fires it.
- **Shadow mode also does not fire it.** The `e2e` job runs only when the deploy step really ran (`deployed == 'true'`, which requires the `MIGRATION_CUTOVER` variable to be `true`). In shadow mode the gate stays unreported.
- **FE has an in-repo `vercel-deploy.yml`** (not org-shared). It triggers on push to `qa` and `production` and on dispatch. It has the same shape: a `deploy` job (Vercel build and deploy, gated by `MIGRATION_CUTOVER`) and an `e2e` job that fires the same gate. On `production` the e2e job runs only the QA team's smoke check.
- **Gate owner:** the test-automation repo runs `post-deploy-e2e.yml`. The workflow comments attribute the plan to Dipen ("Post-Deploy E2E Gate Plan"). The trigger GitHub App (`AUTOMATION_TESTING_TRIGGER_APP_ID`) can start that one workflow and nothing else.

## Actionable lessons

1. "Expected — waiting for status to be reported" on BE means the gate has not been fired. Usually a `deploy-qa` run has not happened, or it ran in shadow mode.
2. Backend release order: **build-qa → deploy-qa** (QA, fires gate) → review and approval → **build-prod → deploy-production** (production). Run history from 2026-11-07 matches this.
3. Frontend needs no manual deploy step. `vercel-deploy` runs on push to `qa` and `production`.

## Open / unverified (flag, do not assert)

- The gate reports on the **deployed** commit SHA. The production PR branch is a separate cherry-picked commit (different SHA, because `qa` and `production` diverged). It is unconfirmed whether the BE production PR's required gate clears from the QA e2e run or needs something else. Confirm with the gate owner (Dipen). The FE is not affected the same way, because its production push fires the gate.

## Related

- [`../workflows/release-flow.md`](../workflows/release-flow.md)
- [`../repos/ik-ai-mock-interview-be/conventions.md`](../repos/ik-ai-mock-interview-be/conventions.md)
- [`../repos/ik-ai-mock-interview-fe/conventions.md`](../repos/ik-ai-mock-interview-fe/conventions.md)
