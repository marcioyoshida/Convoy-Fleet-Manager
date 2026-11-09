# ADR 0009: stdout is an artifact, stderr is a log, the exit code is a receipt

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/debate-review.md §1; ADR 0001

## Context

The pod log endpoint returns one merged stream and rotates at 10 MiB by default. Routing data through it loses most of a large stdout. Exit codes vanish when the pod is garbage-collected.

## Decision

Remote stages write stdout to their artifact and stderr to the container log; `2>&1` is honoured inside the container. The session watches the object and persists exit code, reason and timestamps into the receipt before TTL deletion. Interactive processes are bare Pods with stdin and tty, attached over WebSocket.

## Consequences

`$?` reads the receipt, never the API. Terminal output of a pipeline is stderr live plus the final artifact, honouring early close.

## Amendment 1 · 2026-11-09 · receipts survive a lost link (user story US-10, L10.2)

**Context.** The decision has the session watch the object and persist the exit code before TTL deletion. When the session cannot reach the cluster for longer than the TTL, the object is gone before anyone recorded how it ended, and the receipt has no exit code.

**Decision.** The receipt is persisted where the object completes, by the node agent or a per-device receipt spool, at the moment of completion. TTL deletion is deferred until the session acknowledges the receipt: the Job carries a finalizer that the acknowledgement removes, or `ttlSecondsAfterFinished` is set only on acknowledgement, whichever the platform chooses and `explain` prints. When the link returns the spool drains into the session's receipts in order, and the audit stream marks each entry with both timestamps.

**Consequences.** A site can finish a night's work with no link and lose nothing but latency. A spool has a size; a site with no acknowledgement for longer than the platform's spool retention raises a hotplug event rather than silently dropping receipts. ROADMAP M10 gains the spool.

## Rules · 2026-11-09 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L4.2** (US-04) the toolbox provides `convoy-metric key=value` writing to `/sys/receipt/$pid/metrics`; receipts carry a `metrics` map flagged self-reported (see US-07 L7.2).
- **L8.5** (US-08) `replay` runs under the caller's identity, which needs read on the held closure only; the replay receipt links to the original and records both identities.
- **L16.2** (US-16) the receipt records node and attempt per partition and per group member; `status <pid>` lists partitions with node, attempt and kernel state (from the node agent) and `top` surfaces partitions stuck in D state; `ps --node <name>` and `ps -o node` exist.
