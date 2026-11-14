# US-36 · Too big to stage: a training run that reads 1.4 PB it can never hold, through a window, four times

| Field | Value |
|---|---|
| Primary persona | ML |
| Secondary personas | DE, SEC, FIN, PE, SRE |
| Workload class | streaming inputs: a gang training run whose input (1.4 PB in 40 million objects, owned by another group) is too large to stage onto any computer's file system and is read as a moving window, once per epoch, on the GB200 cluster in the same region and on the university Slurm cluster through its transfer nodes |
| Walkthrough | written against [ADR 0029](../adr/0029-a-computers-file-systems-are-in-its-contract-staging-is-a-planned-copy.md) decisions 1, 2, 4, 5, 6 and 8 and its open item on inputs too large to stage, [ADR 0001](../adr/0001-pipe-is-a-staged-artifact.md) and its amendment 1, [ADR 0012](../adr/0012-logs-and-queues-are-devices.md) decision 2, the pipeline compiler §8.5, [ADR 0004](../adr/0004-completion-contract-and-exit-codes.md) amendments 1, 2 and 3, [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md) decisions 4, 5 and 7, [ADR 0017](../adr/0017-approvals-are-objects-on-the-process.md) decision 2, [ADR 0025](../adr/0025-a-computer-is-a-device-kubernetes-is-one-driver.md) decisions 2, 4 and 8, [ADR 0026](../adr/0026-a-registry-is-a-device-a-tag-is-a-name-for-a-digest.md) (pins), [ADR 0028](../adr/0028-memory-has-a-distance-and-a-domain-is-claimed-whole.md) decision 9, and the command map rows for `cat`, `du` and `fuser`; it follows US-35 |
| Gaps already closed there | G4 (gang contract) |
| Milestones needed | M3, M4, M5, M6, M7, M8 |
| Findings in this document | L36.1 – L36.19 |

## 1. Story

**As an** ML engineer, **I want** to train on a dataset far larger than any disk I can get, reading it as a stream at the rate my GPUs need, from where it lives, **so that** I do not choose between a plan that refuses because it cannot stage 1.4 PB and a run that reads in place at a third of the speed my GPUs could go, re-reads 40 % of the data after every restart, skips the shards it cannot read and still exits 0, and leaves seven terabytes of other people's faces on node disks when it ends.

**Context.** After US-35 the platform runs ADRs 0001 to 0031. The `fleet-data` group owns `/s3/dashcam`: 1.4 PB of dashcam frames in 40 million objects in us-east-1, mostly single JPEGs with some tar shards, `class=restricted` because frames show faces and plates. A lifecycle policy, in `fleet-data`'s Terraform, moves prefixes not read for 90 days to an archive storage class. `fleet-data`'s own production readers use the same prefixes. `kenji` (group `vision`) trains a video model for four epochs on 32 GPUs: 8 replicas of 4, with 4 data-loader workers each. ADR 0029 lets the planner read an input in place or stage it; neither fits an input this size on a computer that cannot reach it, and in place says nothing about rate.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Trains the model | `kenji` | user `kenji`, group `vision` | |
| Owns the dataset | `ravi` | user `ravi`, group `fleet-data` | runs production readers on the same prefixes |
| Owns residency and the class | `amara` | user `amara`, group `security` | |
| Operates `hpc-sp` | the university's HPC team | none | transfer nodes are the only way out |
| Pays | FinOps | group `finops` | requests and egress |

## 3. Preconditions

- **Capabilities** (`caps`): on `gb200`: local NVMe per node, `dra`, `kueue`, same region as the dataset; on `hpc-sp`: `reach` to object stores only through four transfer nodes, `/scratch` with a 60 TB quota (US-33).
- **Packages**: the training image uses a common streaming data-loader library, which can read objects or tar shards and keeps its own local cache.
- **Platform defaults** `explain` must print: for an input read as a stream, the manifest it reads and how it was pinned, the rate the stage needs and the rate the source, link and relays can give, the window and where it lives, how the stream is partitioned, the requests and bytes per epoch and their price, and what happens to the window when the process ends.

## 4. Main path

