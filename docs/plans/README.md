# Pettirosso — Master Plan Index

**Status**: Draft v3 · **Date**: 2026-08-08 · **Supersedes**: migration-plan v2 phase ordering where they conflict
**Inputs**: docs/robyn-scaffold-requirements.md (with extraction-2026-08-08.md impact table applied) · docs/robyn-gap-change-report.md · docs/plans/extraction-2026-08-08.md
**Process**: every sprint doc hardened against atm-core sprint-planning-guidelines.md; nothing is built until its sprint doc passes review.

## Organizing concepts

1. **The app gradient** (extraction §1): Tier 1 Python-only prototype → Tier 2 contract hardening in Rust → Tier 3 performance migration to Rust. Each phase must leave the gradient cheaper to traverse; the scaffold is *usable* when the wizard can emit a working Tier-1 and Tier-2 app (end of Phase E).
2. **One operation layer**: CLI, MCP, and web are protocol adapters over the same command registry; the CLI is a native HTTP client (UDS/TCP) to the running server.
3. **Optional components via the capability manifest**: every subsystem with alternatives is a manifest-selected provider; an all-upstream manifest reproduces stock Robyn.
4. **Vendored core**: `crates/robyn` stays minimal-diff for upstream tracking; all new logic in `pettirosso-*` sibling crates.

## Phase map — each phase adds one critical component

| Phase | Component added | Detail level now | Gate |
|---|---|---|---|
| 0 | Buildable fork: workspace, name reservation, dep matrix, executor probe | **Sprint docs** | upstream tests green |
| A | Capability manifest (+ `pettirosso-error`) — the optional-components mechanism | **Sprint docs** | provider switch with zero code edits; all-upstream manifest = stock Robyn |
| B | Observability provider (`sc-observability`, OTEL feature) | Planning depth | manifest switch `basic` ↔ `sc-observability`; log-shape invariance harness |
| C | Command registry + DU envelope + protocol + native CLI + **Rust MCP** provider | Planning depth | one command, byte-identical envelopes over CLI `--json` / MCP / web |
| D | Database layer (**SeaORM**) + versioning gates + sync v1 | Planning depth | SQLite `local` + Supabase `central` live; unapproved drift = red CI |
| E | Generator/wizard (sc-compose; Tier 1 + Tier 2 output) — **usable-scaffold milestone** | Planning depth (imports archived Phase-G structure) | wizard emits building, testing Tier-1 and Tier-2 apps |
| F | ACP agent hosting (optional component) | Outline | remote agent drives registry commands; self-correction demo |
| G | Reference app acceptance (ATM sync dogfood) | Outline | full lifecycle incl. Tier-1→2 graduation with unchanged snapshots |

**Detail-level rule** (anti-40-sprint rule): only the next executable phase carries full sprint docs. Each later phase is a README at planning depth — real enough to review, naming its own open items — and is expanded into sprint docs only when its predecessor's findings land. Expanding a future phase early is a process violation, not diligence.

## Chokepoints & parallelism

- Chokepoints: A (manifest), C-1 (registry/envelope), D-2 (approval gates).
- B is parallel-safe with early C once A lands. D-1/D-2 are parallel tracks. E depends on C (templates render against real crates) and on the R8 fork resolution for its DB modes. F serialized behind C and D-2.

## Open items register (blocking specific phases)

| # | Item | Blocks |
|---|---|---|
| O-1 | R8 versioning mechanism for model-first (no migration head) | D-2, E (DB-mode templates) |
| O-2 | `pettirosso-protocol` — real crate or premature | C |
| O-3 | `pettirosso-cli-core` — real crate or premature | C, E |
| O-4 | crates.io name for the runtime core (`robyn` collides) | 0 (reservation), C (publishing) |
| O-5 | `_DB_` vs `_SCHEMA_` env naming | D |

## Standing process rules

- Review gate before any build step; sprint docs are the authoritative scope.
- Self-hosting rule from D-2 onward: the scaffold's own schema-affecting changes go through its own approval gate.
- Upstream queue (engage when working code exists): G12 manifest → G1 scaffolder templating → G11 instrumentation (#1276) → G7 env_populator.
