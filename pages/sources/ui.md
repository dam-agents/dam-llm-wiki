---
source: dam-agents/dam
commit: b68af4ad0a0c427c856b0e5ba245feb8c2085a72
files: [packages/ui/src/, docs/architecture/platform-topology.md]
updated: 2026-07-01
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

The sandbox **settings** view exposes the per-agent hibernation timeout as a
minutes field (`0` = never hibernate), showing the *effective* value so an Agent
with no override displays the inherited default rather than a blank
([#1373](https://github.com/dam-agents/dam/pull/1373),
`packages/ui/src/modules/sandboxes/components/hibernation-timeout-field.tsx:14-45 @b68af4a`,
`packages/ui/src/modules/sandboxes/views/sandbox-settings-view.tsx:108-120 @b68af4a`);
see [the idle decision](../concepts/agent-lifecycle.md#the-idle-decision).
Form primitives are being converged onto a shared `FormField` and unified
`select`/`textarea` components under `components/ui/`
([#2130](https://github.com/dam-agents/dam/pull/2130),
[#2115](https://github.com/dam-agents/dam/pull/2115)).

## See also

- [platform-topology](../concepts/platform-topology.md) — UI ↔ api-server protocols.
- [HITL approvals](../concepts/hitl-approvals.md) — the inbox/permission UX this renders.
- [Keycloak](../entities/keycloak.md) — the identity provider UI auth flows through.
