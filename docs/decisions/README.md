# Architecture Decision Records

This directory documents the challenges encountered and the rationale behind each design decision made during the DeepSeek TUI + Hermes Agent integration.

## Decisions

| # | Decision | Key Challenge |
|---|----------|---------------|
| [001](001-why-http-api.md) | Use HTTP runtime API | TUI is interactive-only; Hermes needs programmatic access |
| [002](002-why-thread-polling.md) | Poll thread endpoint instead of SSE | SSE stream never closes; curl buffers SSE output |
| [003](003-why-shell-wrapper.md) | Shell wrapper via `terminal` tool | Avoids complex plugin development; follows existing Hermes patterns |
| [004](004-why-systemd-service.md) | systemd user service for persistence | WSL needs linger for boot-time startup; auto-restart on crash |
| [005](005-why-port-7878.md) | Use default port 7878 | Plan guessed 7777; docs say 7878 |
| [006](006-pitfalls-curl-sse.md) | curl + SSE pitfalls | Multiple debugging issues: buffering, wrong JSON schema, wrong body format |

## Overarching Lessons

1. **Read the actual docs first.** Port numbers, API schemas, and capabilities should be verified against documentation, not assumed.
2. **Test the full pipeline early.** The SSE vs. polling issue wasn't discovered until the wrapper was tested end-to-end.
3. **Prefer simple, reliable mechanisms.** Thread polling is less elegant than SSE but far more reliable in shell scripts.
4. **Match the tool to the integration model.** Hermes's `terminal` tool is the natural integration point — no need for custom plugins.
