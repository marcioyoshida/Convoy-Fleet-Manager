# ADR 0006: Detach is not destroy: umount, kill and rm never delete data

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/debate-review.md (disk verbs); docs/stories.md story 9

## Context

A research note mapped `umount` to PVC deletion. Deleting a claim on a Delete reclaim policy wipes the disk. Kubernetes' PVC-protection finalizer does not refuse deletion; it defers it silently until the last pod exits.

## Decision

`umount` edits the session mount table and affects only future processes. `kill` scales compute to zero and retains volumes. `rm` on a volume or namespace is refused and points to a destructive verb (`rmvol`, with `--with-data` for namespaces) that checks in-use via the volume→pods index, snapshots first by default, and asks for confirmation.

## Consequences

Volumes outlive processes and must appear in `df` with a cost column and a retention policy. Scale-down of StatefulSets must state what claims are retained.
