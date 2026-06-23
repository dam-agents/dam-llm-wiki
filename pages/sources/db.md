---
source: dam-agents/dam
commit: 380cb06d1d60bca40fa703b77e13a16ec96eedf7
files: [packages/db/src/schema.ts, packages/db/src/migrate.ts, packages/db/src/client.ts, packages/db/README.md, docs/architecture/persistence.md, docs/adrs/071-postgres-role-separation.md, docs/notes/postgres-role-operations.md, deploy/helm/platform/templates/postgres.yaml, deploy/helm/platform/templates/postgres-migrate-roles.yaml, deploy/helm/platform/values.yaml]
updated: 2026-06-23
---

# db

The Postgres layer the [api-server](api-server.md) owns end-to-end — the
**Application State** substrate. The controller never reads or writes Postgres
(`docs/architecture/persistence.md @662ebe4`). It holds everything that must be
queryable when no agent pod is running: channel routing/bindings, identity links
+ auth allow-list + API keys, the skills catalog (sources, install records,
publish history), the activity log + agent ownership mirror, and
[schedules](../entities/schedule.md). **Session metadata is not here** — it is
agent-owned on the PVC.

## Layout (`packages/db/src/`)

- `schema.ts` — the authoritative table/index/enum schema.
- `migrate.ts` — migrations run automatically on api-server startup; table/index/
  enum changes are generated from the schema, the reporting (`usage_*`) views are
  hand-written, and original history is squashed to a baseline fresh installs run
  and existing deployments skip (`docs/architecture/persistence.md @662ebe4`,
  `packages/db/README.md @662ebe4`).
- `client.ts` — the Postgres client. A no-database guard asserts every schema
  change was generated.

## Role separation & DBA audit (bundled Postgres)

For the **bundled** in-cluster Postgres, the Helm chart abandoned the single
SUPERUSER role for **three roles** (ADR-071,
`docs/adrs/071-postgres-role-separation.md:1 @380cb06`): `platform_apiserver`
and `platform_keycloak` are `LOGIN NOSUPERUSER` roles, each the sole connection
identity for its own database (`platform` / `keycloak`), while the bootstrap
superuser `platform` is reserved for DBA work
(`deploy/helm/platform/templates/postgres.yaml:36-43 @380cb06`,
`deploy/helm/platform/values.yaml:170-174 @380cb06`). Cross-database isolation
is enforced by revoking `CONNECT` from `PUBLIC` and granting it back only to the
owning role, so a leaked app credential is bounded to one database and cannot
reach the other service's data or escalate
(`postgres.yaml:62-65 @380cb06`). A regression guard raises an exception
(aborting init / failing the migration) if either app role is ever SUPERUSER
(`postgres.yaml:154 @380cb06`).

DBA sessions are **statement-logged** by default — a per-role
`ALTER ROLE platform SET log_statement = 'all'` — while the server-wide
`log_statement` stays at `ddl`, so routine app traffic stays out of the audit
stream (`postgres.yaml:70 @380cb06`). The whole layout is one idempotent
`01-roles.sql`, applied both by the image on first init and by a
`post-install,post-upgrade` Helm-hook Job that converges an existing
single-superuser cluster
(`deploy/helm/platform/templates/postgres-migrate-roles.yaml @380cb06`). The
audit is best-effort, not enforced (a superuser session can re-`SET`
mid-session); operational details are in
`docs/notes/postgres-role-operations.md @380cb06`. This boundary applies to the
bundled instance — a managed/external Postgres is out of scope.

## See also

- [persistence-substrates](../concepts/persistence-substrates.md) — Postgres vs. custom resources vs. PVCs, and the rule for choosing.
- [usage tracking](api-server.md) — the `activity_events` log + SQL `usage_*` views read interface.
