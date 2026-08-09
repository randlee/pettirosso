# Phase F — ACP Agent Hosting (outline)

**Component added**: `agent: acp` provider in `pettirosso-acp` — app-hosted, remotely controlled agent whose toolset is the registry projection.
**Detail level**: outline only. Deliberately last optional component; ACP spec movement is why transport topology is decided *in-phase* (ADR against then-current Zed ACP spec: stdio subprocess vs WebSocket vs both — note the server's TCP transport from Phase C already serves the remote-control story).

- Agent session management: lifecycle, per-worker placement ADR (designated worker vs external process — G9), remote-control auth via A4 resolution order.
- Toolset projection from registry (surface-filtered, same schema records as CLI/MCP).
- `ai.py` wrapped as the alternative `agent` provider; assess its internals as the model-provider layer under the ACP host.
- Optional web chat fronting the same agent session (wizard opt-in).
- Corrective-action demo: agent receives `Validation{expected,got}` and self-corrects (R9.3 payoff).

**Exit**: remote ACP client drives a hosted agent through registry commands including the self-correction demo; manifest switches `acp`/`ai-py`/`none` cleanly.
**Depends on**: A, C, D-2 (agents operate under approval gates).
