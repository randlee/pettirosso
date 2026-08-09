# Sprint 0A — Workspace, CI & Name Reservation

**Phase**: 0 · **Closure type**: Build/structural · **Risk**: Low-medium (wheel/workspace CI friction)
**Status**: Not started · **Recommended agent/model**: Cipher-311d/fast

## Objective

Restructure the fork into a Cargo workspace with the vendored Robyn crate and empty `pettirosso-*` member slots, keep the Maturin wheel build byte-equivalent in behavior, and reserve the `pettirosso` name on both registries.

## Deliverables

1. Workspace layout: `crates/robyn/` (vendored, minimal-diff — moved, not edited) + workspace `Cargo.toml`. No `pettirosso-*` crate content yet; members added by later phases' sprints.
2. Maturin wheel builds from the workspace; wheel installs and serves an upstream example app.
3. CI: upstream Rust + Python test suites green against the workspace layout; workspace `cargo publish --dry-run` job to surface wheel/workspace conflicts early (G4 risk).
4. `pettirosso` reserved on PyPI and crates.io (placeholder 0.0.1). **Non-closure**: the crates.io name for the runtime core itself (O-4) is *not* decided here; only the umbrella name is reserved.
5. Upstream remote configured; documented `git fetch upstream && git merge` procedure with the minimal-diff rule stated in `crates/robyn/README.vendored.md`.

## Paths to delete

- `docs` symlink to upstream marketing site (already replaced by real `docs/` in ced7caa — verify no re-introduction on upstream merge).

## Acceptance criteria

- `cargo build --workspace` and `maturin build` succeed from clean checkout.
- Upstream test suites pass unchanged (same tests, same results as tag v0.88.0).
- `pip install` of the built wheel runs an upstream quickstart app.
- Both registry reservations visible publicly.
- Diff of `crates/robyn/` contents vs upstream v0.88.0 is path-move only (validated by a CI grep/diff gate comparing file hashes).

## Required validation

- CI matrix run (Linux + macOS) on the workspace.
- Hash-comparison script output attached to PR (vendored-diff gate).
