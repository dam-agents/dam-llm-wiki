---
source: dam-agents/dam
commit: b62d21c288162847d7d9918ca7887265448fe2b3
files: [docs/architecture/platform-topology.md, docs/architecture/agent-lifecycle.md, packages/agent-runtime/src/modules/acp/, packages/api-server/src/core/acp-client.ts, packages/api-server/src/config.ts]
updated: 2026-07-02
---

# ACP (Agent Client Protocol)

ACP is the **JSON-RPC 2.0** protocol DAM uses for all live agent interaction —
one logical message per WebSocket frame
(`docs/architecture/platform-topology.md @662ebe4`). It is the harness-agnostic
contract that lets any compatible runtime (Claude Code, Pi Agent, Bob, Codex)
run on the platform (`README.md:55 @662ebe4`). Chat sessions, permission
prompts, tool calls, streaming output, and session list/create/delete/mode all
flow over ACP.

## Relay-only

Clients (UI tabs, channel workers, the CLI) **never dial agent pods directly**.
Every ACP frame is relayed through the [api-server](../sources/api-server.md):
one hop, no fan-out, with the api-server waking a hibernated agent before
forwarding the first frame and rewriting `Authorization` to the per-agent
runtime token (`docs/architecture/platform-topology.md @662ebe4`). The
[agent-runtime](../sources/agent-runtime.md) speaks ACP to the harness child via
`/usr/local/bin/harness-chat`.

## Runtime-mediated resume

`session/resume` is mediated **entirely by the runtime** — the frame never
reaches the harness (`docs/architecture/agent-lifecycle.md @662ebe4`). On the hot
path the runtime engages the channel and advances its cursor with no replay; on
the cold path (after a pod restart) it parks the request, issues its own
`session/load` to rehydrate the harness, and serves all parked waiters from
memory once replay completes. This shields the UI from per-harness capability
gaps — some harnesses (e.g. `pi-agent`) don't implement `unstable_resumeSession`
at all. See [Session](../entities/session.md).

## Session metadata over `_meta.platform`

Session mode/type/`scheduleId`/`threadTs`/`createdAt` are **agent-owned
metadata** surfaced over ACP under `_meta.platform`; switching modes persists
over `session/resume` with no server-side side effect or cross-client broadcast
(`docs/architecture/platform-topology.md @662ebe4`).

## Turn liveness — WS heartbeat, not stream activity

The relay's per-turn timeout derives from a **WebSocket ping/pong heartbeat**, not
from application streaming ([#2214](https://github.com/dam-agents/dam/pull/2214)).
The old 120 s inactivity timer reset only on `sessionUpdate`, so a live-but-silent
turn — an egress-approval hold (default 1800 s), a long tool, or a slow first token
— aborted with a spurious "timed out" while the agent kept running. Now the
api-server pings the agent-runtime WS every 30 s and only a dead/half-open socket
aborts, requiring `MAX_MISSED_PONGS` (2) consecutive unanswered pings (~90 s) to
ride out a GC pause on a healthy pod
(`packages/api-server/src/core/acp-client.ts:15-24,148 @b62d21c`). A configurable
absolute per-turn ceiling (`acpTurnCeilingSeconds`, default 1 h) backstops a
wedged-but-still-ponging agent, validated `>= approvalHoldSeconds` so the two timers
cannot disagree (`packages/api-server/src/config.ts:116-122,198-202 @b62d21c`). This
is shared across the ACP relay, Slack/Telegram channels, and forks via the
`sendPrompt` path.

## See also

- [Session](../entities/session.md) · [agent-lifecycle](agent-lifecycle.md) · [platform-topology](platform-topology.md)
