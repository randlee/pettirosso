# Phase C — Operation Layer: Registry, Envelope, Native CLI, Rust MCP (planning depth)

**Component added**: the keystone — one operation layer projected as CLI/MCP/web adapters. Incorporates the **CLI redesign** (extraction §3), which supersedes migration-plan C-2 and requirements R2.
**Detail level**: planning depth. Largest phase; expand to sprint docs only after 0B findings + Phase A review. Apply the `creating-ai-clis` skill for the CLI/MCP contract layer rather than redesigning it.

## Internal stages

**C-1 Foundation** (chokepoint)
- Schema record types + `CommandRegistry` in `pettirosso-registry`: schema records and implementation bindings registered separately (graduation = one-line binding change, R1.1); surface flags with per-surface modifiers, default all-surfaces.
- `CommandResult<T>` DU envelope + error taxonomy in `pettirosso-error` (six starting variants, extend via minor bumps; edge-contract rule R9.2: internals use `Result<T, DbError>`, single translation point per command).
- PyO3 exposure: envelope as typed Python classes; `app.command()` decorator through generalized executor path (shape depends on 0B executor findings).
- Python command registration is the permanent **plugin mechanism** (gradient §1), not just incubation — API named and documented accordingly.

**C-2 Server transports + native CLI** (supersedes old C-2)
- Server binds **UDS + TCP** on the same Actix HttpServer (0B feasibility note is the input).
- `pettirosso-cli`: native Rust bin, HTTP client, no PyO3 in its execution path; `--json` emits the DU envelope; stable exit-code mapping per variant. Maturin bindings on the CLI for in-process Python consumption. No `robyn.cli:run` console-script for app commands (wizard keeps its own entry point).
- Decide O-2 (`pettirosso-protocol` wire-types crate) and O-3 (`pettirosso-cli-core`) at C-2 sprint-planning time — both are one-page ADRs, not silent choices.
- Migrate A3's hand-wired `capability set` onto the registry (declared carry-forward closes here).

**C-3 MCP provider**
- `rust-actix` provider (rust-mcp-sdk + rust-mcp-actix) mounted via the manifest; `tools/list` = registry projection filtered by surface flags; results carry the envelope (`isError` + tagged JSON). `python` provider stays selectable (G3 revised); N4 enforced by A2.
- Web projection: envelope→HTTP-status mapping; upstream routes vs registry commands = two documented response disciplines (G5).

**C-4 Contract parity**
- Same command over CLI `--json`, MCP, and web returns **byte-identical envelopes** (the one-operation-layer principle made testable).

## Exit criteria

DB-less app with one Rust + one Python command working over all three adapters with parity-identical envelopes; native CLI round-trip over UDS with zero Python in the CLI process; graduation demo (swap a Python binding to Rust, schema record unchanged).

## Open inputs

0B executor findings (G2 — least-known unknown); UDS feasibility note; O-2/O-3/O-4 ADRs.
