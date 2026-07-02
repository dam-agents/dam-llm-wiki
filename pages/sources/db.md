---
source: dam-agents/dam
commit: b62d21c288162847d7d9918ca7887265448fe2b3
files: [packages/db/src/schema.ts, packages/db/src/migrate.ts, packages/db/src/client.ts, packages/db/README.md, docs/architecture/persistence.md, docs/adrs/071-postgres-role-separation.md, docs/notes/postgres-role-operations.md, deploy/helm/platform/templates/postgres.yaml, deploy/helm/platform/templates/postgres-migrate-roles.yaml, deploy/helm/platform/values.yaml]
updated: 2026-07-02
---

# db

The Postgres layer the [api-server](api-server.md) owns end-to-end — the
**Application State** substrate. The controller never reads or writes Postgres
(`docs/architecture/persistence.md @662ebe4`). It holds everything that must be
queryable when no agent pod is running: channel routing/bindings, identity links
+ auth allow-list + API keys, the skills catalog (sources, install records,
publish history), the activity log + agent ownership mirror,
[schedules](../entities/schedule.md), and per-agent user-typed env. **Session
metadata is not here** — it is agent-owned on the PVC.

As of [#1899](https://github.com/dam-agents/dam/pull/1899) (`@d507c05`,
migration `0003_smart_squadron_sinister.sql`) the **`agent_env`** table holds
user-typed env per agent — `(agent_id, name, value)` keyed on `(agent_id, name)`,
indexed by `agent_id` (`packages/db/src/schema.ts:269-279 @d507c05`). It is the
store behind the UI Environment editor and is read by the api-server's
state-builder as `env` [contributions](../concepts/connections-and-contributions.md),
replacing the Agent CR's `spec.env` (the CR field is retained but no longer read).
The companion migration `0004_open_mariko_yashida.sql` adds a nullable `path`
column to both `skill_sources` and `agent_skills` for the
[skill-source subdir](../concepts/skill-resolution.md) feature
(`schema.ts:161 @d507c05`, `:185 @d507c05`).

The **[Experiments](../concepts/experiments.md)** subsystem
([#2033](https://github.com/dam-agents/dam/pull/2033)) adds four tables across
migrations `0005`–`0009`: `experiments` (owner-scoped, `owner+name` unique),
`experiment_arms` (PK `(experiment_id, agent_id)`), and the append-only
`experiment_runs` Run Ledger (`run_number` unique per arm, `score` as opaque
`jsonb`) all land in `0005` (`packages/db/src/schema.ts:474-549 @b62d21c`);
`run_artifacts`, the Candidate blob store holding content inline as `bytea`, lands
in `0006` (`packages/db/src/schema.ts:560-572 @b62d21c`). Three follow-ups reshape
the arm model: `0007` collapses `experiments` to a single `prompt` column, `0008`
adds per-arm `status` + `last_activity_at` (the liveness clock the inactivity sweep
reads, with a partial index on running arms), and `0009` replaces `arm_spec` with
`arm_variation` (`packages/db/src/schema.ts:498-524 @b62d21c`).

The [secrets→connections cutover](../concepts/connections-and-contributions.md#secrets--connections-cutover-2200)
([#2200](https://github.com/dam-agents/dam/pull/2200)) touched **no Postgres
schema** — legacy secrets were K8s Secrets, and the `connections`/`connection_grants`
tables predate it (`packages/db/src/schema.ts:322,348 @b62d21c`).

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
