# Conversation Extraction — "Extending Robyn with optional crates"

**Date**: 2026-08-08 · **Source**: claude-desktop project conversation (Aug 7–8) + archive/sonnet-2026-08-08 branch
**Purpose**: Salvage decisions and open items from the prior session so the plan restart (and the move to Claude Code) loses nothing. Where this document conflicts with the three authoritative design docs in `docs/`, the design docs are stale and must be updated per the "Doc impact" column below.

## 1. Product model — the app gradient (supersedes "two tiers")

- **Tier 1 — Python-only rapid prototype.** Today's Robyn experience: `pip install pettirosso`, Python route/command handlers, zero Rust exposure. Extended by making more of the foundation Rust (MCP, database), but the app developer works entirely in Python against the scaffolded crates. The compiled server is real, just invisible.
- **Tier 2 — Harden the contract.** After scaffolding, the developer starts using cargo to harden parts of the design — schema and CLI contract move into owned Rust crates so the schema can't be edited from the Python environment and the core API doesn't bend.
- **Tier 3 — Performance migration.** Business logic / critical image-processing / math migrated to Rust for performance.
- The gradient is the organizing concept: the scaffold's job is making each step *cheap*, and Python command registration is a permanent **plugin mechanism** (great for command extension at any tier), not merely an incubation stage.

## 2. Crate organization (decided)

- Naming: `pettirosso-*`, flat `crates/` directory (matches atm-core / sc-compose precedent). Many small crates over few large (atm-core ~13-crate granularity), including a dedicated error crate.
- `crates/robyn/` stays **vendored, minimal-diff** from upstream so `git fetch upstream && git merge` stays low-friction. New logic in sibling crates, never edited in.
- Crate list: `pettirosso-manifest` (built once on archive branch, spec proven), `pettirosso-error`, `pettirosso-registry`, `pettirosso-observability`, `pettirosso-db`, `pettirosso-acp`, `pettirosso-cli`, proposed `pettirosso-protocol` (wire types shared by CLI client and server dispatch — proposed, not confirmed).
- `pettirosso` is unclaimed on **both PyPI and crates.io** (verified via registry APIs). Reserve early.

## 3. CLI architecture (decided — major change vs. requirements doc)

- CLI is a small native Rust `[[bin]]` acting as an **HTTP client to the running server**, supporting **UDS (local default) and TCP (remote)** on the same Actix `HttpServer`.
- Maturin bindings on the CLI so Python consumes it **in-process** — never shell out to a Python CLI.
- Consequence: CLI/MCP/HTTP become three protocol adapters in front of **one operation layer** (a principle Rand stated directly: never three implementations that happen to agree). Python interpreter cost is paid once at server start; **O1 (schema caching) is moot**; N3 is true by construction.
- The `creating-ai-clis` skill (proven by a colleague) covers the CLI/MCP contract layer — apply it, don't redesign it.

## 4. Wizard / generator (decided)

- The wizard is a **separate one-shot product**: it runs before any server exists, so it is categorically outside the CLI-as-HTTP-client architecture. Stays Python / InquirerPy / sc-compose, console-script invoked.
- No upgrade mode needed: incremental capability changes are handled by `capability set <cap> <provider>` (Sprint A3 shape), not by re-running the wizard.
- sc-compose renders **actual crate source into the new app's repo** (Tier 2), not just dependency references. Tier 1 output contains no Rust at all.
- Generated Tier-2 topology: `my-app-types`, `my-app-schema` (renamed from `-db`), `my-app-api`, `my-app-cli`, plus `python/` plugin directory.

## 5. Database layer (decided)

- SeaORM 2.0. No custom `Storage` trait — `DatabaseConnection` is already runtime-polymorphic across SQLite/Postgres, which satisfies the historical reason for Storage traits.
- `my-app-schema` exposes a **small set of typed async query functions** as the only sanctioned DB access; raw Entity/Column types never leak. **One path through the code for DB writes** (Rand's principle) — single write entry point for sync/audit/translation/observability correctness.
- Tri-state wizard choice per app: **schema-first / model-first (SeaORM: "entity-first") / none**.

## 6. Requirements-doc impact table (from the prior session's own audit)

| Req | Status | Change needed |
|---|---|---|
| R2 CLI | Superseded | Rewrite around native HTTP-client binary; O1 removed |
| R3/R4 | Mostly holds | Add UDS/TCP dual transport |
| R5 DB | Fork added | schema-first vs model-first per-app wizard choice |
| R7 Topology | Amend | Add `<app>-cli`; rename `<app>-db` → `<app>-schema` |
| R8 Versioning | **Contradicted** | Model-first has no migration head — needs different snapshot target (open design) |
| R9 Envelope | Unchanged | — |
| R11 Generator | Amend | Tier 1 vs Tier 2 output paths distinguished; wizard stays Python |
| R14 Packaging | Tier-blind | Maturin wheel not universal (Tier 2/3 apps may have no Python surface) |
| R15 Manifest | Verified | Sprint A1 built + tested it as specified (archive branch) |

## 7. Open items (carried forward, unresolved)

1. **R8 fork**: versioning/approval-gate mechanism for model-first mode (no migration files → snapshot entity definitions or SeaORM-derived DDL). Blocks generator DB-mode templates.
2. **`pettirosso-protocol`** crate: real or premature?
3. **`pettirosso-cli-core`** crate: real or premature?
4. **crates.io name for the runtime core** (`robyn` collides with an unrelated MLIR crate).
5. **`ManifestError` → `pettirosso-error`** extraction (small, mechanical, needs a nod).
6. Config naming: `_DB_` vs `_SCHEMA_` env prefix.

## 8. Salvage inventory from archive/sonnet-2026-08-08

- `pettirosso-manifest` crate: built and tested to Sprint A1 spec — **spec is proven**; code will be rebuilt (or cherry-picked after review), not trusted by default.
- Phase-A sprint docs (A1–A4): structurally sound, stale crate names — re-derived in this plan.
- Phase-G sprint docs (G0–G6): generator plan with correct dependency graph — imported as Phase E here. G0 (`dev-render.py` + `templates/minimal-app/`) is unreviewed spike code documented retroactively; review before reuse.
- Everything else on the branch: discard.

## 9. Process (unchanged, absolute)

No code, no commits, no independent action until the plan is reviewed and Rand specifies the step. Sprint docs follow `atm-core/.claude/skills/plan-hardening/sprint-planning-guidelines.md`.
