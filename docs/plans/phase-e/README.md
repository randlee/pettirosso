# Phase E — Generator & Wizard (planning depth) — usable-scaffold milestone

**Component added**: the generator itself. Imports the archived Phase-G structure (`archive/sonnet-2026-08-08:docs/plans/phase-G/`) — its split rationale and dependency graph were sound; content is re-based on Phases A–D reality at expansion time.
**Detail level**: planning depth. The scaffold is **usable** when this phase exits.

## Fixed scope decisions (extraction §4)

- Wizard is a one-shot Python/InquirerPy/sc-compose product with its own console-script; categorically outside the CLI-as-HTTP-client architecture. No upgrade mode — `capability set` (Phase C registry command) covers incremental change.
- sc-compose renders **owned crate source** into the new repo (Tier 2), not dependency references; Tier 1 output contains no Rust.
- Tier-2 topology: `my-app-types`, `my-app-schema`, `my-app-api`, `my-app-cli`, `python/` plugin dir.
- Tier-1 output must still be a complete working app (pip-installable experience, compiled foundation invisible).

## Sprint skeleton (from archived G0–G6, renumbered E0–E6)

| Sprint | Title | Notes |
|---|---|---|
| E0 | Review of archived render→build spike (`dev-render.py`, `templates/minimal-app/`) | Unreviewed spike code documented retroactively on the archive branch; review before any reuse — reuse is not assumed. |
| E1 | Wizard package location & entry point | Informed by E0 review outcome. |
| E2 | sc-compose integration module | |
| E3 | Four-crate template topology (Tier 2) | Templates render against real Phase C/D crates, incl. scaffold-version stamping (R8.5). |
| E4 | DB generation modes (schema-first / model-first / none) | Blocked on O-1 resolution (D-2 ADR). |
| E5 | Tier 1 output path | Parallel-safe with E3/E4 after E2. |
| E6 | Interactive wizard UX | Wraps a working non-interactive core. |

## Exit criteria (usable scaffold)

Wizard emits (a) a Tier-1 app that installs via pip and runs Python commands over CLI/MCP/web, and (b) a Tier-2 app whose four crates build, pass generated CI (wheel where applicable, snapshot suite, semver-checks), and demonstrate the Tier-1→2 graduation step with unchanged snapshots (N5).

## Open inputs

O-1 (blocks E4) · O-3 (`pettirosso-cli-core`, decided in Phase C, consumed by E3's `my-app-cli` template) · Tier-blind R14 rewrite (extraction §6) lands in the requirements doc before E3 sprint docs.
