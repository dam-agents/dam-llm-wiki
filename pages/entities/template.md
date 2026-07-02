---
source: dam-agents/dam
commit: b62d21c288162847d7d9918ca7887265448fe2b3
files: [docs/ubiquitous-language.md, docs/architecture/persistence.md, docs/architecture/agent-lifecycle.md, packages/api-server/src/modules/agents/services/agents-service.ts, packages/api-server-api/src/modules/templates/types.ts, packages/api-server-api/src/modules/templates/schemas.ts, packages/api-server/src/modules/agents/domain/spec-assembly.ts, deploy/helm/platform/templates/agent-templates.yaml]
updated: 2026-07-02
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

## Per-template scheduling

A template can override two scheduling fields (`runtimeClassName` + `nodeSelector`)
on the agents it creates; **all other
scheduling stays chart-wide** (ADR-073). `TemplateSpec` gains
`runtimeClassName?: string` (overrides the chart-wide runtime class, e.g. a GPU
Kata class; empty = inherit) and `nodeSelector?: Record<string, string>` (labels
merged onto the chart-wide selector; empty = inherit)
(`packages/api-server-api/src/modules/templates/types.ts:54-57 @b62d21c`,
`packages/api-server-api/src/modules/templates/schemas.ts:56-57 @b62d21c`). At
create time the api-server copies both from the template spec straight onto the new
Agent's spec
(`packages/api-server/src/modules/agents/domain/spec-assembly.ts:30-31 @b62d21c`),
and Helm renders them into the template ConfigMap only when set
(`deploy/helm/platform/templates/agent-templates.yaml:46-52 @b62d21c`). The
`k-search-local` template uses this to request the cluster's GPU Kata runtime class
and GPU node label (see [K-Search](../sources/agents.md#k-search--a-workload-harness-on-platform-base)).
The merge-vs-replace semantics live on [Agent](agent.md#per-template-scheduling).

## See also

- [Agent](agent.md) · [agent-lifecycle](../concepts/agent-lifecycle.md) · [agents (harness images)](../sources/agents.md)
