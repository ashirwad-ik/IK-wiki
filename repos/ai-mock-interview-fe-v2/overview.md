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

**Key files**
- `src/hooks/useTerminateOnUnload.ts` — `beforeunload` sets `returnValue` for the native prompt; `pagehide` (bfcache-guarded via `event.persisted`) persists `aimi-abandoned` and PATCHes `ABORTED` with `keepalive`. `popstate` swallowed. No card-queue plumbing anymore.
- `src/hooks/useFocusLeaveOnce.ts` (new) — fires once per mount on `visibilitychange → hidden` or `document.mouseleave` (`clientY ≤ 0`, debounced 450 ms). `beforeunload` + `mouseenter` cancel the pending mouseleave trigger. **`window.blur` is deliberately not listened to** (Chrome also fires blur when it opens the native prompt → would re-introduce the Cancel-shows-card regression).
- `src/components/interview/room-dialogs.tsx` — end-click variant `sm:max-w-3xl` with full-width countdown + 2-col body (miss | how-to); non-blocking mic restart on Keep interviewing; keyboard shortcuts suppressed while open.
- `src/components/interview/stage.tsx` — `LocalTile` uses `useLocalParticipant().isMicrophoneEnabled` (reactive), so the mic icon re-renders when the track mutes.
- Report-unlock copy: "11:15 of 45:00" is **elapsed interview time** (quarter of duration), not wall-clock.

**Mic cascade (BE/worker side) — still unresolved.** `disconnectOnPageLeave: false` removes the FE's contribution on the native-prompt path (room no longer torn down on Cancel), but the broader LiveKit disconnect → `empty_timeout` → `room_finished` → `ABORTED` path still exists for genuine disconnects. The 2026-11-13 two-part fix (BE `empty_timeout` + webhook guard, AND agent grace period in `uplevel-livekit-worker`) is still waiting on manager alignment. See [`../ik-ai-mock-interview-be/gotchas.md`](../ik-ai-mock-interview-be/gotchas.md) → "LiveKit empty_timeout + room_finished cascade" and [`../../services/livekit.md`](../../services/livekit.md) → "Stay / hitch cascade (unresolved)".

**INV-1184** is a duplicate of INV-1186 (end-click card); close as such.

Tickets: [`../../tickets/INV-1183.md`](../../tickets/INV-1183.md) · [`../../tickets/INV-1185.md`](../../tickets/INV-1185.md) · [`../../tickets/INV-1186.md`](../../tickets/INV-1186.md).

## Redesign pending (2026-11-10)

The PM is completely redesigning the frontend; new artifacts incoming. FE pivots to a new PR, so the current report gate and PR #8 may be superseded. Local testing uses email-ID login (no Google auth).

## Artifact polish (2026-11-19, qa)

INV-1201 PresenceCard and INV-1186 BeforeYouGo now match the artifact literally; INV-1202 no-report loader fixed at the API source.

- **PresenceCard** (`src/components/interview/room-dialogs.tsx`): hand icon, `Are you still there?` title, `Irene hasn't heard from you for N minute(s)…` body, `Check N of 3` label under progress bar, footer `Next check in 1 minute` (checks 1–2) / `Ending in 1 minute` (check 3), hand-icon + `I'm here` button. One flat surface — shared `DialogFooter` has `border-t` + `bg-muted/50` baked in, so PresenceCard uses a plain flex row instead. Transparent `fixed inset-0 z-40` click-blocker swallows pointer events behind it while keeping Radix `modal={false}` (no focus/scroll trap; agent/timer/LiveKit untouched).
- **useIdle** (`src/hooks/useIdle.ts`): single `IDLE_CHECK_INTERVAL_MS = 60_000`, three strikes at 60/120/180 s, auto-end at 240 s (**1 minute after the 3rd check**). `strike > IDLE_MAX_STRIKES` is the terminal post-onEnd state; `isFinal` is `>`, not `>=`, so Check 3 still shows the rescue button.
- **No-report loader fix at the hook, not the page** (`src/apis/useGetReportByIdApi.ts`): treat `NOT_STARTED` as terminal alongside `FAILED`. Report page falls through to `<NoAnalysis>` on the first response — no 4-min PENDING poll on aborted/no-report sessions, works on reload and deep links without depending on history cache being warm. Earlier history-cache short-circuit (commit `5a40c72`) was reverted.
- **`NoAnalysis` copy restored** to the PM-approved original ("No report for this session" / "ended too quickly"). No `reason`/`endReason` branching — off-brief.
- **BeforeYouGoDialog colors**: countdown digits `text-amber-900` (brown — matches chocolate-on-peach); "What you'll miss" items are `bg-background` + `border-border/60` (no muted fill); "How to get there" panel `bg-primary/10` + `border-primary/15` (light translucent blue, replacing the dark `bg-primary-soft`); each tip wrapped in its own `bg-background` + `border-primary/10` rounded card inside the panel.

Push cadence: direct pushes to `qa` only; qa→main release PR deferred until the full set is signed off. Auto-mode classifier blocks `git push origin qa` by default — a `permissions.allow` entry (`Bash(git push origin qa:*)`) is required to unblock, and the standing rule is **push only when the user says so in the prompt**.
