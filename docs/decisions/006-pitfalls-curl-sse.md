# ADR-006: curl + SSE Pitfalls (Debugging Notes)

## Status

Resolved — documented as reference

## Context

During development, the wrapper script initially used the SSE events endpoint (`/events`) to stream agent output in real-time. This approach had multiple problems.

## Problems Encountered

### 1. curl Buffers SSE Output

Even with `curl -N` (no-buffer flag), curl may buffer SSE data. The `-N` flag disables the progress meter's output buffering, but curl's internal buffering of HTTP response bodies can still apply.

### 2. SSE Stream Never Closes

The `/events` endpoint is a **long-lived stream**. It does not close after turn completion. A script reading from it will hang indefinitely.

### 3. JSON Body Format

The initial wrapper passed JSON as a positional argument to curl instead of via `-d`:
```bash
# WRONG — JSON treated as URL path segment
curl -X POST "$SERVER/v1/threads/{id}/turns" '{"prompt":"..."}'

# CORRECT — JSON in request body
curl -X POST "$SERVER/v1/threads/{id}/turns" -d '{"prompt":"..."}'
```

### 4. Wrong Request Body Schema

The turn creation endpoint expects `{"prompt": "...", "max_turns": N}`, NOT `{"messages": [...]}` (OpenAI chat format). The DeepSeek TUI API is NOT the OpenAI chat completions API.

```bash
# WRONG — OpenAI format
curl ... -d '{"messages": [{"role":"user","content":"..."}]}'

# CORRECT — DeepSeek TUI format
curl ... -d '{"prompt": "...", "max_turns": 20}'
```

## Resolution

Switched to thread polling for completion detection. SSE is available for real-time streaming if needed, but the wrapper uses polling for reliability.

## Lesson

Always verify API contracts against the actual documentation (`docs/RUNTIME_API.md`), not against assumptions based on similar APIs (OpenAI).
