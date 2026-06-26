---
source: dam-agents/dam
commit: d507c05fb3683c901473b5166766db03ce14fb29
files: [docs/ubiquitous-language.md, docs/architecture/agent-lifecycle.md, docs/architecture/security-and-credentials.md, packages/controller/api/v1/fork_types.go]
updated: 2026-06-26
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

## See also

- [Agent](agent.md) · [agent-lifecycle](../concepts/agent-lifecycle.md) · [zero-trust-credential-gateway](../concepts/zero-trust-credential-gateway.md)
