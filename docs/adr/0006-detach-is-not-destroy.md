# ADR 0006: Detach is not destroy: umount, kill and rm never delete data

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/debate-review.md (disk verbs); docs/stories.md story 9

## Context

A research note mapped `umount` to PVC deletion. Deleting a claim on a Delete reclaim policy wipes the disk. Kubernetes' PVC-protection finalizer does not refuse deletion; it defers it silently until the last pod exits.

## Decision

`umount` edits the session mount table and affects only future processes. `kill` scales compute to zero and retains volumes. `rm` on a volume or namespace is refused and points to a destructive verb (`rmvol`, with `--with-data` for namespaces) that checks in-use via the volume→pods index, snapshots first by default, and asks for confirmation.

## Consequences

Volumes outlive processes and must appear in `df` with a cost column and a retention policy. Scale-down of StatefulSets must state what claims are retained.

Amendment note 2026-11-08: `scale` down on a unit with per-replica volumes delegates to the operator or refuses, and in every case retains the volumes (docs/walkthroughs.md, gap G6).

## Amendment 1 · 2026-11-08 · retention classes and legal hold (walkthrough 8)

**Context.** The artifact store fills unless something is garbage-collected, and the decision above says nothing about what GC may delete. A regulated workload needs its receipts, artifacts and the exact toolbox image kept for years and reproducible on demand.

**Decision.** Every receipt and artifact carries a retention class, defaulting to the platform's (ADR 0010). `--retain=<duration>` on a process sets a hold on its receipt, its output artifacts, the input artifacts they reference and the toolbox image digest they pin. GC, `rm`, `rmvol` and image-registry pruning refuse to break a hold and say so. `replay <pid> --verify` re-executes from content ids with the pinned image and reports whether the output ids match.

**Consequences.** Holds pin images, so the registry must be part of the artifact store's GC scope or the hold is incomplete. `du` on `/artifacts` shows held bytes separately with their monthly cost. See docs/walkthroughs.md, gap G9.

## Amendment 2 · 2026-11-09 · a hold extends to inputs (user story US-08, L8.3)

**Context.** Amendment 1 holds a process's receipt, outputs, the input artifacts they reference and the toolbox digest. Inputs that live in external stores (an object store prefix, a topic's offset range, a table) are not artifacts; they can change or vanish, and a hold on outputs alone does not make `replay --verify` possible in year three.

**Decision.** `--retain` extends to external inputs in one of two ways, chosen per device and printed by `explain --retain` with its monthly price: a copy of the input closure into the held artifact store of the stage's residency (ADR 0001 amendment 2), or an object lock on the device for the hold's duration where the device supports one (`caps: object-lock`). A topic's offset range is always copied (ADR 0012 decision 3). The receipt records which mechanism held each input and where. A hold that can do neither is refused with the reason, before the run.

**Consequences.** The held closure is complete, so `replay` under a later identity needs read on the held store or the locked objects and nothing else (US-08, L8.5). The cost of a hold is dominated by its inputs for large reads, which is why the price is printed first.
