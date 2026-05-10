# ADR-004: Why a systemd User Service for the Runtime API

## Context

The DeepSeek TUI runtime API server (`deepseek-tui serve --http`) is a long-running process. It needs to:
- Start automatically (no manual `deepseek-tui serve` after every reboot)
- Restart on failure
- Run without a login session (WSL doesn't always have a logged-in user)

## Decision

Run the runtime API as a systemd user service with linger enabled.

## Rationale

- **systemd user service**: Standard Linux service management. Handles startup, restart, logging.
- **Linger**: `loginctl enable-linger $USER` allows user services to start at boot without a login session. Critical for WSL where there's no traditional login.
- **Restart on failure**: If the process crashes, systemd restarts it automatically (5s delay).
- **Logging**: `journalctl --user -u deepseek-tui -f` provides structured logs.

## Service Configuration

```ini
[Service]
Type=simple
ExecStart=%h/.cargo/bin/deepseek-tui serve --http --host 127.0.0.1 --port 7878 --workers 2 --auth-token deepseek-local-7878
Restart=on-failure
RestartSec=5
Environment=PATH=%h/.cargo/bin:/usr/bin:/bin
Environment=RUST_LOG=warn
```

- **Workers: 2** — allows concurrent turn processing
- **RUST_LOG=warn** — reduces noise; errors still logged
- **Bind: 127.0.0.1** — localhost only, no external exposure

## Consequences

- Requires systemd (standard on Ubuntu 24.04, available in WSL2 with Ubuntu)
- Requires linger to be enabled (one-time setup)
- Service survives WSL restarts

## Alternatives Considered

- **Background process (`nohup` / `&`)**: No auto-restart, no logging, fragile.
- **Docker container**: Overhead for a single binary. systemd is simpler.
- **Supervisor/supervisord**: Extra dependency when systemd is already available.
