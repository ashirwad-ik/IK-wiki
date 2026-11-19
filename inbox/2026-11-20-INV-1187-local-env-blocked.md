# INV-1187 / PDC-490 — hint feature: local-env session ended, blockers for tomorrow

**Date:** 2026-11-20
**Branches:** `feat/INV-1187-hint-feature` on all three repos. **Nothing pushed.**
**Status:** code written & smoke-tested in isolation; cannot end-to-end locally until blockers below cleared.

## What landed (all local, uncommitted to remote)

- **ik-ai-mock-interview-be** (commit `0bcd8c8`, rebased on fresh `origin/main`):
  - Migration `migrations/versions/2026_11_19_1200_add_interview_hints_table.py` (rev `017_add_interview_hints`, down_rev `016_add_interview_questions`). Applied + validated against local docker-compose MySQL.
  - Table `ai_mock_interview_hints (id, interview_id, question_index, tier, hint_id UNIQUE, text, created_at)`.
  - `routers/interview_hints_router.py`: `POST /api/v1/ai_mock_interview/interview/{id}/hint` (worker → BE, idempotent on `hint_id`, Behavioural rejects WALKTHROUGH), `GET /hints/summary` (FE report page).
  - `services/interview/hint_service.py` carries the PDC-490 penalty table (DSA/Backend/ML/Behavioural tier→score-dim mapping).
  - FE pulls `/hints/summary` separately; the DynamoDB report-write path is intentionally untouched.
- **uplevel-livekit-worker** (commit `e710801`):
  - RPC `request_hint` registered on the agent participant (`services/hint_service.py`).
  - Tier decision from live context (transcript tail, code buffer, prior hints) with Behavioural cap at APPROACH and soft 12-press circuit breaker.
  - Langfuse prompt `mock-interview-hint-v1` with a `HINT_PROMPT_FALLBACK` constant in `prompts.py` if Langfuse is unreachable. The 2744-line `INSTRUCTION_PROMPT` was NOT rewritten — the "NO GRADUATED HINTING LEVELS" block replaced with a 3-line pointer to the handler.
  - Posts to BE via `X-Internal-Api-Key: CENTRAL_AUTH_INTERNAL_API_KEY`.
- **ai-mock-interview-fe-v2** (commit `d2e34ca`, plus `fd48835` test-mode later stripped by `INV-1187: remove local test-mode scaffolding`):
  - `useAgentRequestHintRpc` hook performs the RPC; adds `{moduleHint, questionIndex}` + `escalateHint`.
  - `HintCard` (inline, bottom-left of the interview room, matches the PDC-490 mockup — "Hint N · Tier" title, orange tier dots, info line, "Still stuck? NextTier" secondary CTA, "Got it" dismiss).
  - Dock Hint button with per-question numeric badge; ladder resets on question change.
  - Report page wires `hints_used` from `/hints/summary`.
  - Analytics event `hint_requested` + testid contract updates.

## Known contract mismatches to fix before QA push

1. Worker POSTs to `/api/v1/interview/{id}/hint` but BE mounts under `/api/v1/ai_mock_interview/interview/{id}/hint`. Patched locally on disk, not committed yet on the worker branch.
2. FE expects `hints_used` on the main report response; BE only exposes it at `/hints/summary`. Pick a side before QA — either add a `useGetHintsSummary` hook in FE or merge the two fields into the main report payload.
3. The real `interview-room.tsx` hard-codes `moduleHint='DSA'`; the worker must publish the module on the agent participant's attributes before Behavioural rounds render the correct copy.

## What was stripped at session end (local-only scaffolding removed)

- FE `src/__mocks__/hint-mocks.ts`, `src/app/hint-test/*` (standalone harness), the `NEXT_PUBLIC_HINT_TEST_MODE` branch in the hook, the env flag itself.
- `.env.local` reverted to original QA-pointing values.
- Docker containers `uplevel_mysql` + `uplevel_localstack` removed (no volumes, data was throwaway).
- Local MySQL sandbox seed (`sandbox_local_ashirwad` user + SANDBOX role row) — died with the container.
- BE and worker commits preserved; nothing deleted from the feature branches.

## Blockers that prevented real end-to-end locally (today's session)

