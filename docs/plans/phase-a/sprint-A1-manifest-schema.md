# Sprint A1 — Manifest Schema, pettirosso-error & Provider Trait

**Phase**: A · **Closure type**: Data/type modeling · **Risk**: Low
**Status**: Spec proven on archive/sonnet-2026-08-08 (built + tested 2026-08-07); NOT reviewed; rebuild or cherry-pick only after review.
**Recommended agent/model**: Cipher-311d/fast

## Objective

Define the capability manifest as data and the provider lifecycle as a trait in `pettirosso-manifest`, with errors in a new `pettirosso-error` crate (resolves extraction §7 item 5 in the affirmative — matches atm-core's dedicated-error-crate pattern). Zero runtime wiring into Robyn; types only.

## Deliverables

1. `pettirosso-error` crate: `ManifestError` (+ shape decided in review for how later crates' errors join — sample below is the proposal).
2. `pettirosso-manifest`: `McpProvider { Python | RustActix | None }`, `ObservabilityProvider { Basic | ScObservability }`, `AgentProvider { AiPy | Acp | None }` (serde, kebab-case), aggregated in `Manifest` with `Default` = scaffold Rust-provider defaults. **No fourth capability.**
3. `scaffold.toml` round-trip (de)serialization with unknown-key rejection.
4. `ProviderLifecycle` trait: `configure`/`start`/`shutdown` signatures only, no implementations.

## Code samples

```rust
// pettirosso-error/src/lib.rs
#[derive(Debug, thiserror::Error)]
pub enum ManifestError {
    #[error("unknown provider `{got}` for capability `{capability}`")]
    UnknownProvider { capability: String, got: String },
    #[error("manifest parse error: {0}")]
    Parse(#[from] toml::de::Error),
}

// pettirosso-manifest/src/lib.rs
pub trait ProviderLifecycle {
    fn configure(&mut self, manifest: &Manifest) -> Result<(), pettirosso_error::ManifestError>;
    fn start(&mut self) -> Result<(), pettirosso_error::ManifestError>;
    fn shutdown(&mut self);
}
```

Enum/`Manifest` shapes: reproduce the archived A1 doc verbatim (proven by its test run) modulo the `pettirosso-error` import.

## Paths to delete

None.

## Acceptance criteria

- Round-trip property test: `Manifest` → toml → `Manifest` identity; unknown provider string yields `UnknownProvider`, never panic or silent default.
- `Manifest::default()` = `mcp: rust-actix, observability: sc-observability, agent: acp`.
- No dependency from `pettirosso-manifest` on `crates/robyn` (grep gate: `robyn` absent from its Cargo.toml).

## Required validation

- `cargo test -p pettirosso-manifest -p pettirosso-error`
- `cargo semver-checks` baseline recorded for both crates.