```bash
# Monday 08:00 · kenji, group vision
$ explain batch --gang --replicas=8 --gpu=4 --checkpoint=/ckpt --on hpc-sp \
    torchrun train.py --data /s3/dashcam/ --epochs=4
#  hpc-sp cannot reach /s3 (reach: transfer nodes) and cannot stage 1.4 PB into 60 TB;
#  there is no third choice; the plan is refused                                         (L36.1, L36.15)
$ explain …same… --on gb200
#  input /s3/dashcam: in place; building the manifest lists 40 million objects and takes
#  3 hours and 40 000 list requests                                                      (L36.18)
#  nothing about the rate the GPUs need or the 160 million GETs over four epochs         (L36.6, L36.14)

# Monday 11:00 · the run starts on gb200
#  each of the 32 loader workers reads the whole dataset, so every shard is read 32 times (L36.13)
#  GPUs average 35 % utilisation: the stream is slower than the GPUs; ps says running      (L36.14)
#  the library's own cache fills ~/.cache on the nodes' root disks; two nodes reach disk
#  pressure                                                                                 (L36.10)
#  fleet-data's production readers get "slow down" responses: kenji's stream takes 80 % of
#  the prefixes' request rate                                                               (L36.11)

# Monday 23:00 · the exchanged credential expires after 12 hours; every read fails; the
#  loader retries and then skips                                                           (L36.5)
# Tuesday · fleet-data's lifecycle policy moves 2023 prefixes to the archive class;
#  reads of those objects fail; the loader skips them                                     (L36.2)
# Wednesday 04:00 · a node fault; the gang restarts from /ckpt/step-90000 (ADR 0004 amendment 1)
#  the loader's position in the epoch is not in the checkpoint; it starts the epoch again
#  and reads 40 % of it a second time                                                       (L36.4)
#  fleet-data re-encoded 2 million frames on Tuesday; the window still serves the old ones (L36.7)
# Friday · the run exits 0; the model saw 91 % of the data; nothing says which 9 % it missed (L36.8)
#  7 TB of restricted frames stay on the NVMe of 18 nodes, readable by the next tenant    (L36.12)
#  a second run with the same configuration is not comparable: its shuffle seed was never
#  recorded                                                                                 (L36.19)

# The next week · kenji tries hpc-sp again through the transfer nodes by hand
#  a relay on one of the four transfer nodes; it goes down on day 2; half the ranks starve,
#  the others wait at the collective, and the run burns its allocation at 0 %              (L36.9)
#  a colleague preprocesses with a pipe:
$ cat /s3/dashcam/**/*.jpg | python preprocess.py > /s3/dashcam-small/
#  the plan begins to stage a 1.4 PB artifact                                            (L36.3)
#  FinOps asks why the same 1.4 PB was paid for four times                                (L36.16)
#  the same run reads at 9 GB/s one day and 3 GB/s the next                              (L36.17)
```

