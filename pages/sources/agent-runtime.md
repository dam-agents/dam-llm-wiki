---
source: dam-agents/dam
commit: d34c21a008d3b868fc260838374836ac88fb0807
files: [packages/agent-runtime/src/, packages/agent-runtime-api/src/, docs/architecture/platform-topology.md, docs/architecture/agent-lifecycle.md]
updated: 2026-06-24
---

# agent-runtime

The per-agent **pod process**. It runs the ACP WebSocket server and spawns the
underlying harness binary via the harness-script contract — two executables at
fixed paths, `/usr/local/bin/harness-chat` (the ACP subprocess) and
`/usr/local/bin/harness-terminal` (PTY-attached); otherwise the harness is
opaque (`docs/architecture/agent-lifecycle.md @662ebe4`). The pod holds **zero
credential Secrets** and has no admitted route to TCP 80/443 except its paired
gateway (`docs/architecture/platform-topology.md @662ebe4`).

## Responsibilities

- **Chat (ACP)** — accept one relayed ACP WebSocket, speak JSON-RPC 2.0 to the
  harness, and mediate `session/resume`/`session/load` itself so the UI is
  shielded from per-harness capability gaps. Each session is an append-only
  in-memory log (≤2 MB) fanned out to engaged channels. See
  [Session](../entities/session.md) and [ACP](../concepts/agent-client-protocol.md).
- **Terminal** — one WebSocket per `sessionId` on `/api/terminal`, a PTY running
  `harness-terminal`, relayed as a binary `OP_INPUT`/`OP_OUTPUT`/`OP_RESIZE`/
  `OP_EXIT` frame protocol with 30 s reattach grace.
- **SSH** — `/api/ssh` spawns a per-connection OpenSSH `sshd -i` as the agent
  user, relaying raw bytes; rebuilds `~/.ssh/environment` from the live injected
  env so SSH gets the same egress routing as the harness.
- **Runtime channel** — calls the api-server `hello` on boot/reconnect, accepts
  `applyState` deliveries, materializes declarative file/env contributions under
  `$HOME`, and dispatches runtime events (schedule triggers, workspace seeding).
  See [connections](../concepts/connections-and-contributions.md).
- **In-pod tRPC** — a scoped router (via the api-server tRPC proxy) for file
  operations and skill install/uninstall/scan/publish/listLocal.
- **File import** — extracts a streamed tarball on the harness port into the PVC.
- **Pod service** — supervises at most one optional background process the image
  provides (e.g. claude-code's local model gateway), with SIGHUP reload.

## Code layout (`packages/agent-runtime/src/`)

`server.ts` (entrypoint), `core/`, and `modules/`: `acp/`, `runtime-channel/`,
`skills/`, `import/`, `files.ts`, `git.ts`, `ssh.ts`, `pod-service.ts`,
`config.ts` (`packages/agent-runtime/src/modules/ @662ebe4`). The tRPC wire
contract lives in the sibling `agent-runtime-api` package
(`packages/agent-runtime-api/src/router.ts @662ebe4`).

## See also

- [agent-lifecycle](../concepts/agent-lifecycle.md) — the session/wake/hibernate machinery this implements.
- [Session](../entities/session.md) — the session model and resume mediation.
- [zero-trust-credential-gateway](../concepts/zero-trust-credential-gateway.md) — why the pod has no credentials of its own.
