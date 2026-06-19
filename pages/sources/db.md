---
source: dam-agents/dam
commit: 662ebe4c88029788829246170e17465c69523521
files: [packages/db/src/schema.ts, packages/db/src/migrate.ts, packages/db/src/client.ts, packages/db/README.md, docs/architecture/persistence.md]
updated: 2026-06-19
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

## See also

- [persistence-substrates](../concepts/persistence-substrates.md) — Postgres vs. custom resources vs. PVCs, and the rule for choosing.
- [usage tracking](api-server.md) — the `activity_events` log + SQL `usage_*` views read interface.
