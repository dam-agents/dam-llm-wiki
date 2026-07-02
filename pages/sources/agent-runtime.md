---
source: dam-agents/dam
commit: b62d21c288162847d7d9918ca7887265448fe2b3
files: [packages/agent-runtime/src/, packages/agent-runtime-api/src/, docs/architecture/platform-topology.md, docs/architecture/agent-lifecycle.md]
updated: 2026-07-02
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
  `$HOME`, and dispatches runtime events (schedule triggers, workspace seeding,
  experiment-trigger). As of [#1270](https://github.com/dam-agents/dam/pull/1270)
  (`@b62d21c`, ADR-072) contribution kinds **and** event kinds are unified into one
  `drivers:` map resolved through a single plugin registry — built-ins default-on
  from a runtime-owned table, `impl` defaulting to the kind name
  (`packages/agent-runtime/src/modules/runtime-channel/manifest.ts:105-164 @b62d21c`,
  `packages/agent-runtime/src/modules/runtime-channel/compose.ts:106-115 @b62d21c`).
  This adds the **harness-config** driver — a one-shot write of per-agent
  model/mode/config into the harness's own config file, with a catalog advertised
  on `hello`, a live current-config read, and optional OpenAI-style model discovery
  (`packages/agent-runtime/src/modules/runtime-channel/drivers/harness-config-plugin.ts:26-34,48-102 @b62d21c`,
  `packages/agent-runtime/src/modules/runtime-channel/infrastructure/model-discovery.ts:20-64 @b62d21c`).
  See [connections](../concepts/connections-and-contributions.md).
- **In-pod tRPC** — a scoped router (via the api-server tRPC proxy) for file
  operations and skill install/uninstall/scan/publish/listLocal.
- **File import** — extracts a streamed tarball on the harness port into the PVC.
- **Pod service** — supervises at most one optional background process the image
  provides (e.g. claude-code's local model gateway), with SIGHUP reload.
- **Exec-only mode** — when `PLATFORM_EXEC_ONLY=1` (set on
  [Run](../entities/run.md) executor pods, added in
  [#2120](https://github.com/dam-agents/dam/pull/2120)), the same binary boots a
  stripped server: it serves one `/api/exec` WebSocket — a single PTY-spawned
  command speaking the terminal frame protocol — and **skips the runtime-channel
  `hello`** entirely. This is the in-pod backend of the `dam-run` CLI; the
  command argv arrives as URL query, never persisted in the `Run` CR
  (`packages/agent-runtime/src/server.ts:56-62,403-435 @70c53ae`,
  `packages/agent-runtime/src/modules/exec.ts:12-18 @70c53ae`).

## Code layout (`packages/agent-runtime/src/`)

`server.ts` (entrypoint), `core/`, and `modules/`: `acp/`, `runtime-channel/`,
`skills/`, `import/`, `files.ts`, `git.ts`, `ssh.ts`, `exec.ts` (the exec-only
`/api/exec` command runner), `pod-service.ts`, `config.ts`
(`packages/agent-runtime/src/modules/ @70c53ae`). The tRPC wire contract lives in
the sibling `agent-runtime-api` package
(`packages/agent-runtime-api/src/router.ts @662ebe4`).

## See also

- [agent-lifecycle](../concepts/agent-lifecycle.md) — the session/wake/hibernate machinery this implements.
- [Session](../entities/session.md) — the session model and resume mediation.
- [Run](../entities/run.md) — the exec-only executor pod that boots this binary for `dam-run`.
- [zero-trust-credential-gateway](../concepts/zero-trust-credential-gateway.md) — why the pod has no credentials of its own.
