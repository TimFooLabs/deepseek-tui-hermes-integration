# DeepSeek TUI + Hermes Agent Integration

Integration of [DeepSeek TUI](https://github.com/Hmbown/DeepSeek-TUI) as an autonomous coding agent within the [Hermes Agent](https://hermes-agent.nousresearch.com) ecosystem. This repo documents the full integration: architecture, setup, challenges encountered, and the rationale behind every design decision.

## What It Does

Hermes Agent delegates coding tasks to DeepSeek TUI via a lightweight shell wrapper (`deepseek-agent`) that talks to DeepSeek TUI's runtime HTTP API. This gives Hermes access to DeepSeek V4 Pro's full agent capabilities — file editing, shell execution, git operations, sub-agent orchestration, and streaming reasoning — all running locally.

## Architecture

```
Hermes Agent
  │
  │  terminal tool
  ▼
deepseek-agent (shell wrapper)
  │
  │  HTTP (127.0.0.1:7878)
  ▼
deepseek-tui serve --http (systemd user service)
  │
  │  OpenAI-compatible streaming
  ▼
DeepSeek API (V4 Pro)
```

### Component Map

| Component | Location | Purpose |
|-----------|----------|---------|
| `deepseek-agent` wrapper | `~/.local/bin/deepseek-agent` | Hermes-facing CLI; handles thread/turn lifecycle |
| Runtime API server | `~/.config/systemd/user/deepseek-tui.service` | Persistent HTTP server for headless agent access |
| Hermes skill | `~/.hermes/skills/autonomous-ai-agents/deepseek-tui/SKILL.md` | Instructions for Hermes to use the integration |
| DeepSeek config | `~/.deepseek/config.toml` | API key, model settings, LSP config |

## Quick Start

```bash
# 1. Start the runtime API server
systemctl --user enable --now deepseek-tui

# 2. Verify it's running
curl http://127.0.0.1:7878/health

# 3. Delegate a task from Hermes
deepseek-agent "Add input validation to the /api/login endpoint" \
  --workspace /path/to/project \
  --model deepseek-v4-pro \
  --max-turns 15 \
  --timeout 600
```

## Documentation

- **[Setup Guide](docs/setup.md)** — Full installation and configuration
- **[Architecture Decisions](docs/decisions/)** — ADR-style docs for every significant choice
- **[Troubleshooting](docs/troubleshooting.md)** — Known issues and solutions
- **[API Reference](docs/api-reference.md)** — Runtime API contract (verified)

## Key Files in This Repo

```
├── README.md
├── docs/
│   ├── setup.md
│   ├── troubleshooting.md
│   ├── api-reference.md
│   └── decisions/
│       ├── 001-why-http-api.md
│       ├── 002-why-thread-polling.md
│       ├── 003-why-shell-wrapper.md
│       ├── 004-why-systemd-service.md
│       ├── 005-why-port-7878.md
│       └── 006-pitfalls-curl-sse.md
├── scripts/
│   └── deepseek-agent          # The wrapper script
└── configs/
    ├── deepseek-tui.service    # systemd unit file
    └── config.toml.example     # DeepSeek config template
```
