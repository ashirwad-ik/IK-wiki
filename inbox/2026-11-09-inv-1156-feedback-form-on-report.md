# 2026-11-09 — INV-1156 feedback form on report (FE)

**Repo:** `ik-ai-mock-interview-fe` · **Branch:** `INV-1156-feedback-form-on-report` · **Commit:** cec2845
**PR:** https://github.com/ikickstart/ik-ai-mock-interview-fe/pull/17 (raised)
**Related:** INV-1157 (BE, `feedback-status` GET endpoint) — PR under review.

## What changed

- The interview feedback form (rating + comment) moved from the interview session surface to the report page, shown as a popup over a blurred report. The report is revealed only after submit.
- Reused the existing `FeedbackModalWithRating` and the existing submit POST (`useSubmitRatingApi`). No new form.
- Normal interview end now routes straight to `/report/{roomId}`. The feedback modal was removed from the session surface for the normal flow.
- Invite flow (`isInviteFlow`) unchanged: still shows on-session feedback and thank-you.
- New hook `useGetFeedbackStatusApi` (GET feedback-status) gates the report. Fails open (shows the form) if status cannot be determined.
- Files: `src/constants/apiConstants.ts` (FEEDBACK_STATUS), `src/apis/useGetFeedbackStatusApi.ts` (new), `src/app/report/[id]/page.tsx` (gate), `src/components/modals/FeedbackModalWithRating.tsx` (optional `onFeedbackSubmitted` and `clearSession`), `src/components/livekit/VideoConferenceComponent/VideoConferenceComponent.tsx` and `src/components/livekit/CustomControlBar/CustomControlBar.tsx` (route to report on normal end).

## Gotchas learned

1. **FE and BE URL shape mismatch.** FE calls MI endpoints without a user segment (`/ai_mock_interview/interview/{id}/rating`, `.../report`, `.../feedback-status`). The BE code routes for these are under `/user/{user_id}/interview/{id}/...`. The deployed QA backend accepts the no-user shape (probably a gateway rewrite or older routing), so FE works on QA but not against a raw local BE. INV-1157 adds a short-path alias on the BE (`/interview/{interview_id}/feedback-status`, identity from the auth cookie, user segment ignored). Harmless in prod.
2. **Local full-stack login is blocked by cookie domain.** The Sentinel session cookie is scoped to `.interviewkickstart.com`, so it cannot be stored or sent on `localhost`. Real login does not work on a local FE, whichever BE it points at. For local UI checks, the logged-in state can be seeded in browser storage (`auth_state`, `auth_user`, `auth_api_called` keys) so the provider treats the session as authenticated without a backend call. True end-to-end verification needs a deployed `*.interviewkickstart.com` environment.
3. **gh CLI account selection.** `GH_ACCOUNT=...` is silently ignored by the installed `gh`. `gh auth switch` is blocked (it changes global state). Working per-call method: `GH_TOKEN="$(gh auth token --user <account>)" gh ...`. Global agent instructions were corrected to this.

## Follow-ups

- Confirm the INV-1157 BE alias before relying on the local BE for report gating.
- Confirm the FE report gate behaviour on QA once INV-1157 is deployed.
