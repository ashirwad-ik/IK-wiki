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

## Stay flow + mic cascade (INV-1184 / 1185 / 1186, 2026-11-13)

Tab-close flow: `beforeunload` shows Chrome's native "Leave site?" prompt; on Cancel, the in-app "Before you go" card opens (queued via `setTimeout` in `beforeunload`, cleared by `pagehide`). On real leave, `pagehide` PATCHes `ABORTED` with keepalive and sets `localStorage.aimi-abandoned` for the next-visit feedback card (INV-1183).

**Shipped on the QA PR (today):**
- `src/hooks/useTerminateOnUnload.ts` — guard on `event.persisted` in `pagehide`. Chrome puts the page into bfcache while the native prompt is shown; the legacy handler PATCHed `ABORTED` on bfcache entry, killing live interviews whenever Chrome touched bfcache.
- `src/components/interview/stage.tsx` — `LocalTile` uses `useLocalParticipant().isMicrophoneEnabled` (reactive) instead of `participant.isMicrophoneEnabled` (static read). Mic icon now re-renders when the track mutes.
- `src/components/interview/room-dialogs.tsx` — end-click variant narrowed to `sm:max-w-3xl` with full-width countdown + 2-col body (miss | how-to). Tab-close keeps `sm:max-w-5xl` 3-col with trophy card.
- Report-unlock copy: "11:15 of 45:00" is **elapsed interview time** (artifact: quarter of 45-min duration). Not wall-clock.

**Unresolved (FE can't fix alone) — mic after Stay:**

During the native "Leave site?" prompt Chrome mutes the local `MediaStreamTrack` at the browser level and briefly hitches WebRTC. Four FE strategies tried:
1. `setMicrophoneEnabled(true)` — LiveKit `unmute()` no-ops against the browser-level mute.
2. `LocalAudioTrack.restartTrack()` — races with LiveKit's cached publication.
3. `unpublishTrack(track, stopOnUnpublish=true)` + `setMicrophoneEnabled(true)` — leaves `useTrackToggle` in `pending: true`, control-bar mic button becomes unclickable permanently.
4. `setMicrophoneEnabled(false)` → `setMicrophoneEnabled(true)` (non-destructive toggle, current code in `interview-room.tsx` `onStay`) — button stays clickable but mic still doesn't recover.

Worse, the hitch cascades on the BE and often ABORTs the interview before the user even clicks Stay — see [`../ik-ai-mock-interview-be/gotchas.md`](../ik-ai-mock-interview-be/gotchas.md) → "LiveKit empty_timeout + room_finished cascade" and [`../../services/livekit.md`](../../services/livekit.md) → "Stay / hitch cascade (unresolved)". Proper fix is two-part (BE empty_timeout + webhook guard, AND agent grace period in `uplevel-livekit-worker`). Waiting on manager alignment before shipping Part 1 standalone.

## Redesign pending (2026-11-10)

The PM is completely redesigning the frontend; new artifacts incoming. FE pivots to a new PR, so the current report gate and PR #8 may be superseded. Local testing uses email-ID login (no Google auth).
