---
source: dam-agents/dam
commit: b62d21c288162847d7d9918ca7887265448fe2b3
files: [packages/api-server-api/src/modules/experiments/types.ts, packages/api-server-api/src/modules/experiments/router.ts, packages/api-server-api/src/modules/experiments/schemas.ts, packages/api-server/src/modules/experiments/compose.ts, packages/api-server/src/modules/experiments/candidate-route.ts, packages/api-server/src/modules/experiments/domain/arm-status.ts, packages/api-server/src/modules/experiments/domain/experiment-rollup.ts, packages/api-server/src/modules/experiments/domain/trial-prompt.ts, packages/api-server/src/modules/experiments/infrastructure/experiments-repository.ts, packages/api-server/src/modules/experiments/infrastructure/trial-launcher.ts, packages/api-server/src/modules/experiments/services/experiments-service.ts, packages/api-server/src/modules/experiments/services/experiment-arm-sweeper.ts, packages/api-server/src/modules/artifacts/services/artifact-service.ts, packages/api-server/src/modules/artifacts/infrastructure/postgres-artifact-store.ts, packages/api-server/src/apps/harness-api-server/mcp-endpoint.ts, packages/api-server/src/config.ts, packages/api-server/src/index.ts, packages/agent-runtime/src/modules/runtime-channel/drivers/experiment-trigger-plugin.ts, packages/db/src/schema.ts, packages/ui/src/modules/experiments/internal-only.ts, packages/ui/src/modules/platform/lib/routes.ts]
updated: 2026-07-02
---

# Experiments

