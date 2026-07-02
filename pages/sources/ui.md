---
source: dam-agents/dam
commit: b62d21c288162847d7d9918ca7887265448fe2b3
files: [packages/ui/src/, docs/architecture/platform-topology.md]
updated: 2026-07-02
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
`providers`, `egress-rules`, `repos`, `skills`, `files`, `templates`,
`terms`, `usage`, `settings`, `platform`, plus an internal-only `experiments`.

The `secrets/` module was removed in the
[secrets→connections cutover](../concepts/connections-and-contributions.md#secrets--connections-cutover-2200)
([#2200](https://github.com/dam-agents/dam/pull/2200)) along with the
`connections-picker` and monolithic `add-agent-dialog`; provider setup now flows
through `providers/` + `connections/`, with Anthropic-key validation calling
`connections.testAnthropic`
(`packages/ui/src/modules/connections/api/mutations.ts:57-63 @b62d21c`).

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

The session **Config panel** exposes a **Model settings** section
(`ModelSettingsPanel`) — one dropdown per catalog option group (model / mode /
effort), shown only for agents whose harness advertises a `harness-config` catalog
(`packages/ui/src/modules/sessions/components/model-settings-panel.tsx:32-72 @b62d21c`).
The displayed value is a **live read** from the agent (no optimistic update): a
change fires the one-shot `harnessConfig.set` mutation, polls `settled`, then
re-reads current in one step; "Applies to new sessions" — a running session keeps
its startup settings
(`packages/ui/src/modules/agents/api/harness-config.ts:50-71 @b62d21c`). This
replaced the old ACP-based session-config popover (deleted
`session-config-popover.tsx`, `use-acp-config-cache.ts`, `store/session-config.ts`
[#1270](https://github.com/dam-agents/dam/pull/1270)); model/mode/config now ride
the outbound runtime channel, not an ACP session. See
[connections](../concepts/connections-and-contributions.md#harness-config--per-agent-modelmodeconfig).

A new **`experiments/`** module (list / detail / wizard views) is **internal-only**,
hidden behind the localStorage flag `platform-debug:show-experiments`; both the
route resolver and the nav rail gate on it
(`packages/ui/src/modules/experiments/internal-only.ts:1 @b62d21c`,
`packages/ui/src/modules/platform/lib/routes.ts:79-83 @b62d21c`). See
[Experiments](../concepts/experiments.md).

## See also

- [platform-topology](../concepts/platform-topology.md) — UI ↔ api-server protocols.
- [HITL approvals](../concepts/hitl-approvals.md) — the inbox/permission UX this renders.
- [Keycloak](../entities/keycloak.md) — the identity provider UI auth flows through.