| Step | Convoy object | Store or computer object | Where the state lives |
|---|---|---|---|
| the input | a mount of `/s3/dashcam` | 40 million objects, some archived | the store |
| the manifest | none; a listing at plan time | object versions and ETags | the plan |
| the window | none | the loader's cache in `~/.cache` | the node's root disk |
| the position in the stream | none | the loader's memory | the pods |
| a relay | none | a process on a transfer node | the university |
| the lifecycle policy | none | the bucket's lifecycle rules | `fleet-data`'s Terraform |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe: staged artifact, content id, cache | ● | a pipe over 1.4 PB (L36.3); a window keyed by content id (L36.7); a manifest of 40 million objects (L36.18) | AC-36.3, AC-36.7, AC-36.18 |
| A2 | Command: class, annotation, executable fields | ○ | the reader library's skip and cache behaviour (L36.8, L36.10) | AC-36.8 |
| A3 | Session: mount table, env, cwd, attach and detach | – | | |
| A4 | Backend selection | ● | a third choice beside in place and staged (L36.15); rate in placement (L36.14) | AC-36.14, AC-36.15 |
| A5 | Completion contract | ● | 91 % read and exit 0 (L36.8); the position in the stream at a gang restart (L36.4) | AC-36.4, AC-36.8 |
| A6 | Device tree | ○ | the window as a device (L36.12) | AC-36.12 |
| A7 | Signals and lifecycle | ○ | a restart resumes the stream (L36.4) | |
| A8 | Secrets as references | ● | a credential that expires mid-stream (L36.5) | AC-36.5 |
| A9 | Cost | ● | requests per epoch, small objects, epochs (L36.6, L36.16) | AC-36.6 |
| A10 | Identity and authorization | ● | reading another group's store at most of its rate (L36.11) | AC-36.11 |
| A11 | Platform defaults and `caps` | ● | `stage_max`, `stream_share`, `stream_margin`, `small_object_max` | AC-36.3, AC-36.6 |
| A12 | Storage | ● | archive classes under a stream (L36.2); the window (L36.12) | AC-36.2, AC-36.12 |
| A13 | Networking | ● | relays on transfer nodes (L36.9) | AC-36.9 |
| A14 | Time and queueing | ○ | rate against GPUs (L36.14) | |
| A15 | Audit, receipts, retention and `replay` | ● | shards read and skipped, the seed and the partition map (L36.8, L36.13, L36.19) | AC-36.19 |
| A16 | Durability | ○ | a stream's position across restarts (L36.4) | |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative verb, declarative owner | a lifecycle policy, in another group's Terraform, archives objects under a running stream | ADR 0026 pins hold images against lifecycle policies | **L36.2** |
| P2 | POSIX semantics that cannot hold | `cat` of 1.4 PB into a pipe | ADR 0001: every pipe is a staged artifact; `\|=` is opt-in | **L36.3** |
| P3 | Orphaned or lost state | the position in the stream at a gang restart | ADR 0004 amendment 1 restarts from the checkpoint | **L36.4** |
| P4 | Identity leak or escalation | a credential that expires 12 hours into a 4-day stream | ADR 0016 decision 4: exchanged per open | **L36.5** |
| P5 | Invisible cost | 160 million GETs over four epochs | L33.16 warns on file counts for a copy | **L36.6** |
| P6 | Lying cache | the window serves frames re-encoded since | ADR 0029 decision 5 verifies staged copies | **L36.7** |
| P7 | Silent failure | the loader skips unreadable shards and the run exits 0 | none | **L36.8** |
| P8 | Partial fleet | one of four relays goes down and half the ranks starve | ADR 0029 decision 1's `reach` names transfer nodes | **L36.9** |
| P9 | Created elsewhere | the loader library's own cache on the root disk | L32.11: stores inside a process count in its bound | **L36.10** |
| P10 | Multi-party action | a stream takes 80 % of another group's store rate | ADR 0016 decision 7 grants by group; nothing about rate | **L36.11** |
| P11 | Data at rest | 7 TB of restricted frames left on node disks | ADR 0029 decision 8 removes staged copies | **L36.12** |
| P12 | Process-tree inheritance | 32 workers each read the whole dataset | ADR 0004 amendment 3 | **L36.13** |

Seven more questions without a probe: is there a streamed input at all (**L36.1**, open); does placement know the rate a stage needs (**L36.14**, rule); does ADR 0029's choice have a third answer (**L36.15**, amend); does Convoy cache a dataset across epochs (**L36.16**, accepted); can a stream's rate be promised (**L36.17**, accepted); how is a manifest of 40 million objects pinned (**L36.18**, rule); and is a streamed run reproducible (**L36.19**, rule).

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| objects in an archive class | plan (L36.2) | 3, or a priced restore step | the objects and their class | restore, or a smaller manifest |
| pipe above `stage_max` | plan (L36.3) | 3 | the size and the `\|=` spelling | `\|=` |
| credential refresh fails | the controller (L36.5) | none; the stream pauses with an event | the device, the expiry | fix the trust; the stream resumes |
| shards unreadable | the stream (L36.8) | 0 degraded at the floor, 1 below it | shards read and skipped per epoch | rerun the epoch, or accept |
| a relay down | the stream (L36.9) | none while others carry it; `starved` below the floor | relays used, rate achieved | none |
| stream slower than the stage needs | `ps` (L36.14) | none | required and achieved rate, accelerator idle share | another computer, a packed form |
| another group's store above `stream_share` | plan (L36.11) | 3 until approved | the share and the owner's approval | `approve` |

## 8. Acceptance criteria

