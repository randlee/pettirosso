# Phase A — Capability Manifest & Install Options: Sprint Plan Index

**Status**: Draft v2 · **Date**: 2026-08-08 · **Hardened against**: sprint-planning-guidelines.md (atm-core)
**Component added**: the optional-components mechanism (R15/G12). Re-derivation of the archived Phase-A plan with real crate names, the `pettirosso-error` decision surfaced, and CLI-redesign fallout applied.
**Prior art**: Sprint A1 was built and tested to spec on `archive/sonnet-2026-08-08` — the spec is proven; the code is rebuilt or cherry-picked only after this plan passes review.

## Split rationale (unchanged from v1, still valid)

Four sprints, four closure types touching non-overlapping boundaries: data/type modeling (A1), a high-risk refactor of Robyn's most-trafficked startup path (A2), wizard/UX + incremental-change command (A3), config-resolution plumbing (A4). A single "Phase A done" claim would hide slips in A3/A4 behind A1/A2 landing.

| Sprint | Title | Status | Closure type |
|---|---|---|---|
| A1 | Manifest schema, `pettirosso-error`, provider trait | Spec proven (archive) | Data/type modeling |
| A2 | Runtime binding & boot-path refactor | Not started | Runtime refactor (high risk) |
| A3 | Wizard manifest write + `capability set` | Not started | CLI/UX |
| A4 | Config resolution order | Not started | Config plumbing |

## Dependency relations

| Sprint | Relation | Depends on | Rationale |
|---|---|---|---|
| A1 | must_follow | 0A | Needs workspace member slots. |
| A2 | must_follow | A1 | Consumes manifest types; boot path reads `Manifest`. |
| A3 | must_follow | A1; parallel_safe with A2 | Wizard code vs boot path — no shared files, contracts, or ownership. |
| A4 | must_follow | A2 | Resolution order overrides manifest values at startup. |

Merge-forward trigger for all must_follow edges: parent development pushed (not QA) → merge parent into child before every dev/fix round.

## Phase exit criteria

- `mcp: python` ↔ `mcp: none` switch end-to-end with zero code edits.
- All-upstream manifest passes the upstream test suite unchanged (upstream-compat mode).
- Wizard writes a manifest; `capability set` edits one on an existing app.
- Boot-path diff stays surgical (G12 is upstream-candidate — reviewer checks diff size/shape).
