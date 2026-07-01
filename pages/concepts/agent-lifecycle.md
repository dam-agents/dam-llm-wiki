---
source: dam-agents/dam
commit: b68af4ad0a0c427c856b0e5ba245feb8c2085a72
files: [docs/architecture/agent-lifecycle.md, packages/controller/pkg/reconciler/resources.go, packages/controller/pkg/reconciler/hibernation.go, packages/controller/pkg/reconciler/idlechecker.go, packages/controller/api/v1/agent_types.go, packages/api-server/src/modules/agents/domain/spec-assembly.ts, packages/api-server/src/modules/agents/infrastructure/agent-mappers.ts, packages/api-server/src/modules/runtime-delivery/services/state-builder.ts]
updated: 2026-07-01
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
  back on failure. As of [#1899](https://github.com/dam-agents/dam/pull/1899)
  (`@d507c05`) the controller composes pod env from **platform wiring + any
  chart-level platform defaults only** — `spec.env` is no longer read
  (`packages/controller/pkg/reconciler/resources.go:113-114 @d507c05`). Everything
  tied to an Agent's *configuration* — user-typed env (the UI Environment editor),
  connection-/secret-derived env, and template env — rides the **runtime channel**
  as `env`-kind [contributions](connections-and-contributions.md) instead, applied
  at the next idle turn with **no pod roll**. User-typed and template env are
  stored in Postgres `agent_env`; the CR's `spec.env` field is retained but no
  longer read. Template env is seeded into `agent_env` at **create time only** —
  editing a Template never re-flows into a running Agent.
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
- **Hibernate** — see [The idle decision](#the-idle-decision) below. Idle long
  enough → scale to zero. PVC, Secret, Service, NetworkPolicy persist;
  ephemeral-filesystem changes are lost.
- **Delete** — the api-server deletes the CR; the controller tears down owned
  resources and explicitly reclaims PVCs (not cascade-deleted by K8s). Schedules
  are independent Postgres rows and survive as orphans unless cascaded.

## The idle decision

Hibernation scales an idle Agent's StatefulSets to zero to reclaim CPU and
memory; the next activity wakes it. Whether an Agent is "idle" is **derived from
observed activity, never stored**, and the derivation is deliberately split
across **two independent checks that must both agree** before a pod scales down
(`docs/architecture/agent-lifecycle.md @b68af4a`):

1. **Activity annotations** — the controller's idle checker runs
   `shouldRun(annotations, effectiveTimeout, now)`, the *same* gate the reconciler
   uses to scale *up*, so scale-down and scale-up can never disagree. It keeps the
   Agent awake while `active-session` is `"true"` or `last-activity` is within the
   idle timeout, and **fails open** — a missing/unparseable stamp (or a
   non-positive timeout) keeps it running, so hibernation only ever follows a
   *positive* idle signal, never absent data
   (`packages/controller/pkg/reconciler/hibernation.go:26-42 @b68af4a`).
2. **agent-runtime's live `idle` flag** — only if check 1 says idle does the
   checker probe the pod's `/api/status` (3 s timeout). The runtime is
   authoritative about its own idleness and reports one boolean; an unreachable
   pod counts as *not busy*, permitting hibernation
   (`packages/controller/pkg/reconciler/idlechecker.go:88-102 @b68af4a`).

The scan runs on a timer of **1/6 of the timeout, clamped to 30 s–5 min**
(`packages/controller/pkg/reconciler/idlechecker.go:62-72 @b68af4a`).

The runtime `idle` flag is false while a prompt turn is in flight, prompts queue,
an agent-initiated request awaits the client, or a **terminal (PTY)** is open. It
does **not** see SSH (its own `sshd`, outside PTY tracking) — an SSH session
leans entirely on the `active-session` annotation — and an attached *chat* with
no turn running reads as `idle` (the flag tracks work, not watchers; such a chat
stays awake via `active-session`). What the probe uniquely catches is in-flight
work no connection holds and `last-activity` no longer covers: a scheduled run
outlasting the timeout, or a turn still running after its tab closed.

**The blind spot — background work.** Every signal tracks sessions and
connections, never the processes behind them. Work detached from a session (a
background job outliving its terminal, an async task a tool kicked off) moves none
of them: both checks agree the Agent is idle and the controller hibernates the pod
**mid-job, killing the work**. This is deliberate — the platform *can* see
processes running but can't tell a working batch job from an always-on model
gateway or a leaked orphan, so a process-based keep-awake signal would either pin
the pod open forever or let one leaked process defeat hibernation. It keys only on
what it *can* attribute — sessions and connections — and accepts the blind spot.

**The per-agent hibernation timeout.** Since the platform can't *detect* that
work, it lets an operator *budget* for it. Each Agent carries an optional
`spec.hibernationTimeout` (`*metav1.Duration`): unset inherits the chart-wide
default, a positive value sets a per-agent idle window, and **`"0s"` disables
hibernation** so the Agent never scales down
(`packages/controller/api/v1/agent_types.go:38-40 @b68af4a`). The controller
resolves the effective value (override else global) via `effectiveIdleTimeout` and
feeds it to the same `shouldRun` gate used for scale-up and scale-down
(`packages/controller/pkg/reconciler/hibernation.go:44-50 @b68af4a`,
`packages/controller/pkg/reconciler/idlechecker.go:117-128 @b68af4a`). The
[sandbox settings](../sources/ui.md) expose it as a minutes field showing the
*effective* value (an Agent with no override displays the inherited default), and
the api-server surfaces the same as `effectiveHibernationTimeoutMin`
([#1373](https://github.com/dam-agents/dam/pull/1373),
`packages/api-server/src/modules/agents/domain/spec-assembly.ts:4-10 @b68af4a`,
`packages/api-server/src/modules/agents/infrastructure/agent-mappers.ts:148-151 @b68af4a`).
It is a blunt instrument, not a fix for the blind spot: a longer window (or `0s`)
keeps a known-background-work Agent alive but doesn't make the work visible, and
there is no auto-reclaim — a long or disabled timeout holds CPU, memory, and the
harness open until lowered by hand.

## Sessions inside the pod

The harness child runs for the pod's lifetime, not per-connection. See
[Session](../entities/session.md) for the in-memory log, resume mediation,
terminal/SSH models, and mode switching.

## Forks

[Forks](../entities/fork.md) are the third durable concept: a per-turn run of an
Agent derivative that reconciles to a Kubernetes **Job** (run-to-completion),
used for Slack foreign repliers.

## Run executors (`dam-run`)

A [Run](../entities/run.md) is a different ephemeral shape: a **single-command
executor** behind the in-pod `dam-run` CLI, added in
[#2120](https://github.com/dam-agents/dam/pull/2120). `dam-run <cmd>` runs the
command in a *separate* sandbox pod that shares the calling pod's image,
configuration, and RWX workspace, with stdio streamed through a PTY so it reads
as a local invocation (`docs/architecture/agent-lifecycle.md @70c53ae`). The pod
boots [agent-runtime](../sources/agent-runtime.md) in **exec-only mode** (an
`/api/exec` endpoint plus health, no runtime-channel hello) and stands up no
infrastructure of its own — just a bare Pod plus one egress NetworkPolicy into
the parent's existing gateway, whose credentials and egress boundary it borrows
wholesale. The flow is synchronous over one WebSocket; when the stream closes the
api-server deletes the `Run` and K8s GC reaps the executor. Unlike a Fork it
runs as the **parent Agent's own owner**, so deleting the Agent cascade-deletes
any in-flight `Run` (`docs/architecture/agent-lifecycle.md @70c53ae`).

## See also

- [platform-topology](platform-topology.md) · [persistence-substrates](persistence-substrates.md) · [connections-and-contributions](connections-and-contributions.md)
- [Fork](../entities/fork.md) · [Run](../entities/run.md) — the two ephemeral Agent-derived runs.
