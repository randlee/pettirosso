# Sprint A2 — Runtime Binding & Boot-Path Refactor

**Phase**: A · **Closure type**: Runtime refactor · **Risk**: High (Robyn's most-trafficked startup path)
**Status**: Not started · **Recommended agent/model**: arch-ctm/deep-reasoning

## Objective

Robyn's boot path consults the manifest instead of unconditionally importing `mcp.py` / `ai.py` / logging setup. Two binding layers: cargo features gate Rust providers at compile time; the runtime manifest wires Python providers at startup. `crates/robyn` edits stay minimal-diff (vendored rule) — wiring lives in a thin hook, not scattered edits.

## Deliverables

1. Manifest load at server boot: locate `scaffold.toml` (path resolution defers to A4; this sprint uses cwd + explicit path arg only), parse via `pettirosso-manifest`, fail fast with `ManifestError` on invalid content; **absent file = all-upstream defaults** (upstream-compat is the zero-config behavior).
2. Python-side gating: `robyn/__init__` imports `mcp.py` / `ai.py` / logging setup only when the manifest selects them. Exactly one MCP provider can be active (N4) — enforced at load, not by convention.
3. Cargo feature stubs for Rust providers (`provider-mcp-rust-actix`, `provider-obs-sc`, `provider-agent-acp`) gating empty modules — real content arrives in Phases B/C/F.
4. Boot-hook code sample in this doc before build (guidelines: boundary contracts need signatures) — added at review time from 0B findings; imports must reference `pettirosso_manifest`, not placeholder names (archived A2's samples referenced `scaffold-core` and would not compile).

## Paths to delete

None (upstream `mcp.py`/`ai.py` remain — they are the `python`/`ai-py` providers).

## Non-closure

- No provider does anything new yet; `basic` observability etc. behave exactly as stock Robyn. This sprint closes *selection*, not provider behavior.

## Acceptance criteria

- All-upstream manifest (or no manifest file): upstream test suite passes unchanged.
- `mcp: none`: server boots, `mcp.py` never imported (assert via import-hook test).
- Invalid manifest: process exits non-zero with the `ManifestError` message, before binding any port.
- Diff gate: `crates/robyn/` changes confined to the boot hook + feature declarations; reviewer checks upstream-PR viability of the diff.

## Required validation

- Upstream test suite in CI under three manifests: absent, all-upstream, `mcp: none`.
- Multi-worker check: manifest read once pre-fork, selections identical in every worker (log worker-id + selection at startup, compare).
