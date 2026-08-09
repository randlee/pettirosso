# Robyn Gap/Change Report

**Status**: Draft v1 · **Date**: 2026-08-07 · **Base**: Robyn v0.88.0 · Changes required in Robyn itself to meet the scaffold requirements, with disposition (fork-only vs. upstream-candidate). Overall strategy: **fork as the product**, with individually upstreamable slices where changes are generic.

## G1 — Scaffolder: copytree → template rendering  *(Change · upstream-candidate)*

`robyn --create` (`robyn/cli.py`, verified) is an InquirerPy wizard followed by `shutil.copytree` from `robyn/scaffold/<project_type>`. Copytree cannot parameterize: no app-specific crate names, no version stamping, no conditional file emission. Replace with sc-compose rendering via its pip bindings (in-process, no subprocess plumbing); wizard answers become the Jinja2 context; YAML frontmatter drives conditionals. The wizard itself is extended with scaffold questions (app name, surfaces, DB/connections, existing schema crates, OTEL, ACP). A generic templating scaffolder (minus our templates) is a reasonable upstream PR.

## G2 — Generalize command registration  *(Addition + refactor · fork-only)*

Robyn's `FunctionInfo` + router registration and the executor/`TaskLocals` machinery are route-specific. Extract/generalize so a `CommandRegistry` reuses the same Python-callable dispatch path for non-HTTP invocation (CLI, MCP, agent). New Rust module + PyO3 exports (`app.command()` decorator). The executors themselves likely need a non-Actix entry point (invoke a Python handler outside a request context) — inspect `src/executors/` for hidden Actix coupling early in Phase 1.

## G3 — MCP overlap: `mcp.py` vs. rust-mcp-actix  *(Refactor to capability · upstream-candidate — revised, see G12)*

Robyn ships a Python-side MCP server (`robyn/mcp.py`, JSON-RPC 2.0). The scaffold's MCP surface is Rust (rust-mcp-actix) projecting the command registry. Shipping both active violates N4. **Revised disposition**: rather than fork-only removal, both become providers of an `mcp` capability under the manifest mechanism (G12) — `python` (upstream default) or `rust-actix` (scaffold default) or `none`. N4 is satisfied because the manifest selects exactly one. This converts an unupstreamable deletion into an upstreamable mechanism-plus-default.

## G4 — Workspace restructure  *(Change · fork-only)*

Robyn builds as a single Rust crate under Maturin. The scaffold needs a Cargo workspace: the Robyn crate plus scaffold core crates (registry/envelope, schema/versioning, generator support, observability glue, later sync and ACP). Maturin continues building the Python-facing wheel; core crates are also published to crates.io for derivative apps to pin. Watch: keeping the wheel build and workspace publishing from fighting each other in CI.

## G5 — Error path / envelope integration  *(Change · fork-only)*

Robyn's `Response` is status + headers + body with no structured error contract. The web projection (R4) maps DU envelope variants to HTTP status mechanically, which touches the response-construction path in `src/types/` and the executor error handling. Route handlers (upstream-style) keep working unchanged; *registry commands* exposed over HTTP get the envelope treatment. Two response disciplines coexist — document the boundary clearly.

## G6 — Dependency verification & additions  *(Verification + additions · partially upstream-candidate)*

Additions: rust-mcp-sdk + rust-mcp-actix, SeaORM, insta, clap, sc-observability (+ optional OTEL), sc-compose (Python dep for the generator). Verify compatibility against the pinned stack — Actix-web 4.4.2 / actix-http 3.3.1 / Tokio 1.40 / PyO3 0.27.2 / dashmap 5.4.3. Risk to check first: rust-mcp-actix's minimum Actix version; if it forces Actix-web ≥4.5+, that bump is a Phase 0 isolated commit (upstream-candidate, upstream is behind current Actix anyway). SeaORM's SQLx transitively pins its own Tokio compatibility — expected fine on 1.40, verify.

## G7 — `env_populator` extension  *(Change · upstream-candidate)*

Robyn already loads `.env` at CLI startup. Extend to the full resolution order (CLI flag → env → config file → default) and the generated naming conventions (`<APP>_DB_<CONNECTION>_*`). Small, generic, plausibly upstreamable.

