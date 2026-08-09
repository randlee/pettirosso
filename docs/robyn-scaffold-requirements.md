# Robyn Scaffold — Requirements Document

**Status**: Draft v1 · **Date**: 2026-08-07 · **Base**: Robyn v0.88.0 fork (Actix-web 4.4.2 / Tokio 1.40 / PyO3 0.27.2 / Maturin)

## 1. Purpose & Vision

A launch-pad scaffold for derivative applications built on a Robyn fork. The scaffold generates per-app crate workspaces that provide database schema, CLI commands, MCP tool schemas, a web surface, and embedded ACP agents from a single source of truth. Infrastructure crates built against the scaffold are portable and reusable across future products (e.g. `atm-schema` consumed from crates.io by multiple apps).

**Lifecycle model (organizing principle):** Python is the incubation layer; Rust is the stabilization layer. Derivative apps consume the scaffold's Rust crates as a read-only foundation, prototype new commands in Python where iteration is cheap, and graduate stable commands to Rust behind an unchanged schema record. Python can *add*, never *alter*.

## 2. Terminology

- **DU envelope** — the discriminated-union `CommandResult<T>` result type. *Docs must never abbreviate this as "DI"*: Robyn ships `dependency_injection.py` and the collision is guaranteed to confuse.
- **Schema record** — a serializable data description of a command (name, params, types, surfaces, help). Schema is data, not code.
- **SchemaModule** — the trait contract a database schema crate exports (migrations, manifest, version).
- **Surface** — a projection of the command registry: CLI, MCP, web, ACP agent tools.
- **Connection** — a named database backend entry in the connection registry (e.g. `local`, `central`).

## 3. Functional Requirements

### R1 — Command Registry (foundation)

1.1 A Rust-owned `CommandRegistry` holds schema records and implementation bindings as *separate* registrations. Graduating a command from Python to Rust is a one-line binding change behind an unchanged schema record.
1.2 Schema records are serializable data: name, parameters (names/types/optionality/help), surface exposure, version linkage. One record projects to the clap CLI tree, MCP `tools/list`, web/OpenAPI, and the ACP agent toolset.
1.3 Surface exposure is controlled by schema fields — `surfaces: {cli, mcp, web}` with per-surface modifiers (e.g. `web: read_only`). Default is all surfaces; the common case needs no annotation. Rare `cli-only` / `mcp-only` / `web-read-only` cases are field-driven, never code-driven.
1.4 Python registers commands via a decorator (`app.command()`) mirroring Robyn's route registration, dispatched through the existing `FunctionInfo` + executor machinery (`TaskLocals` async context).
1.5 App-specific native Rust commands are compiled in via app crates and never touch the Python interpreter (fast path).

### R2 — CLI Surface

2.1 The CLI is generated from the registry (clap tree derived from schema records). No hand-maintained argument definitions.
2.2 CLI runs without MCP: load-on-demand, zero agent-context overhead.
2.3 Pure-Rust commands must dispatch and print `--help` without starting the Python interpreter.
2.4 Python-registered command schemas are cached to disk at registration/startup so `--help` and dispatch *decisions* never require the interpreter; *invocation* of a Python command starts the interpreter (accepted cost during incubation). [Default decision — flag if you want the alternative: always-interpreted for Python commands.]
2.5 A `--json` mode emits the identical DU envelope as MCP, so agents driving the CLI get the same contract.
2.6 Stable exit-code mapping per error variant.

### R3 — MCP Surface

3.1 MCP served in Rust via `rust-mcp-sdk` + `rust-mcp-actix` on the existing Actix server (no adapter boilerplate).
3.2 `tools/list` is a projection of the registry filtered by surface flags; tool results carry the DU envelope as structured content with `isError`.
3.3 The fork must not ship two MCP implementations: Robyn's Python `mcp.py` is superseded (see gap report G3).

### R4 — Web Surface

4.1 Registry commands are exposed over HTTP through the same projection mechanism; `web: read_only` and related modifiers enforced at the routing layer.
4.2 HTTP status codes map mechanically from DU envelope variants; response body is the envelope.

### R5 — Database Layer

5.1 ORM/abstraction: **SeaORM over SQLx**. Rationale: multi-backend requirement neutralizes SQLx per-backend compile-time macros; SeaORM's `DatabaseConnection` is runtime-polymorphic, entities are backend-neutral, migrations run against any backend. Raw SQLx pool remains the hot-path escape hatch.
5.2 Supported backends: SQLite (local) and PostgreSQL (server), provider-neutral. Supabase is the first-prototype target; Azure Database for PostgreSQL and bare Postgres are equivalent targets. MSSQL is out of scope.
5.3 Multiple live backends in one process: a named **connection registry** in config (`local`, `central`, …); commands and entity code target connections by name.
5.4 Per-connection `pooling` hint (session | transaction). Transaction mode (Supabase Supavisor default pooler) disables the prepared-statement cache; session/direct connections need nothing.
5.5 TLS-required is the default for `central` connections.
5.6 The connection registry must be multi-process aware: pools are created per worker process (Robyn `processpool` model), never shared across fork boundaries.

