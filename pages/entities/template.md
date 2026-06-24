---
source: dam-agents/dam
commit: d34c21a008d3b868fc260838374836ac88fb0807
files: [docs/ubiquitous-language.md, docs/architecture/persistence.md, docs/architecture/agent-lifecycle.md]
updated: 2026-06-24
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

A Template contributes at **create time only**: the api-server's
`assembleSpecFromTemplate` step copies template image/mounts/env into the new
Agent's `spec`. The controller never reads the Template again at pod start, so
**editing a Template never re-flows into a running Agent** — there is no
"template envs" runtime layer (`docs/architecture/agent-lifecycle.md @662ebe4`).
The default Claude Code template persists the workspace and `$HOME`.

## See also

- [Agent](agent.md) · [agent-lifecycle](../concepts/agent-lifecycle.md) · [agents (harness images)](../sources/agents.md)
