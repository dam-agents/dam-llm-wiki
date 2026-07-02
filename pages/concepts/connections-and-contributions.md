---
source: dam-agents/dam
commit: b62d21c288162847d7d9918ca7887265448fe2b3
files: [docs/ubiquitous-language.md, docs/architecture/agent-lifecycle.md, docs/architecture/connections.md, docs/architecture/persistence.md, packages/api-server-api/src/modules/connections/providers.ts, packages/agent-runtime-api/src/modules/runtime/types.ts, packages/api-server/src/modules/runtime-delivery/services/state-builder.ts, packages/api-server/src/modules/agents/infrastructure/agent-env-repository.ts, packages/api-server/src/modules/connections/domain/gitconfig-contribution.ts, packages/api-server/src/modules/connections/domain/catalog.ts, packages/api-server/src/modules/connections/domain/connection-sds.ts, packages/api-server/src/modules/connections/services/connections-service.ts, packages/api-server/src/modules/connections/migration/secrets-to-connections.ts, packages/api-server/src/modules/connections/infrastructure/github-identity.ts, packages/api-server/src/modules/connections/services/oauth-flow.ts, packages/agent-runtime/src/modules/runtime-channel/manifest.ts, packages/agent-runtime/src/modules/runtime-channel/dispatcher.ts, packages/agent-runtime/src/modules/runtime-channel/event-dispatcher.ts, packages/agent-runtime/src/modules/runtime-channel/compose.ts, packages/agent-runtime/src/modules/runtime-channel/drivers/harness-config-plugin.ts, packages/agent-runtime/src/modules/runtime-channel/infrastructure/model-discovery.ts, packages/agents/pi-agent/runtime-manifest.yaml, packages/platform-base/runtime-manifest.yaml]
updated: 2026-07-02
---

# Connections & contributions

The **runtime channel** is how the [api-server](../sources/api-server.md)
delivers declarative configuration and one-shot events to a running
[agent-runtime](../sources/agent-runtime.md) pod, via a transactional
**outbox + worker** (`docs/ubiquitous-language.md @4a48ae2`,
`docs/architecture/agent-lifecycle.md @4a48ae2`).

