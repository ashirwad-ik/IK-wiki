# 2026-11-10 — INV-1157 BE QA-verified; FE redesign pending

## State
- **BE PR #32** (`ikickstart/ik-ai-mock-interview-be`, `INV-1157-feedback-status-production` → `production`): OPEN, QA-verified, awaiting manager merge + `deploy-production`. Contains short-path endpoint + `login_source` kwarg fix + `Depends(get_service(...))` no-lambda fix.
  - Endpoint: `GET /ai_mock_interview/interview/{interview_id}/feedback-status` → `{ "is_feedback_submitted": bool }`.
  - `automation-testing-gate` sits at "Expected — waiting" on production (external QA E2E posts only in QA flow, SHA-scoped). Manager merges with admin override.
- **FE PR #8** (`ikickstart/ai-mock-interview-fe-v2` → `main`): OPEN, but PM is **completely redesigning** the frontend. New FE work will go on a **new PR** per the PM's design once artifacts arrive; #8 may be superseded.

## Rules reinforced
- Production: never create a new production branch/PR — always push to the single existing open production PR (#32).
- QA: reuse the current open QA branch/PR; only create a new one once the previous QA PR is merged.
- User runs all GitHub Actions and merges; Claude only raises/updates PRs.
- Sentry check: only after `deploy-qa` + after hitting the endpoint, not at build.

## Next
Waiting on PM's FE redesign artifact before any FE work begins.