- **AC-36.2** Given a streamed input, then the plan checks the storage class of every object in the manifest; objects in an archive class are restored first as a planned, priced step or the plan is refused; the stream holds a read pin on its manifest for the run, which the store's lifecycle honours where it can, and `explain` warns where it cannot.
- **AC-36.3** Given a pipe whose source read exceeds the platform default `stage_max`, then the plan is refused with exit 3 and the `|=` spelling; `|=` from a device read is a stream over a pinned manifest and produces no artifact.
- **AC-36.4** Given a streamed input and `--checkpoint`, then the stream's position (manifest id, partition map, epoch, shuffle seed, offset per partition) is written with each checkpoint, and a restart resumes exactly there.
- **AC-36.5** Given a stream that outlives a credential, then the controller refreshes it through the group's service identity for the process's lifetime, independent of any session; a refresh that fails pauses the stream and raises an event before the credential expires.
- **AC-36.6** Given a streamed input, then `explain` prices its requests and bytes per epoch and in total; above `small_object_max` objects it warns and offers a packed form (shards by content id, produced once as a staged artifact).
- **AC-36.7** Given a window, then its entries are keyed by the content ids of the pinned manifest and verified; a new manifest invalidates every entry whose id changed.
- **AC-36.8** Given a stream, then shards read and skipped are counted per epoch against the manifest; below the stage's floor it exits 1, at or above it exits 0 degraded; the receipt lists the skipped shards.
- **AC-36.9** Given a computer whose reach goes through relays, then a stream spreads across every relay the computer declares and fails over; below the stage's rate floor the process shows `starved` with the reason.
- **AC-36.10** Given a reader library with its own cache, then the platform toolbox points it at the window device, and writes elsewhere count against the process's ephemeral bound.
- **AC-36.11** Given a stream from a device owned by another group, then `explain` shows its share of the device's declared rate; above `stream_share` it needs the owner's approval, and the owner sees it in `fuser`.
- **AC-36.12** Given a window, then it is an `ephemeral` file system or a `/dev/shm` cache of the process, encrypted when the data is `class=restricted`, never shared across groups, and removed when the process ends.
- **AC-36.13** Given a stream opened by a process tree, then the plan partitions it across the members; each reads its partition; reads outside it are counted; the receipt records the partition map.
- **AC-36.14** Given a streamed input, then `explain` estimates the rate the stage needs (from its annotation or a measured sample) against the store, link and relay rates; placement drops computers below the needed rate times `stream_margin`; `ps` shows the achieved rate and the accelerators' idle share.
- **AC-36.15** Given an input, then the planner chooses `in place`, `staged` or `streamed` and says why; for `streamed`, the capacity check is on the window, not the input.
- **AC-36.18** Given a manifest above `list_max` objects, then it is built from the store's inventory with version ids and ETags, not by listing, and its id is the hash of the manifest.
- **AC-36.19** Given a streamed input, then the shuffle seed and the partition map are pinned and recorded; without them the stage is `deterministic: false`.

## 9. Persona lenses

