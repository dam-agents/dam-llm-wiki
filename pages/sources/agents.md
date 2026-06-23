---
source: dam-agents/dam
commit: 380cb06d1d60bca40fa703b77e13a16ec96eedf7
files: [packages/agents/, packages/platform-base/, docs/architecture/agent-lifecycle.md]
updated: 2026-06-23
---

# agents — harness container images

`packages/agents/` holds one subdirectory per supported **harness**, each
building a container image from the shared `platform-base` image plus a
harness-specific layer (`packages/agents/ @662ebe4`,
`packages/platform-base/ @662ebe4`). The platform contract every image must
satisfy is two executables at fixed paths — `/usr/local/bin/harness-chat`
(spawned as the ACP subprocess for chat sessions) and
`/usr/local/bin/harness-terminal` (PTY-attached for terminal sessions) — beyond
which [agent-runtime](agent-runtime.md) treats the harness as opaque
(`docs/architecture/agent-lifecycle.md @662ebe4`).

## Shipped harnesses

| Dir | Harness | Notes |
| --- | --- | --- |
| `claude-code/` | Claude Code | Ships a **pod-service** local model gateway (`model-gateway.mjs`/`.sh`) to front Anthropic-compatible upstreams; `harness-chat.sh`, `harness-terminal.sh`, `pod-service.sh` (`packages/agents/claude-code/ @662ebe4`). |
| `codex/` | Codex | `system-config.toml`, harness scripts (`packages/agents/codex/ @662ebe4`). |
| `pi-agent/` | Pi Agent | Multi-provider; ships a `pi-dynamic-providers` extension (`packages/agents/pi-agent/ @662ebe4`). |
| `bob/` | Bob | IBM enterprise harness; includes an ACP shim `bob-acp-shim.mjs` (`packages/agents/bob/ @662ebe4`). |

Each harness dir carries a `Dockerfile`, the two `harness-*.sh` scripts, a
`runtime-manifest.yaml`/`tasks.toml`, and a seed `workspace/` (with a
`.config/mise/` toolchain). Any ACP-compatible runtime can be added as a new
harness ("bring your own harness", `README.md:55 @662ebe4`).

## See also

- [agent-lifecycle](../concepts/agent-lifecycle.md) — the harness-script contract and pod-service supervision.
- [Template](../entities/template.md) — how an image is selected at agent create time.
