---
source: dam-agents/dam
commit: b62d21c288162847d7d9918ca7887265448fe2b3
files: [docs/architecture/agent-lifecycle.md, docs/ubiquitous-language.md, packages/agent-runtime/src/modules/acp/infrastructure/session-metadata-store.ts, packages/agent-runtime/src/modules/acp/services/acp-runtime.ts]
updated: 2026-07-02
---

# Session

One conversation with the agent harness, with its own lifecycle and metadata
(`docs/ubiquitous-language.md @662ebe4`). Sessions are **agent-owned** — their
metadata lives in agent-runtime's `.platform/` state file on the
[PVC](../concepts/persistence-substrates.md), not in Postgres, and is surfaced
over [ACP](../concepts/agent-client-protocol.md) `_meta.platform`
(`docs/architecture/agent-lifecycle.md @662ebe4`).

## Chat sessions

The harness child runs for the **pod's lifetime**, not per-connection; multiple
ACP channels (UI tabs, the Slack worker, the trigger handler) attach
concurrently and engage by the `sessionId` on each frame
(`docs/architecture/agent-lifecycle.md @662ebe4`). Each session is an
append-only **in-memory log** (≤2 MB soft cap, truncation sentinel for older
history) with a per-channel cursor; `session/load` serves from the log on hit,
the on-disk store on cold start. When a session goes idle (no engaged channel,
no active/queued prompt, no pending agent request) the runtime sends
`session/close` and reaps the subprocess; the next attach respawns it.
`session/resume` is runtime-mediated — see [ACP](../concepts/agent-client-protocol.md).

## Terminal sessions

A different model: at most one WebSocket per `sessionId` on `/api/terminal`, a
PTY running `harness-terminal`, a binary frame protocol, 30 s reattach grace, no
log/fan-out/resume. One viewer at a time; the harness's own on-disk store (e.g.
`~/.claude/projects/.../<HARNESS_SESSION_ID>.jsonl`) is the durable record.

## SSH sessions

Unrelated to the session/mode machinery — no `sessionId`, no DB row, no harness
involvement. A per-connection `sshd -i` relays raw bytes; an open connection
pins the agent `active-session` so it won't hibernate.

## Last-activity timestamp

The session-metadata store also records a per-session **`lastActivityAt`**: when
a real prompt arrives, the runtime stamps it as genuine activity, and the value
is surfaced over ACP as the session's `updatedAt`
([#2049](https://github.com/dam-agents/dam/pull/2049),
`packages/agent-runtime/src/modules/acp/services/acp-runtime.ts:1227-1229,1405 @70c53ae`,
`packages/agent-runtime/src/modules/acp/infrastructure/session-metadata-store.ts:76-87 @70c53ae`).
This lets the session list **sort and display by last message**, rather than by
the harness JSONL's file-modification time (which a passive `session/load` would
bump). It is set only by a prompt, and is preserved across metadata `set` writes.

## Mode & continuity

Mode (chat ↔ terminal) is **metadata-only** — a UI hint, persisted over
`session/resume` with no harness effect and no cross-client broadcast.
[Schedule](schedule.md) sessions are typed (`schedule_cron`): *fresh* schedules
open a new session per fire; *continuous* schedules resume one session across
fires. An [Experiment](../concepts/experiments.md) Trial is likewise a typed
session, its metadata carrying the optional `experimentId` that scopes the Arm's
loop (`packages/agent-runtime/src/modules/acp/infrastructure/session-metadata-store.ts:8 @b62d21c`).

## See also

- [ACP](../concepts/agent-client-protocol.md) · [agent-lifecycle](../concepts/agent-lifecycle.md) · [Schedule](schedule.md)
