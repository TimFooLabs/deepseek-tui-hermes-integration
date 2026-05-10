# ADR-001: Why Use the HTTP Runtime API Instead of the CLI

## Status

Accepted

## Context

DeepSeek TUI provides two interfaces:
1. **Interactive TUI** — terminal UI via `deepseek` (requires a PTY, interactive only)
2. **Headless HTTP API** — `deepseek serve --http` exposing a REST-like API

Hermes Agent needs to delegate tasks programmatically. It cannot interact with a terminal UI.

## Decision

Use the HTTP runtime API (`deepseek serve --http`) as the integration point.

## Rationale

- The TUI requires a PTY and interactive input. Hermes has no way to drive a terminal UI programmatically.
- The HTTP API is explicitly designed for programmatic access — thread-based model with turn submission and event streaming.
- The API is documented in `docs/RUNTIME_API.md` (366 lines) with a stable contract.
- This mirrors how Hermes integrates with Claude Code (CLI subprocess) but adapted for DeepSeek TUI's architecture.

## Consequences

- Requires a persistent server process (solved with systemd user service)
- Adds a network hop (localhost only, negligible latency)
- Must manage thread lifecycle (create, poll, extract results)

## Alternatives Considered

- **Direct `deepseek --message` one-shot mode**: Doesn't exist in this version. The `deepseek` CLI dispatches to the TUI runtime; there's no built-in headless mode without the HTTP server.
- **PTY automation (expect/script)**: Fragile, error-prone, and unnecessary when a proper HTTP API exists.
