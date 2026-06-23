---
source: dam-agents/dam
commit: 380cb06d1d60bca40fa703b77e13a16ec96eedf7
files: [packages/controller/main.go, packages/controller/api/v1/, packages/controller/pkg/reconciler/, docs/architecture/platform-topology.md]
updated: 2026-06-23
---

# controller

A stateless **Go** reconciler (client-go) — the platform's Kubernetes operator.
It watches the `Agent` and `Fork` custom resources (`agent-platform.ai/v1`) plus
agent-labelled pods, and reconciles each agent's running infrastructure. It
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

## Code layout

- `api/v1/` — CRD Go types: `agent_types.go`, `fork_types.go`, the group-version
  info, and generated deepcopy/schema code (`packages/controller/api/v1/ @662ebe4`).
- `pkg/reconciler/` — the reconcile logic, one file per concern:
  `agent_reconciler.go`, `envoy.go`, `gateway.go`, `network_policy.go`,
  `authorization_policy.go`, `extauthz_service.go`, `fork.go`/`fork_resources.go`,
  `hibernation.go`, `idlechecker.go`, `warmpool.go`, `service_account.go`,
  `status.go`, `iptables_init.go`, `np_gate_init.go`, `pod_termination.go`
  (`packages/controller/pkg/reconciler/ @662ebe4`).
- `pkg/config/`, `pkg/crdcheck/`, `pkg/types/`, `main.go` — config, CRD
  presence checks, shared types, entrypoint.

Installing the CRDs requires cluster-admin at install time; the chart layout is
under `deploy/helm/platform/ @662ebe4`.

## See also

- [agent-lifecycle](../concepts/agent-lifecycle.md) — create → wake → hibernate → delete, driven by this reconciler and the api-server.
- [persistence-substrates](../concepts/persistence-substrates.md) — the `spec`/`status` ownership split and the warm PVC pool.
- [zero-trust-credential-gateway](../concepts/zero-trust-credential-gateway.md) — the NetworkPolicies and AuthorizationPolicies it renders.
