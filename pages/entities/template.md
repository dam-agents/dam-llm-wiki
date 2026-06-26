---
source: dam-agents/dam
commit: d507c05fb3683c901473b5166766db03ce14fb29
files: [docs/ubiquitous-language.md, docs/architecture/persistence.md, docs/architecture/agent-lifecycle.md, packages/api-server/src/modules/agents/services/agents-service.ts]
updated: 2026-06-26
---

# Template

A **read-only catalog blueprint** defining the base image, mounts, env, and
resources for creating an [Agent](agent.md) (`docs/ubiquitous-language.md @662ebe4`).

## Not a CRD

Templates are deliberately **not** custom resources: they are **chart-rendered
ConfigMaps**, loaded by the [api-server](../sources/api-server.md) at boot,
read-only and never reconciled by the controller
(`docs/architecture/persistence.md @662ebe4`). They fail the "controller
reconciles it" test for CRD-hood — only the api-server reads them.

## Create-time only

A Template contributes at **create time only**: the api-server copies template
image/mounts into the new Agent's `spec`, and — since
[#1899](https://github.com/dam-agents/dam/pull/1899) (`@d507c05`) — seeds the
template's **env into Postgres `agent_env`** rather than `spec.env`, so it rides
the runtime channel like user env (any caller-supplied env is ordered last, so a
user value wins a same-named template default)
(`packages/api-server/src/modules/agents/services/agents-service.ts:281-284 @d507c05`,
`:345-351 @d507c05`). The controller never reads the Template again at pod start,
so **editing a Template never re-flows into a running Agent** — there is no
"template envs" runtime layer (`docs/architecture/agent-lifecycle.md @d507c05`).
The default Claude Code template persists the workspace and `$HOME`.

## See also

- [Agent](agent.md) · [agent-lifecycle](../concepts/agent-lifecycle.md) · [agents (harness images)](../sources/agents.md)
