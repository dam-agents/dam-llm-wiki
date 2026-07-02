---
source: dam-agents/dam
commit: b62d21c288162847d7d9918ca7887265448fe2b3
files: [docs/architecture/observability.md, deploy/helm/platform/values.yaml, deploy/helm/platform/Chart.yaml, deploy/helm/platform/templates/clickstack/collector.yaml, deploy/helm/platform/templates/clickstack/authorizationpolicy.yaml, deploy/helm/platform/templates/_helpers.tpl, deploy/helm/platform/templates/agent-templates.yaml, deploy/helm/platform/templates/controller/deployment.yaml, packages/controller/pkg/reconciler/envoy.go, packages/controller/pkg/config/config.go]
updated: 2026-07-02
---

# Observability (agent telemetry)

An **optional, bundled subsystem** that receives and stores the OpenTelemetry
signals (logs, traces, metrics) agents emit — the substrate for answering *how
agents run*: token consumption, cost, per-sub-agent breakdown
(`docs/architecture/observability.md @b68af4a`). It was added in
[#2030](https://github.com/dam-agents/dam/pull/2030) (backend) and
[#2028/#2129](https://github.com/dam-agents/dam/pull/2129) (the Claude Code
export path).

It is **disabled by default** (`clickstack.enabled: false`) — a heavy, multi-pod
stack that would receive nothing until agents are wired to export, so operators
opt in per install (`deploy/helm/platform/values.yaml:245-246 @b68af4a`). When
disabled, none of it exists.

## Topology and roles

Implemented with **ClickStack**: a columnar telemetry store (ClickHouse) fronted
by an exploration UI (HyperDX), plus a document store backing that UI's own
application state (`docs/architecture/observability.md @b68af4a`). Three roles:

- **Collector** — an OpenTelemetry collector that receives OTLP and writes the
  signals into the store. It is **platform-owned**, deliberately *not* the
  collector ClickStack bundles: the platform runs its own with a fixed config and
  no key enforcement (`deploy/helm/platform/values.yaml:248-260 @b68af4a`,
  `deploy/helm/platform/templates/clickstack/collector.yaml @b68af4a`). It holds
  no upstream credentials; it only ingests telemetry.
- **Telemetry store** (ClickHouse) — a columnar analytical database built for
  high-volume, high-cardinality, time-series telemetry. The only place telemetry
  lives.
- **Exploration UI** (HyperDX) — reads the store directly; its own app state
  (dashboards, saved views) lives in a separate document store that holds no
  telemetry.

The whole stack sits inside the cluster trust boundary. The ClickStack subchart
is a Helm dependency gated by `clickstack.enabled`, and the third-party images
(HyperDX, ClickHouse server/keeper, busybox) are pinned to the
`quay.io/dam-agents` mirror (`deploy/helm/platform/Chart.yaml:8-14 @b68af4a`,
`deploy/helm/platform/values.yaml:261-300 @b68af4a`).

## Operators are install-time infrastructure

The telemetry store and the UI's app-state store are managed by Kubernetes
**operators** driven by custom resources. Like the service mesh and the
certificate manager, those operators and their CRDs install out-of-band at
cluster-install time (`deploy/tasks.toml`), not by the platform chart — Helm
never upgrades a chart's CRDs on an existing release, so CRD-owning
infrastructure stays outside the chart. The chart ships only the custom resources
and the platform-owned collector, which roll with an ordinary platform upgrade
(`docs/architecture/observability.md @b68af4a`).

## Access control: the mesh, not ingestion tokens

ClickStack's default posture secures the collector with an ingestion key the UI
issues — application-layer, token-based. The platform deliberately does **not**
use that. The collector is gated the same way as the rest of the platform: by the
**service mesh**, through an Istio `AuthorizationPolicy` that admits only the
platform's own namespaces (the release namespace and the agent namespace) and
denies everything else — Istio ALLOW is deny-by-default, enforced L4 by ztunnel
(`deploy/helm/platform/templates/clickstack/authorizationpolicy.yaml:16-27 @b68af4a`).
This keeps telemetry on the same SPIFFE-principal model as the rest of the
[credential gateway](zero-trust-credential-gateway.md) rather than a parallel
token scheme. Making the mesh the sole gate is *why* the collector is
platform-owned: ClickStack's bundled collector takes its config (including the
key check) dynamically from the UI, so that config cannot be the boundary here.
The UI is unaffected — it reads the store directly, not the collector.

## Agent export

Harnesses **export telemetry themselves** over OTLP — the platform does not
scrape or tap it. Export is wired for the **Claude Code harness only** for now; a
per-template `telemetry` flag opts a harness in, so others follow by flipping it
(`deploy/helm/platform/templates/agent-templates.yaml:63-66 @b68af4a`).