As of [#2200](https://github.com/dam-agents/dam/pull/2200) (`@b62d21c`) the
Connection/Contribution model is the **sole, shipped credential model** — the
standalone `secrets` module and the parallel `OAuthAppDescriptor`/`ProviderPreset`
registries have been retired into it (see [the cutover](#secrets--connections-cutover-2200)).

## Model

- **Connection Template** — a code-level catalog entry shipping defaults
  (pre-filled `AuthConfig` + `Contribution[]` + input fields). Premade (GitHub,
  Anthropic) and "Custom" (MCP/OAuth/Header) share one shape.
- **Connection** — `{ auth, contributions, inputs, templateId? }`, no `kind`
  discriminator; identity is the contributions + auth it carries.
- **Contribution** — one typed unit a Connection emits per Agent when granted:
  `env`, `egress-allow`, `egress-inject`, `file`, `mcp-entry`, `skill-ref`
  (extensible discriminated union,
  `packages/agent-runtime-api/src/modules/runtime/types.ts:3-9 @380cb06`). A
  `file` contribution carries a `mergeMode` (e.g. `section-marker`) so the
  runtime can update one block without clobbering the rest of the file
  (`packages/agent-runtime-api/src/modules/runtime/types.ts:61 @380cb06`).
- **AuthConfig** — `oauth` | `header` | `none`; `header` covers any
  header-injected static credential, distinguished by `headerName` + `valueFormat`.

### Storage

A Connection is one Postgres `connections` row (`id, owner,
templateId, name, inputs, auth, contributions`) plus one per-Connection K8s Secret
holding the raw `value` and the baked `host-*.sds.yaml` SDS files the
[Envoy gateway](zero-trust-credential-gateway.md) reads
(`packages/api-server/src/modules/connections/services/connections-service.ts:368-388 @b62d21c`,
`packages/api-server/src/modules/connections/domain/connection-sds.ts:24-53 @b62d21c`).
The code-declared template catalog (`buildCatalog`) covers Anthropic, OpenAI, IBM
LiteLLM, Bob, Modal, GitHub OAuth/PAT/Enterprise, Slack, and ~16 Google services
(`packages/api-server/src/modules/connections/domain/catalog.ts:740 @b62d21c`).

## The `env` contribution — three sources, user wins

The `env` kind is no longer just credential-derived. As of
[#1899](https://github.com/dam-agents/dam/pull/1899) (`@d507c05`) **three sources**
feed it (`docs/architecture/connections.md @d507c05`):

- **user-typed env** — the UI Environment editor, stored in Postgres `agent_env`
  (`agent_id`, `name`, `value`), the editor sending the complete replacement set
  (`packages/api-server/src/modules/agents/infrastructure/agent-env-repository.ts:35-49 @d507c05`);
- **connection-derived env** — a credential placeholder the [gateway](zero-trust-credential-gateway.md)
  swaps on the wire;
- **secret-derived env** — from the Agent's mounted credential Secrets.

For user-typed and other non-credential config env the contribution carries the
**literal value** (in the `placeholder` field); only credential-derived env
carries a placeholder the gateway rewrites. This replaces the Agent CR's
`spec.env`: the controller no longer reads it (see [agent-lifecycle](agent-lifecycle.md)),
so an env change applies over the runtime channel at the next idle turn with no
pod roll. The state-builder emits **user env first** in the contribution list,
and because the on-pod env driver is *first-occurrence-wins*, user env shadows a
same-named connection or secret env
(`packages/api-server/src/modules/runtime-delivery/services/state-builder.ts:60-71 @d507c05`,
`:87-100 @d507c05`). The `env` contribution itself remains a wire-level kind
(`packages/agent-runtime-api/src/modules/runtime/types.ts @d507c05`).

## Provider presets — one source of truth

Distinct from the proposed model above, **provider presets** are today's
code-level catalog of the model providers a user can connect (Anthropic,
IBM LiteLLM, OpenAI, Bob), each carrying its auth modes, default env mappings,
and [Envoy](../entities/envoy-gateway.md) injection config (header/query-param,
placeholder rewritten on the wire — see
[zero-trust-credential-gateway](zero-trust-credential-gateway.md)). As of
@4a48ae2 these are **unified into one registry**, `PROVIDERS`, in
`packages/api-server-api/src/modules/connections/providers.ts:108 @4a48ae2` —
the single source of truth the server catalog, UI, and CLI all derive from
(`packages/api-server-api/src/modules/connections/providers.ts:1 @4a48ae2`).
The previously duplicated `provider-templates.ts` copies in both `ui` and `cli`
were deleted, and the type/helper exports moved out of the `secrets` module into
this new `connections/providers` module. The template↔provider relation
(`templateIdForProvider`, `providerTypeForTemplateId`) is derived from each
preset's `modes[].templateId` rather than hand-maintained
(`packages/api-server-api/src/modules/connections/providers.ts:203 @4a48ae2`).

## Secrets → Connections cutover (#2200)

[#2200](https://github.com/dam-agents/dam/pull/2200) (`@b62d21c`) deleted the
standalone `secrets` module across api-server, api-server-api, ui, and cli
(`k8s-secrets-port`, `secrets-service`, the GitHub-PAT grouping, the UI
`connections-picker`/`add-agent-dialog`, the secrets tRPC router). Every credential
is now a [Connection](#storage). The SDS YAML the gateway reads is produced by
`connection-sds.ts` (`buildConnectionSdsFields`), replacing the deleted
`secrets/infrastructure/k8s-secrets-port.ts`
(`packages/api-server/src/modules/connections/domain/connection-sds.ts:24-53 @b62d21c`).
A one-time, idempotent, boot-time migration drains legacy K8s Secrets into
Connections and flips agent grants (see
[api-server → boot migration](../sources/api-server.md#boot-time-secrets--connections-migration)),
with an inert `legacy-secret-env-source` safety net for the upgrade window
(`packages/api-server/src/modules/connections/migration/secrets-to-connections.ts:16-27 @b62d21c`).
This cutover touched **no Postgres schema** — the `connections`/`connection_grants`
tables predate it.

> **Flagged for lint (contradiction, not reconciled).** The same PR marks ADR-051
> "Accepted" declaring a *clean break with no migration choreography*, yet the
> shipped code performs a full staged boot-time migration
> (`migrateSecretsToConnections`, wired in
> `packages/api-server/src/index.ts:216-240 @b62d21c`). Both the ADR and the
> security doc were touched in the same commit; code is ground truth per this
> wiki's contradiction policy, so the wiki follows the code and surfaces the
> tension rather than flattening it.

## GitHub identity → git commit attribution

A concrete use of the `file` contribution: when a **GitHub** OAuth connection
finishes its token exchange, the api-server resolves the connected user's
identity by calling `GET https://api.github.com/user` with the fresh access
token (`packages/api-server/src/modules/connections/infrastructure/github-identity.ts:33 @380cb06`).
The display name is `name ?? login` with control characters stripped (so it
cannot inject extra gitconfig sections), and the email falls back to GitHub's
no-reply form `${id}+${login}@users.noreply.github.com` when the account email
is private — keeping a real primary email out of commit history
(`.../github-identity.ts:48-49 @380cb06`). From that identity it builds a `file`
contribution at `$HOME/.gitconfig` (`ini`, `mergeMode: section-marker`, a
`[user]` block) and **upserts** it — replacing any prior gitconfig contribution
at the same path
(`packages/api-server/src/modules/connections/domain/gitconfig-contribution.ts:4-15 @380cb06`),
so the agent's git commits are attributed to the connected user.

The whole identity step is **best-effort**: it only runs for the `github`
template (`.../services/oauth-flow.ts:146 @380cb06`), and because the token is
already minted, a failed lookup is wrapped in try/catch — it logs a
`securityLog` warning and never fails the connection, with identity landing on
the next successful re-auth (`.../services/oauth-flow.ts:179 @380cb06`). On
success it persists via `repo.updateContributions`, bumps the runtime Version,
and enqueues a re-push to every granted Agent (`.../services/oauth-flow.ts:171 @380cb06`).

## Delivery — the outbox state machine

`applyState` carries a **State Slice** (a full declarative snapshot of the
Agent's desired contributions, with a content hash for short-circuiting) and an
**Events** slice (one-shot directives like `trigger`)
(`docs/ubiquitous-language.md @4a48ae2`). Each Agent has a monotonic **Version**
bumped on every contribution edit or event insert. Key states:

- **Settled Version** — the last version whose apply cycle ran to termination
  (success *or* with failures); recorded as `runtime_state_outbox.last_settled_version`.
- **Last Applied Version / Hash** — the last *fully-applied* version/hash;
  advances only on a failure-free settle. The server rejects older state pushes
  (cross-replica race defense) and skips retransmitting an unchanged slice.
- **Apply Failures** — the drivers that failed the latest settle; drives the
  capped background retry and the degraded `contributionFailures` badge. A
  version bump clears them.
- **Contributions Settled** — "the agent terminated reconciliation for the
  current Version." Does *not* assert per-driver success. Drives the retry sweep;
  on this iteration it does **not** gate readiness (the apply worker dispatches
  only to a `Ready` agent).

A pod's `hello` is **presence-only** — it signals the worker to dispatch; the
worker dispatches only to a `Ready` agent, so an apply never targets a pod that's
down or rolling (`docs/architecture/agent-lifecycle.md @4a48ae2`).

## Agent-side: the unified `drivers:` manifest (ADR-072)

On the pod, the runtime channel dispatches every contribution and event through a
per-harness `drivers:` map. As of
[#1270](https://github.com/dam-agents/dam/pull/1270) (`@b62d21c`) that map is
**unified**: every kind — contribution *or* event — is a driver entry resolved the
same way through the plugin registry. The event path is no longer a hardcoded
switch; event kinds bind through `plugin.bindEvent` exactly as contribution kinds
bind through `plugin.bind`
(`packages/agent-runtime/src/modules/runtime-channel/event-dispatcher.ts:37-55 @b62d21c`,
`packages/agent-runtime/src/modules/runtime-channel/dispatcher.ts:43-63 @b62d21c`),
both fed from one resolved binding map
(`packages/agent-runtime/src/modules/runtime-channel/compose.ts:106-115 @b62d21c`).

- **Built-ins are on by default.** A runtime-owned table (`BUILTIN_DRIVERS`)
  registers each built-in kind with a default binding; a manifest entry is needed
  only to *configure* a kind, *override* its `impl`, or *disable* it with `false`.
  `resolveDrivers` = default-on built-ins ∪ manifest declarations ∖ disables,
  throwing on an unknown kind
  (`packages/agent-runtime/src/modules/runtime-channel/manifest.ts:105-164 @b62d21c`).
- **`impl` defaults to the kind name**; the one built-in whose impl differs is
  `schedule-reset`, bound to impl `trigger` — the historical mapping preserved so
  older manifests still resolve
  (`packages/agent-runtime/src/modules/runtime-channel/manifest.ts:124,138-140 @b62d21c`).
- **Capabilities are derived, not declared** — advertised contribution kinds come
  from the resolved drivers (defaults ∪ declared ∖ opted-out); event kinds are
  advertised in full because a kind with no active driver simply no-ops at dispatch
  (`packages/agent-runtime/src/modules/runtime-channel/compose.ts:122-127 @b62d21c`).
  `platform-base`'s manifest declares `drivers: {}` — every built-in on by default;
  pi-agent drops its four inherited driver entries and declares only its
  harness-config (`packages/agents/pi-agent/runtime-manifest.yaml:7-40 @b62d21c`).

### harness-config — per-agent model/mode/config

**harness-config** is a driver (default-*off* until a manifest binds it, since it
needs a per-harness config) that carries two capabilities beyond a single event
handler: it **presents a catalog** (advertised on `hello`) and **answers a live
current-config read**
(`packages/agent-runtime/src/modules/runtime-channel/drivers/harness-config-plugin.ts:26-34 @b62d21c`).
On a `harness-config` event it performs a **one-shot key-targeted write** into the
harness's own config file (claude-code `~/.claude/settings.json`, pi
`~/.pi/agent/settings.json`) and **never re-asserts it** — like `workspace-seed`,
the file stays the user's to edit
(`packages/agent-runtime/src/modules/runtime-channel/drivers/harness-config-plugin.ts:48-102 @b62d21c`).
It is therefore **not persisted state**: the change rides as a one-shot outbox
event (30-day TTL), not part of the reconciled `applyState` snapshot, so a running
session keeps its startup settings and *new* sessions pick up the change
(`packages/api-server/src/modules/harness-config/services/harness-config-service.ts:42-57 @b62d21c`).
When a manifest declares `modelDiscovery.urlEnv` the read enumerates models live
from the provider's OpenAI-style `/v1/models` endpoint over the agent's egress,
falling back to the static catalog on any failure
(`packages/agent-runtime/src/modules/runtime-channel/infrastructure/model-discovery.ts:20-64 @b62d21c`).
The user-facing surface is the session Config panel's Model settings (see
[ui](../sources/ui.md)); the [api-server](../sources/api-server.md) `harness-config`
module inserts the event, while the live current-config read is served by
agent-runtime directly.

## See also

- [agent-lifecycle](agent-lifecycle.md) (trigger fire rides this rail) · [Schedule](../entities/schedule.md) · [persistence-substrates](persistence-substrates.md)
