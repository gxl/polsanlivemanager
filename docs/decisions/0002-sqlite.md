# ADR 0002 — SQLite as the v1 database

- Status: Accepted
- Date: 2026-10-19

## Context
Pilot scale, single deployable (ADR-0001), a small team, and a strong wish for a zero-ops
database that also makes tests fast and leaves no containers behind (docs/testing.md, Resource hygiene).

## Decision
Use one SQLite database file (WAL mode) through EF Core, with one `DbContext` per module and a
table-name prefix per module (`identity_*`, `courses_*`, `scheduling_*`, `attendance_*`, `video_*`).

## Consequences
- Buys: zero server to operate or back up beyond one file; in-memory SQLite in integration
  tests (fast, no Docker); simple local setup.
- Costs: one writer at a time (fine at pilot scale, WAL helps readers); no schemas, so ownership
  is a naming convention checked by FD-2; EF Core on SQLite cannot compare/order `DateTimeOffset`
  in queries — all times are stored as UTC `DateTime` via a value converter; limited ALTER TABLE
  (EF migrations rebuild tables); single-node only, so no horizontal scaling of the host.

## Alternatives considered
- **PostgreSQL** — recommended at bootstrap (schemas per module, timestamptz, concurrency);
  declined for v1 in favour of zero ops.
- **SQL Server / MySQL** — no organisational requirement for either.

## Revisit triggers
Write contention or `SQLITE_BUSY` under load; need for > 1 host instance; multi-tenancy;
backup/HA requirements a single file cannot meet. Move = new ADR; data access is already
confined per module, so the change is per-DbContext.
