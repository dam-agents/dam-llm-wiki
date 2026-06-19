---
source: dam-agents/dam
commit: 662ebe4c88029788829246170e17465c69523521
files: [docs/architecture/platform-topology.md, docs/architecture.md]
updated: 2026-06-19
---

# Platform topology

DAM runs as **four long-lived subsystems** on Kubernetes plus per-agent paired
pods (`docs/architecture/platform-topology.md @662ebe4`):

- [controller](../sources/controller.md) — Go, reconciles the `Agent`/`Fork` CRs.
- [api-server](../sources/api-server.md) — TypeScript, brokers user requests and
  relays agent traffic.
- per-agent [agent-runtime](../sources/agent-runtime.md) + **gateway** pods — the
  agent process and its egress proxy ([Envoy gateway](../entities/envoy-gateway.md)).
- [ui](../sources/ui.md) — React, served by the api-server.

Agent pods are **stateless and ephemeral**; durable state lives on per-agent
PVCs, which is what makes scale-to-zero hibernation safe. The controller and
api-server **never talk directly** — they coordinate through the K8s API using
the `spec`/`status` subresource split so writes never contend.

## Key edges

| Edge | Protocol | Purpose |
| --- | --- | --- |
| ui → api-server | tRPC / HTTP | Resource CRUD, uploads |
| ui → api-server | WebSocket (ACP, JSON-RPC 2.0) | Live session, permissions, streaming, session list/create/delete |
| ui → api-server | WebSocket (binary frames) | Terminal session |
| cli → api-server | tRPC + WS frames | Agent resolution, auth, `dam chat` attach |
| api-server → agent-runtime | WebSocket (ACP) / HTTP (tRPC) | Chat relay (one hop) + in-pod file ops |
| agent-runtime → api-server (harness port, via gateway) | HTTP | MCP tools, runtime-channel `hello` |
| gateway → api-server | gRPC | HITL ext_authz Check |
| controller → K8s API | watch / status writes | Reconciliation |
| api-server → K8s API | REST | Spec writes, pod wake |

ACP frames are JSON-RPC 2.0, one logical message per WebSocket frame.

## Two-port api-server

The **public port** is user-authenticated (tRPC, REST, ACP relay); the
**harness port** is cluster-internal with no user auth (MCP, `hello`). They
share no routes. All ACP is relay-only — the UI never dials pods directly.

## K8s resource model

Controller-reconciled domain resources are CRDs under `agent-platform.ai/v1`
([Agent](../entities/agent.md), [Fork](../entities/fork.md)), each with a status
subresource: `spec` is api-server-owned user intent, `status` is
controller-owned observed state whose `Ready` condition is the api-server's sole
routing signal. Two domain resources are deliberately **not** CRDs:
[Templates](../entities/template.md) (chart-rendered ConfigMaps) and
[Schedules](../entities/schedule.md) (Postgres rows) — see
[persistence-substrates](persistence-substrates.md).

## See also

- [agent-lifecycle](agent-lifecycle.md) · [zero-trust-credential-gateway](zero-trust-credential-gateway.md) · [persistence-substrates](persistence-substrates.md)
