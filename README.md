# IK-wiki — Interview Kickstart second brain

Compact knowledge base for IK engineering. Maintained by agents, read by humans and agents.

## What's in here

| Folder | Contents |
|---|---|
| `repos/` | Per-repo overview, architecture, conventions, gotchas |
| `services/` | Shared services (Cognito, LiveKit, S3, DynamoDB, Sentinel, Sentry) |
| `integrations/` | Cross-repo contracts (FE ↔ BE) |
| `tickets/` | Active tickets. Distilled on Done → `tickets/_done/` |
| `env/` | Env-var registry — **names and purpose only, never values** |
| `workflows/` | Release flow, story pointing |
| `inbox/` | Recent session digests (pre-merge traces) |
| `index.md` | **Start here.** Catalog of every page |

## For humans

1. Open [`index.md`](./index.md) — pick the page you need.
2. Edit any file directly — agents reconcile via lint.
3. Commit messages: `type: subject` (`docs:`, `fix:`, `learn:`).

## For agents

Read [`CLAUDE.md`](./CLAUDE.md) — defines read order, update triggers, and secret rules.

**Scope: IK only.** Personal projects are in a separate wiki.