- **ML**: wants the plan to say whether the GPUs will be fed, a restart that resumes the epoch where it was, and a count of the data the model actually saw.
- **DE**: wants a stream from my store kept to a share of its rate, and archived objects handled before they fail.
- **SEC**: wants restricted frames encrypted on node disks and gone when the run ends.
- **FIN**: wants requests and epochs priced before the run, and a packed form offered when small objects make it expensive.
- **PE**: wants reader caches on the window device, not root disks, and relays used together.
- **SRE**: wants a starved run to say so, not run at 35 % for four days.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L36.1 | none | An input too large to stage, which a computer must read as a moving window, has no design. ADR 0029 chooses between reading in place and staging; ADR 0001 stages every pipe; ADR 0012's follow covers topics only. Nothing describes a stream's manifest, its window, the rate it must sustain, how it is partitioned, where its position is kept, how it crosses a computer's relays, or how much of it was actually read. | **open**. The record asked for: a `streamed` input mode beside `in place` and `staged`; a pinned manifest by version ids from the store's inventory; a window device of the process with a declared size and location; a required rate against the source, link and relay rates; partitioning across a process tree; the stream's position in checkpoints; relays through the computer's `reach`; credentials refreshed for long reads; completeness counted per epoch against the manifest | ADR 0029 d1, d2, d4; ADR 0001; ADR 0012 d2; compiler §8.5 |
| L36.2 | P1 | The data owner's lifecycle policy archived prefixes under a running stream, and reads failed. | **rule**: the plan checks the storage class of every object in a stream's manifest; archived objects are restored first as a planned, priced step, or the plan is refused; the stream holds a read pin on its manifest for the run, which the store's lifecycle honours where it can, and `explain` warns where it cannot | ADR 0026; ADR 0010 a1; ADR 0029 d4 |
| L36.3 | P2 | `cat` of 1.4 PB into a pipe began staging a 1.4 PB artifact. | **rule**: a pipe whose source read exceeds `stage_max` is refused with exit 3 and the `\|=` spelling; `\|=` from a device read is a stream over a pinned manifest and produces no artifact | ADR 0001; ADR 0012 d2; compiler §8.5 |
| L36.4 | P3 | After a gang restart the loader started the epoch again and read 40 % of it twice. | **rule**: with `--checkpoint`, the stream's position (manifest id, partition map, epoch, shuffle seed, offset per partition) is written with each checkpoint, and a restart resumes exactly there | ADR 0004 a1; ADR 0001 |
| L36.5 | P4 | The exchanged credential expired 12 hours into the stream, and every read failed. | **rule**: a stream's credential is refreshed by the controller through the group's service identity for the process's lifetime, independent of any session; a refresh that fails pauses the stream and raises an event before expiry | ADR 0016 d4, d5; ADR 0020 d4; ADR 0027 |
| L36.6 | P5 | Four epochs over 40 million objects were 160 million GETs, priced nowhere. | **rule**: `explain` prices a stream's requests and bytes per epoch and in total; above `small_object_max` objects it warns and offers a packed form (shards by content id, produced once as a staged artifact) | ADR 0010; ADR 0029 rules (L33.16) |
| L36.7 | P6 | The window served frames that had been re-encoded since. | **rule**: window entries are keyed by the content ids of the pinned manifest and verified; a new manifest invalidates every entry whose id changed | ADR 0001; ADR 0029 d5 |
| L36.8 | P7 | The loader skipped unreadable shards, the model saw 91 % of the data, and the run exited 0. | **rule**: a stream counts shards read and skipped per epoch against the manifest; below the stage's floor it exits 1, at or above it 0 degraded; the receipt lists the skipped shards | ADR 0004 a2; ADR 0002 |
| L36.9 | P8 | One of four relays went down, and half the ranks starved while the rest waited. | **rule**: a stream through a computer's relays spreads across every relay the computer declares in its `reach` and fails over; below the stage's rate floor the process shows `starved` with the reason | ADR 0029 d1; ADR 0025 d8 |
| L36.10 | P9 | The loader library's own cache filled the nodes' root disks. | **rule**: the platform toolbox points reader libraries' caches at the process's window device; writes elsewhere count against its ephemeral bound | ADR 0028 d9; ADR 0029 d1 |
| L36.11 | P10 | The stream took 80 % of another group's store rate, and their production readers were throttled. | **rule**: `explain` shows a stream's share of the source device's declared rate; above `stream_share` a stream from another group's device needs the owner's approval, and the owner sees it in `fuser` | ADR 0017 d2; ADR 0016 d7 |
| L36.12 | P11 | 7 TB of restricted frames stayed on the NVMe of 18 nodes after the run. | **rule**: a window is an `ephemeral` file system or a `/dev/shm` cache of the process, encrypted when the data is `class=restricted`, never shared across groups, and removed when the process ends | ADR 0029 d6, d8; ADR 0028 d9 |
| L36.13 | P12 | Each of 32 loader workers read the whole dataset. | **rule**: a stream opened by a process tree is partitioned across its members by the plan; each reads its partition; reads outside it are counted; the receipt records the partition map | ADR 0004 a3; ADR 0003 |
| L36.14 | none | GPUs ran at 35 % for four days because the stream was slower than they were, and nothing said so. | **rule**: `explain` estimates the rate a streamed stage needs (from its annotation or a measured sample) against the store, link and relay rates; placement drops computers below that rate times `stream_margin`; `ps` shows the achieved rate and the accelerators' idle share | ADR 0025 d2, d4; ADR 0023 |
| L36.15 | none | On a computer that could neither reach the dataset nor hold it, the plan had no third choice. | **amend** ADR 0029 decision 2: the planner chooses `in place`, `staged` or `streamed` for each input and says why; for `streamed` the capacity check is on the window, not the input, and the rate check of L36.14 applies | ADR 0029 d2 |
| L36.16 | none | The same 1.4 PB was paid for four times, once per epoch. | **accepted**: a stream is not cached beyond its window; each epoch reads again and is priced again; where an input fits on a reachable file system, staging it is the planner's other choice | ADR 0029 d2 |
| L36.17 | none | The same run read at 9 GB/s one day and 3 GB/s the next. | **accepted**: the rate of a shared store, link or relay is not promised; estimates come from earlier receipts and are marked as such | ADR 0025 d2 |
| L36.18 | none | Building the manifest listed 40 million objects at plan time, for three hours. | **rule**: a manifest above `list_max` objects is built from the store's inventory with version ids and ETags, not by listing, and its id is the hash of the manifest; content ids of objects are the store's, not recomputed | ADR 0001; ADR 0029 d4; command map `du` |
| L36.19 | none | A second run with the same configuration was not comparable, because its shuffle seed was never recorded. | **rule**: a stream's shuffle seed and partition map are pinned and recorded; without them the stage is `deterministic: false` (ADR 0001 amendment 1) | ADR 0001 a1 |
