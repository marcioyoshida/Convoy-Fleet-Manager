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

Note · 2026-11-10: a held artifact that contains a leaked secret is quarantined (readers narrowed, never a cache hit), never deleted or rewritten; ADR 0007 amendment 5 refuses `--purge` on it and names the hold.

## Amendment 2 · 2026-11-09 · a hold extends to inputs (user story US-08, L8.3)

**Context.** Amendment 1 holds a process's receipt, outputs, the input artifacts they reference and the toolbox digest. Inputs that live in external stores (an object store prefix, a topic's offset range, a table) are not artifacts; they can change or vanish, and a hold on outputs alone does not make `replay --verify` possible in year three.

**Decision.** `--retain` extends to external inputs in one of two ways, chosen per device and printed by `explain --retain` with its monthly price: a copy of the input closure into the held artifact store of the stage's residency (ADR 0001 amendment 2), or an object lock on the device for the hold's duration where the device supports one (`caps: object-lock`). A topic's offset range is always copied (ADR 0012 decision 3). The receipt records which mechanism held each input and where. A hold that can do neither is refused with the reason, before the run.

**Consequences.** The held closure is complete, so `replay` under a later identity needs read on the held store or the locked objects and nothing else (US-08, L8.5). The cost of a hold is dominated by its inputs for large reads, which is why the price is printed first.

## Rules · 2026-11-09 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L1.5** (US-01) retention class `partial`, default seven days, marked in `ls -l /artifacts` and priced in the receipt.
- **L5.6** (US-05) the session pre-creates PVCs named `<pid>-<index>` and mounts them by index through the downward API; they are retained on restart and on `kill` (ADR 0006); M3 exit criterion gains this case.
- **L6.2** (US-06) `rmdir` refuses a non-empty namespace; `rm -r` lists everything that would be lost and requires `--with-data` and, for namespaces with volumes, a second approval (US-03 L3.1); reclaim policy Retain is the platform default.
- **L8.6** (US-08) holds copy the image by digest into a platform-held registry device (`/dev/registry/by-id/held`), or refuse when none exists.
  Note · 2026-11-13: the draft [ADR 0026](0026-a-registry-is-a-device-a-tag-is-a-name-for-a-digest.md) makes `held` one kind of registry device and pins every digest a hold covers, a receipt within retention names or a unit runs (decision 6).
- **L11.4** (US-11) `explain --retain` prices the copy per month in the band.
- **L26.3** (US-26) `--hold` on a telemetry read stores the result as an artifact, so a later `replay` reads it rather than a store that has since downsampled (full rule in ADR 0023).
- **L27.9** (US-27) `umount` or decommission of a store with holds is refused, naming each hold; `hold move <store>` copies held artifacts to a store with `caps: object-lock` or `bucket-lock`, sets the target lock to the original expiry, verifies content ids, and records the chain of custody (source, target, checksums, times, actors) in the hold's receipt.
- **L28.5, L28.6** (US-28) a plan may not destroy or replace a data-holding resource the change did not name; a partial apply registers `iac: partial` and is priced until completed or removed (full rules in ADR 0024).
- **L30.3, L30.4** (US-30) `rm` of a digest pinned by a running unit, a receipt within retention or a hold is refused; lifecycle policies that would expire a pinned digest are reported and the digest protected first (full rules in [ADR 0026](0026-a-registry-is-a-device-a-tag-is-a-name-for-a-digest.md)).
- **L31.17** (US-31) a hold, a branch unit, a lend or a grant raises `expiring` to its owning group before it lapses (full rule in [ADR 0027](0027-convoy-tells-a-group-about-its-own-objects.md)).
