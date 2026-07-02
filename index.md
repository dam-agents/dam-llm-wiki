# Wiki index

Content catalog. One line per page, grouped by category. Maintained by `ingest`
and `lint`. Each line links a page title to its path under `pages/` (by category)
and ends with a one-line hook.

This wiki documents **[dam-agents/dam](pages/sources/dam.md)** — the DAM platform
for running agent harnesses headless in the cloud.

## Sources

- [dam — DAM agent platform (overview)](pages/sources/dam.md) — what DAM is, repo shape, module map, and where to start.
- [api-server](pages/sources/api-server.md) — TypeScript backend: user API, ACP relay, sole writer of spec + Postgres.
- [controller](pages/sources/controller.md) — Go Kubernetes operator reconciling the Agent/Fork/Run CRs.
- [agent-runtime](pages/sources/agent-runtime.md) — per-agent pod process: ACP server, terminal/SSH, runtime channel.
- [ui](pages/sources/ui.md) — React + Vite single-page app and the only management surface.
- [cli](pages/sources/cli.md) — the `dam` command-line client.
- [agents](pages/sources/agents.md) — per-harness container images (claude-code, codex, pi-agent, bob, nous, openevolve, k-search).
- [db](pages/sources/db.md) — the Postgres schema, migrations, and client.

## Entities

- [Agent](pages/entities/agent.md) — the durable, owned, runnable resource (UI: "Sandbox").
- [Template](pages/entities/template.md) — read-only blueprint copied into an Agent at create time.
- [Session](pages/entities/session.md) — one agent-owned conversation; chat, terminal, and SSH models.
- [Schedule](pages/entities/schedule.md) — a cron/RRULE/heartbeat task attached to an Agent (Postgres row).
- [Fork](pages/entities/fork.md) — per-turn, Job-shaped run impersonating a Slack foreign replier.
- [Run](pages/entities/run.md) — ephemeral single-command executor behind the in-pod `dam-run` CLI; a bare Pod borrowing the parent's gateway.
- [Envoy gateway](pages/entities/envoy-gateway.md) — per-agent egress proxy that injects credentials on the wire.
- [Keycloak](pages/entities/keycloak.md) — the in-cluster OIDC identity authority.

## Concepts

- [Platform topology](pages/concepts/platform-topology.md) — the four subsystems, paired pods, and protocols.
- [Agent lifecycle](pages/concepts/agent-lifecycle.md) — create → wake → trigger → hibernate → delete.
- [Zero-trust credential gateway](pages/concepts/zero-trust-credential-gateway.md) — the defining security model.
- [Persistence substrates](pages/concepts/persistence-substrates.md) — Postgres vs. custom resources vs. per-agent PVCs.
- [Observability (agent telemetry)](pages/concepts/observability.md) — optional bundled ClickStack backend: OTel collector + columnar store + exploration UI, gated by the mesh, with gateway-stamped trusted attribution.
- [ACP (Agent Client Protocol)](pages/concepts/agent-client-protocol.md) — the JSON-RPC relay contract for live agent I/O.
- [HITL approvals & egress rules](pages/concepts/hitl-approvals.md) — the human-in-the-loop gate on egress and tool calls.
- [Connections & contributions](pages/concepts/connections-and-contributions.md) — the runtime channel, outbox, and apply state machine.
- [E2E testing](pages/concepts/e2e-testing.md) — Playwright specs driving a real k3s cluster with a scriptable mock agent.
- [Skill resolution](pages/concepts/skill-resolution.md) — which folders DAM scans to discover/install skills from a source (`skills/`, `.claude/skills/`, `.agents/skills/`; repo root as fallback), or a single pinned subdir (`path`).
- [Experiments](pages/concepts/experiments.md) — race competing R&D harnesses against one goal; the platform starts Arms, captures scored Runs over MCP, and presents the comparison (internal-only UI).
