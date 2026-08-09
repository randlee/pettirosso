# Sprint A3 — Wizard Manifest Write & `capability set`

**Phase**: A · **Closure type**: CLI/UX · **Risk**: Low
**Status**: Not started · **Recommended agent/model**: Cipher-311d/fast

## Objective

New apps get a manifest from the wizard; existing apps change providers with `capability set`. Scope note from the CLI redesign: the **wizard is a separate one-shot Python product** (InquirerPy, console-script) and is *not* subject to the CLI-as-HTTP-client architecture; `capability set` here is hand-wired Python and is **migrated onto the command registry in Phase C** (explicit carry-forward, tracked in Phase C plan, not silent).

## Deliverables

1. Wizard capability questions (observability / mcp / agent) appended to `robyn --create`; answers written to `scaffold.toml` via `pettirosso-manifest` serialization (through the PyO3 boundary or a minimal Python mirror — decide in review; mirror risks drift, PyO3 requires A2's bindings — flag as the sprint's one open design point).
2. `capability set <capability> <provider>`: validates against the enum sets, rewrites `scaffold.toml`, prints old → new. Refuses unknown capability/provider with the same wording as `ManifestError`.
3. Wizard defaults = scaffold defaults (Rust providers); an explicit `--upstream` preset writes the all-upstream manifest.

## Paths to delete

None.

## Acceptance criteria

- Fresh wizard run produces a `scaffold.toml` that A2's loader accepts (round-trip integration test wizard → boot).
- `capability set mcp none` on a generated app then boot: `mcp.py` not imported.
- Invalid input paths covered by tests (unknown capability, unknown provider, unwritable file).

## Required validation

- End-to-end pytest: wizard (non-interactive answers mode) → generated dir → boot under the three A2 CI manifest configurations.