An **Experiment** races several AI-driven R&D harnesses against one goal and
compares what each produced. The platform's role is deliberately narrow: it
**starts Arms, captures Runs, and presents the comparison**. It never runs the
optimization loop and never interprets a Score — a Score is opaque, captured but
never normalized or ranked across Arms
(`packages/api-server-api/src/modules/experiments/types.ts:45-59 @b62d21c`,
`packages/api-server/src/modules/experiments/domain/experiment-rollup.ts:8-11 @b62d21c`).
Introduced by [#2033](https://github.com/dam-agents/dam/pull/2033).

## The bet: no bespoke harness

There is no experiment-specific harness. A generic [Agent](../entities/agent.md)
runs its own iterate-and-score loop and reports back, driven by two things the
platform already had:

- **The interface is exposed as MCP tools.** Experiments adds two tools to the
  in-pod platform-outbound MCP server — `record_run` (append a scored Run) and
  `finish_arm` (declare the Arm finished)
  (`packages/api-server/src/apps/harness-api-server/mcp-endpoint.ts:502,587 @b62d21c`).
  MCP is the one tool-calling surface every modern harness already speaks, so any
  harness image can report back with no per-harness client. Neither tool takes an
  Experiment or Arm id — attribution is automatic (see below).
- **The contract rides in the Trial prompt.** The composed prompt carries the
  full reporting contract inline as an autonomous-trial directive, so a generic
  harness runs unattended to completion and reports every scored candidate the
  moment its score lands
  (`packages/api-server/src/modules/experiments/domain/trial-prompt.ts:10-13 @b62d21c`).
  It lives in the session-scoped prompt, not a skill: a skill would reach only
  Claude-family harnesses and would leak reporting nags into every non-experiment
  session (`packages/api-server/src/modules/experiments/domain/trial-prompt.ts:1-9 @b62d21c`).

The subsystem lives in the [api-server](../sources/api-server.md) Experiments
context, which mirrors the [Connections](connections-and-contributions.md) owner +
grant-to-many-Agents model — an Experiment belongs to a user and references many
Agents through its Arms.

## Resources

Contract types are in `api-server-api`
(`packages/api-server-api/src/modules/experiments/types.ts @b62d21c`):

- **Experiment** — owner-scoped; holds the shared prompt, a lifecycle status, and
  the Run Ledger (`packages/api-server-api/src/modules/experiments/types.ts:21-30 @b62d21c`).
- **Arm** — one competitor: a reference to one owned Agent plus its Arm Variation.
  Keyed `(experimentId, agentId)`, so the same Agent can't be two Arms of one
  Experiment, but the same harness image can back many raced Agents
  (`packages/api-server-api/src/modules/experiments/types.ts:36-42 @b62d21c`).
- **Arm Variation** — optional free text appended verbatim to the shared prompt at
  launch under an `Arm variation:` header; opaque to the platform, no length cap
  (`packages/api-server-api/src/modules/experiments/schemas.ts:3,17 @b62d21c`,
  `packages/api-server/src/modules/experiments/domain/trial-prompt.ts:19-25 @b62d21c`).
- **Trial** — the single [Session](../entities/session.md) an Arm's Agent opens on
  start; the harness runs its loop here. Carried as `SessionType.ExperimentTrial`
  (`packages/agent-runtime/src/modules/runtime-channel/drivers/experiment-trigger-plugin.ts:31-33 @b62d21c`).
- **Run** — one Ledger entry per loop iteration: a Score plus a Candidate, numbered
  monotonically within its Arm
  (`packages/api-server-api/src/modules/experiments/types.ts:48-59 @b62d21c`).
- **Candidate** — the artifact a Run produced, retrievable as a download (see below).
- **Score** — the number a harness reports; stored as opaque `jsonb`, never
  normalized (`packages/db/src/schema.ts:535 @b62d21c`).
- **Run Ledger** — the append-only `experiment_runs` table; the one genuinely new
  persistence primitive (`packages/db/src/schema.ts:527-549 @b62d21c`).

## Trial flow: launch → wake

Start marks each Arm `running`
(`packages/api-server/src/modules/experiments/infrastructure/experiments-repository.ts:412-422 @b62d21c`),
composes each Trial prompt (shared prompt + Arm Variation + autonomous-trial
directive), and launches it
(`packages/api-server/src/modules/experiments/services/experiments-service.ts:48-79 @b62d21c`).
Launch reuses the runtime channel's trigger/wake primitive: an
`experiment-trigger` event is bumped onto the agent's outbox, the outbox is
enqueued, and the Agent is woken
(`packages/api-server/src/modules/experiments/infrastructure/trial-launcher.ts:20-33 @b62d21c`)
— see [agent-lifecycle](agent-lifecycle.md) and the
[runtime channel](connections-and-contributions.md). On the pod, a runtime-channel
driver handles the event by opening a fresh `ExperimentTrial` session and
submitting the composed task
(`packages/agent-runtime/src/modules/runtime-channel/drivers/experiment-trigger-plugin.ts:25-36 @b62d21c`).
A Trial that fails to launch fails its Arm immediately rather than waiting for the
liveness sweep — an Arm that never started can never report or finish
(`packages/api-server/src/modules/experiments/services/experiments-service.ts:64-77 @b62d21c`).

## Attribution: no id from the caller

The harness passes no Experiment or Arm id. The MCP endpoint resolves the Agent's
identity from the URL `:id`, which the mesh waypoint guarantees matches the network
principal (a label lookup only)
(`packages/api-server/src/apps/harness-api-server/mcp-endpoint.ts:646-664 @b62d21c`).
On each tool call the platform resolves the caller's *active* Arm from that
verified `agentId`
(`packages/api-server/src/apps/harness-api-server/mcp-endpoint.ts:516,591 @b62d21c`):
`resolveActiveArm` returns the Arm of the owner's single `running` Experiment that
contains this Agent AND is itself still `running`, or null — owner-scoped so a
leaked agentId can't reach another tenant, and status-scoped so a finished/stopped
Arm resolves to null
(`packages/api-server/src/modules/experiments/infrastructure/experiments-repository.ts:495-531 @b62d21c`).
Both tools are rejected (`CONFLICT`) once the calling Arm is no longer `running` —
the ledger is closed after Stop or completion
(`packages/api-server/src/modules/experiments/services/experiments-service.ts:232-250,264-282 @b62d21c`).
`finish_arm` is success-only: a harness that gives up simply stops and is caught by
the liveness sweep
(`packages/api-server/src/apps/harness-api-server/mcp-endpoint.ts:588 @b62d21c`).

## Candidate storage: a capped blob in Postgres

On a Run, `record_run` reads the candidate file from the Agent's workspace over the
harness file API and hands the bytes to the api-server
(`packages/api-server/src/apps/harness-api-server/mcp-endpoint.ts:523-559 @b62d21c`).
The api-server stores it **inline as a `bytea` blob** in the `run_artifacts` table
— not on a PVC and not in object storage (agents are egress-locked, so S3 is
unreachable)
(`packages/db/src/schema.ts:560-572 @b62d21c`,
`packages/api-server/src/modules/artifacts/infrastructure/postgres-artifact-store.ts:7-31 @b62d21c`).
Living in Postgres is what makes a Candidate **downloadable while the producing
Agent is hibernated**: Postgres is independent of agent pods, and the api-server —
its sole reader/writer — is always running
(`packages/db/src/schema.ts:551-558 @b62d21c`). The artifact service enforces a
hard size cap before the blob reaches storage — `maxArtifactBytes`, default
**10 MiB** (`MAX_ARTIFACT_BYTES`)
(`packages/api-server/src/modules/artifacts/services/artifact-service.ts:23-30 @b62d21c`,
`packages/api-server/src/config.ts:157-162 @b62d21c`). See
[persistence-substrates](persistence-substrates.md) and [db](../sources/db.md).

**Download is owner-scoped.** The candidate route resolves the Run *through the
owner's Experiment first* — a non-owned Experiment reads back as 404, so the opaque
storage key is never trusted from the client
(`packages/api-server/src/modules/experiments/candidate-route.ts:21-48 @b62d21c`).

## Completion and liveness

Per-Arm status is the source of truth for completion. An Arm moves
`pending → running` at launch, then to one terminal state: **completed**
(`finish_arm`), **failed** (Inactivity Deadline or launch failure), or **stopped**
(the user Stopped the Experiment while the Arm was running)
(`packages/api-server-api/src/modules/experiments/types.ts:8-19 @b62d21c`). An
**Experiment becomes completed once every Arm is terminal**, regardless of the mix
— there is no Experiment-level `failed`, because the platform reports the
comparison, it never judges the whole Experiment failed
(`packages/api-server/src/modules/experiments/domain/arm-status.ts:3-22 @b62d21c`,
`packages/api-server/src/modules/experiments/infrastructure/experiments-repository.ts:161-179 @b62d21c`).
Every terminal transition locks the experiment row then the arm row (a shared lock
order, no deadlock), requires both `running`, and flips the Experiment if that was
the last non-terminal arm — an atomic conditional transition
(`packages/api-server/src/modules/experiments/infrastructure/experiments-repository.ts:181-227 @b62d21c`).

The **Inactivity Deadline** is the liveness guarantee that lets a started
Experiment always reach a terminal state. A background sweep marks a `running` Arm
`failed` if it records no Run and never calls `finish_arm` within a configured
window (`experimentArmInactivitySeconds`, default **1 hour**); the clock
(`last_activity_at`) resets on each Run
(`packages/api-server/src/config.ts:171-175 @b62d21c`,
`packages/api-server/src/modules/experiments/infrastructure/experiments-repository.ts:466-474 @b62d21c`).
Without it, a harness that crashes, forgets to finish, or hibernates mid-loop would
strand its Arm forever — the one-shot Trial prompt is never re-issued
(`packages/api-server/src/modules/experiments/services/experiment-arm-sweeper.ts:1-21 @b62d21c`).
The sweep is safe on every api-server replica: each reap re-checks the deadline
under the row lock, so a race no-ops on the already-terminal row, and a
**randomized start offset** keeps replicas from scanning in lockstep
(`packages/api-server/src/modules/experiments/services/experiment-arm-sweeper.ts:62-104 @b62d21c`).

Stop is the distinct user-driven terminal path: in one transaction it flips the
Experiment to `stopped` and moves every still-`running` Arm to `stopped`
(`packages/api-server/src/modules/experiments/infrastructure/experiments-repository.ts:310-355 @b62d21c`).

## Where the code lives

- **Contract** (resources, service interface, MCP tool inputs):
  `packages/api-server-api/src/modules/experiments/`.
- **Implementation** (service, repository, Trial prompt, launcher, liveness sweep):
  `packages/api-server/src/modules/experiments/`; Candidate blob storage in the
  sibling `packages/api-server/src/modules/artifacts/`; MCP tools in
  `packages/api-server/src/apps/harness-api-server/mcp-endpoint.ts`.
- **Trial-side event handler:**
  `packages/agent-runtime/src/modules/runtime-channel/drivers/experiment-trigger-plugin.ts`.
- **Tables:** `packages/db/src/schema.ts` (migrations `0005`–`0009`). See [db](../sources/db.md).
- **UI (internal-only):** `packages/ui/src/modules/experiments/`, hidden behind the
  localStorage flag `platform-debug:show-experiments`
  (`packages/ui/src/modules/experiments/internal-only.ts:1 @b62d21c`,
  `packages/ui/src/modules/platform/lib/routes.ts:79-83 @b62d21c`). See [ui](../sources/ui.md).

## See also

- [api-server](../sources/api-server.md) · [ui](../sources/ui.md) · [db](../sources/db.md)
- [connections & contributions](connections-and-contributions.md) — the owner/grant
  model and the trigger/wake primitive Experiments reuses.
- [agent lifecycle](agent-lifecycle.md) — wake-on-trigger, hibernation.
- [persistence substrates](persistence-substrates.md) — why the Candidate is a
  Postgres blob.
