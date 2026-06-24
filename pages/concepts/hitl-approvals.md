---
source: dam-agents/dam
commit: d34c21a008d3b868fc260838374836ac88fb0807
files: [docs/ubiquitous-language.md, docs/architecture/security-and-credentials.md]
updated: 2026-06-24
---

# HITL approvals & egress rules

Human-in-the-loop approval gates either a **credentialed egress request**
(ext_authz, via the [credential gateway](zero-trust-credential-gateway.md)) or a
**harness tool call** (acp_native). Approvals persist in the `pending_approvals`
table (`docs/ubiquitous-language.md @662ebe4`).

## Vocabulary

- **Approval / Pending Approval** — a user-pending decision; pending ones live in
  the **Inbox** (top-level page, sidebar bell + badge, per-Agent tray).
- **Verdict** — `allow_once`, `allow`, `deny_once`, or `deny`. The `*_once`
  verdicts resolve only the held call (no rule written); `allow`/`deny` also
  write a permanent **egress rule** for ext_authz.
- **Held Call** — an ext_authz request blocking on the api-server awaiting a
  verdict, up to `approvalHoldSeconds` (default 30 min); the durable pending row
  outlives the hold.
- **Synth Frame** — a synthetic ACP `session/request_permission` frame the relay
  injects into an attached client WS for an ext_authz approval; the synthetic
  session id has an `_egress:` prefix so the UI routes it to the inbox rather
  than the in-session permission queue.
- **Action Outcome** — what a mutation reports: `applied`,
  `rule_written_expired`, or `not_actionable` (unknown/foreign/already-settled,
  deliberately indistinguishable), plus the egress rule written (or `null`).

## Egress rules

An **Egress Rule** is a persistent allow/deny keyed on
`(agent, host, method, path_pattern)`, matched on **every** ext_authz Check
before any user prompt (`docs/ubiquitous-language.md @662ebe4`). A **Rule Match**
finds the most-specific active rule; a miss falls through to the ext_authz
Gate's pending-approval flow. The same flow fails **closed** — a blocked Check
returns 403 with no prompt (`docs/architecture/security-and-credentials.md @662ebe4`).

## The ext_authz Gate

The application service running Envoy's HTTP ext_authz Check: rule lookup →
pending-row creation → synth-frame fan-out → synchronous hold → wake-up →
expiry. acp_native rows resolve via a **Wrapper Response** forwarded by whichever
replica holds the agent's upstream WS.

## See also

- [zero-trust-credential-gateway](zero-trust-credential-gateway.md) · [ACP](agent-client-protocol.md) · [api-server](../sources/api-server.md) (`approvals`/`egress-rules` modules)
