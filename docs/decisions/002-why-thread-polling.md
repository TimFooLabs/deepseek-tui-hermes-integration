# ADR-002: Why Thread Polling Instead of SSE Event Streaming

## Status

Accepted

## Context

The DeepSeek TUI runtime API provides two ways to monitor turn progress:

1. **SSE stream** — `GET /v1/threads/{id}/events?since_seq=N` returns a long-lived Server-Sent Events stream
2. **Thread polling** — `GET /v1/threads/{id}` returns the full thread state including turn status and items

## Decision

Use thread polling (`GET /v1/threads/{id}`) to detect turn completion and extract results.

## Rationale

The SSE endpoint has two critical problems for scripted use:

1. **It never closes.** The `/events` stream is a long-lived connection that stays open after turn completion. A script reading from it will hang indefinitely waiting for more events.
2. **curl buffers SSE output.** Even with `--no-buffer` (`-N`), curl may buffer SSE data, making real-time detection unreliable.

Thread polling avoids both issues:
- Each request is independent and returns immediately
- The response includes `turns[0].status` which transitions from `in_progress` → `completed`/`failed`
- The `items[]` array contains the full agent response text

Polling interval: 5 seconds. This is a good balance between responsiveness and API load.

## Consequences

- Slight delay in completion detection (up to 5s polling interval)
- More API calls than SSE (but each is lightweight)
- Simpler, more reliable code

## Alternatives Considered

- **SSE with curl --no-buffer**: Tested. curl still buffers. The stream doesn't close on completion. Unreliable for scripted completion detection.
- **SSE with a proper SSE client (e.g., Python sseclient)**: Would work but adds a dependency. Polling with curl is simpler and uses tools already available.
- **Webhook/callback pattern**: Not supported by the API.
