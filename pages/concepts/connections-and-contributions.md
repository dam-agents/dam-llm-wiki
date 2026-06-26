---
source: dam-agents/dam
commit: d507c05fb3683c901473b5166766db03ce14fb29
files: [docs/ubiquitous-language.md, docs/architecture/agent-lifecycle.md, docs/architecture/connections.md, docs/architecture/persistence.md, packages/api-server-api/src/modules/connections/providers.ts, packages/agent-runtime-api/src/modules/runtime/types.ts, packages/api-server/src/modules/runtime-delivery/services/state-builder.ts, packages/api-server/src/modules/agents/infrastructure/agent-env-repository.ts, packages/api-server/src/modules/connections/domain/gitconfig-contribution.ts, packages/api-server/src/modules/connections/infrastructure/github-identity.ts, packages/api-server/src/modules/connections/services/oauth-flow.ts]
updated: 2026-06-26
---

# Connections & contributions

The **runtime channel** is how the [api-server](../sources/api-server.md)
delivers declarative configuration and one-shot events to a running
[agent-runtime](../sources/agent-runtime.md) pod, via a transactional
**outbox + worker** (`docs/ubiquitous-language.md @4a48ae2`,
`docs/architecture/agent-lifecycle.md @4a48ae2`).

> The unified Connection/Contribution model below is **proposed, in-flight
> design** per the glossary — structure (subtype axes, push channel, capability
> negotiation) is still being grilled (`docs/ubiquitous-language.md @4a48ae2`).
> It generalises today's split between `OAuthAppDescriptor` and `ProviderPreset`.

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

## See also

- [agent-lifecycle](agent-lifecycle.md) (trigger fire rides this rail) · [Schedule](../entities/schedule.md) · [persistence-substrates](persistence-substrates.md)