## G8 — `ai.py` relationship to ACP agents  *(Refactor to capability · partially upstream-candidate — revised, see G12)*

Robyn ships experimental `ai.py` (OpenAI/Anthropic/Google agent integration). The scaffold's agent story is ACP-hosted agents whose toolset is the registry projection. **Revised disposition**: both become providers of an `agent` capability under the manifest (G12) — `ai-py` (upstream's) or `acp` (scaffold's) or `none`. Revisit at Phase 9 whether `ai.py` internals are salvageable as the model-provider layer under the ACP host.

## G9 — Multi-process worker awareness  *(Constraint · fork-only)*

Robyn's `processpool` spawns worker processes. The connection registry (R5.6), sc-observability pipeline, and any agent session state must be per-worker: pools and loggers initialized post-fork, never inherited. Sync watermarks live in the DB (safe); agent sessions need a placement decision in Phase 9 (single designated worker vs. external process).

## G10 — Scaffold templates directory  *(Addition · fork-only)*

`robyn/scaffold/` currently holds copytree project types (no-db, sqlite, postgres, mongo, sqlalchemy, prisma, sqlmodel). Scaffold templates for `<app>-types`/`<app>-db`/`<app>-api` workspaces replace/extend this tree in sc-compose form. Decide whether upstream's simple project types remain (rendered trivially) for drop-in compatibility — recommended yes, cheap goodwill for tracking upstream.

## G11 — Upstream observability vacuum: issue #1276  *(Opportunity · upstream-candidate)*

Upstream issue sparckles/Robyn#1276 (open, Jan 2026, unanswered) asks Robyn's direction on metrics/tracing/structured logging and surfaces three problems our Phase 7 design answers structurally: (a) the `after_request(response)` hook can't assemble standard HTTP metrics (method + route + status + duration) — moot when instrumenting at the Rust/Actix layer where all four coexist; (b) `const=True` routes bypass Python and thus Python-side instrumentation — invisible to Rust-side instrumentation (same fast-path logic as our interpreter-free CLI commands); (c) multi-process metrics isolation — resolved by per-worker init + per-worker OTEL export (G9), avoiding `prometheus_client` multiprocess mode. Disposition: carve a neutral upstream slice from Phase 7 — Rust-layer HTTP instrumentation on the `tracing` ecosystem without the sc-observability dependency — and engage on #1276 with it. sc-observability remains the fork's opinionated layer on top. Benefit: shrinks long-term fork-maintenance surface and aligns upstream instrumentation points with our design.

## G12 — No capability manifest: subsystems are hardcoded  *(Addition · upstream-candidate)*

Robyn has no mechanism to declare alternative implementations of a subsystem — `mcp.py`, `ai.py`, templating, sessions are hardcoded module imports. Swapping Python MCP for Actix MCP is surgery, not configuration. Add a **capability manifest** (R15): capabilities with real alternatives (`mcp`, `observability`, `agent`) declare selectable providers; cargo features bind Rust providers at compile time, the runtime manifest wires Python providers at startup. Scaffold defaults select Rust providers; upstream-provider selection reproduces stock Robyn. This is the mechanism that makes G3 and G8 upstreamable (mechanism + default rather than deletion) and gives upstream a structural answer to "in-tree vs third-party" questions (#1276 Q1). Keep scope tight: a provider selector, not a plugin framework.

## Summary Table

| # | Area | Kind | Disposition |
|---|------|------|-------------|
| G1 | Scaffolder rendering | Change | Upstream-candidate |
| G2 | Command registration | Addition/refactor | Fork-only |
| G3 | mcp.py → capability provider | Refactor | Upstream-candidate |
| G4 | Workspace restructure | Change | Fork-only |
| G5 | Envelope/error path | Change | Fork-only |
| G6 | Dependencies | Verify/add | Partial upstream |
| G7 | env_populator | Change | Upstream-candidate |
| G8 | ai.py → capability provider | Refactor | Partial upstream |
| G9 | Multi-process | Constraint | Fork-only |
| G10 | Template tree | Addition | Fork-only |
| G11 | Observability upstream slice (#1276) | Opportunity | Upstream-candidate |
| G12 | Capability manifest | Addition | Upstream-candidate |
