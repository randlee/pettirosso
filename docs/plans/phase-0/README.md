# Phase 0 — Fork Setup & Groundwork: Sprint Plan Index

**Status**: Draft v1 · **Date**: 2026-08-08 · **Hardened against**: sprint-planning-guidelines.md (atm-core)
**Component added**: a buildable, upstream-tracking fork with room for `pettirosso-*` crates.

## Split rationale

Two sprints, two closure types: 0A is structural/build work (workspace, CI, registry reservations); 0B is investigation producing written findings (dependency matrix, executor probe). Bundling them would let the investigation slip silently behind "the fork builds".

| Sprint | Title | Closure type | Recommended agent |
|---|---|---|---|
| 0A | Workspace, CI & name reservation | Build/structural | Cipher-311d/fast |
| 0B | Dependency matrix & executor coupling probe | Investigation/findings | arch-ctm/deep-reasoning |

## Dependency relations

| Sprint | Relation | Depends on | Rationale |
|---|---|---|---|
| 0A | — | — | Leaf. |
| 0B | parallel_safe | 0A | Reads upstream source + registries only; touches no files 0A owns. Merge before Phase A kickoff. |
