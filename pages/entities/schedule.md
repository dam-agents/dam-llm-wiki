---
source: dam-agents/dam
commit: 662ebe4c88029788829246170e17465c69523521
files: [docs/architecture/agent-lifecycle.md, docs/architecture/persistence.md, docs/ubiquitous-language.md]
updated: 2026-06-19
---

# Schedule

A time-triggered task attached to an [Agent](agent.md) — either cron/RRULE-based
or a **Heartbeat** (interval, internally converted to cron)
(`docs/ubiquitous-language.md @662ebe4`).

## Postgres, not a CRD

Schedules are **Postgres rows** owned by the [api-server](../sources/api-server.md);
the controller plays no part (`docs/architecture/persistence.md @662ebe4`). They
store the RRULE, quiet hours, task payload, session mode, and firing
bookkeeping, and survive [Agent](agent.md) deletion as orphans unless the
deletion path explicitly cascades.

## Firing

Each schedule is armed as one delayed job on a Redis-backed queue, re-armed after
every fire (`docs/architecture/agent-lifecycle.md @662ebe4`). The next occurrence
is computed from the cron/RRULE expression in its timezone, **skipping**
occurrences inside an enabled quiet-hours window (suppressed, not deferred — a
schedule whose every occurrence is quiet is rejected at save time). When due, the
api-server commits a `trigger` event to the runtime outbox and pokes the Agent
awake; the event is durable and the schedule re-arms regardless of delivery
outcome. A per-schedule last-fire timestamp on the PVC prevents double-runs. See
[agent-lifecycle](../concepts/agent-lifecycle.md) and
[connections](../concepts/connections-and-contributions.md).

## Session continuity

- **Fresh** — every fire creates a new [Session](session.md) (`session/new`);
  the schedule accumulates a browseable list.
- **Continuous** — first fire creates a session, every later fire
  `session/resume`s it; one session, history retained. A `schedule-reset` event
  clears the binding so the next fire starts fresh.

## See also

- [agent-lifecycle](../concepts/agent-lifecycle.md) · [connections-and-contributions](../concepts/connections-and-contributions.md) · [Session](session.md)
