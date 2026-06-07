# ADR-0006 — Command ingestion is idempotent by comment id; the poll cursor is not canonical

- **Date:** 2026-06-07
- **Status:** Accepted

## Context

Tracker commands (`/approve`, `/request-changes`, `/reject`) arrive by
polling issue comments on each `tick`, paced by a per-card cursor + ETag
stored in a `.cursor.json` sidecar. The cursor is the one deliberately
non-canonical piece of run state (it tracks non-command comments too, so
it can't be derived from the event log). That makes it tempting to treat
the cursor as the exactly-once boundary for command ingestion — but it
isn't crash-safe: a tick that appends `command_received` and then crashes
*before* persisting the advanced cursor will, on the next tick, re-read
the same comment and append a **duplicate** `command_received` into the
append-only log.

## Decision

The canonical idempotency key for a command is the tracker's **stable
comment id**, recorded on every `command_received` event. The fold treats
a `command_received` whose comment id is already in the log as a no-op, so
re-ingestion after a crash is harmless. The poll cursor is demoted to a
pure **fetch optimization** (fewer comments pulled per tick) and is never
a correctness boundary — worst case after a lost cursor is a re-fetch that
the comment-id dedup absorbs.

This keeps the command path crash-safe **by replay**, consistent with the
`step_dispatched`-before-execute discipline (ADR-0003), rather than
relying on a sidecar surviving a crash.

## Considered options

- **Cursor as source of truth (rejected).** Simplest, but not crash-safe:
  a crash between appending the event and persisting the cursor duplicates
  the command.
- **Comment-id dedup in the log (chosen).** One extra field per command
  event buys idempotent ingestion that survives crashes and never depends
  on sidecar durability.

## Consequences

- Every `command_received` carries `comment_id`.
- **First-valid-wins per gate** falls out naturally: once the first valid
  decision appends `gate_decided` and closes the gate, later command
  comments target an already-closed gate and fail validation (audit-only).
- The engine's own posted comments (gate prompts) are recorded with their
  comment id + bot `actor`, so they can never be re-ingested as commands.
