# ADR 0009: stdout is an artifact, stderr is a log, the exit code is a receipt

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/debate-review.md §1; ADR 0001

## Context

The pod log endpoint returns one merged stream and rotates at 10 MiB by default. Routing data through it loses most of a large stdout. Exit codes vanish when the pod is garbage-collected.

## Decision

Remote stages write stdout to their artifact and stderr to the container log; `2>&1` is honoured inside the container. The session watches the object and persists exit code, reason and timestamps into the receipt before TTL deletion. Interactive processes are bare Pods with stdin and tty, attached over WebSocket.

## Consequences

`$?` reads the receipt, never the API. Terminal output of a pipeline is stderr live plus the final artifact, honouring early close.
