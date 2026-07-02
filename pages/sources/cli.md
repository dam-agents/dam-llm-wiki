---
source: dam-agents/dam
commit: b62d21c288162847d7d9918ca7887265448fe2b3
files: [packages/cli/src/, docs/architecture/cli.md, docs/ubiquitous-language.md]
updated: 2026-07-02
---

# cli (`dam`)

The `dam` command-line client — an npm-distributed Node package
(`@dam-agents/cli`) that talks to a configured hosted Platform deployment from
the user's terminal (`docs/ubiquitous-language.md @4a48ae2`,
`packages/cli/package.json @4a48ae2`). It uses the same tRPC surface as the UI
for agent resolution and auth, and the same binary terminal-frame protocol for
`dam chat` attach (`docs/architecture/platform-topology.md @4a48ae2`).

## Configuration & auth

- **Active Host** — the target Server URL, resolved from `--server` flag, env
  var, then `config.toml`, in that order (`docs/ubiquitous-language.md @4a48ae2`).
- **Auth Store** — per-host credentials at `$XDG_STATE_HOME/dam/auth.toml`
  (mode 0600, atomic writes), written by `dam auth login` (Keycloak device
  grant; the public client is `platform-cli`).
- **Token Provider** — the single service every authenticated verb calls to get
  a valid bearer, with proactive refresh and `invalid_grant` → clear-creds.
- **Compat check** — compares the local version against the server's
  `minClientVersion`: `Ok` / `BehindMinClient` (hard refuse) / `BehindCurrent`
  (soft warn), using the unauthenticated `version` endpoint.
- **Agent Ref / Resolver** — addresses an agent by ID (anything starting with
  the reserved `agent-` prefix) or by exact case-sensitive name.

## Verb groups (`packages/cli/src/modules/`)

`agent`, `auth`, `channel`, `chat`, `approval`, `connection`, `egress`, `file`,
`import`, `schedule`, `skill`, `ssh`, `template`, plus `cli`/`shared`. Entrypoint
`src/bin.ts`; each group is composed and added to the commander program in
`src/compose.ts` (`packages/cli/src/compose.ts:142-155 @380cb06`).

### `channel` — messenger channel bindings

`dam channel` manages an Agent's Slack/Telegram bindings
(`packages/cli/src/modules/channel/compose.ts:53 @380cb06`): `channel available`
lists the providers the operator enabled host-wide, `channel list <agent>` shows
an agent's connected channels, `channel slack connect|disconnect` and `channel
telegram connect|disconnect` bind/unbind a channel
(`channel/compose.ts:71,78 @380cb06`), and `channel allow|disallow` edit the
agent's `allowedUserEmails` allow-list that gates who may drive it
(`channel/compose.ts:92-93 @380cb06`). Connect verbs run a **client-side
precheck** that refuses early if the provider isn't enabled, rather than
round-tripping a server `PRECONDITION_FAILED`
(`packages/cli/src/modules/channel/commands/precheck.ts:21-34 @380cb06`).

### Credentials via Connections only (#2200)

As of the [secrets→connections cutover](../concepts/connections-and-contributions.md#secrets--connections-cutover-2200)
([#2200](https://github.com/dam-agents/dam/pull/2200)) the `dam` CLI no longer
dual-reads legacy secrets: `create-interactive` grants credentials only through
`connections.setAgentConnections`, and a GitHub PAT is a single `github-pat`
connection rather than a client-side-grouped secret pair — the `group-github-pats`
helper and its test were deleted
(`packages/cli/src/modules/agent/commands/create-interactive.ts @b62d21c`).

### Setup-time connection config inputs

`dam connection connect` accepts repeatable `-c, --config key=value` flags to
set a template's optional config inputs (e.g. `--config model=premium-shell`),
validated against each input's `enumValues`/`pattern`; a blank value clears the
input and unknown keys error
(`packages/cli/src/modules/connection/domain/config-inputs.ts:39-62 @380cb06`).

### Plainer server-rejection reporting

When the server rejects a command, the CLI now prints the server's actual reason
instead of the misleading "cannot reach server": a `TransportError` carrying a
`serverCode` (set when the server returned a tRPC error code) prints the plain
reason, and only a genuine connectivity failure gets the old text
(`packages/cli/src/modules/shared/trpc/print.ts:20-24 @380cb06`,
`shared/trpc/classify.ts:22-25 @380cb06`). Auth rejections route through a
tailored hint — `DAM_TOKEN was rejected…` vs. `run \`dam auth login\` first`
(`packages/cli/src/modules/shared/auth-message.ts:5-13 @380cb06`).

## See also

- [platform-topology](../concepts/platform-topology.md) — the protocols the CLI speaks.
- [Keycloak](../entities/keycloak.md) — the identity provider behind `dam auth login`.
- [Session](../entities/session.md) — note: session CRUD moved to agent-owned ACP; some CLI terminal-resolution paths still reference dropped `sessions.*` procedures (`docs/architecture/platform-topology.md @4a48ae2`).
