---
source: dam-agents/dam
commit: 662ebe4c88029788829246170e17465c69523521
files: [docs/architecture/agent-lifecycle.md]
updated: 2026-06-19
---

# Agent lifecycle

An [Agent](../entities/agent.md) is the durable, owned, runnable resource: a
single CR holds definition and runtime state, and its StatefulSet scales between
0 and 1 replica as it hibernates and wakes
(`docs/architecture/agent-lifecycle.md @662ebe4`). Three actors drive it: the
**UI** (the only management surface), the api-server's **scheduler**, and the
controller's **idle checker**. There is **no stored desired state** —
running-vs-hibernated is observed status the controller derives from activity.

## Phases

- **Create** — the [api-server](../sources/api-server.md) writes a new Agent CR
  (image/mounts copied from a [Template](../entities/template.md) at create time
  if any, plus env, secret refs, allowed users). The
  [controller](../sources/controller.md) reconciles the paired resources. A
  private-registry pull Secret, if any, is written *before* the CR and rolled
  back on failure. Pod env composes three layers (platform envs →
  `credentialEnvVars` → `agent.env`, last wins). Template env contributes at
  create time only — editing a Template never re-flows into a running Agent.
- **Wake** — every caller routes through a single **reachability primitive**:
  the controller-published `Ready` condition is the authoritative "can I call
  this pod?". It pokes the `last-activity` annotation (the reconciler scales up
  recently-active agents), single-flights concurrent waits, and bumps activity
  on every successful call. Three wake paths: connect-driven (ACP frame, UI tab
  or channel inbound), schedule-driven (trigger event + poke, no wait), and
  skills-management-driven. Bounded wakes give up after two minutes.
- **Trigger fire** — [Schedules](../entities/schedule.md) are Postgres rows armed
  as delayed Redis jobs; the next occurrence is computed from cron/RRULE in the
  schedule's timezone, skipping enabled quiet-hours windows (suppressed, not
  deferred). A due fire inserts a `trigger` event into the runtime outbox (same
  txn that bumps the agent version), pokes the agent awake, and the delivery
  worker pushes `applyState` once `Ready`. Every event carries a TTL so a
  long-down agent doesn't replay a backlog. agent-runtime's handler opens an ACP
  session and submits the task as a prompt; a per-schedule last-fire timestamp on
  the PVC prevents double-runs. Fresh schedules `session/new` each fire;
  continuous schedules `session/resume` the same session.
- **Hibernate** — the idle checker probes agent-runtime's `/api/status`; the
  runtime is authoritative about its own `idle` flag (false while a turn runs,
  prompts queue, a request awaits a client, or a terminal/SSH is open). Idle
  long enough → scale to zero. PVC, Secret, Service, NetworkPolicy persist;
  ephemeral-filesystem changes are lost.
- **Delete** — the api-server deletes the CR; the controller tears down owned
  resources and explicitly reclaims PVCs (not cascade-deleted by K8s). Schedules
  are independent Postgres rows and survive as orphans unless cascaded.

## Sessions inside the pod

The harness child runs for the pod's lifetime, not per-connection. See
[Session](../entities/session.md) for the in-memory log, resume mediation,
terminal/SSH models, and mode switching.

## Forks

[Forks](../entities/fork.md) are the third durable concept: a per-turn run of an
Agent derivative that reconciles to a Kubernetes **Job** (run-to-completion),
used for Slack foreign repliers.

## See also

- [platform-topology](platform-topology.md) · [persistence-substrates](persistence-substrates.md) · [connections-and-contributions](connections-and-contributions.md)
