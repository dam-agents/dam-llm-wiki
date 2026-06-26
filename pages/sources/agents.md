---
source: dam-agents/dam
commit: d507c05fb3683c901473b5166766db03ce14fb29
files: [packages/agents/, packages/platform-base/, docs/architecture/agent-lifecycle.md]
updated: 2026-06-26
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
| `nous/` | Nous | Hypothesis-driven experimentation agent; built **FROM** the `claude-code` image, not `platform-base` (`packages/agents/nous/ @ca1877a`). See below. |

Each harness dir carries a `Dockerfile`, the two `harness-*.sh` scripts, a
`runtime-manifest.yaml`/`tasks.toml`, and a seed `workspace/` (with a
`.config/mise/` toolchain). Any ACP-compatible runtime can be added as a new
harness ("bring your own harness", `README.md:55 @662ebe4`).

## Nous — a harness layered on another harness

`nous/` (added in [#1382](https://github.com/dam-agents/dam/pull/1382),
`@ca1877a`) is the first harness that does **not** build on `platform-base`
directly: its Dockerfile is `FROM` the `claude-code` image
(`ARG BASE_IMAGE=platform-claude-code`), inheriting the `claude` CLI, the model
gateway, and CA trust, then adds a Python venv at `/opt/nous-venv` with the
[`nous`](https://github.com/AI-native-Systems-Research/agentic-strategy-evolution)
package `uv pip`-installed straight from its public GitHub repo (pinned via
`ARG NOUS_REF`, default `v0.4.0`; no vendoring)
(`packages/agents/nous/README.md @ca1877a`). It runs **Nous** — a deterministic
Python orchestrator (`nous run campaign.yaml`) that drives two Claude agent roles
through a hypothesis → experiment → findings loop against a target repo it
clones, patches, and measures (`packages/agents/nous/AGENTS.md @ca1877a`).

Both `harness-chat`/`harness-terminal` entrypoints are **inherited unchanged**
from the claude-code base — behaviour is customised through context, not harness
scripts: a Nous-oriented `AGENTS.md` as chat system context, the bundled `nous`
skill (CLI + campaign-authoring reference), and the upstream wiki slash-commands
(`post-campaign`, `index-wiki`, `visualize-*`, `suggest-next`) vendored into
`~/.claude/commands/` (`packages/agents/nous/README.md @ca1877a`). The chat agent
drives the `nous` CLI: `--auto-approve` (so the two human gates auto-pass —
`NOUS_ALLOW_AUTO_APPROVE=1` is baked in), launched as a background process, with
campaign artifacts under `$NOUS_CAMPAIGN_PARENT` (`~/nous-campaigns`) on the
persistent `$HOME` so a hibernation-killed run **resumes on the next turn**
(`nous resume`) (`packages/agents/nous/AGENTS.md @ca1877a`).

It maps onto the platform's keyless model: Nous's Claude SDK calls and `git`/`gh`
authenticate through the [Envoy credential gateway](../entities/envoy-gateway.md)
— the pod holds no credentials. A stdlib-only
[`nous-channel-bridge.py`](https://github.com/dam-agents/dam/blob/main/packages/agents/nous/nous-channel-bridge.py)
relays each campaign DESIGN/FINDINGS gate summary into the agent's bound
Slack/Telegram thread via Nous's own `channels:` webhook feature pointed at
`127.0.0.1:8765`, calling the per-agent `send_channel_message` MCP tool — no
external egress, no webhook secret on disk (`packages/agents/nous/README.md @ca1877a`).
CI builds it per-arch after `merge-agents` (it pulls its claude-code base by the
same per-commit tag) and publishes the multi-arch manifest to
`quay.io/dam-agents/nous`; the template ships `experimental: true`
(`packages/agents/nous/README.md @ca1877a`, `.github/workflows/cd.yml @d34c21a`).

## See also

- [agent-lifecycle](../concepts/agent-lifecycle.md) — the harness-script contract and pod-service supervision.
- [Template](../entities/template.md) — how an image is selected at agent create time.
