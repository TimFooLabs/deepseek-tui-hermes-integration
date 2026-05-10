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

```bash
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/deepseek-tui.service << 'EOF'
[Unit]
Description=DeepSeek TUI Runtime API Server
Documentation=https://github.com/Hmbown/DeepSeek-TUI/blob/main/docs/RUNTIME_API.md
After=network.target

[Service]
Type=simple
ExecStart=%h/.cargo/bin/deepseek-tui serve --http --host 127.0.0.1 --port 7878 --workers 2 --auth-token deepseek-local-7878
Restart=on-failure
RestartSec=5
Environment=PATH=%h/.cargo/bin:/usr/bin:/bin
Environment=RUST_LOG=warn

[Install]
WantedBy=default.target
EOF

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

## Step 5: Verify End-to-End

```bash
# Health check
curl http://127.0.0.1:7878/health
# Expected: {"status":"ok"}

# Test delegation
deepseek-agent "Say hello and confirm you're working" --timeout 60 --verbose
```

## Step 6: Hermes Skill

The skill at `~/.hermes/skills/autonomous-ai-agents/deepseek-tui/SKILL.md` tells Hermes how to use the integration. Hermes invokes `deepseek-agent` via its `terminal` tool — no custom tool definition needed.
