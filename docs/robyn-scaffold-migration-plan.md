# Robyn Scaffold — Migration Project Plan (Detailed)

**Status**: v2 · **Date**: 2026-08-07 · **Supersedes**: robyn-scaffold-project-plan.md (v1 outline)
**Companion docs**: robyn-scaffold-requirements.md (R1–R15) · robyn-gap-change-report.md (G1–G12)

**Ordering principle**: Capability manifest first (Phase A) — it converts every later subsystem swap from surgery into provider installation, keeps the fork continuously shippable, and preserves upstream-compatibility mode from day one. Foundations (command registry, DU envelope) are slotted where dependency order forces them.

---

## Phase 0 — Fork Setup & Groundwork

**Objective**: A buildable fork with room for scaffold crates and verified dependencies.

**Work items**
- 0.1 Fork sparckles/Robyn @ v0.88.0; CI green on upstream test suite.
- 0.2 Cargo workspace restructure (G4): Robyn crate + placeholder scaffold crates (`scaffold-core`, `scaffold-registry`, `scaffold-db`, `scaffold-acp`). Maturin wheel build unchanged; workspace publishing dry-run to catch wheel/workspace CI conflicts early.
- 0.3 Dependency compatibility matrix (G6): rust-mcp-sdk/rust-mcp-actix, SeaORM, insta, clap, sc-observability(+OTEL) against Actix-web 4.4.2 / actix-http 3.3.1 / Tokio 1.40 / PyO3 0.27.2 / dashmap 5.4.3. **First check**: rust-mcp-actix minimum Actix version — if it forces a bump, land it as an isolated, upstreamable commit.
- 0.4 Executor coupling probe (G2 risk): inspect `src/executors/` for Actix request-context assumptions that would block non-HTTP invocation of Python handlers. Findings feed Phase C sizing.

**Exit criteria**: Fork builds + passes upstream tests; matrix documented; executor findings written up.
**Risks**: Forced dependency bumps cascading (mitigate: isolated commits); wheel/workspace CI friction.

---

## Phase A — Capability Manifest & Install Options  *(R15, G12; revises G3/G8 dispositions)*

**Objective**: Subsystems become declared capabilities with selectable providers; stock-Robyn behavior reproducible by manifest selection. This is the enabler for Phases B, C, E.

**Work items**
- A.1 Manifest schema (data, serde): capability → provider selection. Initial capability set: `observability` (`basic` | `sc-observability`), `mcp` (`python` | `rust-actix` | `none`), `agent` (`ai-py` | `acp` | `none`). Nothing else — provider selector, not a plugin framework.
- A.2 Provider binding, two layers: cargo features gate Rust providers at compile time; runtime manifest wires Python providers at app startup. Startup refactor: `robyn/__init__` + Rust server boot consult the manifest instead of unconditionally importing `mcp.py` / `ai.py` / logging setup.
- A.3 Provider trait(s) in `scaffold-core`: minimal lifecycle (configure, start, shutdown) + capability-specific interface stubs (filled by Phases B/C/E).
- A.4 Install options: extend `robyn --create` wizard (InquirerPy, existing) with capability questions; manifest written into generated/app config. Existing apps: `robyn capability set <cap> <provider>` command (early registry seed — hand-wired now, migrated onto the real registry in Phase C).
- A.5 Upstream-compat verification: manifest selecting all-upstream providers passes the upstream test suite unchanged.
- A.6 Config resolution groundwork (R13, G7): CLI flag → env → config file → default, reusing `env_populator`; manifest values overridable per that order.

**Exit criteria**: `mcp: python` vs `mcp: none` switch works end-to-end with no code edits; all-upstream manifest = green upstream tests; wizard writes manifests.
**Dependencies**: Phase 0.
**Risks**: Startup-path refactor touches Robyn's most-trafficked code; keep diffs surgical for upstream-PR viability (G12 is upstream-candidate).

---

## Phase B — sc-observability Wiring  *(R10, G9, G11; first provider through the manifest)*

**Objective**: `observability: sc-observability` is a working provider selection; instrumentation lives at the Rust layer.

