---
source: dam-agents/dam
commit: 662ebe4c88029788829246170e17465c69523521
files: [docs/architecture/persistence.md]
updated: 2026-06-19
---

# Persistence substrates

DAM persists state on **three durable substrates**, split between platform and
agent (`docs/architecture/persistence.md @662ebe4`):

| Substrate | Owner / writer | Holds |
| --- | --- | --- |
| **Postgres** (Application State) | api-server only | Channel routing, identity/auth + API keys, skills catalog, activity log + agent mirror, [schedules](../entities/schedule.md). Queryable with no pod running. See [db](../sources/db.md). |
| **Custom resources** (Infra State) | `spec`: api-server · `status`: controller | [Agent](../entities/agent.md), [Fork](../entities/fork.md). |
| **Per-Agent PVCs** | agent process | Workspace + `$HOME`: git checkouts, tool caches, agent memory, skills, and the **session-metadata state file** (sole source of truth for per-session mode/type/scheduleId). See [Session](../entities/session.md). |

The controller never touches Postgres; the api-server never writes `status`. The
agent's only durable surface is the PVC — everything the platform knows *about*
the agent is mirrored onto Postgres or the CR, never written by the agent.

## Choosing Postgres vs. the K8s API

A resource belongs on a **CRD iff the controller reconciles it**. If only the
api-server reads/writes it, it belongs in **Postgres** — the spec/status
single-writer split exists only to coordinate api-server and controller, so
without a controller reader it has no purpose. This is why
[Schedules](../entities/schedule.md) live in Postgres and
[Templates](../entities/template.md) never became a CRD.

## CR ownership & annotations

`spec` = user intent (api-server-written, K8s-validated at admission); `status`
= observed state (controller-written via the status subresource); conditions
(`Ready`, `AgentPodReady`, `GatewayPodReady`, `Reconciled`) are the source of
truth. High-frequency signals (last-activity, active-session marker, roll
trigger) live on **annotations**, independently patchable.

## Lifetime

PVC, Postgres, and CR all survive pod restart, hibernate, wake, and api-server/
controller restart. On **Agent delete**: the api-server marks the agent row
deleted and removes the CR; the controller removes the PVCs explicitly (opting
out of StatefulSet retain-on-delete). Schedules survive as orphans unless
cascaded; sessions follow the PVC.

## Warm PVC pool

First-start PVC provisioning is slow, so the controller can keep a **warm pool**
of pre-provisioned, already-bound spare PVCs (per size, immediate-binding
StorageClass, disabled by default). A new Agent **claims** a matching spare
atomically (stamping the owner label, removing the available marker); a claimed
spare becomes an ordinary per-Agent PVC. Unclaimed spares carry a pool label but
**no owner label**, so the orphan-PVC sweep ignores them.

## Security boundary

The PVC is a **shared mutable surface** across every session, trigger, fork, and
channel-driven prompt on the same Agent — treat workspace contents as
adversarial input (a scheduled job can plant a file that prompt-injects a later
session). The platform does not sandbox writes within the workspace; mitigations
are the [credential gateway](zero-trust-credential-gateway.md), NetworkPolicy,
and [forks](../entities/fork.md).

## See also

- [platform-topology](platform-topology.md) · [agent-lifecycle](agent-lifecycle.md) · [db](../sources/db.md)
