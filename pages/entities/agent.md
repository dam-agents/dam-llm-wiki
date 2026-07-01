---
source: dam-agents/dam
commit: b68af4ad0a0c427c856b0e5ba245feb8c2085a72
files: [docs/ubiquitous-language.md, docs/architecture/persistence.md, packages/controller/api/v1/agent_types.go, packages/controller/pkg/reconciler/resources.go]
updated: 2026-07-01
---

# Agent

The **durable, owned, runnable resource** — definition, runtime state, and
lifecycle in one (`docs/ubiquitous-language.md @662ebe4`). The former
`Instance`/template pair was collapsed into it; there is no separate instance
resource. The UI surfaces it to users as a **Sandbox**, but "Agent" is the
domain/code term.

## Shape

A single `Agent` custom resource (`agent-platform.ai/v1`,
`packages/controller/api/v1/agent_types.go @662ebe4`) with the
[spec/status split](../concepts/persistence-substrates.md):

- **`spec`** (api-server-written) — image, mount declarations, secret refs,
  image-pull secret ref, granted secret + connection IDs, allowed users.
  Optionally derived from a [Template](template.md) at create time. The `spec.env`
  field is **retained but no longer read** since
  [#1899](https://github.com/dam-agents/dam/pull/1899) (`@d507c05`): user-typed env
  now lives in Postgres `agent_env` and reaches the pod over the runtime channel,
  not the pod spec — the controller projects only chart-level platform defaults
  into pod env (`packages/controller/pkg/reconciler/resources.go:113-114 @d507c05`).
  See [agent-lifecycle](../concepts/agent-lifecycle.md). An optional
  `spec.hibernationTimeout` (`*metav1.Duration`) overrides the chart-wide idle
  timeout for this Agent — omitted inherits the default, a positive value sets a
  per-agent idle window, and `"0s"` disables hibernation entirely
  ([#1373](https://github.com/dam-agents/dam/pull/1373),
  `packages/controller/api/v1/agent_types.go:38-40 @b68af4a`); the UI writes it in
  minutes and both controller and api-server resolve the *effective* value. See
  [the idle decision](../concepts/agent-lifecycle.md#the-idle-decision).
- **`status`** (controller-written) — observed conditions (`Ready`,
  `AgentPodReady`, `GatewayPodReady`, `Reconciled`).
- **annotations** — high-frequency signals: `last-activity`, active-session
  marker, roll trigger.

There is **no stored desired state** — running-vs-hibernated is observed status
derived from activity (`docs/architecture/persistence.md @662ebe4`). The
reserved ID prefix `agent-` is minted by the controller; the api-server forbids
Agent names beginning with it, and the CLI uses it as the ID-vs-name split.

## Runs as

A paired set of pods reconciled by the [controller](../sources/controller.md):
the [agent-runtime](../sources/agent-runtime.md) pod and its
[Envoy gateway](envoy-gateway.md) pod, both scaling 0↔1. Durable state is the
per-Agent [PVC](../concepts/persistence-substrates.md).

## See also

- [agent-lifecycle](../concepts/agent-lifecycle.md) · [Template](template.md) · [Fork](fork.md) · [Session](session.md) · [Schedule](schedule.md)
