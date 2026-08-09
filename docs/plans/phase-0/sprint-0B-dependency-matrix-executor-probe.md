# Sprint 0B — Dependency Matrix & Executor Coupling Probe

**Phase**: 0 · **Closure type**: Investigation/findings · **Risk**: Medium (findings may force Phase C redesign)
**Status**: Not started · **Recommended agent/model**: arch-ctm/deep-reasoning

## Objective

Produce two written findings documents that de-risk Phases A–C sizing: a dependency compatibility matrix and an executor-coupling analysis. No production code changes.

## Deliverables

1. `docs/findings/dependency-matrix.md`: rust-mcp-sdk/rust-mcp-actix, SeaORM 2.0, insta, clap, sc-compose (pip), sc-observability(+OTEL) verified against the pinned stack (Actix-web 4.4.2 / actix-http 3.3.1 / Tokio 1.40 / PyO3 0.27.2 / dashmap 5.4.3). **First check**: rust-mcp-actix minimum Actix version — if it forces a bump, record the exact bump as a proposed isolated, upstreamable commit (do not land it in this sprint).
2. `docs/findings/executor-coupling.md`: analysis of `crates/robyn/src/executors/` for Actix request-context assumptions blocking non-HTTP invocation of Python handlers (G2). Must answer: can a Python handler be invoked with `TaskLocals` but without an `HttpRequest`? Findings feed Phase C sprint sizing.
3. UDS feasibility note (new, from CLI redesign): confirm the pinned Actix-web version binds UDS and TCP on one `HttpServer` builder; record any constraint for Phase C.

## Paths to delete

None.

## Acceptance criteria

- Each matrix row states: version tested, compatible yes/no, constraint, evidence (build log or doc citation) — no row left "assumed".
- Executor findings answer the G2 question with file/line citations, classified as: reusable as-is / needs new entry point / needs refactor.
- UDS note cites the Actix API used.

## Required validation

- Throwaway build experiments live in a scratch branch, never merged; findings docs are the only merged artifact.
