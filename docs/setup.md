# Setup Guide

## Prerequisites

- **Rust toolchain** — DeepSeek TUI is written in Rust; you'll build from source
- **DeepSeek API key** — from https://platform.deepseek.com/api_keys
- **Hermes Agent** — already installed and running
- **WSL2 Ubuntu 24.04** — the target environment (glibc 2.35)

## Step 1: Build DeepSeek TUI from Source

Prebuilt binaries require glibc 2.39+, but Ubuntu 24.04 ships glibc 2.35. Building from source is required.

```bash
# Install Rust if not present
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"

# Clone and build
git clone https://github.com/Hmbown/DeepSeek-TUI.git
cd DeepSeek-TUI
cargo build --release

# Install binaries
cp target/release/deepseek ~/.cargo/bin/
cp target/release/deepseek-tui ~/.cargo/bin/

# Verify
deepseek --version    # → deepseek 0.8.26
deepseek-tui --version
```

> **Why source build?** Prebuilt binaries are linked against glibc 2.39+. Ubuntu 24.04 has glibc 2.35. Attempting to run prebuilt binaries produces a cryptic "version `GLIBC_2.39' not found" error. Building from source against the host's glibc is the fix.

## Step 2: Configure DeepSeek

```bash
mkdir -p ~/.deepseek
cat > ~/.deepseek/config.toml << 'EOF'
api_key = "sk-your-key-here"
model = "deepseek-v4-pro"
max_reasoning = true
lsp = true
snapshots = true
EOF
```

Verify with:
```bash
deepseek doctor
deepseek doctor --json
```

## Step 3: Install the Runtime API Server (systemd)

### 3a. Generate a runtime token

The runtime API is protected by a bearer token that you generate locally. Never
put it in a command line (`ps` shows command lines to every local user), a
script, or a committed file.

```bash
# Create a secret, user-owned env file
mkdir -p ~/.config/deepseek-tui
install -m 600 /dev/null ~/.config/deepseek-tui/runtime.env

# Generate and store the token
printf 'DEEPSEEK_RUNTIME_TOKEN=%s\n' "$(openssl rand -hex 32)" \
  >> ~/.config/deepseek-tui/runtime.env
chmod 600 ~/.config/deepseek-tui/runtime.env
```

A template lives at [`.env.example`](../.env.example). If you run the service as
a **system** unit instead of a user unit, use a root-owned file:
`sudo install -m 600 -o root -g root /dev/null /etc/deepseek-tui/runtime.env`
and point `EnvironmentFile=` at it.

> **Why an env file?** The server reads `DEEPSEEK_RUNTIME_TOKEN` when
> `--auth-token` is omitted (verified against upstream v0.8.26). If the env file
> is missing or blank, the server does **not** start unauthenticated — it
> generates an ephemeral token for that process, which no client knows. Clients
> then fail with 401/403 instead of silently working; see
> [Troubleshooting](troubleshooting.md#runtime-api-returns-401403-after-a-reboot).

### 3b. Install the unit

```bash
mkdir -p ~/.config/systemd/user
cp configs/deepseek-tui.service ~/.config/systemd/user/deepseek-tui.service
```

The unit ships as:

```ini
[Service]
Type=simple
EnvironmentFile=%h/.config/deepseek-tui/runtime.env
ExecStart=%h/.cargo/bin/deepseek-tui serve --http --host 127.0.0.1 --port 7878 --workers 2
Restart=on-failure
RestartSec=5
Environment=PATH=%h/.cargo/bin:/usr/bin:/bin
Environment=RUST_LOG=warn
```

```bash
# Enable linger so the service runs without login
sudo loginctl enable-linger $USER

# Start and enable
systemctl --user daemon-reload
systemctl --user enable --now deepseek-tui
systemctl --user status deepseek-tui
```

## Step 4: Install the Hermes Wrapper

```bash
cp scripts/deepseek-agent ~/.local/bin/deepseek-agent
chmod +x ~/.local/bin/deepseek-agent
```

The wrapper requires `DEEPSEEK_RUNTIME_TOKEN` in its environment and exits
immediately with a clear error if it is unset. For interactive use, export it
from the env file rather than duplicating the value:

```bash
export DEEPSEEK_RUNTIME_TOKEN="$(sed -n 's/^DEEPSEEK_RUNTIME_TOKEN=//p' ~/.config/deepseek-tui/runtime.env)"
```

Hermes invokes the wrapper through its `terminal` tool, so export the variable
wherever Hermes's shell environment is configured.

## Step 5: Verify End-to-End

```bash
# Health check (unauthenticated)
curl http://127.0.0.1:7878/health
# Expected: {"status":"ok"}

# Auth check — must NOT return 401/403
curl -s -o /dev/null -w '%{http_code}\n' \
  -H "Authorization: Bearer $DEEPSEEK_RUNTIME_TOKEN" \
  http://127.0.0.1:7878/v1/threads
# Expected: 200

# Test delegation
deepseek-agent "Say hello and confirm you're working" --timeout 60 --verbose
```

## Step 6: Hermes Skill

The skill at `~/.hermes/skills/autonomous-ai-agents/deepseek-tui/SKILL.md` tells Hermes how to use the integration. Hermes invokes `deepseek-agent` via its `terminal` tool — no custom tool definition needed.
