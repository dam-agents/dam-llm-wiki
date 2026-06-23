---
source: dam-agents/dam
commit: 380cb06d1d60bca40fa703b77e13a16ec96eedf7
files: [docs/architecture/platform-topology.md, docs/architecture/security-and-credentials.md]
updated: 2026-06-23
---

# Envoy gateway (per-agent gateway pod)

A per-agent **Envoy** pod paired with the [agent-runtime](../sources/agent-runtime.md)
pod — the enforcement point of the
[zero-trust credential gateway](../concepts/zero-trust-credential-gateway.md)
(`docs/architecture/platform-topology.md @662ebe4`). It is the agent's **only**
admitted route to TCP 80/443.

## What it mounts and does

It mounts the owner's credential K8s Secrets, the cert-manager-issued leaf TLS
material, and the rendered Envoy bootstrap ConfigMap (all controller-rendered),
then (`docs/architecture/security-and-credentials.md @662ebe4`):

1. Terminates the agent's egress TLS (the agent trusts the leaf cert via the CA
   at `/etc/platform/ca/ca.crt`).
2. Reads SNI and, on per-host L7 filter chains, **injects credentials** on the
   wire (headers and/or URL query params), then forwards to a `STRICT_DNS`
   cluster pinned to the host — the agent's inner `Host` can't misroute.
3. Gates each credentialed request through the api-server's **ext_authz Check**
   ([HITL approvals](../concepts/hitl-approvals.md)).

SNI-miss traffic is TCP passthrough. The agent pod **never** mounts Secret bytes.

## Identity & network

The gateway pod stays in the Istio ambient mesh and carries a SPIFFE workload
cert whose SA name equals the Agent (or [fork](fork.md)) name; per-Agent
AuthorizationPolicies admit it to the harness waypoint
(`/api/agents/<id>/*`) and the per-Agent ext-authz Service. Its NetworkPolicy
admits ingress only from the paired agent pod and egress only to upstreams, the
ext_authz port, and DNS (`docs/architecture/security-and-credentials.md @662ebe4`).

## See also

- [zero-trust-credential-gateway](../concepts/zero-trust-credential-gateway.md) · [HITL approvals](../concepts/hitl-approvals.md) · [controller](../sources/controller.md) (renders it) · [Keycloak](keycloak.md)
