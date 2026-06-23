---
source: dam-agents/dam
commit: 380cb06d1d60bca40fa703b77e13a16ec96eedf7
files: [docs/ubiquitous-language.md, docs/architecture/persistence.md, packages/controller/api/v1/agent_types.go]
updated: 2026-06-23
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

- **`spec`** (api-server-written) — image, mount declarations, env, secret refs,
  image-pull secret ref, granted secret + connection IDs, allowed users.
  Optionally derived from a [Template](template.md) at create time.
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