- **Rides the agent's ordinary egress.** The exporter honours the agent's
  `HTTPS_PROXY`, so telemetry leaves through the paired
  [gateway pod](../entities/envoy-gateway.md) over HTTPS to the collector — the
  same MITM-terminating chain that performs the [trusted
  attribution](#trusted-attribution). No new network path, no credential in the
  export.
- **Config travels the harness env rail.** The OTLP environment (enable flag,
  per-signal exporters, endpoint, protocol, flush intervals) is emitted by the
  `platform.agentTelemetry.env` chart helper — `CLAUDE_CODE_ENABLE_TELEMETRY=1`,
  `OTEL_{METRICS,LOGS,TRACES}_EXPORTER=otlp`, `OTEL_EXPORTER_OTLP_ENDPOINT=https://<collector>:4318`
  over `http/protobuf`, plus 1 s export intervals and the enhanced-telemetry beta
  flag (`deploy/helm/platform/templates/_helpers.tpl:332-355 @b68af4a`).
- **Signals.** Metrics, logs, and traces (the last via the enhanced-telemetry
  beta) over OTLP/HTTP. **Content bodies are not exported** — prompt text, tool
  arguments, and raw API bodies stay off; only structural telemetry (durations,
  model/tool names, token/cost counters, span shape) leaves the agent
  (`docs/architecture/observability.md @b68af4a`).

## Trusted attribution

Telemetry only answers *whose agents ran* if each record is reliably tied to the
producing agent — and agents run untrusted code, so the attribution cannot be
taken from the agent's own telemetry. It comes instead from the
platform-controlled path the telemetry already travels.

Every agent's egress leaves through its **paired gateway pod**, the only admitted
route, whose Envoy config is controller-rendered, not agent-writable. When
telemetry is enabled, that gateway MITM-terminates any agent OTLP bound for the
collector on the collector SNI and stamps a trusted `x-platform-agent-id` header,
**overwriting** anything the agent set — the value is fixed in this gateway's own
per-agent config (`packages/controller/pkg/reconciler/envoy.go:855-909 @b68af4a`).
The collector then promotes that header to a `platform.agent.id` resource
attribute with a **delete-then-set** processor: it unconditionally drops any
agent-supplied `platform.agent.id`, then sets it *only* from the trusted header,
so a forged value can never survive
(`deploy/helm/platform/templates/clickstack/collector.yaml:47-53 @b68af4a`).

The guarantee is **attribution, not content integrity**: an agent can misreport
its own numbers (inflate a token count) but can only ever pollute *its own*
telemetry — never make it appear under another agent. Telemetry carrying no
`platform.agent.id` (the platform's own components) is never attributed to a
user. The attribute is namespaced under the permanent `platform` codename so it
never collides with OpenTelemetry semantic-convention or agent-self-reported
`agent.*` attributes. For a [Fork](../entities/fork.md), the header carries the
*fork's own* instance ID (not the parent's), so fork telemetry attributes to the
fork even though the fork's gateway dials the parent's ext-authz Service
(`packages/controller/pkg/reconciler/envoy.go:1131-1144 @b68af4a`).

The collector host is added to the gateway's leaf-cert SAN so the agent's TLS
client validates the intercept cert, and the leaf is issued even for an instance
with no credential Secrets. The whole chain renders only when the backend is
configured (`TelemetryEnabled()` — collector host non-empty) **and** the
collector host doesn't collide with a credentialed chain host (two chains sharing
`server_names` is a fatal Envoy error)
(`packages/controller/pkg/reconciler/envoy.go:1165 @b68af4a`,
`packages/controller/pkg/config/config.go:265-270 @b68af4a`). The controller
learns the collector host from `PLATFORM_TELEMETRY_COLLECTOR_HOST`, set by the
chart only when `clickstack.enabled`
(`deploy/helm/platform/templates/controller/deployment.yaml:82-91 @b68af4a`,
`packages/controller/pkg/config/config.go:156-157 @b68af4a`).

## Persistence

The telemetry store is a **fourth durable substrate** beyond the three in
[persistence-substrates](persistence-substrates.md) (Postgres, the CRs, the
per-Agent PVC), sitting outside that platform/agent split — neither the agent nor
the controller touches it. Both the telemetry data and the UI app-state persist
on operator-managed volumes that survive pod restart and a chart uninstall;
losing them loses telemetry history and nothing else
(`docs/architecture/observability.md @b68af4a`).

## Relationship to logging and usage-tracking

Distinct from two neighbours and non-overlapping: structured operational logs +
the real-identity security audit trail go to stdout (a *logging* concern);
pseudonymized usage analytics live in Postgres as an append-only activity log
read through SQL views (a *usage-tracking* concern, see the usage views in
[db](../sources/db.md)). Telemetry here is the OpenTelemetry-native, explorable
pipeline: a different store (columnar, not Postgres), a different shape (OTLP
logs/traces/metrics), and a different read surface (the exploration UI). Postgres
cannot serve high-volume telemetry — which is the reason this subsystem exists
(`docs/architecture/observability.md @b68af4a`).

## See also

- [zero-trust-credential-gateway](zero-trust-credential-gateway.md) — the gateway
  that MITM-terminates and stamps the trusted attribution header.
- [persistence-substrates](persistence-substrates.md) — the telemetry store as
  the fourth (optional) durable substrate.
- [platform-topology](platform-topology.md) · [agents](../sources/agents.md) — the
  Claude Code export path.
