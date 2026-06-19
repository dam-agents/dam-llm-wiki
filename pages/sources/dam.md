---
source: dam-agents/dam
commit: 662ebe4c88029788829246170e17465c69523521
files: [README.md, docs/architecture.md, docs/ubiquitous-language.md, package.json, pnpm-workspace.yaml, packages/]
updated: 2026-06-19
---

# dam — DAM agent platform (source overview)

**DAM** runs agent harnesses like Claude Code **headless in the cloud**: each
agent executes continuously in an isolated container, keeps running after you
disconnect, and routes every credentialed network call through a
policy-enforced gateway so the agent process never holds upstream credentials
(`README.md:9-31 @662ebe4`). Users reach their agents from a **Web UI**, the
**`dam` CLI**, **Slack/Telegram channels**, and **schedules**, and any runtime
that speaks [ACP](../concepts/agent-client-protocol.md) can run as a harness —
Claude Code, Pi Agent, Bob, and Codex ship in-repo (`README.md:33-53 @662ebe4`).

## Repository shape

A pnpm + mise monorepo of mostly TypeScript with a Go control plane
(`package.json @662ebe4`, `pnpm-workspace.yaml @662ebe4`). Roughly 1,280 tracked
files: ~700 `.ts`, ~120 `.tsx`, ~56 `.go`, plus Helm YAML, ADRs, and docs.

| Top-level | What lives there |
| --- | --- |
| `packages/` | All deployable code (see the module map below). |
| `deploy/helm/platform/` | The Helm chart that installs the whole platform on Kubernetes, including the `Agent`/`Fork` CRDs. |
| `docs/` | Architecture (`docs/architecture/`), 70 ADRs (`docs/adrs/`), security notes, strategy, and the [ubiquitous-language glossary](#domain-glossary). |
| `.agents/` / `.claude/` | Skill definitions the repo itself ships for its own agents. |
| `scripts/`, `tasks.toml`, `mise.toml` | Dev/build task runners. |

## Module map (`packages/`)

The platform is four long-lived subsystems plus their shared API contracts and
supporting packages (`docs/architecture/platform-topology.md @662ebe4`):

- [api-server](api-server.md) — the TypeScript backend: user-facing tRPC + REST,
  the ACP relay, channels, schedules, approvals, skills, connections, usage.
- [controller](controller.md) — the Go Kubernetes operator reconciling the
  `Agent` and `Fork` custom resources into pods, services, and policies.
- [agent-runtime](agent-runtime.md) — the per-agent pod process: ACP WebSocket
  server, terminal/SSH relays, the runtime channel, and in-pod skills/files.
- [ui](ui.md) — the React + Vite single-page app served by the api-server.
- [cli](cli.md) — the `dam` command-line client (npm-distributed Node package).
- [agents](agents.md) — per-harness container images (claude-code, codex,
  pi-agent, bob) built on a shared `platform-base`.
- [db](db.md) — the Postgres schema, migrations, and client used by api-server.
- `api-server-api` / `agent-runtime-api` — shared tRPC routers and types that
  define the wire contracts between the UI/CLI, api-server, and agent pods.
- `keycloak-theme` — the login/account theme for the in-cluster
  [Keycloak](../entities/keycloak.md) identity provider.
- `platform-base` — the base container image all harness images extend.
- `e2e` — browser-driven end-to-end tests. `dev-config` — shared dev tooling.

## How it fits together

- **Architecture** — start at [platform-topology](../concepts/platform-topology.md)
  for the component/protocol map, then [agent-lifecycle](../concepts/agent-lifecycle.md)
  (create → wake → trigger → hibernate → delete).
- **Security** — the defining property is the
  [zero-trust credential gateway](../concepts/zero-trust-credential-gateway.md):
  the agent pod has no route to the internet except its paired Envoy gateway,
  which injects credentials on the wire and gates each call through
  [HITL approvals](../concepts/hitl-approvals.md).
- **State** — [persistence-substrates](../concepts/persistence-substrates.md)
  covers the three-way split: Postgres, custom resources, and per-agent PVCs.
- **Core resources** — [Agent](../entities/agent.md), [Template](../entities/template.md),
  [Session](../entities/session.md), [Schedule](../entities/schedule.md),
  [Fork](../entities/fork.md).

## Domain glossary

`docs/ubiquitous-language.md @662ebe4` is the authoritative glossary, organized
by bounded context (Agents, Channels, Forks, Skills, Approvals, Egress Rules,
Connections, Secrets, Terms, Usage Tracking, CLI). Read it before reasoning
about any domain term — the same word (e.g. *Skill Source*) deliberately means
different things in the api-server vs. agent-runtime contexts.
