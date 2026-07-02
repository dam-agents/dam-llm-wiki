---
source: dam-agents/dam
commit: b62d21c288162847d7d9918ca7887265448fe2b3
files: [packages/controller/api/v1/run_types.go, packages/controller/pkg/reconciler/run.go, packages/controller/pkg/reconciler/run_resources.go, packages/controller/pkg/reconciler/ephemeral_pod.go, packages/api-server/src/modules/runs/services/runs-service.ts, packages/api-server/src/apps/harness-api-server/harness-run-relay.ts, packages/agent-runtime/src/modules/exec.ts, packages/agent-runtime/src/server.ts, packages/platform-base/dam-run.mjs, packages/platform-base/Dockerfile, docs/architecture/agent-lifecycle.md, docs/architecture/security-and-credentials.md]
updated: 2026-07-02
---

# Run

An **ephemeral, single-command executor** derived from an [Agent](agent.md),
backing the in-pod **`dam-run`** CLI: `dam-run <cmd>` runs the command in a
*separate* sandbox pod that shares the calling pod's image, configuration, and
RWX `$HOME`/workspace, with stdio streamed through a PTY so it reads like a local
invocation (`docs/architecture/agent-lifecycle.md @70c53ae`,
`packages/platform-base/dam-run.mjs:2-5 @70c53ae`). The CLI ships in
`platform-base` (`COPY … dam-run.mjs /usr/local/bin/dam-run`), so it is on `PATH`
in every harness image (`packages/platform-base/Dockerfile:92 @70c53ae`). It is a
`Run` CR under
`agent-platform.ai/v1` — added in
[#2120](https://github.com/dam-agents/dam/pull/2120) — and is the third
controller-reconciled resource alongside [Agent](agent.md) and [Fork](fork.md).

## What it's for

`dam-run` lets an agent run a command in a *clean* copy of its own sandbox —
same image and credentials, same files — without disturbing its live process.
The classic use is parallel or throwaway work (build/test sweeps, a long
command) that should not block or pollute the agent's main session.

## Lighter than a Fork

A `Run` carries **only** a parent agent reference (`spec.agentName`); the command
argv is deliberately **not** stored on the CR — it travels only over the exec
WebSocket, so no command bytes ever land in etcd
(`packages/controller/api/v1/run_types.go:7-18 @70c53ae`). The executor pod is
generic. Phases are Pending / Ready / Failed / Completed; the controller is the
sole writer of `status` (phase, `podIP`, structured error)
(`packages/controller/api/v1/run_types.go:22-48 @70c53ae`).

Unlike a [Fork](fork.md) — which reconciles to a Kubernetes **Job**, gets its
**own** ServiceAccount, gateway, cert, and AuthorizationPolicies — a `Run`
reconciles to a **bare Pod** that runs as the *parent Agent's own owner* and
routes egress through the **parent's already-running** gateway. It therefore
needs no gateway/cert/SA/AuthorizationPolicy of its own — just the executor pod
plus **one** egress NetworkPolicy admitting it to the parent gateway
(`packages/controller/pkg/reconciler/run.go:32-38 @70c53ae`,
`packages/controller/pkg/reconciler/run_resources.go:16-24 @70c53ae`). The Fork
and Run reconcilers share the ephemeral-pod builder `buildEphemeralAgentPod`;
the kind-specific differences are identity/type labels, a few env vars, and the
workload kind (Job vs. Pod) (`packages/controller/pkg/reconciler/ephemeral_pod.go:92-112 @70c53ae`).

## exec-only mode

The executor pod boots [agent-runtime](../sources/agent-runtime.md) in
**exec-only mode** (`PLATFORM_EXEC_ONLY=1`): it serves a single `/api/exec`
WebSocket and **skips the runtime-channel `hello`** — it exists only to run one
command (`packages/controller/pkg/reconciler/run_resources.go:48-51 @70c53ae`,
`packages/agent-runtime/src/server.ts:56-62,466-468 @70c53ae`). `attachExec` spawns the
forwarded argv under a PTY and speaks the binary terminal-frame protocol
(`OP_INPUT`/`OP_OUTPUT`/`OP_RESIZE`/`OP_EXIT`), reporting signals as exit code
`128+signum` (`packages/agent-runtime/src/modules/exec.ts:12-18,66-76 @70c53ae`).
The executor is trusted to run the agent-supplied argv verbatim — the agent can
already run arbitrary code in its own sandbox, so this is the point, not a new
trust boundary (`packages/agent-runtime/src/modules/exec.ts:12-18 @70c53ae`).

## The synchronous flow

One `dam-run` invocation is one WebSocket, end to end
(`docs/architecture/agent-lifecycle.md @70c53ae`):

1. `dam-run` derives its target from `PLATFORM_MCP_URL` (`…/mcp` → `…/run`,
   `http` → `ws`) and dials the api-server **harness** port, passing argv (base64
   JSON), cwd, and tty size as URL query — the dependency-free WHATWG
   `WebSocket` can't set headers
   (`packages/platform-base/dam-run.mjs:25-36 @70c53ae`).
2. The api-server **run relay** writes a `Run` CR (owner-refed to the parent
   Agent by uid), waits for the controller-published pod IP, then dials the
   executor's `/api/exec` and relays terminal frames both ways
   (`packages/api-server/src/apps/harness-api-server/harness-run-relay.ts:97-130 @70c53ae`,
   `packages/api-server/src/modules/runs/services/runs-service.ts:51-97 @70c53ae`).
3. When the stream closes (command exits, or `dam-run` dies) the api-server
   **deletes** the `Run`, and K8s GC reaps the owner-refed executor pod +
   NetworkPolicy (`packages/api-server/src/apps/harness-api-server/harness-run-relay.ts:60-70 @70c53ae`).

## Lifetime & cleanup backstops

The `Run` is owner-refed to the **parent Agent**, so deleting the Agent
cascade-deletes any in-flight `Run`
(`packages/api-server/src/modules/runs/services/runs-service.ts:51-74 @70c53ae`).
Two backstops cover a `Run` the api-server never cleaned up (e.g. a crash
mid-stream):

- **Controller hard-lifetime reaper** — any `Run` older than `RunMaxLifetime`
  (60 min) is deleted regardless; a `dam-run` needing longer is out of scope for
  this lifetime model (`packages/controller/pkg/reconciler/run.go:20-29,69-76 @70c53ae`).
  A pod that doesn't reach Ready within `RunPodReadyTimeout` (120 s) is failed
  (`packages/controller/pkg/reconciler/run.go:21-23,127-130 @70c53ae`).
- **Harness-boot sweep** — a fresh api-server process holds no live relays, so it
  deletes every `Run` CR it finds at boot (their executors would otherwise run
  untethered) (`packages/api-server/src/modules/runs/services/runs-service.ts:103-106 @70c53ae`).

## Security & recursion bound

The executor holds **no credential bytes** and has **no SA, gateway, cert, or
AuthorizationPolicy** of its own; its egress boundary is *exactly* the parent's
(same credentials, same ext-authz/HITL gate) because it routes through the
parent's gateway (`docs/architecture/security-and-credentials.md @70c53ae`,
`packages/controller/pkg/reconciler/run_resources.go:41-43 @70c53ae`). Credentials
reach it the same way forks get them: placeholder env + on-wire injection at the
shared gateway (`packages/controller/pkg/reconciler/ephemeral_pod.go:172-175 @70c53ae`).
`dam-run` adds **no new privilege** — it only dials the harness port the agent
can already reach, pinned to the agent's own SA at the waypoint, so an agent can
spawn executors only for itself
(`docs/architecture/security-and-credentials.md @70c53ae`).

Because that identity reaches the parent's full harness surface — *runs
included* — a command inside an executor can spawn its own `dam-run` (nesting
depth N = N concurrent runs). With no per-run SA to scope it, recursion is
bounded **not structurally but by a per-agent concurrent-run cap**
(`MAX_CONCURRENT_RUNS_PER_AGENT = 16`, counted in-memory in the single-replica
api-server) — a deliberate trade for dropping the per-run gateway and its mesh
identity (`packages/api-server/src/apps/harness-api-server/harness-run-relay.ts:11-20 @70c53ae`).

## See also

- [Fork](fork.md) — the other ephemeral Agent-derived CR; Job-shaped, isolated SA, run-to-completion. Shares the ephemeral-pod builder.
- [Agent](agent.md) · [agent-lifecycle](../concepts/agent-lifecycle.md) · [zero-trust-credential-gateway](../concepts/zero-trust-credential-gateway.md)
- [agent-runtime](../sources/agent-runtime.md) — exec-only mode · [api-server](../sources/api-server.md) — the run relay · [controller](../sources/controller.md) — the RunReconciler.
