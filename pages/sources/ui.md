---
source: dam-agents/dam
commit: 662ebe4c88029788829246170e17465c69523521
files: [packages/ui/src/, docs/architecture/platform-topology.md]
updated: 2026-06-19
---

# ui

A **React + Vite** single-page app served by the [api-server](api-server.md),
and the only management surface — creating, configuring, hibernating, and
deleting agents all flow through it (`docs/architecture/agent-lifecycle.md @662ebe4`).
It uses **tRPC over HTTP** for resource management and permission flows, and
opens **one ACP WebSocket per active session** for bidirectional agent
communication — permission prompts, tool calls, and streaming output all share
that connection (`docs/architecture/platform-topology.md @662ebe4`). The largest
package in the repo (~240 files).

## Code layout (`packages/ui/src/`)

Top-level wiring: `main.tsx`, `app.tsx`, `api.ts`, `trpc.ts`, `auth.ts`,
`store.ts`, `query-client.ts`, `brand.ts` (`packages/ui/src/ @662ebe4`).
Feature modules under `src/modules/` mirror the api-server's bounded contexts
(`packages/ui/src/modules/ @662ebe4`):

`acp`, `agents`/`sandboxes`, `sessions`, `schedules`, `approvals`, `connections`,
`providers`, `secrets`, `egress-rules`, `repos`, `skills`, `files`, `templates`,
`terms`, `usage`, `settings`, `platform`.

> The UI surfaces the **Agent** domain object under the user-facing name
> **Sandbox** (`docs/ubiquitous-language.md @662ebe4`); "Agent" remains the
> code/domain term — hence both `agents/` and `sandboxes/` modules exist.

## See also

- [platform-topology](../concepts/platform-topology.md) — UI ↔ api-server protocols.
- [HITL approvals](../concepts/hitl-approvals.md) — the inbox/permission UX this renders.
- [Keycloak](../entities/keycloak.md) — the identity provider UI auth flows through.
