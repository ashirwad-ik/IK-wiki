# IK-wiki — Interview Kickstart second brain

A compact, living knowledge base for IK engineering work. Written and maintained by AI agents (Claude Code / Codex), read by both humans and agents.

> **One-sentence rationale:** tickets close; repositories compound. This wiki captures what we learn about our repositories, services, and their integrations so the next person (or agent) working on them doesn't start from zero.

## What's in here

| Folder | What it holds |
|---|---|
| `repos/` | One sub-folder per IK repository — overview, architecture, conventions, gotchas, lessons. |
| `services/` | Shared external services we depend on (Cognito, LiveKit, S3, DynamoDB, Sentinel, Sentry). |
| `integrations/` | Contracts between our own repos (e.g. frontend ↔ backend). |
| `decisions/` | Lightweight ADRs — "why we picked X over Y". |
| `tickets/` | Live tickets being worked. Distilled on Done, archived to `tickets/_done/`. |
| `env/` | **How we handle environment variables and secrets — read this before contributing.** |
| `inbox/` | Raw digests from recent agent sessions (pre-merge). Mostly a trace; the real wiki is the pages. |
| `index.md` | Catalog of every page with a one-line summary. **Start here when browsing.** |
| `log.md` | Append-only changelog. What changed and when. |

## How to use it (humans)

1. Open [`index.md`](./index.md) — pick the page you want.
2. Prefer browsing in Obsidian (point it at this folder) if you want wikilinks + graph view. Any Markdown reader works.
3. If something is wrong, **just edit the file**. Commit directly. Agents will reconcile via lint.
4. If a piece of context is missing, raise it in a session — the next wiki-update cycle will file it.

## How to use it (agents)

Read [`CLAUDE.md`](./CLAUDE.md). It defines read order, update triggers, models, and secret rules. Everything an agent needs to work with this wiki is there.

## Environment variables — how we handle them

**This is the one thing to get right.** The wiki records what every variable *is* and *where* its value lives. The wiki never contains the value itself.

### What you'll find in the wiki
- **Variable names** — `AWS_ACCESS_KEY_ID_1`, `COGNITO_USER_POOL_ID`, `NEXT_PUBLIC_MI_API_BASE_URL`, etc.
- **Purpose** — one line per variable: what it configures, which service it points at.
- **Where the real value lives** — team password manager, AWS Secrets Manager, ECS task definition, a repo's own untracked `.env.local`, etc.
- **Naming gotchas** — e.g. the backend reads `AWS_ACCESS_KEY_ID_1` (with the `_1` suffix), not the generic name; the README in that repo is stale.

### What you will never find in the wiki
- API keys, tokens, passwords, JWTs.
- Connection strings with embedded credentials.
- Pre-signed URLs or any URL with secret query params.

### Enforcement
- Agent rules in [`CLAUDE.md`](./CLAUDE.md) prohibit writing secret values and require redaction on ingest from Jira / Slack / tickets.
- `.gitignore` blocks `.env*`, `*.key`, `*.pem`, `secrets/` — belt + braces, so a stray paste can't be committed.
- The authoritative registry is [`env/README.md`](./env/README.md). All variable references in page bodies should point back to it.

### If you spot a secret in the wiki
Delete it in a commit (the value will remain in git history — treat it as leaked). Then: rotate the credential, and tell the team in your usual channel. Don't try to rewrite history on a shared remote.

## How the wiki stays current

Four events trigger an automatic update — no scheduled jobs, no manual drudgery:

1. **A PR is raised** (`gh pr create` succeeds) — the strongest signal that something worth recording just happened.
2. **A session wraps up** on IK work.
3. **A thread is about to be context-compacted** — safety flush before detail is lost.
4. **The user runs `/wiki`** — manual flush, typically used before starting a fresh thread.

Each trigger writes a short digest to `inbox/`, merges into the right pages, appends to `log.md`, and pushes to GitHub — all in one cycle. Details in [`CLAUDE.md`](./CLAUDE.md).

For tickets, agents call the Atlassian MCP **only after a PR exists**, **only at status transitions**, and **only for comment deltas** — not polling, not refetching, not consuming tokens for no reason.

## Contributing (humans)

- Edit any file directly. Keep pages short — one topic per page. Prefer links to duplication.
- Dated inline notes are fine: `- 2026-11-08 — feature X ships behind flag Y`.
- Treat every paste as potentially containing a secret. The `.gitignore` is a safety net, not a substitute for checking.
- Commit messages: `type: subject` (`docs:`, `fix:`, `learn:`) — the wiki is prose-first, so conventional-commit-lite is enough.

## Scope

**IK only.** Personal projects live in a separate wiki under `~/Desktop/Ethan/`. This wiki never references them, and no personal context is ingested here.

---

Questions, corrections, additions → just edit. The wiki prefers "wrong and current" to "right and stale."
