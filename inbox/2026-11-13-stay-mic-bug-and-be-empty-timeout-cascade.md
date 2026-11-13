# 2026-11-13 — "Stay in the interview" mic bug + BE empty_timeout cascade

Context: INV-1184 / 1185 / 1186 (tab-close + "Before you go" card) on `ai-mock-interview-fe-v2`. QA PR opened earlier today with FE fixes; one FE bug + one BE-driven UX bug remain and are blocked on a cross-repo change.

## What shipped (FE, QA PR)

Repo: `ai-mock-interview-fe-v2`.

- **bfcache guard on `pagehide`** — `src/hooks/useTerminateOnUnload.ts`. Chrome fires `pagehide` with `event.persisted === true` when the native "Leave site?" prompt puts the page in bfcache. The legacy handler unconditionally PATCHed `ABORTED`, killing live interviews whenever Chrome touched bfcache. Now `event.persisted` short-circuits the handler: only terminate when the page is really going away. Also clears the pending `onStay` timer on bfcache entry so it re-arms on the next `beforeunload`.
- **Reactive mic icon on local tile** — `src/components/interview/stage.tsx`. `LocalTile` used `participant.isMicrophoneEnabled` (a static read); icon didn't re-render when the track muted. Switched to `useLocalParticipant().isMicrophoneEnabled` so the tile stays in sync with the control bar.
- **End-click card UI matched to artifact** — `src/components/interview/room-dialogs.tsx`. `isEndClick` variant now narrower (`sm:max-w-3xl` vs tab-close's `sm:max-w-5xl`), full-width horizontal countdown panel + 2-col body (miss | how-to). Tab-close keeps 3-col grid with trophy card.
- **Report unlock copy** — the "11:15 of 45:00" number is **elapsed interview time**, not wall-clock. From the artifact: report unlocks at a quarter of the interview duration (45 min × ¼ = 11:15 elapsed).

## What is NOT fixed (blocked, FE can't solve alone)

Bug: user clicks "Stay in the interview" on the browser's native prompt → mic icon shows disabled in the profile tile, agent stops talking, interview often ends up ABORTED.

### Why FE alone cannot fix it

1. During Chrome's native "Leave site?" dialog, Chrome mutes the local `MediaStreamTrack` at the browser level and briefly hitches the WebRTC connection. We cannot stop Chrome from doing this.
2. Four FE recovery strategies tried, all fail:
   - `setMicrophoneEnabled(true)` → LiveKit's `unmute()` no-ops against the browser-level `MediaStreamTrack.muted`.
   - `LocalAudioTrack.restartTrack()` → races with LiveKit's cached publication.
   - `unpublishTrack(track, true)` + `setMicrophoneEnabled(true)` → leaves `useTrackToggle` in `pending: true` forever, control-bar mic button becomes unclickable.
   - `setMicrophoneEnabled(false)` → `setMicrophoneEnabled(true)` (non-destructive toggle, current code) → button stays clickable but mic still doesn't recover.

### The full cascade (root cause chain)

```
Chrome hitch on cancelled beforeunload
  → agent worker (uplevel-livekit-worker) sees candidate "disconnect" and leaves the room
  → room goes empty
  → LiveKit's empty_timeout fires (small default)
  → LiveKit sends room_finished webhook
  → BE webhook handler marks interview ABORTED (no grace, no min-duration check)
```

By the time the user clicks "Stay", the agent is gone and the DB row is `ABORTED`. Mic recovery on FE is therefore necessary but not sufficient.

## Proper fix (two-part — both required)

**Part 1 — this BE repo (`ik-ai-mock-interview-be`), NOT YET IMPLEMENTED:**

- `services/agent/livekit_adapter.py` line ~81: add `empty_timeout=120` (seconds) to `RoomConfiguration` so a brief hitch does not immediately destroy the room.
- `services/webhook/terminate_session_webhook.py`: add a minimum-duration guard before marking `ABORTED` on `room_finished`. If the interview is under ~30s or shows a clear hitch pattern, don't abort — treat as a transient disconnect.

**Part 2 — `uplevel-livekit-worker` repo (SEPARATE repo, not in this workspace):**

- Agent must wait 30–60s before leaving when the candidate disconnects. If they reconnect inside the window, resume conversation with LLM state intact. Otherwise leave as before.

**If only Part 1 ships:** interview is no longer ABORTED on hitch, mic may recover with the new empty_timeout room staying alive — but agent is gone, so candidate sees a silent room with no interviewer. Part 1 alone stops the ABORTED rows piling up but does not fix UX.

## Decision point (waiting on user / manager)

- User asked for a manager-ready plain-English explanation of the two-part fix before any BE code change. Drafted and delivered in-session.
- Blocked until: (a) manager confirms scope of Part 1 + points to the `uplevel-livekit-worker` repo for Part 2, OR (b) user says "ship Part 1 now."

## Files touched today (FE)

- `src/hooks/useTerminateOnUnload.ts` — bfcache guard, pending timer cleanup.
- `src/components/interview/stage.tsx` — `useLocalParticipant` for reactive mic icon on `LocalTile`.
- `src/components/interview/room-dialogs.tsx` — end-click card width + layout rework.
- `src/components/interview/interview-room.tsx` — non-destructive mic toggle on `onStay` (temporary; does not actually recover the mic — see cascade above).

## Files identified in BE for the fix (NOT edited)

- `ik-ai-mock-interview-be/services/agent/livekit_adapter.py` — `RoomConfiguration` + `RoomAgentDispatch(agent_name="uplevel-livekit-worker")` at line ~81. Where `empty_timeout` goes.
- `ik-ai-mock-interview-be/services/webhook/terminate_session_webhook.py` — `terminate_if_in_progress(..., ABORTED)` at line ~44. Where the min-duration guard goes.
- `ik-ai-mock-interview-be/routers/webhook_router.py` line ~26 — `room_finished` event dispatch.

## Related tickets

- INV-1184 — tab-close / terminate-on-unload hook.
- INV-1185 — "Before you go" / "Stay in the interview" card.
- INV-1186 — end-click card variant.
- INV-1182 — early-exit feedback on next visit (not started).
- INV-1187 — hint-focused, needs BE (not started).

## Gotchas worth remembering

- **`useTrackToggle({ source: Track.Source.Microphone })` is sticky.** If you call `unpublishTrack(track, stopOnUnpublish=true)` without immediately re-publishing cleanly, `pending` never flips back to `false` and the control-bar mic button is dead. Any FE recovery must preserve the publication or fully reset the toggle.
- **Chrome's native "Leave site?" dialog puts the page into bfcache.** `pagehide` fires with `persisted === true`. Any keepalive / terminate call there kills the interview even if the user clicked Cancel. Always guard on `event.persisted`.
- **`participant.isMicrophoneEnabled` is a static read.** It does not subscribe to the participant's events. Use `useLocalParticipant()` from `@livekit/components-react` for a reactive value.
- **LiveKit `RoomConfiguration(agents=[RoomAgentDispatch(agent_name=...)])` is applied once at room creation.** If the dispatched agent leaves (worker process exits the room), it is not automatically re-dispatched when a new participant joins. Re-dispatch would need an explicit server-side call.
- **Agent worker is in a separate repo (`uplevel-livekit-worker`)**, not accessible from this workspace. Do not look for the agent's grace-period logic here.
