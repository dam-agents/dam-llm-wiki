---
source: dam-agents/dam
commit: b62d21c288162847d7d9918ca7887265448fe2b3
files: [docs/ubiquitous-language.md, docs/architecture/agent-lifecycle.md, docs/architecture/security-and-credentials.md, packages/controller/api/v1/fork_types.go]
updated: 2026-07-02
---

# Fork

An **ephemeral, per-turn execution environment** derived from an [Agent](agent.md)
that impersonates a foreign user for the duration of **one Slack turn**
(`docs/ubiquitous-language.md @662ebe4`). It is the third durable concept in the
Agents bounded context, alongside [Template](template.md) and Agent.

## Job-shaped, run-to-completion

Unlike an Agent, a `Fork` CR (`packages/controller/api/v1/fork_types.go @662ebe4`)
reconciles to a Kubernetes **Job**, not a StatefulSet — it runs to completion and
is never woken, hibernated, or kept warm (`docs/architecture/agent-lifecycle.md @662ebe4`).
Forks are owner-refed to the parent Agent CR, so Kubernetes garbage-collects them
automatically. A **Fork Phase** is Pending, Ready, Failed, or Completed.

## Why it exists — the foreign replier

When a user other than the Agent owner replies in a bound Slack thread (a
**Foreign Replier** in the Agent's `allowedUsers`, with a **Foreign Sub**), the
api-server creates a Fork so the turn runs under **that replier's** credentials,
not the owner's (`docs/architecture/security-and-credentials.md @662ebe4`). The
fork's [gateway](envoy-gateway.md) pod mounts Secrets selected by
`agent-platform.ai/owner=<replier-sub>`; the credential boundary is preserved.

## Identity isolation

A fork pair gets its **own** per-fork ServiceAccount, distinct from the parent's,
with narrow per-fork AuthorizationPolicies admitting it only to
`/api/agents/<parent>/mcp` and the parent's ext-authz Service — so a compromised
fork can't impersonate the parent (`docs/architecture/security-and-credentials.md @662ebe4`).
Its `agent` label still points at the parent so traffic resolves under the
parent's egress rules; its own pair key isolates it.

## vs. Run

The [Run](run.md) (the `dam-run` executor, added in
[#2120](https://github.com/dam-agents/dam/pull/2120)) is the *other* ephemeral
Agent-derived CR, and the two share the controller's ephemeral-pod builder. But
they make **opposite identity trades**: a Fork is a Job with its **own** isolated
SA and gateway; a Run is a bare Pod that runs as the **parent's own owner** and
borrows the parent's gateway, holding no SA or policy of its own
(`docs/architecture/security-and-credentials.md @70c53ae`). A Fork runs to
completion (Job); a Run lives only as long as its `dam-run` stream.

## See also

- [Agent](agent.md) · [Run](run.md) · [agent-lifecycle](../concepts/agent-lifecycle.md) · [zero-trust-credential-gateway](../concepts/zero-trust-credential-gateway.md)
