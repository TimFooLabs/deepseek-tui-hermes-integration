# Runtime API Reference (Verified)

DeepSeek TUI's runtime API is documented in `docs/RUNTIME_API.md` in the [DeepSeek-TUI repo](https://github.com/Hmbown/DeepSeek-TUI). This is a verified subset used by the Hermes integration.

## Base URL

```
http://127.0.0.1:7878
```

## Authentication

All `/v1/*` routes require a bearer token:

```
Authorization: Bearer $DEEPSEEK_RUNTIME_TOKEN
```

The token is generated locally (`openssl rand -hex 32`), stored in the service
env file (`~/.config/deepseek-tui/runtime.env`, mode `600`), and read by the
server when `--auth-token` is omitted. It is never hardcoded or committed. See
[Setup, Step 3](setup.md#3a-generate-a-runtime-token).

`GET /health` is unauthenticated. If no token is configured, the server
generates an ephemeral one for that process rather than serving `/v1/*` open.

## Endpoints

### Health Check

```
GET /health
→ {"status": "ok"}
```

### Create Thread

```
POST /v1/threads
Body: {"model": "deepseek-v4-pro", "workspace": "/path/to/project", "mode": "agent", "auto_approve": false}
→ {"id": "thread_abc123", "model": "...", "workspace": "...", ...}
```

### Submit Turn

```
POST /v1/threads/{thread_id}/turns
Body: {"prompt": "Your task here", "max_turns": 20, "stream": false}
→ {"id": "turn_xyz789", "status": "queued", ...}
```

### Get Thread (Polling)

```
GET /v1/threads/{thread_id}
→ {
    "id": "thread_abc123",
    "turns": [{"id": "...", "status": "completed", ...}],
    "items": [
      {"kind": "agent_message", "text": "The agent's response..."},
      ...
    ]
  }
```

**This is the primary polling endpoint.** Check `turns[0].status` for completion. Extract agent text from `items[i].text` where `items[i].kind == "agent_message"`.

### Get Events (SSE — Not Used by Wrapper)

```
GET /v1/threads/{thread_id}/events?since_seq=0
→ (long-lived SSE stream, does not close on completion)
```

### Interrupt Turn

```
POST /v1/threads/{thread_id}/turns/{turn_id}/interrupt
→ {"status": "interrupted"}
```

### Delete Thread

```
DELETE /v1/threads/{thread_id}
→ 204 No Content
```

## Turn Statuses

| Status | Meaning |
|--------|---------|
| `queued` | Waiting to start |
| `in_progress` | Agent is working |
| `completed` | Done — check `items[]` for results |
| `failed` | Error occurred |
| `interrupted` | Manually interrupted |
| `canceled` | Canceled before completion |

## Event Types (SSE)

| Event | When |
|-------|------|
| `thread.started` | Thread created |
| `turn.started` | Turn begins processing |
| `item.started` | Agent begins generating a response item |
| `item.delta` | Streaming text chunk |
| `item.completed` | Response item finished |
| `turn.lifecycle` | Turn state change |
| `turn.completed` | Turn finished |
| `turn.failed` | Turn errored |
| `approval.required` | Tool call needs approval (if not in yolo mode) |
