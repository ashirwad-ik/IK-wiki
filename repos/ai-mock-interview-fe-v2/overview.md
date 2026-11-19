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

## Exit surfaces (INV-1183 / 1185 / 1186, 2026-11-16)

Three exit paths, one surface each. They don't stack.

| Trigger | Surface | Hook |
|---|---|---|
| Tab close / refresh / Ctrl+W | Native "Leave site?" prompt only | `useTerminateOnUnload` (INV-1183) |
| Tab-switch / window-blur / cursor-to-tab-bar | In-app "Before you go" card, once per interview | `useFocusLeaveOnce` (INV-1185) |
| In-room End button below 25% elapsed | End-click confirmation card | INV-1186 |

LiveKit: `<LiveKitRoom options={{ disconnectOnPageLeave: false }}>` — agent stays connected across the native prompt; Cancel resumes the interview with no in-app card and no mic-recovery hop.

Key hooks: `useTerminateOnUnload` (beforeunload + pagehide ABORTED patch), `useFocusLeaveOnce` (visibilitychange + mouseleave, **no `window.blur`** — Chrome fires blur on native prompt). End-click card in `room-dialogs.tsx` (`sm:max-w-3xl`). Report-unlock copy: "11:15 of 45:00" = elapsed interview time, not wall-clock.

**Mic cascade (BE/worker side) — still unresolved.** See [`../../services/livekit.md`](../../services/livekit.md#stay--hitch-cascade-unresolved-2026-11-13).

**INV-1184** is a duplicate of INV-1186 (end-click card); close as such.

Tickets: [`../../tickets/INV-1183.md`](../../tickets/INV-1183.md) · [`../../tickets/INV-1185.md`](../../tickets/INV-1185.md) · [`../../tickets/INV-1186.md`](../../tickets/INV-1186.md).

## Redesign pending (2026-11-10)

The PM is completely redesigning the frontend; new artifacts incoming. FE pivots to a new PR, so the current report gate and PR #8 may be superseded. Local testing uses email-ID login (no Google auth).

## Artifact polish (2026-11-19, qa)

INV-1201 PresenceCard and INV-1186 BeforeYouGo now match the artifact literally; INV-1202 no-report loader fixed at the API source.

- **PresenceCard**: hand icon, 3-strike idle check (60/120/180s, auto-end 240s), flat surface (no DialogFooter), transparent click-blocker with Radix `modal={false}`.
- **useIdle**: `IDLE_CHECK_INTERVAL_MS = 60_000`, `isFinal` is `>` not `>=` so Check 3 shows rescue button.
- **No-report fix**: `useGetReportByIdApi` treats `NOT_STARTED` as terminal → no 4-min PENDING poll. Earlier history-cache short-circuit reverted.
- **BeforeYouGoDialog colors**: brown countdown digits, translucent blue "How to get there" panel, per-tip cards.

Push cadence: direct to `qa` only; qa→main PR deferred until signed off. Push only when user says so.
