---
source: dam-agents/dam
commit: 662ebe4c88029788829246170e17465c69523521
files: [docs/architecture/platform-topology.md, docs/architecture/agent-lifecycle.md, packages/agent-runtime/src/modules/acp/]
updated: 2026-06-19
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

## See also

- [Session](../entities/session.md) · [agent-lifecycle](agent-lifecycle.md) · [platform-topology](platform-topology.md)
