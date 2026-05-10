# ADR-003: Why a Shell Wrapper Instead of a Hermes Tool Plugin

## Context

Hermes Agent supports two ways to add capabilities:
1. **Tool plugins** — registered in `config.yaml`, called directly by the LLM
2. **Shell commands** — invoked via the existing `terminal` tool

## Decision

Implement the integration as a shell wrapper script (`deepseek-agent`) invoked through Hermes's existing `terminal` tool. No custom tool plugin.

## Rationale

- **Simplicity**: A shell script is easier to write, debug, and modify than a Hermes tool plugin.
- **No config changes**: Adding a tool plugin requires modifying `config.yaml` and restarting the gateway. A shell script works immediately.
- **Hermes pattern**: Hermes already uses this pattern for other integrations (e.g., Claude Code is invoked via `terminal` tool, not as a native plugin).
- **Portability**: The wrapper works outside Hermes too — any script or agent can call it.
- **Iteration speed**: Patching a shell script is faster than rebuilding/reconfiguring a plugin.

The Hermes skill (`SKILL.md`) documents how to use the wrapper. The LLM reads the skill and knows to call `deepseek-agent` via the `terminal` tool.

## Consequences

- Slightly higher latency (shell process spawn + HTTP overhead)
- No type-safe tool schema (the LLM must construct the command line from skill instructions)
- Works reliably in practice — the skill provides exact invocation templates

## Alternatives Considered

- **Native Hermes tool plugin**: Would provide better UX (structured inputs, type checking) but requires more complex setup and gateway restart. Can be added later if the shell wrapper proves insufficient.
- **Python script instead of bash**: Would be more maintainable for complex logic, but bash + curl + jq is sufficient and has fewer dependencies.
