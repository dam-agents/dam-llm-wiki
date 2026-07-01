---
source: dam-agents/dam
commit: b68af4ad0a0c427c856b0e5ba245feb8c2085a72
files: [docs/architecture/security-and-credentials.md, docs/architecture.md, packages/api-server/src/apps/harness-api-server/harness-run-relay.ts, packages/controller/pkg/reconciler/envoy.go, packages/controller/pkg/config/config.go]
updated: 2026-07-01
---

# Zero-trust credential gateway

The defining security property of DAM. Three rules carry the model
(`docs/architecture/security-and-credentials.md @662ebe4`):

1. **Agents never hold upstream credentials.** Real tokens (GitHub, Anthropic,
   Slack, internal gateways) live in K8s Secrets labelled with the owner's `sub`.
   The [Envoy gateway](../entities/envoy-gateway.md) pod injects them on the wire;
   the agent pod never mounts Secret bytes.
2. **Identity flows from [Keycloak](../entities/keycloak.md).** The api-server
   validates the JWT and stamps `agent-platform.ai/owner=<sub>` on every resource;
   the controller's label selector refuses to mount any other owner's Secret.
3. **Two boundaries, layered.** Agent → gateway is gated at the **kernel** by a
   per-pair NetworkPolicy; gateway → api-server (harness + ext-authz) is gated at
   the **mesh** by per-Agent Istio AuthorizationPolicies on the gateway pod's
   SPIFFE principal.

## The wire path

The agent sets `HTTPS_PROXY=http://<agent>-gateway:<port>`, but the value is
**decorative** — Kubernetes admits no other route to TCP 80/443
(`docs/architecture/platform-topology.md @662ebe4`). Every egress arrives at the
paired gateway as HTTP CONNECT. Envoy terminates the agent's TLS with a
per-agent leaf cert (cert-manager-issued; CA trusted by the agent at
`/etc/platform/ca/ca.crt`), reads SNI, and per-host filter chains inject the
configured header(s)/query-param and forward to a `STRICT_DNS` cluster pinned to
the host — so the agent's inner `Host` header can't misroute a credential
(route-confusion exfiltration is structurally closed). SNI-miss chains do TCP
passthrough. Hosts with a credential surface as L7 chains; hosts without surface
as L4 passthrough.

## HITL ext_authz

Each credentialed request goes through an **ext_authz Check** against the
api-server over a per-Agent ext-authz Service (`<release>-extauthz-<id>`); the
AuthorizationPolicy on each Service admits only the matching SA principal, so the
calling agent is proven cryptographically before the Check arrives. The handler
looks up the matching [egress rule](hitl-approvals.md) and allows, denies, or
**holds** the request open while the user decides in the inbox.
`failure_mode_allow: false` — a blocked Check fails **closed** (403, no prompt).
See [HITL approvals](hitl-approvals.md).

## Boundaries in detail

- **Per-pair agent-egress NetworkPolicy** (`<id>-agent-egress`) — the agent pod
  opts out of ambient mesh, so the kernel sees real IPs; it admits only DNS-less
  egress to the paired gateway's Envoy port (the agent resolves nothing itself).
- **Agent ingress NetworkPolicy** — admits the agent port only from api-server
  (ACP/tRPC relay) and controller (idle probe); agent-runtime serves
  unauthenticated on the assumption this kernel gate is the auth boundary.
- **Mesh AuthorizationPolicies** — harness via the api-server waypoint
  (ALLOW gateway SA → `/api/agents/<id>/*`), ext-authz per-Agent Service
  (ALLOW matching SA). Agent pods carry no SPIFFE identity and no SA token.
- **Forks** get their **own** per-fork SA, narrowly scoped to
  `/api/agents/<parent>/mcp` and the parent's ext-authz Service, so a compromised
  fork can't impersonate the parent.
- **[Run](../entities/run.md) executors** (`dam-run`) take the *opposite* trade:
  no SA, gateway, cert, or AuthorizationPolicy of their own — a single egress
  NetworkPolicy routes the executor through the **parent's** existing gateway, so
  its egress boundary is *exactly* the parent's (same credentials, same
  ext-authz/HITL gate). `dam-run` adds no new privilege — it only dials the
  harness port the agent can already reach, pinned to the agent's own SA at the
  waypoint, so an agent spawns executors only for itself. The cost: that identity
  reaches the parent's full harness surface — runs included — so recursion is
  bounded by an api-server **per-agent concurrent-run cap (16)**, not
  structurally (`docs/architecture/security-and-credentials.md @70c53ae`,
  `packages/api-server/src/apps/harness-api-server/harness-run-relay.ts:11-20 @70c53ae`).

## Telemetry egress (trusted attribution)

When the optional [telemetry backend](observability.md) is enabled, the gateway
gains one more L7 chain — but it injects **identity, not credentials**. It
MITM-terminates agent OTLP on the collector SNI and stamps a trusted
`x-platform-agent-id` header with `OVERWRITE_IF_EXISTS_OR_ADD`, replacing anything
the agent set; the value is fixed in this gateway's controller-rendered config, so
an agent can never forge another's identity
(`packages/controller/pkg/reconciler/envoy.go:855-909 @b68af4a`). No ext_authz
(platform-internal traffic, not user egress) and no credential injection; the
collector host is added to the leaf SAN so the agent's TLS validates the intercept
cert. The chain renders only when the backend is configured (`TelemetryEnabled()`)
and the collector host doesn't collide with a credentialed chain host
(`packages/controller/pkg/reconciler/envoy.go:1165 @b68af4a`,
`packages/controller/pkg/config/config.go:265-270 @b68af4a`). A
[Fork](../entities/fork.md)'s telemetry carries the *fork's own* instance ID, even
though its gateway dials the parent's ext-authz Service
(`packages/controller/pkg/reconciler/envoy.go:1131-1144 @b68af4a`).

## Credential storage notes

One K8s Secret per `(owner, connection)`. A GitHub PAT is *two* `generic`
Secrets sharing a display name: the `api.github.com` half (`Bearer`, projects
`GH_TOKEN`) and the `github.com` half (`Basic base64("x-access-token:"+PAT)` for
`git clone`). A single OAuth token can inject on multiple hosts with different
schemes from one Secret via the `injection-hosts` annotation. **Image-pull
credentials** are a structurally separate, agent-scoped class consumed by the
kubelet, never by Envoy.

## See also

- [Envoy gateway](../entities/envoy-gateway.md) · [HITL approvals](hitl-approvals.md) · [Keycloak](../entities/keycloak.md) · [persistence-substrates](persistence-substrates.md) (workspace is outside the trust boundary)
- [Fork](../entities/fork.md) · [Run](../entities/run.md) — the two ephemeral runs and their opposite identity trades (own SA vs. borrowed parent gateway).
- [db](../sources/db.md) — a distinct boundary: per-service NOSUPERUSER Postgres roles isolate the platform's own backing store (ADR-071), separate from this agent↔upstream egress gateway.
