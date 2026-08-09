# Claude Instructions for pettirosso

## ⚠️ CRITICAL: Process Rules (absolute, non-negotiable)

1. **Review-before-build gate.** No code is written, committed, or independently
   acted upon until the sprint doc for that sprint has passed Rand's review.
   Gaps in instructions are never filled with independent judgment about
   efficiency — ask instead.
2. **The sprint plan is authoritative.** Scope lives in `docs/plans/`; prompts
   and summaries may project it but never replace or narrow it.
3. **Anti-40-sprint rule.** Only the next executable phase carries full sprint
   docs. Later phases stay at planning depth until predecessor findings land.
   Expanding a future phase early is a process violation, not diligence.
4. **Never silently resolve an open item.** O-1…O-5 are tracked in
   `docs/plans/README.md`; each is closed by an explicit decision or ADR, never
   by code that happens to pick a side.
5. Sprint docs are hardened against
   `docs/plans/` guidelines inherited from
   `atm-core/.claude/skills/plan-hardening/sprint-planning-guidelines.md`
   (to be copied repo-local when `.claude/skills` is installed).

## What this project is (and is not)

**Pettirosso is a scaffolding tool, not a product suite.** It is a fork of the
Robyn web framework (Rust: Actix-web / Tokio / PyO3 / Maturin) that generates
starter application repositories. Pettirosso itself is pure infrastructure and
wiring — **no business logic ever**. Scaffolding forces business logic outside
infrastructure; keeping that boundary clean is what makes development fast.

The prior planning failure mode to avoid: treating the generator, the fork,
and the generated apps as three separate products to build to completion.

## The three-tier app gradient (organizing concept)

- **Tier 1** — Python-only rapid prototype. Compiled Rust foundation (server,
  MCP, DB) is invisible; developer works entirely in Python.
- **Tier 2** — Developer uses `cargo` after scaffolding to harden the schema
  and CLI contract in Rust.
- **Tier 3** — Business logic / critical image-processing / math migrated to
  Rust for performance.

Every phase must leave the gradient cheaper to traverse. Python command
registration is a permanent plugin mechanism, not merely an incubation stage.

## Repository layout & rules

- `crates/robyn/` (after Sprint 0A): **vendored, minimal-diff** from upstream
  sparckles/Robyn v0.88.0. Never edit it directly; new logic goes in sibling
  `pettirosso-*` crates. Upstream tracking is plain
  `git fetch upstream && git merge`.
- New crates: flat `crates/` directory, `pettirosso-*` naming, many small
  crates (atm-core granularity), dedicated `pettirosso-error` crate.
- `docs/` — three authoritative design documents (requirements, gap report,
  migration plan) **plus** `docs/plans/extraction-2026-08-08.md`, which
  records decisions that supersede parts of them (CLI redesign, `<app>-schema`
  rename, tier gradient). Read the extraction report first.
- `docs/plans/README.md` — master plan index and the entry point for all work.
- `archive/sonnet-2026-08-08` branch — unreviewed prior work. Reference only;
  never merge or cherry-pick from it without explicit instruction.

## Branch & PR rules

- Work lands on `develop`; PRs target `main`; Rand reviews every PR.
- **`gh` quirk**: this repo is a GitHub fork of sparckles/Robyn until the fork
  relationship is detached (Sprint 0A deliverable 6). Until then, ALWAYS
  owner-qualify: `gh pr create -R randlee/pettirosso ...` or API with
  `head=randlee:<branch>` — otherwise `gh` targets sparckles/Robyn.
- Never push anything to sparckles/Robyn. Upstream PRs happen only from the
  upstream queue (`docs/plans/README.md`) when Rand says so.

## Architecture decisions (locked — do not re-litigate)

- **One operation layer**: CLI / MCP / web are protocol adapters over one
  command registry; never three implementations that agree.
- **CLI**: native Rust binary, HTTP client to the running server over UDS
  (local default) + TCP (remote); Maturin bindings for in-process Python use.
  The wizard is a separate one-shot Python/InquirerPy/sc-compose product.
- **DB**: SeaORM 2.0, no custom Storage trait; `<app>-schema` exposes typed
  async query functions as the *only* sanctioned DB access; **one write path**.
- **Optional components**: capability manifest (`scaffold.toml`) selects
  providers; all-upstream manifest reproduces stock Robyn.
- Apply the proven `creating-ai-clis` skill for the CLI/MCP contract layer
  rather than redesigning it.

## Terminology (use exactly)

- **DU envelope** — discriminated-union `CommandResult<T>`. NEVER abbreviate
  "DI" (collides with Robyn's `dependency_injection.py`).
- **Schema record** — serializable data description of a command. Schema is
  data, not code.
- **Surface** — a projection of the registry: CLI, MCP, web, ACP.
- **SchemaModule** — trait contract a DB schema crate exports.
- **Model-first** — Rand's term for SeaORM's "entity-first" workflow.
- **Capability manifest** — provider-selection config.

## Environment notes

- Repo path contains a space (`/Volumes/Extreme Pro/github/pettirosso`) —
  quote it in every shell command.
- External SSD: confirm the volume is mounted before assuming the repo is
  missing.
- Persistent cross-session state: capture significant decisions/progress to
  open-brain (`capture_thought`) at session midpoints and ends.

## Agents & skills: not yet installed

Rust agents and skills are installed via Rand's existing installation
process — do NOT copy them from atm-core or hand-author them. Rand runs the
installation when ready. Until `.claude/agents/` and `.claude/skills/` exist
here: single-session Claude Code, working on `develop` feature branches,
following the process rules above. The sprint-planning guidelines remain
readable at their atm-core path (see rule 5) in the meantime.
