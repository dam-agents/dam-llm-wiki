---
source: dam-agents/dam
commit: d34c21a008d3b868fc260838374836ac88fb0807
files: [packages/e2e/playwright/playwright.config.ts, packages/e2e/playwright/src/config.ts, packages/e2e/playwright/src/lib/auth.ts, packages/e2e/playwright/src/lib/api-client.ts, packages/e2e/playwright/src/lib/agents.ts, packages/e2e/playwright/src/lib/connections.ts, packages/e2e/playwright/src/lib/fixtures.ts, packages/e2e/playwright/src/tests/01-auth.spec.ts, packages/e2e/playwright/src/tests/05-injection.spec.ts, packages/e2e/playwright/src/tests/06-api-keys.spec.ts, packages/e2e/playwright/src/tests/07-slack.spec.ts, packages/e2e/playwright/src/tests/08-session-delete.spec.ts, packages/e2e/agents/mock/src/main.ts, packages/api-server/src/modules/e2e/services/e2e-service.ts, tasks.toml]
updated: 2026-06-24
---

# E2E testing

DAM's end-to-end tests live in the **`e2e`** package and drive the *real,
fully-deployed platform* — UI, [api-server](../sources/api-server.md),
[controller](../sources/controller.md), [Keycloak](../entities/keycloak.md),
Postgres, Istio and the [Envoy gateway](../entities/envoy-gateway.md) — from a
real browser. They are **[Playwright](https://playwright.dev)** specs that run
against a throwaway single-node k3s cluster, with a **mock agent harness** standing
in for any real LLM so the assertions are deterministic
(`packages/e2e/playwright/playwright.config.ts @9d1bc99`, `tasks.toml:18-99 @9d1bc99`).

The `e2e` package has three parts (`packages/sources/dam.md:54 @0158357`):

- **`playwright/`** — the test specs and helper library.
- **`agents/mock/`** — the deterministic mock harness image (a scriptable ACP
  agent that replaces Claude Code et al. in the cluster).
- **`agents/mock-api/`** — the tRPC contract the mock exposes for scripting it.

## What a run looks like

`mise run e2e` is the from-scratch CI path: nuke any prior test VM, install a
fresh cluster, run the specs, tear the VM down on exit (`tasks.toml:18-99 @9d1bc99`).
The cluster is provisioned by the shared `e2e:install` task with the e2e Helm
overlay (`values-e2e.yaml`), mock-only agents, and the e2e control API enabled
(`tasks.toml:103-165 @9d1bc99`). Two faster variants exist for iteration:

- **`e2e:loop`** — rerun against a warm, persistent `platform-k3s-test` VM that is
  never deleted; optionally rebuild components (`--rebuild=apiserver,ui,...`) and
  wipe data, then run the specs. It does **not** heal a wedged cluster — it fails
  loud (`tasks.toml:205-284 @9d1bc99`).
- **`e2e:reset`** — data wipe only (drop+recreate the platform DB, delete agent
  resources, clear stored Playwright auth); leaves the cluster running
  (`tasks.toml @9d1bc99`).

The full from-scratch path nukes the same shared VM name, so the next `e2e:loop`
bootstraps a fresh one (`CLAUDE.md` of the source, "E2E tests" section).

## The specs are a serial pipeline

Playwright is configured **non-parallel** — `fullyParallel: false`, `workers: 1`,
`retries: 0` — and the projects form a dependency chain so each builds on the
state the previous one left behind
(`packages/e2e/playwright/playwright.config.ts:7-67 @9d1bc99`):

| # | Project | Depends on | What it proves |
| --- | --- | --- | --- |
| 01 | `auth` | — | Log in through the real Keycloak redirect, accept terms, **save the browser session** to `.auth/user.json` (`tests/01-auth.spec.ts:7-33 @9d1bc99`). |
| 02 | `connection` | auth | Create a custom-header [connection](./connections-and-contributions.md) in the UI (`tests/02-connection.spec.ts @9d1bc99`). |
| 03 | `agent` | connection | Create a mock [Agent](../entities/agent.md) with the connection attached (`tests/03-agent.spec.ts @9d1bc99`). |
| 04 | `messages` | agent | Exchange chat messages with the agent, incl. mid-turn queued prompts (`tests/04-messages.spec.ts @9d1bc99`). |
| 05 | `injection` | messages | Verify the [credential gateway](./zero-trust-credential-gateway.md) end to end (below) (`tests/05-injection.spec.ts:14-67 @9d1bc99`). |
| 06 | `api-keys` | auth | Pure-API authorization matrix for scoped API keys — no browser (`tests/06-api-keys.spec.ts @9d1bc99`). |
| 07 | `slack` | injection | Foreign-user Slack mention → [Fork](../entities/fork.md) → reply lands in the thread, **and** the fork's egress carries the *foreign* user's injected credential (per-owner isolation across a fork: `testUser2`'s own connection sentinel comes back, not the owner's) (`tests/07-slack.spec.ts:147-166 @380cb06`). |
| 08 | `session-delete` | agent | Deleting the active [Session](../entities/session.md) from the sidebar clears it **without a page refresh** (the row vanishes on its own) and a session started right after appears as a fresh active row — regression cover for #1084/#1688 (`tests/08-session-delete.spec.ts:21-79 @0ffb6d6`). Creates and deletes its own session, so it leaves no residue and depends on `agent` only to gate on a running agent (`playwright.config.ts:67-75 @0ffb6d6`). |

Subsequent projects reuse the saved `storageState`, so they start already logged
in (`playwright.config.ts:29-66 @9d1bc99`). The `api-keys` project is the
exception: it mints its own JWT and never touches the browser session, depending
on `auth` only to gate on a healthy cluster (`playwright.config.ts:52-60 @9d1bc99`).

Run artifacts are heavy by design for debugging a distributed system: `trace: "on"`,
`video: "on"`, screenshots on failure, plus an HTML report
(`playwright.config.ts:12-21 @9d1bc99`). On failure the `mise run e2e` wrapper also
dumps full cluster diagnostics — nodes, pods, events, and pod logs — before teardown
(`tasks.toml:42-78 @9d1bc99`).

## How a spec talks to the platform

Two channels, both pointed at the deployed cluster (`PLATFORM_BASE_URL`, default
`http://localhost:4444`/`:5555` in CI) (`packages/e2e/playwright/src/config.ts:1-19 @9d1bc99`):

1. **The browser** — Playwright drives the real React UI through Chromium.
2. **A typed tRPC client** — `createApiClient(token)` builds a `@trpc/client`
   typed by the api-server's `AppRouter`, so specs can set up state and assert on
   the backend directly (`src/lib/api-client.ts:7-16 @9d1bc99`). Tokens come from
   a Keycloak **password grant** against the test users `dev`/`dev2`
   (`src/lib/auth.ts:14-32 @9d1bc99`, `src/config.ts:11-19 @9d1bc99`).

Because pods come up asynchronously, the helpers **poll** rather than assume
timing — e.g. `waitForAgentRunning` polls `agents.list`/`agents.get` until the
agent reaches `running`, with a 180s budget (`src/lib/agents.ts:5-37 @9d1bc99`).

## The mock agent and the `e2e` control API

Real LLM harnesses are non-deterministic, so the e2e cluster runs a **scriptable
mock harness** instead. The mock is an ACP agent whose every reply is dictated by
a script; `MOCK_DEFAULT_REPLY` seeds a default, and tests overwrite the script per
test (`packages/e2e/agents/mock/src/main.ts:1-26 @9d1bc99`). Specs script it
through an **`e2e` control namespace** on the api-server's tRPC router — e.g.
`api.e2e.setScript` makes the next turn return an exact string
(`src/lib/agents.ts:67-86 @9d1bc99`), and `api.e2e.getEnv` / `api.e2e.performFetch`
read pod state and drive egress from inside the agent
(`tests/05-injection.spec.ts:21-66 @9d1bc99`).

That control surface is **gated and off by default** — it is only mounted when the
api-server is started with the e2e flag (`E2E_ENABLED`), which the e2e Helm overlay
turns on and production never does (`packages/api-server/src/config.ts @9d1bc99`,
`deploy/helm/platform/values.yaml @9d1bc99`). On the api-server side each `e2e.*`
procedure opens a WebSocket to the target agent pod's ACP endpoint
(`ws://<pod>/api/acp`) and proxies the call to the mock's `scriptedMock` router
(`packages/api-server/src/modules/e2e/services/e2e-service.ts:30-50 @9d1bc99`).
Slack control (`fireMention`, `fireCommand`, `readOutbound`, `resetOutbound`) is
only present when a Slack e2e control is wired into the deployment, and throws
otherwise (`e2e-service.ts:22-28 @9d1bc99`).

## Two specs worth understanding

- **05 — credential injection** is the end-to-end proof of the
  [zero-trust credential gateway](./zero-trust-credential-gateway.md). It checks
  two "rails" at once: the *env rail*, where the agent process sees only a
  **placeholder** value for the connection's injected env var (`api.e2e.getEnv`
  returns the placeholder, never the real secret); and the *egress rail*, where a
  request the agent makes with that placeholder header comes back from the echo
  endpoint carrying the **real sentinel value** — proving Envoy swapped the
  credential in on the wire, outside the pod (`tests/05-injection.spec.ts:14-67 @9d1bc99`).

- **06 — API-key authorization** walks one real agent through its whole lifecycle
  holding three differently-scoped keys at once (`agents:manage` owns CRUD,
  `agents:operate` drives the live agent but can't manage it, `agents:read` can
  only look), asserting every allow/deny verdict against an agent that actually
  exists (`tests/06-api-keys.spec.ts:9-169 @9d1bc99`).

## See also

- [Agent lifecycle](./agent-lifecycle.md) — the create→running→delete path the
  specs exercise.
- [Zero-trust credential gateway](./zero-trust-credential-gateway.md) — what spec 05 verifies.
- [Fork](../entities/fork.md) — the per-turn run that spec 07's Slack mention triggers.
- [api-server](../sources/api-server.md) — hosts the gated `e2e` control router.
