# ADR 0029: A computer's file systems are in its contract, and staging is a planned copy

**Status:** Draft · 2026-11-14 · rules from user story US-33 added 2026-11-14
**Supersedes / relates to:** open finding L33.1 (a computer's own file systems, and staging into and out of them, are not in the design); ADR 0025 decisions 2, 4, 7 and 9 and its open item on moving data to a computer's own file system (the computer contract, placement as one ordered filter, the credential exchange per kind, cost in the computer's unit); ADR 0001 and its amendment 2 (content ids, the artifact store per topology); ADR 0003 amendment 1 (placement by locality); ADR 0004 and its amendment 3 (the completion contract, the inherited contract); ADR 0006 (detach is not destroy, holds); ADR 0011 decision 9 and amendments 1, 3 and 4 (`/dev/fs`, drivers from packages, `residency`, `disconnected`); ADR 0015 (tables, snapshots and commits); ADR 0016 decisions 3, 4 and 7 (objects belong to a group, the exchange per open, approvals by a group); ADR 0017 decision 2 (gates from policy); ADR 0024 decisions 1 and 8 (devices installed by modules); ADR 0028 decision 10 (spill devices); walkthrough 2; the command map rows for `cp` (and L27.5), `mv`, `ls` and `du`

## Context

Some computers read their inputs where they are; others can only read their own file systems. A Kubernetes pod reads an object store over the network; a university's Slurm compute nodes often have no route out and read Lustre; a function has a small ephemeral disk; a batch job may get a Lustre file system linked to an object store, loaded on first read and exported back asynchronously; an on-premises Spark cluster may still have HDFS. ADR 0025 made every one of these a computer and left one item open: getting the data there before a stage and back after it.

US-33 ran a genomics pipeline through that gap. 38 TB of patient reads were copied by hand over a link that dropped at 61 % and started again from nothing, through 2.1 million small files that slowed a university's metadata server, priced nowhere. The copies sat under the cluster's one service account, readable by another group, then filled a quota the lab shared with its students. Two stages on the same computer moved 31 TB out to S3 and back. Stages exited 0 with their outputs not delivered, `mv` lost 212 files, a file system made by a job definition billed for three weeks after its run, a rerun trusted a copy the university's purge had removed, a 14 GB input went to a 10 GB disk, and walkthrough 2 copies 1.8 TiB into HDFS every day for a job that can read S3.

Facts that constrain the design:

- Object stores are strongly consistent, and table formats give atomic commits, so the main reasons HDFS was coupled to compute (locality on slow networks, rename as a commit) are gone where the computer can reach the store.
- Where a computer cannot reach the store, or reads the same data many times, or reads millions of small files, a copy on its own file system is faster and sometimes the only way.
- File systems on computers Convoy does not operate have their own policies: purges, quotas shared with people outside Convoy, transfer nodes, metadata limits.
- A copy can be deleted or changed under you; a path proves nothing, a content id does.

## Decision

1. **A computer's file systems are in its contract.** The computer contract (ADR 0025 decision 2) gains a field, `fs`, that every driver fills: each file system a stage's tasks can read or write, with its kind (`scratch`, `project`, `ephemeral`, `linked`, `hdfs`, `shared`), capacity per task and in total, quota and whether it is shared with people outside Convoy (and who owns it), purge policy, throughput and metadata limits, residency, the isolation it enforces (`per-user`, `per-group` or `none`), and for a `linked` file system the object store it is linked to and how it loads and exports. The field also says what the computer's tasks can reach directly (`reach`): which devices, and through what (the network, transfer nodes, nothing). `/sys/compute/<id>/fs/` holds it:

   ```
   /sys/compute/slurm-hpc-sp/fs/
   ├── scratch   lustre  kind=scratch  purge=21d-no-access  isolation=none  residency=br
   ├── project   lustre  kind=project  quota=60T shared=allocation:lab (outside Convoy)  isolation=none
   └── reach     /dev/obj: via transfer nodes (globus); compute nodes: none
   /sys/compute/batch-awsbatch-123456789012-sa-east-1/fs/
   └── linked    fsx-lustre  kind=linked  per-process  link=/dev/obj/…  load=on-first-read  export=async
   /sys/compute/fn-lambda-123456789012-sa-east-1/fs/
   └── tmp       kind=ephemeral  per-invocation=10G
   ```

   A spill device (ADR 0028 decision 10) is a file system of kind `ephemeral` in this field.

2. **The planner decides between reading in place and staging, and says why.** For each input of a stage, the planner reads in place when the computer can reach the device and the stage reads it in one pass. It stages when the computer cannot reach the device, or when the estimate shows a copy is cheaper or faster for the reads the stage will make (repeated reads, random access, many small files). `explain` prints, per input, `in place` or `staged` and the reason. In the ordered filter of ADR 0025 decision 4, a computer whose file systems cannot hold a stage's staged inputs and expected outputs per task is removed before ranking, and ranking counts the time and price of staging. Walkthrough 2's copy into HDFS becomes a read in place unless the plan shows otherwise.

3. **Staging is steps of the stage, compiled per kind.** Stage-in and stage-out are steps of the stage in the plan and in the process, never separate commands the user must remember. The driver compiles them to what the computer has: a transfer service through its transfer nodes, a copy job that the scheduler runs before and after the stage, a linked file system's load and export, a function's download to its ephemeral disk, a copy job into a volume on Kubernetes. Above `transfer_min` the provider's transfer service is used (L27.5). `ps` shows a process as `staging-in` or `staging-out` with the files done and left.

4. **A transfer is a manifest of content ids from a pinned source, and it resumes.** Stage-in reads a pinned snapshot of each source (object versions, a table snapshot, ADR 0015), writes the snapshot to the receipt, and builds a manifest of files and their content ids; a change to the source during the process does not reach it. Transfers are per file and resumable; a stage starts only when its whole manifest is verified at the target, and an interrupted link (ADR 0011 amendment 4) resumes from what is verified. `explain` of any staging prints its file count, its request cost and the target's metadata limit, and warns above it.

5. **A staged copy is a cache entry, verified before use.** A staged copy is keyed by content id on the target file system and listed under `/sys/compute/<id>/fs/<fs>/staged/`. A later process reuses it, within the group that owns it, or across groups only where the source device grants both. It is verified against its checksums before a stage uses it; a copy that is missing, changed or past the file system's declared purge is staged again, never trusted from a path or a receipt. Convoy never touches files to keep them from a purge.

6. **Staged data keeps its residency and its group.** Every file system a stage copies to carries a residency and is checked against the source's, as a mount is (ADR 0011 amendment 3). On a computer whose isolation is `none` or whose identity is `shared`, staged data goes only into a per-group directory the computer enforces; data from a mount marked `class=restricted` is never staged to a computer that cannot enforce one. Staged copies belong to the process's group (ADR 0016 decision 3).

7. **A stage succeeds only when its outputs arrive.** A stage that writes to a computer's own file system writes to a directory made for the process, and its stage-out ships only files the process wrote; other files found there are listed with their owners and not shipped. The stage succeeds only when every output is committed and verified by content id at its destination; an asynchronous export is part of the stage, and table outputs are committed through the table (ADR 0015). Otherwise the stage ends with exit 1 and lists what is missing. Artifacts passed between stages of one process tree on the same computer stay on its file system (ADR 0001 amendment 2 as revised by L33.13); only the tree's declared outputs, checkpoints and receipts go to the topology's store.

8. **What staging creates, the process owns, and it ends with the process.** Staged copies and file systems made for a process are objects of that process, listed in `/sys/proc/<pid>/staged`, and removed after the platform default `stage_keep` once the process ends in any way (exit, `kill`, deadline, an unreachable computer) unless a hold covers them (ADR 0006); the receipt records each copy, where it was and when it was removed. A file system for one process is made by the computer's driver through the mechanism the computer declares (a storage class, a burst buffer, a per-job allocation); a file system that outlives processes is a device installed by a module (ADR 0024 decision 1). Convoy never creates one by calling the provider directly. A file system no process uses is shown in `/sys` with its daily price.

9. **Staging is priced, and a shared quota is asked for.** `explain` prices each staging copy (egress, requests, and the target's capacity-hours) inside the stage's band, in the computer's own unit (ADR 0025 decision 9), and gives the copy's time as an estimate from earlier receipts, marked as such. For a file system with a quota shared with people outside Convoy, `explain` prints what is left, and staging more than the platform default `stage_share_max` of it needs an approval from the group that owns the allocation (ADR 0017 decision 2).

10. **`cp`, `mv`, `ls` and `du` keep their meaning across devices.** A copy by hand into a computer's file system is the same transfer as a stage-in and leaves the same kind of staged copy, owned by the user's group. `mv` across devices copies everything, verifies it at the destination, and only then deletes the sources; a failure part-way deletes nothing and lists what was copied. On a `linked` file system, `ls -l` and `/sys` show each file's state (`stub`, `loaded`, `modified`, `exported`), `du` reports loaded and total bytes, and `explain` counts unloaded bytes as a stage-in.

## Declined

| Proposal | Why not | What replaces it |
|---|---|---|
| one global file system that every computer mounts | reach, latency, residency and policy differ by computer, and a single namespace hides them | each computer's `fs` and `reach` (decision 1) |
| coupling storage to compute by default, as HDFS did | object stores and table formats remove the reasons where the store is reachable | read in place unless staging is cheaper or required (decision 2) |
| staging as a command the user writes | forgotten, unpriced, unverified, never cleaned up | stage-in and stage-out as steps of the stage (decision 3) |
| trusting a staged copy by its path or a receipt | purges and edits happen under it | verification by content id before use (decision 5) |
| touching files to keep them from a purge | it defeats the operator's policy on a computer Convoy does not run | staging again (decision 5) |
| Convoy creating a provider's file system | the owner is a module or the computer's driver | the declared mechanism (decision 8) |

## Consequences

- L33.1 is drafted here, and ADR 0025's open item on moving data to a computer's own file system is settled by it. [US-33](../user-stories/US-33-the-data-has-to-be-there-first.md) was applied on 2026-11-14: L33.13 revised ADR 0001 amendment 2 inline (artifacts between stages of one tree on one computer stay on its file system), which decision 7 assumes; its sixteen rules are in the Rules section below, with pointers in ADRs 0001, 0003, 0004, 0006, 0009, 0010, 0011, 0015, 0016, 0017, 0024 and 0025 and a note in walkthrough 2; L33.17 is accepted and is decision 9's estimate.
- ADR 0025 decision 2 gains the `fs` field and decision 4's filter gains the capacity check and the staging cost in ranking. ADR 0003 amendment 1's locality becomes the choice between reading in place and staging. ADR 0011 decision 9's `/dev/fs` devices are file systems that outlive processes; ADR 0028 decision 10's spill devices are `ephemeral` file systems. Walkthrough 2 reads S3 in place unless the plan shows a copy is better. The command map rows for `cp`, `mv`, `ls` and `du` read as decision 10.
- New paths: `/sys/compute/<id>/fs/`, `/sys/compute/<id>/fs/<fs>/staged/`, `/sys/proc/<pid>/staged`. New platform defaults: `stage_keep`, `stage_share_max`. New `caps`: `fs:<kind>`, `reach`.
- ROADMAP: the choice between reading in place and staging, the capacity check and staging steps in the plan are M3; staged copies as cache entries by content id and pinned sources are M4; the `fs` field, linked-file states in `ls` and `du`, and `mv` across devices are M5; per-group staging directories and `class=restricted` are M7; cleanup after `stage_keep` and idle file systems shown with their price are M8; the per-kind staging drivers and transfer services are M9.
- Open, to be settled against this record: interactive sessions on a scheduler that work on staged data (`salloc`, a notebook on a GPU node); inputs too large to stage that must be streamed; computers at the edge whose file system holds the only copy; a story that probes this record.


Note · 2026-11-14: the draft [ADR 0030](0030-an-interactive-process-holds-a-computer-while-someone-is-attached.md) settles the open item on interactive sessions over staged data: an interactive process is placed near its data like a stage, and the inputs it reads are recorded with their snapshots (decisions 7, 8).

Note · 2026-11-14: the draft [ADR 0031](0031-an-edge-computer-is-reached-by-contact-and-known-by-what-it-delivers.md) settles the open item on edge computers holding the only copy: decision 7's rule gains the `done-local` state before `delivered`, the device keeps its outputs until delivery, and `ls` says `only copy on <device>`.
## Rules · 2026-11-14 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L33.2** (US-33) a staged copy is a cache entry keyed by content id, verified against its checksums before use; a copy that is missing, changed or past the file system's declared purge is staged again; Convoy never touches files to avoid a purge (decision 5).
- **L33.3** (US-33) every file system a stage copies to carries a residency and is checked like a mount; staged copies are removed when the process tree ends unless held, and the receipt records each copy, its place and its removal (decisions 6, 8).
- **L33.4** (US-33) `explain` prices each staging copy (egress, requests, the target's capacity-hours) inside the stage's band; a file system made for a process is removed with it; one no process uses is shown in `/sys` with its daily price (decisions 8, 9).
- **L33.5** (US-33) a stage with outputs on a computer's own file system succeeds only when every output is committed and verified by content id at its destination; an asynchronous export is part of the stage; otherwise exit 1 with the missing outputs listed (decision 7).
- **L33.6** (US-33) `mv` across devices copies everything, verifies it at the destination, and only then deletes the sources; a failure part-way deletes nothing and lists what was copied (decision 10).
- **L33.7** (US-33) when a process ends in any way, its staged copies and the file systems made for it are removed after `stage_keep` unless held; `/sys/proc/<pid>/staged` lists them until then (decision 8).
- **L33.8** (US-33) on a computer with `identity=shared` or isolation `none`, staged data goes only into a per-group directory the computer enforces; data from a mount marked `class=restricted` is never staged to a computer that cannot enforce one (decision 6).
- **L33.9** (US-33) transfers are per file by content id and resumable; a stage starts only when its whole manifest is verified at the target; `ps` shows `staging-in` with files done and left (decisions 3, 4).
- **L33.10** (US-33) a stage-out ships only files the process wrote, in a directory made for the process; other files there are listed with their owners and not shipped (decision 7).
- **L33.11** (US-33) a file system that outlives one process is a device installed by a module; one for a single process is made by the computer's driver through the mechanism the computer declares, never by Convoy calling the provider (decision 8).
- **L33.12** (US-33) a computer declares which file systems have a quota shared outside Convoy; `explain` prints what is left; staging above `stage_share_max` of it needs an approval from the group that owns the allocation (decision 9).
- **L33.14** (US-33) placement compares a stage's staged input and expected output with each computer's file-system capacity per task and removes those that cannot hold them before ranking (decision 2).
- **L33.15** (US-33) a stage reads its input in place when the computer can reach it, unless the plan shows staging is cheaper or faster for the reads it will make; `explain` prints the choice and why (decision 2).
- **L33.16** (US-33) `explain` of a copy prints its file count, its request cost and the target file system's declared metadata limit, and warns above it (decision 4).
- **L33.18** (US-33) a stage-in reads a pinned snapshot of the source (object versions, a table snapshot), recorded in the receipt; changes to the source during the process do not reach it (decision 4).
- **L33.19** (US-33) on a file system linked to an object store, `ls -l` and `/sys` show each file's state (`stub`, `loaded`, `modified`, `exported`), `du` reports loaded and total bytes, and `explain` counts unloaded bytes as a stage-in (decision 10).
- **L34.7** (US-34) an interactive process records the snapshot of each input it reads, as a stage pins its own (full rule in [ADR 0030](0030-an-interactive-process-holds-a-computer-while-someone-is-attached.md)).