1. **Google OAuth redirect allowlist** does not include `localhost:3000` — Google button on the login page fails. Only `*.interviewkickstart.com` is allowed in Google Cloud Console.
2. **Auth cookies are domain-scoped to `.interviewkickstart.com`** — even once signed in on a brand hostname, the browser won't attach them to `localhost:8000` calls. So FE on `local.interviewkickstart.com` can't talk to local BE without extra proxy wiring.
3. **mkcert + hosts file + `npm run proxy` setup was not completed** (needed for the real login path).
4. **QA LiveKit dispatch** uses default agent name `mock-interview-agent`. Local worker ran under `mock-interview-agent-local-ashirwad` for isolation, so a QA FE interview never reaches the local worker — meaning full QA UI can't exercise local hint code without also patching FE's dispatch call. Staging LiveKit credentials from the existing BE `.env` returned 401; fell back to QA LiveKit keys with name isolation.
5. **AWS writes** (S3 transcripts, DynamoDB report) — redirected to LocalStack during the session via `AWS_ENDPOINT_URL=http://localhost:4566` and dummy `test/test` creds. Works without code changes because boto3 ≥ 1.28 reads that env natively. Containers now torn down.
6. **Sandbox auth works** but shows the candidate/invite UI, not the full dashboard — not acceptable for "QA-like" testing per Ashirwad's feedback.

## What's needed to resume tomorrow (asks for manager / DevOps)

- **Google OAuth** add `https://local.interviewkickstart.com` (and `https://local.dexity.com`) to the OAuth redirect URI allowlist in Google Cloud Console, AND to the Cognito app client's callback URL list. Without this, no amount of local proxying gets Google login working.
- **Cognito app client callback URLs**: same — add the local brand URLs.
- **Secrets Ashirwad still needs:**
  - `OPENAI_API_KEY` for the worker (none provided in the `.env-main` DevOps shared).
  - Confirmation that the shared `LANGFUSE_PUBLIC_KEY` / `LANGFUSE_SECRET_KEY` (US region) are OK for read-only prompt fetches during local dev, and that writing to the Langfuse prompt registry is out-of-bounds. Confirmed intent today: fetch-only.
  - Staging LiveKit URL + keys that actually work — the pair in the existing BE `.env` (`API26uN6n6ND42m` on `ai-mock-interview-staging-wabyyhm7.livekit.cloud`) returned 401 against LiveKit Cloud today. Either staging project rotated keys or the URL changed.
- **QA test Cognito user with email+password** (not Google-only) — would unblock local login without needing OAuth redirect changes.
- **QA LiveKit dispatch mechanism — explicit vs auto**: confirm that if Ashirwad changes local worker `AGENT_NAME` back to `mock-interview-agent` to receive QA dispatches, the QA worker and his local worker would race for the same room (bad), or whether per-room explicit dispatch means only one gets the room.
- **Deployment of the worker to QA** — the `uplevel-livekit-worker` repo has no GitHub Actions deploy workflow visible. DevOps please confirm how the worker ships to QA.
- **MIGRATION_CUTOVER gating** — who flips `vars.MIGRATION_CUTOVER` in GitHub Actions, and when, for the INV-1187 migration to run on QA.
- **Who runs the migration on QA DB** — Ashirwad has no RDS access. Confirm handoff process + notice period.

## Schema / migration push plan (for DevOps alignment)

Migration `017_add_interview_hints` is additive-only: new table, no column changes, no backfill. Safe against concurrent writes on QA RDS. Local Docker MySQL 8.4 validated the DDL via `scripts/migration_helper.py migrate`. On QA:

1. PR merges to `qa` branch with `MIGRATION_CUTOVER` NOT yet set (just deploys the code).
2. DevOps runs `python scripts/migration_helper.py migrate` against QA MySQL (needs AWS access to the ECS task / jump host).
3. Verify `017_add_interview_hints` is current head via `python scripts/migration_helper.py status`.
4. Flip `MIGRATION_CUTOVER=true` or re-deploy once the migration completes, so the new endpoint is actually usable.
5. Worker needs its own deploy pass (unknown mechanism — DevOps).

**Open question:** should hint-affected scoring apply retroactively to existing completed interviews, or only to interviews started after the feature ships? Ashirwad's vote: only new interviews. Needs a one-line PM confirm.

## Linked

- Jira: PDC-490 (PRD) / INV-1187 (Ashirwad's dev ticket)
- Related standing rule: never auto-push IK code repos without explicit per-task authorization ([[CLAUDE.md]])
