# ADR 0009: stdout is an artifact, stderr is a log, the exit code is a receipt

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/debate-review.md §1; ADR 0001

## Context

The pod log endpoint returns one merged stream and rotates at 10 MiB by default. Routing data through it loses most of a large stdout. Exit codes vanish when the pod is garbage-collected.

## Decision

Remote stages write stdout to their artifact and stderr to the container log; `2>&1` is honoured inside the container. The session watches the object and persists exit code, reason and timestamps into the receipt before TTL deletion. Interactive processes are bare Pods with stdin and tty, attached over WebSocket.

## Consequences

`$?` reads the receipt, never the API. Terminal output of a pipeline is stderr live plus the final artifact, honouring early close.

Note · 2026-11-10: the draft [ADR 0020](0020-a-session-is-a-journal-not-a-pod.md) decision 1 moves the watcher and receipt writer from the session to the Convoy controller, so a lost session loses no receipt; amendment 1 below is unchanged.

Note · 2026-11-11: the draft [ADR 0023](0023-telemetry-is-a-device-the-shell-reads.md) decision 7 gives stderr its second half: a process's stderr is one path across its life, read from the pod while it exists and from the platform's log device after, and the receipt names the log device and selector that hold it.

## Amendment 1 · 2026-11-09 · receipts survive a lost link (user story US-10, L10.2)

**Context.** The decision has the session watch the object and persist the exit code before TTL deletion. When the session cannot reach the cluster for longer than the TTL, the object is gone before anyone recorded how it ended, and the receipt has no exit code.

**Decision.** The receipt is persisted where the object completes, by the node agent or a per-device receipt spool, at the moment of completion. TTL deletion is deferred until the session acknowledges the receipt: the Job carries a finalizer that the acknowledgement removes, or `ttlSecondsAfterFinished` is set only on acknowledgement, whichever the platform chooses and `explain` prints. When the link returns the spool drains into the session's receipts in order, and the audit stream marks each entry with both timestamps.

**Consequences.** A site can finish a night's work with no link and lose nothing but latency. A spool has a size; a site with no acknowledgement for longer than the platform's spool retention raises a hotplug event rather than silently dropping receipts. ROADMAP M10 gains the spool.

## Rules · 2026-11-09 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L4.2** (US-04) the toolbox provides `convoy-metric key=value` writing to `/sys/receipt/$pid/metrics`; receipts carry a `metrics` map flagged self-reported (see US-07 L7.2).
- **L8.5** (US-08) `replay` runs under the caller's identity, which needs read on the held closure only; the replay receipt links to the original and records both identities.
- **L16.2** (US-16) the receipt records node and attempt per partition and per group member; `status <pid>` lists partitions with node, attempt and kernel state (from the node agent) and `top` surfaces partitions stuck in D state; `ps --node <name>` and `ps -o node` exist.
- **L29.6, L29.11** (US-29) receipts settle each stage in its computer's price model, allocations in their own units; an image conversion records the digest and the converted hash (full rules in [ADR 0025](0025-a-computer-is-a-device-kubernetes-is-one-driver.md)).
- **L30.3, L30.15** (US-30) a receipt's digests stay pullable for its retention; the receipt records every container that ran, injected ones included (full rules in [ADR 0026](0026-a-registry-is-a-device-a-tag-is-a-name-for-a-digest.md)).
- **L31.6, L31.10** (US-31) messages are priced to the group that owns the object; events raised at a disconnected site are spooled with the receipts (full rules in [ADR 0027](0027-convoy-tells-a-group-about-its-own-objects.md)).
- **L33.3, L33.4** (US-33) the receipt records each staged copy, where it was and when it was removed, and settles staging in the stage's band (full rules in [ADR 0029](0029-a-computers-file-systems-are-in-its-contract-staging-is-a-planned-copy.md)).