### R6 — Sync (first-class app pattern)

6.1 Per-table **SyncPolicy** lives in schema metadata (direction, watermark column, origin identity) and is part of the versioned snapshot.
6.2 Origin identity convention: tables opting into sync carry an `origin_host` column; cross-site identity is `(origin_host, local_id)`, eliminating PK collisions on fan-in.
6.3 v1 scope: **watermark-based append-only push** (covers ATM-style event sync). Bidirectional sync of mutable rows (conflict resolution) is explicitly out of scope for v1.
6.4 Sync is an ordinary registry command (`<app> sync --from local --to central`) returning the DU envelope with per-table counts and structured partial-failure variants — automatically available over CLI, MCP, web, and to ACP agents.

### R7 — Crate Topology, Generation & Reuse

7.1 Generated per-app crates:
  - `<app>-types` — domain structs, serde, command schema data. No app-specific deps.
  - `<app>-db` — migrations + query layer implementing `SchemaModule`. *Optional*: apps composing purely from published schema crates skip it.
  - `<app>-api` — the consumer contract: command schema records, surface flags, error taxonomy, registry bindings. The read-only surface handed to Python.
7.2 Apps may reference existing published schema crates (e.g. `atm-schema` from crates.io). Reuse of infrastructure crates across apps is a foundational goal.
7.3 `SchemaModule` trait enables composition: the scaffold merges migrations and manifests from N schema crates into one runner and one registry.
7.4 Migrations use **per-module tracking** (per-crate migration table or namespace). Independent crates' linear sequences never interleave; each module applies its own sequence in isolation, so `atm-schema` upgrades on its own cadence without per-app forks.

### R8 — Versioning & Approval Gates

8.1 Universal mechanism: snapshot tests (insta/cargo-insta pattern) serialize the full schema surface — DB DDL, command schemas, surface flags, error taxonomy, sync policies — and diff against a committed snapshot. Any drift fails CI.
8.2 Approval is an explicit scaffold command (`<app> schema approve`) that regenerates the snapshot, bumps the version, and records the diff in a generated changelog. This is a human gate agents cannot silently bypass.
8.3 Axis semantics:
  - **DB schema**: linear, append-only; version = migration head; drift without a new migration file fails CI.
  - **Consumer API** (`<app>-api`): semver with mechanical diff classification — new command / new optional param = minor; removal, rename, type change, tightened surface flag = major. The approve command refuses a minor bump on a breaking diff.
  - **Scaffold Rust trait surface** (`SchemaModule`, registry traits, `CommandResult`): cargo-semver-checks in CI, tied to crate semver.
8.4 The error taxonomy is part of the versioned consumer API: add variant = minor; remove/restructure = breaking.
8.5 The generator stamps the scaffold version into each generated crate (`[package.metadata.scaffold]`) so derivative apps can be upgraded knowingly when the scaffold evolves.

### R9 — DU Result Envelope (keystone type)

9.1 Canonical `CommandResult<T>`: serde-tagged discriminated union with structured error taxonomy (`Validation {field, expected, got}`, `NotFound`, `Conflict`, `Db`, `Unauthorized`, …).
9.2 The taxonomy is an **edge contract**. Internal layers use plain `Result<T, DbError>`; `SchemaModule` returns typed db errors and nothing more. The command layer is the single translation point and decides, per context, whether a `DbError` is *state* (NotFound on an existence check is a good `Ok` answer) or *failure* worth an envelope variant with a context-specific message. No inter-layer DTO ceremony.
9.3 Agent-actionable by construction: variants carry `retryable` and machine-readable detail (`expected`/`got` on validation) so agents self-correct without human intervention.
9.4 DB errors are translated, never leaked: constraint violation → `Conflict` with the constraint name, not a raw Postgres string.
9.5 The same enum crosses PyO3 as typed Python classes; incubating Python commands return the same envelope, so error behavior is contract-conformant before graduation.

### R10 — Observability

