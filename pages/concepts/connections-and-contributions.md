---
source: dam-agents/dam
commit: 4a48ae298461ef9b999a4d44cd105e8ba76d2bf9
files: [docs/ubiquitous-language.md, docs/architecture/agent-lifecycle.md, docs/architecture/persistence.md, packages/api-server-api/src/modules/connections/providers.ts]
updated: 2026-06-20
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
  `env`, `egress-host`, `file`, `mcp-entry`, `skill-ref` (extensible union).
- **AuthConfig** — `oauth` | `header` | `none`; `header` covers any
  header-injected static credential, distinguished by `headerName` + `valueFormat`.

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
