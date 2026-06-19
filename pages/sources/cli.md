---
source: dam-agents/dam
commit: 662ebe4c88029788829246170e17465c69523521
files: [packages/cli/src/, docs/architecture/cli.md, docs/ubiquitous-language.md]
updated: 2026-06-19
---

# cli (`dam`)

The `dam` command-line client — an npm-distributed Node package
(`@dam-agents/cli`) that talks to a configured hosted Platform deployment from
the user's terminal (`docs/ubiquitous-language.md @662ebe4`,
`packages/cli/package.json @662ebe4`). It uses the same tRPC surface as the UI
for agent resolution and auth, and the same binary terminal-frame protocol for
`dam chat` attach (`docs/architecture/platform-topology.md @662ebe4`).

## Configuration & auth

- **Active Host** — the target Server URL, resolved from `--server` flag, env
  var, then `config.toml`, in that order (`docs/ubiquitous-language.md @662ebe4`).
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

`agent`, `auth`, `chat`, `approval`, `connection`, `egress`, `file`, `import`,
`schedule`, `skill`, `ssh`, `template`, plus `cli`/`shared`
(`packages/cli/src/modules/ @662ebe4`). Entrypoint `src/bin.ts`,
composition in `src/compose.ts`.

## See also

- [platform-topology](../concepts/platform-topology.md) — the protocols the CLI speaks.
- [Keycloak](../entities/keycloak.md) — the identity provider behind `dam auth login`.
- [Session](../entities/session.md) — note: session CRUD moved to agent-owned ACP; some CLI terminal-resolution paths still reference dropped `sessions.*` procedures (`docs/architecture/platform-topology.md @662ebe4`).
