# Phase B — Observability Provider (planning depth)

**Component added**: `observability: sc-observability` as the first real provider through the manifest; Rust-layer HTTP instrumentation.
**Detail level**: planning depth — expanded into sprint docs when Phase A review lands. Parallel-safe with early Phase C-1 once A merges.

## Scope (from migration plan Phase B, unchanged by the CLI redesign)

- B1: `basic` provider = current Robyn logging wrapped as a `ProviderLifecycle` impl (proves the trait on trivial ground).
- B2: `pettirosso-observability` crate wiring sc-observability; **per-worker init post-fork, never inherited** (G9); worker-id field on every record as the silent-init-bug tripwire.
- B3: Rust-layer HTTP instrumentation (method/route/status/duration at the Actix layer; covers `const=True` routes) on the `tracing` ecosystem.
- B4: OTEL as cargo feature + wizard question; per-worker OTEL export.
- B5: PyO3 stdlib-logging bridge (interim per randlee/sc-observability#88), shape-identical fields; removal path documented against #88.
- B6: log-shape invariance harness — same logical event from Rust and Python compares equal in shape (becomes part of the Tier-1→2 graduation contract, N5).
- B7: upstream slice (G11): neutral `tracing` instrumentation branch for sparckles/Robyn#1276, without the sc-observability dependency.

## Exit criteria

Manifest switch `basic` ↔ `sc-observability` with no code edits; shape-identical Rust/Python records; OTEL export verified when feature on; #1276 branch ready (engagement timing Rand's call).

## Known risks / open inputs

- sc-observability Python bindings timeline (external, #88) — bridge fully mitigates.
- Instrumentation must attach in `crates/robyn` middleware without violating the minimal-diff vendored rule — expect one hook, like A2's.
