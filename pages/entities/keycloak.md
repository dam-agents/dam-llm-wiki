---
source: dam-agents/dam
commit: 662ebe4c88029788829246170e17465c69523521
files: [docs/architecture/security-and-credentials.md, packages/keycloak-theme/]
updated: 2026-06-19
---

# Keycloak

The **only identity authority** in DAM — an in-cluster Helm subchart and the
OIDC provider for every authenticated surface
(`docs/architecture/security-and-credentials.md @662ebe4`).

## Auth flow

1. The browser authenticates against Keycloak and obtains a JWT with audience
   `platform-api`.
2. The [UI](../sources/ui.md) sends the JWT to the
   [api-server](../sources/api-server.md) on every tRPC and ACP call; the
   api-server validates it against Keycloak's JWKS.
3. The `sub` claim becomes `agent-platform.ai/owner=<sub>` on every resource the
   user creates — the basis of
   [per-owner credential isolation](../concepts/zero-trust-credential-gateway.md).

There is no token exchange; credential storage is K8s-native and label-scoped.
For headless/CI use the [CLI](../sources/cli.md) accepts a long-lived **API key**
(`pk_` prefix) in the same `Authorization: Bearer` slot; API keys cannot mint or
revoke other keys, so an exfiltrated key cannot escalate. The CLI's `dam auth
login` uses Keycloak device grant against the public `platform-cli` client.

## Audit event source

Keycloak emits login and admin events to pod stdout via its `jboss-logging`
listener (the external log service is the audit source of truth). Login events
are **not** written to the Keycloak DB (high volume); admin-event metadata is
recorded to Postgres but request bodies are not
(`adminEventsDetailsEnabled` off) to avoid storing plaintext credentials
(`docs/architecture/security-and-credentials.md @662ebe4`).

## Theme

The login/account UI theme is the `keycloak-theme` package
(`packages/keycloak-theme/ @662ebe4`).

## See also

- [zero-trust-credential-gateway](../concepts/zero-trust-credential-gateway.md) · [api-server](../sources/api-server.md) · [cli](../sources/cli.md)
