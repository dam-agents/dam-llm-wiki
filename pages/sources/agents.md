---
source: dam-agents/dam
commit: b68af4ad0a0c427c856b0e5ba245feb8c2085a72
files: [packages/agents/, packages/platform-base/, docs/architecture/agent-lifecycle.md]
updated: 2026-07-01
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
| `claude-code/` | Claude Code | Ships a **pod-service** local model gateway (`model-gateway.mjs`/`.sh`) to front Anthropic-compatible upstreams; `harness-chat.sh`, `harness-terminal.sh`, `pod-service.sh` (`packages/agents/claude-code/ @662ebe4`). The **only** harness wired for OpenTelemetry export (via `CLAUDE_CODE_ENABLE_TELEMETRY` + `OTEL_*` env from the chart when a template opts in) — see [observability](../concepts/observability.md). |
| `codex/` | Codex | `system-config.toml`, harness scripts (`packages/agents/codex/ @662ebe4`). |
| `pi-agent/` | Pi Agent | Multi-provider; ships a `pi-dynamic-providers` extension (`packages/agents/pi-agent/ @662ebe4`). |
| `bob/` | Bob | IBM enterprise harness; includes an ACP shim `bob-acp-shim.mjs` (`packages/agents/bob/ @662ebe4`). |
| `nous/` | Nous | Hypothesis-driven experimentation agent; built **FROM** the `claude-code` image, not `platform-base` (`packages/agents/nous/ @ca1877a`). See below. |
| `openevolve/` | OpenEvolve | Evolutionary code-optimization agent; also built **FROM** the `claude-code` image, not `platform-base` (`packages/agents/openevolve/ @a226090`). See below. |

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
— the pod holds no credentials. Nous's **gate summaries**, however, are
OpenAI-format LLM calls, and those bypassed the gateway until
[#2117](https://github.com/dam-agents/dam/pull/2117) (`@9ddc474`): a
`nous-model-gateway.sh` override wraps the claude-code base shim (renamed
`model-gateway-base.sh` at build) so that when `ANTHROPIC_BASE_URL` points at the
in-pod gateway, an unset or LiteLLM `OPENAI_BASE_URL` is repointed to match —
otherwise the summaries fail against the raw LiteLLM URL (intercept-CA TLS + a
model-id 403). It is sourced by `harness-chat`, `harness-terminal`, and the SSH
login profile, so the whole shell env is fixed
(`packages/agents/nous/nous-model-gateway.sh:1-17 @b68af4a`,
`packages/agents/nous/Dockerfile:47-51 @b68af4a`). A stdlib-only
[`nous-channel-bridge.py`](https://github.com/dam-agents/dam/blob/main/packages/agents/nous/nous-channel-bridge.py)
relays each campaign DESIGN/FINDINGS gate summary into the agent's bound
Slack/Telegram thread via Nous's own `channels:` webhook feature pointed at
`127.0.0.1:8765`, calling the per-agent `send_channel_message` MCP tool — no
external egress, no webhook secret on disk (`packages/agents/nous/README.md @ca1877a`).
That bridge depends on a build-time fix
([#2026](https://github.com/dam-agents/dam/pull/2026), `@ecb70c0`): Nous's
runtime reads `campaign.channels` at every gate, but the pinned `v0.4.0` schema
omits the `channels:` property while setting `additionalProperties: false`, so
any campaign using channels is rejected at pre-flight (upstream Nous issue #296).
A `patch-campaign-schema.py` run during `docker build` mutates the *installed*
`campaign.schema.yaml` in place to add the property — idempotent, version-agnostic
(it patches the parsed dict), and self-verifying (asserts a channels-bearing
campaign now validates **and** an unknown key is still rejected), so it cleanly
no-ops once a future `NOUS_REF` ships the property
(`packages/agents/nous/Dockerfile:33-43 @70c53ae`,
`packages/agents/nous/patch-campaign-schema.py @70c53ae`).
CI builds it per-arch after `merge-agents` (it pulls its claude-code base by the
same per-commit tag) and publishes the multi-arch manifest to
`quay.io/dam-agents/nous`; the template ships `experimental: true`
(`packages/agents/nous/README.md @ca1877a`, `.github/workflows/cd.yml @d34c21a`).

## OpenEvolve — the second claude-code-derived harness

`openevolve/` (added in [#2025](https://github.com/dam-agents/dam/pull/2025),
`@a226090`) is the second harness that builds `FROM` the `claude-code` image
(`ARG BASE_IMAGE=platform-claude-code`) rather than `platform-base`: it inherits
the `claude` CLI, the model gateway, and CA trust, then `uv pip`-installs the
[`openevolve`](https://github.com/algorithmicsuperintelligence/openevolve)
package from PyPI into a venv at `/opt/openevolve-venv` (pinned via
`ARG OPENEVOLVE_VERSION`, default `0.2.27`; no vendoring), prepended to `PATH`
so `openevolve-run` shadows the base tools
(`packages/agents/openevolve/Dockerfile @a226090`). It runs **OpenEvolve** — an
open-source AlphaEvolve: an LLM mutates a program, an evaluator scores each
variant against a measurable objective, and a MAP-Elites database keeps a diverse
population of the best, iterating until a winner emerges
(`packages/agents/openevolve/README.md @a226090`).

Like Nous, both `harness-chat`/`harness-terminal` entrypoints are **inherited
unchanged** from the claude-code base — behaviour is customised through context,
not harness scripts: an OpenEvolve-oriented `AGENTS.md` mounted at
`/etc/AGENTS.md` (the claude-code base symlinks `/etc/claude-code/CLAUDE.md` →
`/etc/AGENTS.md`) carries the operating policy, and a bundled `openevolve` skill
is the CLI/config reference (`packages/agents/openevolve/Dockerfile @a226090`,
`packages/agents/openevolve/workspace/.agents/skills/openevolve/SKILL.md @a226090`).
The agent is **conversational** — it takes a target repo and a measurable
objective in chat, authors the three OpenEvolve inputs (`program` with
`EVOLVE-BLOCK` markers, `evaluator` emitting `combined_score`, `config.yaml`),
runs the evolution, and reports the winning variant
(`packages/agents/openevolve/AGENTS.md @a226090`).

Two model paths run through one keyless gateway: the **driver** (Claude Code's
own model, via the inherited gateway) and the **evolution loop**, where
`openevolve-run` calls an **OpenAI-compatible** endpoint injected by the attached
model-provider connection as `OPENAI_BASE_URL` + `OPENAI_API_KEY`; a single
IBM-LiteLLM-class connection feeds both, a pure-OpenAI provider feeds only the
loop (`packages/agents/openevolve/AGENTS.md @a226090`). `git`/`gh`/PR creation
likewise authenticate through the [Envoy credential gateway](../entities/envoy-gateway.md)
host-keyed injection — the pod holds only a placeholder token, and even
LLM-generated evaluator code reaching an allowed GitHub host gets the credential,
so the control surface is the connection's scope
(`packages/agents/openevolve/AGENTS.md @a226090`). Run state (target clone,
checkpoints, `run.pid`/`run.log`) persists under `$OPENEVOLVE_OUTPUT_ROOT`
(`~/openevolve-runs`) on the agent's `$HOME` PVC so a hibernation-killed run
**resumes on the next turn** from its latest checkpoint
(`packages/agents/openevolve/AGENTS.md @a226090`, `packages/agents/openevolve/Dockerfile @a226090`).
CI builds it per-arch after `merge-agents` (it pulls its claude-code base by the
same per-commit tag via `build-openevolve`) and `merge-openevolve` publishes the
multi-arch manifest to `quay.io/dam-agents/openevolve`; the template ships
`category: preconfigured`, `experimental: true`
(`.github/workflows/cd.yml @e54e8f1`, `deploy/helm/platform/values.yaml @e54e8f1`).

## See also

- [agent-lifecycle](../concepts/agent-lifecycle.md) — the harness-script contract and pod-service supervision.
- [Template](../entities/template.md) — how an image is selected at agent create time.
