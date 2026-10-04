# Troubleshooting

## Server Issues

### `ERROR: DeepSeek TUI runtime API not reachable at http://127.0.0.1:7878`

```bash
# Check service status
systemctl --user status deepseek-tui

# Restart
systemctl --user restart deepseek-tui

# View logs
journalctl --user -u deepseek-tui -f

# Verify port is listening
ss -tlnp | grep 7878
```

### Service fails to start after WSL reboot

Ensure linger is enabled:
```bash
sudo loginctl enable-linger $USER
loginctl show-user $USER | grep Linger
# Should show: Linger=yes
```

## Build Issues

### `version 'GLIBC_2.39' not found`

Prebuilt binaries require glibc 2.39+. Ubuntu 24.04 has glibc 2.35. Build from source:
```bash
cargo build --release
```

### `MISSING_COMPANION_BINARY`

Both `deepseek` and `deepseek-tui` must be on PATH:
```bash
export PATH="$HOME/.cargo/bin:$PATH"
which deepseek deepseek-tui
```

### `Unable to find library -ldbus-1`

```bash
# Option 1: Install dev package
sudo apt install libdbus-1-dev

# Option 2: Create symlink
ln -s /lib/x86_64-linux-gnu/libdbus-1.so.3 ~/.local/lib/libdbus-1.so
```

## API Issues

### Turn times out

- Increase `--timeout` (default 600s). Complex tasks may need 900s+.
- Reduce `--max-turns` if the agent is looping.
- Check `journalctl --user -u deepseek-tui` for API errors.

### Agent returns empty response

- Verify API key: `deepseek doctor`
- Check the thread directly:
  ```bash
  curl -s -H "Authorization: Bearer $DEEPSEEK_RUNTIME_TOKEN" \
    http://127.0.0.1:7878/v1/threads/THREAD_ID | jq '.items'
  ```

### Runtime API returns 401/403 after a reboot

The server found no `DEEPSEEK_RUNTIME_TOKEN` at startup and generated an
ephemeral token for that process — it refuses to serve `/v1/*` unauthenticated,
so every client gets 401/403.

```bash
# Does the env file exist, and is it non-empty?
ls -l ~/.config/deepseek-tui/runtime.env
grep -c '^DEEPSEEK_RUNTIME_TOKEN=' ~/.config/deepseek-tui/runtime.env

# Check what the server resolved at startup
journalctl --user -u deepseek-tui -n 20 | grep -i "auth"
```

Fix: regenerate the env file per [Setup, Step 3](setup.md#3a-generate-a-runtime-token),
then `systemctl --user restart deepseek-tui`.

### Wrapper exits immediately: `DEEPSEEK_RUNTIME_TOKEN is not set`

The wrapper no longer ships a default token — this is the fail-fast guard.
Export it in the shell that launches Hermes (see
[Setup, Step 4](setup.md#step-4-install-the-hermes-wrapper)).

### `curl: (56) Recv failure: Connection reset by peer`

The server may be overloaded. Reduce `--workers` in the service file or wait for current turns to complete.

## Wrapper Issues

### Wrapper exits with code 3 (server unreachable)

The systemd service isn't running. Start it:
```bash
systemctl --user start deepseek-tui
```

### Wrapper exits with code 2 (timeout)

The task exceeded the timeout. Either:
- Increase `--timeout`
- Simplify the task
- Break the task into smaller subtasks

### Wrapper exits with code 1 (API/LLM error)

Check:
1. API key validity
2. Model name correctness (`deepseek-v4-pro`)
3. Server logs: `journalctl --user -u deepseek-tui -n 50`
