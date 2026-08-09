# Phase G — Reference App Acceptance: ATM Sync (outline)

**Purpose**: end-to-end dogfood proving the whole scaffold on a real product-shaped app, exercising the full gradient.
**Detail level**: outline only.

- Generate via wizard: composes published `atm-schema`, SQLite `local` + Supabase `central`, sc-observability + OTEL, rust-actix MCP, ACP agent opted in.
- Sync ATM messages SQLite→central; idempotence + fan-in from two machines.
- **Gradient walk (the acceptance headline)**: Tier 1 — incubate one new command in Python; Tier 2 — graduate it to Rust with unchanged snapshots (schema, envelope, log shape — N5); Tier 3 — migrate one hot path (e.g. an image-processing or math routine) to Rust and measure the win.
- Agent-driven operation: ACP agent runs the sync and handles a partial-failure envelope.
- Friction write-up feeding the backlog; upstream PR queue review (G12 → G1 → G11/#1276 → G7).

**Exit**: all of the above pass without scaffold-side patches; friction list produced.
**Depends on**: all phases.
