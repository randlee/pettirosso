# Phase D — Database Layer, Versioning Gates & Sync (planning depth)

**Component added**: SeaORM 2.0 multi-backend DB in `pettirosso-db`, the approval-gate machinery, sync v1.
**Detail level**: planning depth. D-1 and D-2 are parallel tracks; **D-2 gates all later surface area** (self-hosting rule starts here).

## Decisions already fixed (extraction §5)

- SeaORM 2.0; no custom Storage trait — `DatabaseConnection` is the abstraction.
- Generated schema crate is `<app>-schema` (renamed from `<app>-db`): SeaORM entities + migrations + a small set of **typed async query functions as the only sanctioned DB access**; raw Entity/Column types never leak.
- **One path for DB writes** — single write entry point so sync/audit/error-translation/observability have exactly one place to be correct.
- Tri-state per-app wizard choice: schema-first / model-first (SeaORM "entity-first") / none.

## D-1 DB core

Connection registry (named connections, per-worker pools post-fork, TLS default for `central`, Supavisor transaction-mode statement-cache handling); `SchemaModule` trait with **per-module migration tracking** (independent linear sequences — custom runner, no off-the-shelf SeaORM support, size accordingly); composition proof: app-local module + published `atm-schema`; DbError→envelope translation at the command layer with constraint names surfaced in `Conflict`. Resolve O-5 (`_DB_` vs `_SCHEMA_` env naming) here.

## D-2 Versioning & approval (chokepoint)

insta snapshot suite over DDL + command schemas + surface flags + taxonomy + sync policies; `<app> schema approve` as a registry command (regenerate, bump, changelog; refuses minor on breaking diff); cargo-semver-checks on trait surface and `<app>-api`.
**Must resolve O-1 before D-2 sprint docs**: model-first mode has no migration head — snapshot target = entity definitions or SeaORM-derived DDL. This is a real design fork (R8 contradiction), not wording; one ADR required.

## D-3 Sync v1

SyncPolicy in schema metadata (in the D-2 snapshot); `origin_host` convention, identity `(origin_host, local_id)`; watermark append-only push, idempotent under re-run/interruption; `sync` as an ordinary registry command → automatically on all adapters via Phase C.

## Exit criteria

One app live against SQLite `local` + Supabase `central`; two-module migrations independent; unapproved drift on any axis red; both wizard DB modes produce approvable apps; SQLite→Postgres push idempotent, zero fan-in collisions from two origins.
