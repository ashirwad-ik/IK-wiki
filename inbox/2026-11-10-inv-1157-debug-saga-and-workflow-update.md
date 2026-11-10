# 2026-11-10 — INV-1157 debug saga, updated workflow, FE redesign

Repo touched: `ik-ai-mock-interview-be` (INV-1157), `ai-mock-interview-fe-v2` (INV-1156 context). Extends [2026-11-09-inv-1156-1157-feedback-gate-v2.md](2026-11-09-inv-1156-1157-feedback-gate-v2.md).

## State
- BE INV-1157 done and QA-verified. Final production PR **#32** (head `INV-1157-feedback-status-production` -> `production`) open, awaiting manager review, merge, `deploy-production`. Gate sits "Expected" on #32 by design; manager merges with admin override.
- Endpoint: `GET /ai_mock_interview/interview/{interview_id}/feedback-status` -> `{ "is_feedback_submitted": bool }`.
- FE INV-1156 shipped to qa; production PR #8 (`-> main`) open. **PM is completely redesigning the FE** (new artifacts incoming). FE pivots to a new PR; the current gate / #8 may be superseded.

## Three sequential bugs (all masked to client as `{"code":"UNEXPECTED_ERROR","message":"Service unavailable"}` / 503)
1. Wrong path `/user/{user_id}/...` -> 404. Fix: short path.
2. Handler lacked `login_source` kwarg; `@require_authentication()` injects it into every handler -> `TypeError: unexpected keyword argument 'login_source'`. Fix: `login_source: str = None`.
3. `Depends(lambda: get_service(Service))` made `service` the dependency function -> `AttributeError: 'function' object has no attribute 'get_feedback_status'`. Fix: `Depends(get_service(Service))`.
- Unrelated Sentry issue: `/auth/refresh` NotAuthorized (project `AI-MOCK-INTERVIEWER-2Z`), likely test-session expiry.

## Workflow strategy (user-confirmed 2026-11-10)
- QA-first. Reuse the current open QA branch/PR; new QA branch only if the previous QA PR is merged.
- Never create a new production branch/PR; push to the single open production PR (#32). Close stray PRs except production.
- Sentry check only after `deploy-qa` and after hitting the endpoint.
- Local FE testing: email-ID login, no Google auth.
- The user runs GitHub Actions and merges, not Claude.
