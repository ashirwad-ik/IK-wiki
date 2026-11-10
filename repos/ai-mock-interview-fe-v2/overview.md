# ai-mock-interview-fe-v2 — overview

AI Mock Interview **frontend V2**. This is the live FE repo. The older `ik-ai-mock-interview-fe` is V1 (see [`../ik-ai-mock-interview-fe/overview.md`](../ik-ai-mock-interview-fe/overview.md)); do not target it for new work.

- **Remote:** `ikickstart/ai-mock-interview-fe-v2`
- **Branches:** `main`, `qa`. No `production` branch.
- **Deploy:** `.github/workflows/vercel-deploy.yml` on `[qa, main]`. `main` deploys to production (`--prod`); `qa` deploys a preview.
- **QA URL:** https://qa-mi-v2.interviewkickstart.com
- **Release target:** PR to `qa` first, then final PR to **`main`**. See [`../../workflows/release-flow.md`](../../workflows/release-flow.md).
- **Stack:** Next.js App Router, shadcn/ui (Dialog, Button, Textarea), @tanstack/react-query. `InterviewProvider` is in the root layout.
- Endpoints live in `buildEndpoints(brand)` in `src/constants/apiConstants.ts`.

## Feedback gate on the report (INV-1156)

- `/report/[id]` is gated by `useGetFeedbackStatusApi` (GET feedback-status). Until feedback is submitted the report is blurred (`blur-[2px]`, light on purpose) behind `FeedbackRatingDialog`.
- `FeedbackRatingDialog` (`src/components/interview/room-dialogs.tsx`) takes `onSubmitted`; in gate mode it is non-dismissible and a successful submit reveals the report in place.
- `interview-room.tsx` `onInterviewEnd` goes straight to `/report/{id}` for normal candidates. Invite/recruiter flow (`isInviteFlow`) keeps the in-room card.
- While the status loads, the report stays blurred (prevents a flash). Submit sets the query cache optimistically and writes `localStorage` key `aimi-feedback-submitted:{interviewId}` so the gate never re-nags.
- Files: `src/app/(app)/report/[id]/page.tsx`, `room-dialogs.tsx`, `interview-room.tsx`, `src/apis/useGetFeedbackStatusApi.ts`, `apiConstants.ts`, `e2e/app.spec.ts`, `e2e/fixtures.ts`.

## Gotchas

- **Gate reappearing is backend-driven.** If `feedback-status` is not deployed to the environment, the GET reports "not submitted" and the gate returns on reload. Deploy the BE first.
- **e2e fixtures must mock `/feedback-status`.** Unmocked it returns 401 and the app logs out. `signIn` in `e2e/fixtures.ts` mocks it (default submitted). Desktop 40 pass / 1 skip, mobile 2 pass.
- Gate behaviour of `automation-testing-gate` on the production PR: see [`../../workflows/release-flow.md`](../../workflows/release-flow.md).

Ticket: [`../../tickets/INV-1156.md`](../../tickets/INV-1156.md). BE counterpart: [`../../tickets/INV-1157.md`](../../tickets/INV-1157.md).

## Redesign pending (2026-11-10)

The PM is completely redesigning the frontend; new artifacts incoming. FE pivots to a new PR, so the current report gate and PR #8 may be superseded. Local testing uses email-ID login (no Google auth).
