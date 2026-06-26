---
source: dam-agents/dam
commit: d507c05fb3683c901473b5166766db03ce14fb29
files: [packages/ui/src/, docs/architecture/platform-topology.md]
updated: 2026-06-26
---

# ui

A **React + Vite** single-page app served by the [api-server](api-server.md),
and the only management surface — creating, configuring, hibernating, and
deleting agents all flow through it (`docs/architecture/agent-lifecycle.md @4a48ae2`).
It uses **tRPC over HTTP** for resource management and permission flows, and
opens **one ACP WebSocket per active session** for bidirectional agent
communication — permission prompts, tool calls, and streaming output all share
that connection (`docs/architecture/platform-topology.md @4a48ae2`). The largest
package in the repo (~240 files).

## Code layout (`packages/ui/src/`)

Top-level wiring: `main.tsx`, `app.tsx`, `api.ts`, `trpc.ts`, `auth.ts`,
`store.ts`, `query-client.ts`, `brand.ts` (`packages/ui/src/ @4a48ae2`).
Feature modules under `src/modules/` mirror the api-server's bounded contexts
(`packages/ui/src/modules/ @4a48ae2`):

`acp`, `agents`/`sandboxes`, `sessions`, `schedules`, `approvals`, `connections`,
`providers`, `secrets`, `egress-rules`, `repos`, `skills`, `files`, `templates`,
`terms`, `usage`, `settings`, `platform`.

> The UI surfaces the **Agent** domain object under the user-facing name
> **Sandbox** (`docs/ubiquitous-language.md @4a48ae2`); "Agent" remains the
> code/domain term — hence both `agents/` and `sandboxes/` modules exist.

## See also

- [platform-topology](../concepts/platform-topology.md) — UI ↔ api-server protocols.
- [HITL approvals](../concepts/hitl-approvals.md) — the inbox/permission UX this renders.
- [Keycloak](../entities/keycloak.md) — the identity provider UI auth flows through.