10.1 All logging via `sc-observability` (structured logging; OTEL as an optional cargo feature). A wizard question controls OTEL inclusion in generated apps.
10.2 Gap: no Maturin/Python bindings today — tracked as **randlee/sc-observability#88**. Interim: a PyO3 bridge routes Python stdlib `logging` records into the Rust structured logger so incubated commands emit the same log shape from day one; the bridge is deleted when bindings land with no consumer-visible change.
10.3 Log-shape invariance across Python→Rust graduation is a hard requirement.

### R11 — Scaffolding Generator

11.1 Extend Robyn's existing `robyn --create` wizard (InquirerPy) with: app name, surface selection, DB/connection choices, existing-schema-crate references, OTEL y/n, ACP agent y/n.
11.2 Replace `shutil.copytree` with **sc-compose** rendering (in-process via the pip bindings): wizard answers = Jinja2 context; YAML frontmatter drives per-file conditionals (skip `<app>-db`, Dockerfile, etc.).
11.3 Generated output is a Maturin workspace with CI that builds wheels and runs the versioning test suites.

### R12 — Embedded ACP Agents

12.1 The app **hosts** the agent; ACP (Zed's Agent Client Protocol) is the remote-control surface.
12.2 The agent's toolset is the command-registry projection — the same schema records as CLI/MCP. Whatever the transport, agents consume registry commands; the ACP phase must not fork the architecture.
12.3 Optionally the agent services a web chat interface fronting the same agent session.
12.4 Agent role/persona is intentionally open-ended and app-defined. The scaffold provides plumbing only: session management, tool projection, transport. Feature-gated; apps opt in at generation time.
12.5 Detailed transport topology (stdio subprocess vs. WebSocket endpoint vs. both) is decided within the ACP phase against then-current ACP spec state.

### R13 — Configuration & Identity

13.1 Environment is the primary channel at every level; resolution order: **CLI flag → environment → config file → default**.
13.2 Naming convention generated from the connection registry: `<APP>_DB_<CONNECTION>_URL`, `<APP>_DB_<CONNECTION>_POOLING`, etc.
13.3 Config-file fallback reuses Robyn's existing `env_populator` (.env loading already in the CLI startup path).
13.4 Templates render env *references*, never secret values. Secrets never appear in generated files or CLI args.
13.5 Managed-identity auth (Azure) is a later enhancement, not v1.

### R14 — Build & Packaging

14.1 Maturin across all interfaces: Robyn fork itself and every generated app that exposes a Python surface builds as Maturin wheels.
14.2 `<app>-api` carries a feature-gated PyO3 module (or thin `<app>-api-py` binding crate) exposing schema records, the registration decorator, and result types as typed compiled objects — the read-only boundary enforced by the wheel, not convention.
14.3 Generated CI includes wheel builds, snapshot/versioning tests, and cargo-semver-checks.

### R15 — Capability Manifest

15.1 Robyn subsystems with alternative implementations are modeled as **capabilities** with selectable providers, declared in a manifest rather than hardcoded imports: `mcp: rust-actix | python | none`, `observability: sc-observability | basic`, `agent: acp | none`. Only capabilities with real alternatives get entries.
15.2 Two binding layers: cargo features select Rust providers at compile time; the runtime manifest governs Python-side providers and startup wiring.
15.3 The wizard writes the initial manifest; scaffold defaults select Rust providers; a manifest selecting upstream's Python providers reproduces stock Robyn behavior (upstream compatibility mode).
15.4 The manifest is generated app config; its effects on exposed surfaces are already captured by the surface snapshot (R8), so it needs no separate versioning axis.

## 4. Non-Functional Requirements

- **N1** Read-only foundation: derivative-app Python cannot mutate DB schema, command schemas, or MCP tool schemas; enforcement via crate/wheel boundary.
- **N2** Portability: crates built against the scaffold are publishable and consumable by other scaffold apps without modification.
- **N3** CLI startup for pure-Rust commands imposes no Python interpreter cost.
- **N4** Exactly one MCP implementation ships in the fork.
- **N5** Graduation invariance: a Python→Rust command migration changes neither its schema record, its envelope behavior, nor its log shape — proven by unchanged snapshots.

## 5. Out of Scope (v1)

MSSQL support · bidirectional/mutable-row sync and conflict resolution · Azure managed-identity auth · ACP transport finalization ahead of its phase · general CRDT machinery.

## 6. Open Items

- **O1** R2.4 default (cached Python command schemas for interpreter-free `--help`) — confirm or choose always-interpreted.
- **O2** sc-observability Python bindings — external dependency, tracked at randlee/sc-observability#88; interim bridge specified in R10.2.
- **O3** rust-mcp-actix / SeaORM version compatibility against Actix-web 4.4.2 + Tokio 1.40 + PyO3 0.27.2 — verification task in Phase 0 (see gap report G6).
