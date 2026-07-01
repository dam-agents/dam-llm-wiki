---
source: dam-agents/dam
commit: 70c53ae1a47512cfe06c0eb2982102d899e45f5a
files: [packages/controller/main.go, packages/controller/api/v1/, packages/controller/pkg/reconciler/, packages/controller/pkg/config/config.go, docs/architecture/platform-topology.md]
updated: 2026-07-01
---

# controller

A stateless **Go** reconciler (client-go) — the platform's Kubernetes operator.
It watches the `Agent`, `Fork`, and `Run` custom resources (`agent-platform.ai/v1`)
plus agent-labelled pods, and reconciles each agent's running infrastructure. It
writes **only the `status` subresource** of resources it owns and never writes
`spec` — the spec/status split makes write contention with the api-server
structurally impossible (`docs/architecture/platform-topology.md @662ebe4`).

## What it reconciles per Agent

Two paired StatefulSets (agent + gateway, replicas 0↔1 for
hibernate/wake), two Services (agent ACP + `<agent>-gateway` proxy DNS), a
per-pair agent-egress NetworkPolicy, a per-agent ServiceAccount, a per-agent
ext-authz Service, two Istio AuthorizationPolicies, and a per-agent Envoy
bootstrap ConfigMap + leaf-TLS Certificate
(`docs/architecture/platform-topology.md @662ebe4`). It computes agent
**readiness** from the pod pair onto the Agent `status` (`Ready` is the
api-server's sole routing signal) and hibernates idle agents by scaling the
pair to zero — the schedule loop lives in the api-server, not here.

Its **idle checker** (`idlechecker.go`) scans on a timer of 1/6 of the idle
timeout clamped to 30 s–5 min, and hibernates only when the shared `shouldRun`
activity gate *and* a live `/api/status` busy-probe both agree the Agent is quiet
(`packages/controller/pkg/reconciler/idlechecker.go:62-102 @b68af4a`). Each Agent
may override the chart-wide timeout via `spec.hibernationTimeout` (`"0s"` disables
hibernation), resolved by `effectiveIdleTimeout` before the gate
([#1373](https://github.com/dam-agents/dam/pull/1373),
`packages/controller/pkg/reconciler/hibernation.go:26-50 @b68af4a`). See
[the idle decision](../concepts/agent-lifecycle.md#the-idle-decision).

When the optional [telemetry backend](../concepts/observability.md) is on, the
controller reads `PLATFORM_TELEMETRY_COLLECTOR_HOST` and renders a collector
egress chain into every gateway's Envoy bootstrap that MITM-terminates and stamps
the trusted `x-platform-agent-id` header
(`packages/controller/pkg/config/config.go:265-270 @b68af4a`,
`packages/controller/pkg/reconciler/envoy.go:855-909 @b68af4a`).

## Three reconciled CRs

Each CR has its own dynamic informer, work queue, and worker
(`packages/controller/main.go:117-223 @70c53ae`):

- **Agent** — the durable per-agent resource (above); resolved via a getter.
- **Fork** — the Job-shaped per-turn run; see [Fork](../entities/fork.md).
- **Run** — the bare-Pod, single-command executor behind the in-pod `dam-run`
  CLI, added in [#2120](https://github.com/dam-agents/dam/pull/2120). The
  `RunReconciler` is deliberately lighter than the Fork's: the executor runs as
  the **parent Agent's own owner** and routes egress through the parent's
  already-running gateway, so it renders just a bare Pod + one egress
  NetworkPolicy (no gateway/cert/SA/AuthorizationPolicy of its own)
  (`packages/controller/pkg/reconciler/run.go:32-38 @70c53ae`). The api-server
  deletes the `Run` when the `dam-run` stream closes; the controller keeps a
  hard-lifetime reaper (`RunMaxLifetime` 60 min) as a GC backstop for a `Run` the
  api-server never cleaned up (`packages/controller/pkg/reconciler/run.go:20-29,69-76 @70c53ae`).
  A pod-readiness transition re-enqueues the owning `Run` so the Ready+`podIP`
  status is written without waiting for the informer resync — interactive
  `dam-run` startup stays snappy (`packages/controller/main.go:313-322 @70c53ae`).
  See [Run](../entities/run.md).

The Fork and Run workers share a generic `runCachedWorker` (lister-decoded CRs);
the Agent worker resolves via a getter. The single unstructured→typed conversion
point for every CR is the generic `FromCacheObject[T]`
(`packages/controller/main.go:277-309 @70c53ae`,
`packages/controller/pkg/reconciler/crd.go:24-44 @70c53ae`).

## Code layout

- `api/v1/` — CRD Go types: `agent_types.go`, `fork_types.go`, `run_types.go`,
  the group-version info, and generated deepcopy/schema code
  (`packages/controller/api/v1/ @70c53ae`).
- `pkg/reconciler/` — the reconcile logic, one file per concern:
  `agent_reconciler.go`, `envoy.go`, `gateway.go`, `network_policy.go`,
  `authorization_policy.go`, `extauthz_service.go`, `fork.go`/`fork_resources.go`,
  `run.go`/`run_resources.go`, the shared `ephemeral_pod.go` (the Fork/Run pod
  builder), `hibernation.go`, `idlechecker.go`, `warmpool.go`,
  `service_account.go`, `status.go`, `iptables_init.go`, `np_gate_init.go`,
  `pod_termination.go` (`packages/controller/pkg/reconciler/ @70c53ae`).
- `pkg/config/`, `pkg/crdcheck/`, `pkg/types/`, `main.go` — config, CRD
  presence checks, shared types, entrypoint.

Installing the CRDs requires cluster-admin at install time; the chart layout is
under `deploy/helm/platform/ @662ebe4`.

## See also

- [agent-lifecycle](../concepts/agent-lifecycle.md) — create → wake → hibernate → delete, driven by this reconciler and the api-server.
- [persistence-substrates](../concepts/persistence-substrates.md) — the `spec`/`status` ownership split and the warm PVC pool.
- [zero-trust-credential-gateway](../concepts/zero-trust-credential-gateway.md) — the NetworkPolicies and AuthorizationPolicies it renders.
- [Fork](../entities/fork.md) · [Run](../entities/run.md) — the two ephemeral Agent-derived CRs it reconciles.
- [observability](../concepts/observability.md) — the telemetry collector egress chain it renders into each gateway.
