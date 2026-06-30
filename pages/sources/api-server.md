---
source: dam-agents/dam
commit: 70c53ae1a47512cfe06c0eb2982102d899e45f5a
files: [packages/api-server/, packages/api-server-api/, deploy/helm/platform/templates/apiserver/rbac.yaml, docs/architecture/platform-topology.md]
updated: 2026-06-30
---

# api-server

The TypeScript backend and the platform's only user-facing surface. It hosts
the user API, relays all agent traffic, and is the **sole writer of every
`Agent`/`Fork`/`Run` `spec`** and of all Postgres state
(`docs/architecture/platform-topology.md @4a48ae2`,
`docs/architecture/persistence.md @4a48ae2`,
`deploy/helm/platform/templates/apiserver/rbac.yaml:57-62 @70c53ae`).

## Two listeners

- **Public port** — user-authenticated tRPC, REST (OAuth callbacks, health,
  unauthenticated `version`), and the ACP relay WebSocket. Auth accepts either a
  Keycloak JWT or a `pk_`-prefixed API key in the same `Authorization: Bearer`
  slot, dispatched by prefix; ownership is checked here, then `Authorization` is
  rewritten to the per-agent runtime token before forwarding — agent-runtime
  never sees user identity.
- **Harness port** — cluster-internal only, no user auth: MCP tool calls, the
  runtime-channel `hello` from agent pods, and the **`dam-run` relay** — a
  manually-wired WebSocket upgrade on `/api/agents/<id>/run` that stands up an
  ephemeral [Run](../entities/run.md) executor pod and relays its `/api/exec`
  stdio (`packages/api-server/src/apps/harness-api-server/app.ts:67-114 @70c53ae`).

It proxies **all** ACP to agent pods (clients never dial pods directly), waking
a hibernated agent before forwarding the first frame
(`docs/architecture/platform-topology.md @4a48ae2`).

## Module layout (`packages/api-server/src/modules/`)

Domain modules, each its own bounded context
(`packages/api-server/src/modules/ @4a48ae2`):

| Module | Responsibility |
| --- | --- |
| `agents`, `templates`, `forks`, `runs` | Agent CRUD + spec assembly, template catalog, per-turn forks, and ephemeral single-command run executors (the `dam-run` backend). See [Agent](../entities/agent.md), [Template](../entities/template.md), [Fork](../entities/fork.md), [Run](../entities/run.md). |
| `channels` | Slack/Telegram adapters + inbound relay. See [channels](../concepts/platform-topology.md). |
| `schedules` | RRULE/cron schedule loop, quiet hours, firing. See [Schedule](../entities/schedule.md). |
| `approvals`, `egress-rules` | HITL ext_authz gate and persistent egress rules. See [HITL approvals](../concepts/hitl-approvals.md). |
| `connections`, `runtime-delivery` | Unified connection/contribution model + the transactional outbox worker. See [connections](../concepts/connections-and-contributions.md). |
| `secrets`, `secret-store` | K8s-Secret credential storage (owner-labelled), GitHub PAT pairs. |
| `skills`, `repos` | Skill source catalog, install/publish orchestration, git repos. |
| `audit`, `usage` | Structured audit trail and append-only usage tracking. |
| `api-keys`, `terms` | Headless API keys; Terms-of-Use acceptance gate. |
| `files` | Streamed bundled file import proxied to agent-runtime. |

`src/sagas/` holds event-driven projections (e.g. the agent ownership mirror),
`src/bootstrap/` and `src/apps/` wire the two listeners, and `src/proto-gen/`
holds generated gRPC types for the ext_authz Check. The harness app also runs a
**boot sweep** that deletes any `Run` CR a fresh process holds no live relay for
(a crash-leaked executor would run untethered), and caps concurrent `dam-run`
executors at 16 per agent — the **sole** bound on `dam-run` recursion, since
executors borrow the parent's gateway identity rather than getting their own SA
(`packages/api-server/src/apps/harness-api-server/app.ts:100-110 @70c53ae`,
`packages/api-server/src/apps/harness-api-server/harness-run-relay.ts:11-20 @70c53ae`). The wire contracts (tRPC
routers, CRD types, harness router) live in the sibling `api-server-api`
package (`packages/api-server-api/src/ @4a48ae2`).

## See also

- [platform-topology](../concepts/platform-topology.md) — where api-server sits.
- [zero-trust-credential-gateway](../concepts/zero-trust-credential-gateway.md) — the ext_authz Check the api-server answers.
- [persistence-substrates](../concepts/persistence-substrates.md) — what it writes to Postgres vs. the K8s API.
- [Run](../entities/run.md) — the ephemeral executor the harness `dam-run` relay stands up and reaps.
