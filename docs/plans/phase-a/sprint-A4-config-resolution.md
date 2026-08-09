# Sprint A4 — Config Resolution Order

**Phase**: A · **Closure type**: Config plumbing · **Risk**: Low
**Status**: Not started · **Recommended agent/model**: Cipher-311d/fast

## Objective

Implement the full resolution order — CLI flag → environment → config file → default (R13.1) — for manifest values and manifest-file location, reusing Robyn's existing `env_populator` (.env already loads at CLI startup). Small and generic by design: this slice is a G7 upstream candidate.

## Deliverables

1. Resolution function with the four-level order; manifest path resolvable via `--scaffold-manifest` flag and `PETTIROSSO_MANIFEST` env var; individual capability overrides via `PETTIROSSO_CAPABILITY_<CAP>` env vars.
2. A2's loader switched from cwd-only to this resolution (one-line seam swap — A2 deliverable 1 anticipated it).
3. Resolution-order table documented in the fork docs; precedence covered by a matrix test (each level overriding the next).
4. Naming groundwork only for DB (`<APP>_DB_<CONNECTION>_*` vs `_SCHEMA_`, open item O-5): record the convention as *proposed* in docs; actual DB env wiring is Phase D scope, not closed here (explicit non-closure).

## Paths to delete

None.

## Acceptance criteria

- Matrix test: 4-level precedence proven for at least manifest path + one capability override (16-case table acceptable to prune to the meaningful 8).
- `env_populator` behavior for stock Robyn apps unchanged (upstream tests green).
- No secret values ever logged by resolution debug output (grep gate on log fixtures).

## Required validation

- CI runs the matrix test plus upstream suite.