**Work items**
- B.1 `basic` provider = current Robyn logging behavior, wrapped as a provider (proves the A.3 trait on trivial ground).
- B.2 `sc-observability` provider: structured logging wired through scaffold-core; per-worker initialization post-fork (G9 — never inherited across `processpool` boundaries).
- B.3 Rust-layer HTTP instrumentation: method/route/status/duration assembled in the Actix layer (moots upstream's `after_request(response)` limitation; covers `const=True` routes that bypass Python). Built on the `tracing` ecosystem.
- B.4 OTEL as cargo feature + wizard question; per-worker OTEL export (sidesteps prometheus multiprocess mode).
- B.5 PyO3 stdlib-logging bridge (interim per randlee/sc-observability#88): Python `logging` records → Rust structured pipeline, shape-identical fields. Removal path documented against #88 acceptance criteria.
- B.6 Log-shape invariance test harness: same logical event emitted from Rust and Python compares equal in shape. (Becomes part of the Phase C graduation contract.)
- B.7 Upstream slice (G11): extract B.3 as a neutral `tracing`-based instrumentation branch without the sc-observability dependency; engage on sparckles/Robyn#1276 with working code once stable.

**Exit criteria**: Manifest switch `basic` ↔ `sc-observability` with no code edits; Rust/Python records shape-identical; OTEL export verified when feature on; #1276 branch ready (engagement timing at Rand's discretion).
**Dependencies**: Phase A.
**Risks**: sc-observability Python bindings timeline (external, tracked #88 — bridge fully mitigates); per-worker init bugs are silent (add a worker-id field to every record as a tripwire).

---

## Phase C — Command Registry, DU Envelope, MCP & CLI  *(R1–R3, R9, R14; G2, G3, G5)*

**Objective**: The keystone: schema-as-data registry + envelope, projected to MCP (as the `rust-actix` provider) and the CLI. Largest phase; internally staged.

**Work items — C-1 Foundation**
- C.1 Schema record types in `scaffold-registry`: name, params (types/optionality/help), surface flags + modifiers (`web: read_only` etc., default all-surfaces), canonical stable serialization (snapshot-ready).
- C.2 `CommandRegistry`: schema records and implementation bindings registered *separately* (graduation = one-line binding change, R1.1).
- C.3 `CommandResult<T>` DU envelope + error taxonomy (`Validation{field,expected,got}`, `NotFound`, `Conflict`, `Db`, `Unauthorized`, …), serde-tagged; `retryable` + machine-readable detail on every variant (R9.3). Edge-contract discipline documented: internals use `Result<T, DbError>`; single translation point per command (R9.2).
- C.4 PyO3 exposure: envelope as typed Python classes; `app.command()` decorator via generalized `FunctionInfo`/executor path (G2, informed by 0.4 findings — non-HTTP executor entry point).

**Work items — C-2 CLI surface**
- C.5 clap tree generated from registry; interpreter-free dispatch + `--help` for Rust commands (measured cold-start, N3).
- C.6 Python command schema cache on disk (R2.4 default): help/dispatch decisions interpreter-free; invocation starts the interpreter.
- C.7 `--json` envelope mode + stable exit-code mapping per variant. Migrate A.4's hand-wired `capability set` onto the registry.

**Work items — C-3 MCP provider**
- C.8 `rust-actix` MCP provider (rust-mcp-sdk + rust-mcp-actix) mounted on the Actix server via the manifest; `tools/list` = registry projection filtered by surface flags; tool results carry the envelope (`isError` + tagged JSON).
- C.9 N4 check: manifest guarantees exactly one active MCP provider; `python` provider remains selectable for upstream compat (G3 revised).
- C.10 Web projection (R4): registry commands over HTTP; envelope→status mapping (G5); two response disciplines (upstream routes vs registry commands) documented at the boundary.
- C.11 Contract parity test: same command over CLI `--json`, MCP, and web returns byte-identical envelopes.

**Work items — C-4 Generator delivery**
- C.12 sc-compose rendering pipeline replaces copytree (G1): pip bindings in-process; wizard answers = Jinja2 context; frontmatter conditionals; scaffold-version stamping (`[package.metadata.scaffold]`).
- C.13 `<app>-types` / `<app>-api` templates: Maturin workspace, feature-gated PyO3 module in `<app>-api` (read-only boundary via wheel, R14.2), env conventions, CI (wheel build + tests).

**Exit criteria**: Generated DB-less app with one Rust + one Python command working over CLI/MCP/web with parity-identical envelopes; Rust-command cold start interpreter-free; graduation demo (swap a Python binding to Rust, schema record unchanged).
**Dependencies**: Phases 0 (0.4), A; B for instrumented-by-default templates.
**Risks**: Executor generalization (G2) is the least-known unknown — 0.4 exists to de-risk it; scope creep in taxonomy design (timebox: start with the six variants listed, extend via minor bumps later).

---

## Phase D — Database Layer, Versioning Gates & Sync  *(R5–R8; G9)*

**Objective**: Multi-backend DB with composable schema modules, the approval-gate machinery, and sync v1.

**Work items — D-1 DB core**
- D.1 SeaORM integration in `scaffold-db`; raw SQLx pool escape hatch documented.
- D.2 `SchemaModule` trait: migrations, manifest, version. Per-module migration tracking (independent linear sequences, never interleaved, R7.4).
- D.3 Connection registry: named connections from config (`<APP>_DB_<CONNECTION>_URL` / `_POOLING`), per-worker pools post-fork (R5.6/G9), TLS-required default for `central`, per-connection pooling hint disabling statement cache in transaction mode (Supabase Supavisor, R5.4).
- D.4 Composition proof: app-local module + published crate (`atm-schema` from crates.io) merged into one runner + registry.
- D.5 `<app>-db` template + wizard questions (backends, connections, existing schema crates); smoke configs for local SQLite + Supabase.
- D.6 DbError→envelope translation point at the command layer (R9.2/9.4): typed db errors in, context-decided state-vs-failure out; constraint names surfaced in `Conflict`.

**Work items — D-2 Versioning & approval**
- D.7 Snapshot suite (insta): DB DDL + command schemas + surface flags + error taxonomy + sync policies serialized canonically; any drift = red CI (R8.1).
- D.8 `<app> schema approve` (a registry command): regenerates snapshot, bumps version, appends generated changelog; mechanical semver classification — additive=minor, breaking=major; refuses minor on breaking diff (R8.3).
- D.9 cargo-semver-checks in CI for the scaffold trait surface and `<app>-api` (R8.3); scaffold-version stamp verification.
- D.10 Axis semantics enforced: DB linear (version = migration head; drift without migration file fails), API semver, traits via semver-checks.

**Work items — D-3 Sync v1**
- D.11 SyncPolicy schema metadata (direction, watermark column, origin identity) — lives in the D.7 snapshot.
- D.12 `origin_host` convention; cross-site identity `(origin_host, local_id)`; fan-in collision test with two origins.
- D.13 Watermark append-only push engine; idempotent under re-run and interruption.
- D.14 `sync` registry command (`--from local --to central`): per-table counts, structured partial-failure variants — automatically on CLI/MCP/web via Phase C.

**Exit criteria**: One app live against SQLite `local` + Supabase `central` simultaneously; two-module migrations apply independently; unapproved drift on any axis is red; approve flow correct; SQLite→Postgres push idempotent with zero fan-in collisions.
**Dependencies**: Phase C (registry, envelope, templates). D-2 gates everything after it — **install before D-3 and Phase E add surface area**.
**Risks**: Per-module migration tracking has no off-the-shelf SeaORM support (custom runner — size accordingly); Supavisor behavior differences between pooler modes (smoke-test both).

---

## Phase E — ACP Agent Hosting  *(R12; G8 revised)*

**Objective**: `agent: acp` provider — app-hosted, remotely controlled agent whose toolset is the registry projection.

**Work items**
- E.1 Transport topology decision against then-current ACP (Zed) spec: stdio subprocess vs WebSocket endpoint vs both. Decision recorded as ADR; constraint already fixed: agents consume registry commands regardless of transport (R12.2).
- E.2 Agent session management in `scaffold-acp`: lifecycle, per-worker placement decision (single designated worker vs external process — G9), remote-control auth via env-first config (R13).
- E.3 Toolset projection from registry (surface-filtered, same schema records as CLI/MCP).
- E.4 `ai-py` wrapped as alternative `agent` provider (G8); Phase-9 assessment of its internals as the model-provider layer under the ACP host.
- E.5 Optional web chat interface fronting the same agent session; wizard opt-in + template.
- E.6 Corrective-action demo: agent receives `Validation{expected,got}` and self-corrects parameters without human intervention (R9.3 payoff).

**Exit criteria**: Remote ACP client drives a hosted agent through registry commands including the self-correction demo; manifest switches `acp`/`ai-py`/`none` cleanly; chat demo if in scope.
**Dependencies**: Phases A, C; D-2 (agents operate under approval gates).
**Risks**: ACP spec movement between now and this phase (that's why topology is decided *here*); agent/`processpool` placement.

---

## Phase F — Reference App: ATM Sync (Acceptance)

**Objective**: End-to-end dogfood proving the whole scaffold on a real product-shaped app.

**Work items**
- F.1 Generate app via wizard: composes published `atm-schema`, SQLite `local` + Supabase `central`, sc-observability + OTEL, rust-actix MCP, ACP agent opted in.
- F.2 Sync ATM messages SQLite→central; verify idempotence and fan-in from two machines.
- F.3 Full lifecycle: incubate one new command in Python → stabilize → graduate to Rust with unchanged snapshots (schema, envelope behavior, log shape — N5).
- F.4 Agent-driven operation: ACP agent runs the sync and handles a partial-failure envelope.
- F.5 Friction write-up feeding the backlog; upstream PR queue review (G1, G7, G11, G12 candidates).

**Exit criteria**: F.1–F.4 pass without scaffold-side patches; friction list produced.
**Dependencies**: All phases.

---

## Cross-Phase Notes

- **Parallelism**: B is parallelizable with early C-1 once A lands; D-1 and D-2 can run as parallel tracks; E is serialized behind C and D-2. A, C-1, and D-2 are the chokepoints.
- **Continuous shippability**: after Phase A, the fork always runs in upstream-compat mode; every provider lands behind the manifest, so no phase leaves the tree broken.
- **Self-hosting rule**: from D-2 onward, the scaffold's own schema-affecting changes go through its own approval gate.
- **Upstream queue** (engage when working code exists, not before): G12 manifest mechanism → G1 scaffolder templating → G11 instrumentation slice (#1276) → G7 env_populator.
