# US-33 · The data has to be there first: staging a genomics pipeline into a university's scratch, a linked Lustre file system and a function's disk, and back

| Field | Value |
|---|---|
| Primary persona | DE |
| Secondary personas | SEC, FIN, PE, DS, SRE |
| Workload class | data staging: a pipeline whose stages run on computers that read from their own file systems, so the data is copied in before a stage and out after it; 38 TB of patient sequencing reads under `residency=br`, a university Slurm cluster with a purged Lustre scratch and a shared project quota, AWS Batch with an FSx for Lustre file system linked to S3, Lambda with its ephemeral disk, and the on-premises HDFS of walkthrough 2 |
| Walkthrough | written against [ADR 0025](../adr/0025-a-computer-is-a-device-kubernetes-is-one-driver.md) decisions 2, 4, 7 and 9 and its open item on staging, [ADR 0001](../adr/0001-pipe-is-a-staged-artifact.md) and its amendment 2, [ADR 0003](../adr/0003-three-backends-local-first.md) amendment 1, [ADR 0004](../adr/0004-completion-contract-and-exit-codes.md) and its amendment 3, [ADR 0006](../adr/0006-detach-is-not-destroy.md), [ADR 0011](../adr/0011-device-tree-and-mount-table.md) decision 9 and amendments 1, 3 and 4, [ADR 0015](../adr/0015-tables-and-catalogs-are-devices.md), [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md) decisions 3, 4 and 7, [ADR 0017](../adr/0017-approvals-are-objects-on-the-process.md) decision 2, [ADR 0024](../adr/0024-a-device-is-installed-by-terraform.md) decisions 1 and 8, walkthrough 2, and the command map rows for `cp`, `mv`, `du` and `ls`; it follows US-32 |
| Gaps already closed there | none |
| Milestones needed | M3, M4, M5, M8, M9 |
| Findings in this document | L33.1 – L33.19 |

## 1. Story

**As a** data engineer, **I want** a stage that runs on a computer with its own file system to get its inputs there and its outputs back as part of the plan, priced, verified and cleaned up, **so that** 38 TB of patient reads do not cross a university link by hand with nothing saying how long it takes or what it costs, a job that "completed" has really delivered its results, the reads do not stay readable on someone else's scratch for three weeks, and a file system made for one run does not keep billing after it.

**Context.** After US-32 the platform runs ADRs 0001 to 0027. The `genomics` group sequences patients for a hospital network. The raw reads are 412 whole-genome samples, 38 TB in 2.1 million files, in `s3://acme-genomics-br` in sa-east-1, mounted as `/s3/genomics-br` with `residency=br`. The pipeline has four stages:
- **align** (per sample) and **markdup** (per sample) run on `hpc-sp`, the São Paulo university Slurm cluster of US-29. Its compute nodes reach the outside only through the university's data transfer nodes, and it has two Lustre file systems: `/scratch`, where files not accessed for 21 days are purged, and `/project`, where the lab's allocation has a 60 TB quota shared with its students.
- **call** (variant calling, per sample) runs on `batch-awsbatch-123456789012-sa-east-1` with an FSx for Lustre file system linked to an S3 prefix. A file there is loaded from S3 on its first read, and new files are exported back to S3 asynchronously.
- **qc** (per sample, small) runs on `fn-lambda-123456789012-sa-east-1`, whose ephemeral `/tmp` is at most 10 GB.
- **load** writes `/tables/genomics/variants`, an Iceberg table.

The `analytics` team still runs walkthrough 2 on `k8s-capi-dc-sp-01`, copying from S3 into HDFS before every Spark job.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs the pipeline | `mei` | user `mei`, group `genomics` | |
| Reads a shared reference | `noor` | user `noor`, group `research` | also runs on `hpc-sp` |
| Owns the allocation on `hpc-sp` | the lab's principal investigator | none; the university's accounts | quota shared with students outside Convoy |
| Operates `hpc-sp` | the university's HPC team | none | owns the purge and the transfer nodes |
| Owns residency | `amara` | user `amara`, group `security` | patient data |
| Pays | FinOps | group `finops` | |
| Platform | platform team | `svc:platform` | installs devices and drivers |

## 3. Preconditions

- **Capabilities** (`caps`): on `hpc-sp`: `slurm:23.11`, `identity=shared` (L29.5), `operator=external`, file systems `/scratch` (Lustre, purge after 21 days without access) and `/project` (Lustre, a 60 TB quota on the allocation); on the Batch computer: `batch:aws`, `fsx-lustre`; on the Lambda computer: `fn:lambda`, ephemeral disk 10 GB.
- **Packages**: the Slurm, Batch and function drivers of ADR 0025; `hdfs` and `spark-operator` on `dc-sp-01`.
- **Platform defaults** `explain` must print: for every stage, where its inputs are read from, what is copied where before and after it, the size, file count, time and price of each copy, the file system that holds the copy and how long it stays there.

## 4. Main path

```bash
# Monday 08:00 · mei, group genomics
$ explain ./pipeline.sh
stage 1 align    placement: hpc-sp            412 partitions   9 200 core-h (allocation)
stage 2 markdup  placement: hpc-sp            412 partitions   3 100 core-h (allocation)
stage 3 call     placement: awsbatch sa-east-1 412 partitions  band $1 200 – 1 450
stage 4 qc       placement: lambda sa-east-1   412 invocations band $8
stage 5 load     /tables/genomics/variants (Iceberg)
#  nothing says how 38 TB reaches hpc-sp, how long it takes, what it costs, or where it
#  stays afterwards; the computer contract has no field for its file systems              (L33.1)

# 08:10 · the align tasks read s3:// directly; the compute nodes have no route out; 412 fail (L33.1)
$ cp /s3/genomics-br/raw/ /hpc-sp/scratch/genomics/raw/
#  2.1 million files through one transfer node; the university's metadata server slows for
#  everyone; nothing printed the file count or the request cost                           (L33.16)
#  priced nowhere: 38 TB out of sa-east-1                                                  (L33.4)
#  the link drops at 61 % on Monday night; the copy starts again from nothing             (L33.9)
#  one week the copy takes 31 hours, the next 9                                            (L33.17)

# Tuesday · the copy lands in /scratch, owned by the cluster's one service account, mode 0644
#  research's jobs on hpc-sp, under the same account, can read patient reads             (L33.8)
#  mei moves the copy to /project to escape the purge; 38 TB of a 60 TB quota;
#  two students' jobs fail with EDQUOT; the investigator was never asked                   (L33.12)

# Wednesday · align ends; between align and markdup, both on hpc-sp, the artifact contract
#  copies 31 TB of BAM to the topology's store in sa-east-1 and back again              (L33.13)
#  markdup's Slurm jobs end COMPLETED; the copy of their output to S3 fails at 80 %;
#  the stage exits 0                                                                        (L33.5)
#  a postdoc had scp'd their own files into /hpc-sp/scratch/genomics/out; the
#  stage-out ships them to the patients' bucket                                            (L33.10)
$ mv /hpc-sp/scratch/genomics/bam/ /s3/genomics-br/bam/
#  each source file is deleted when its copy returns; 212 multipart uploads never
#  completed, and those BAMs are now in neither place                                       (L33.6)

# Thursday · call on Batch; mei's job definition creates fsx-genomics through the AWS API,
#  linked to s3://acme-genomics-br/bam/                                                     (L33.11)
$ ls -l /batch/fsx-genomics/bam/ | head -3; du -sh /batch/fsx-genomics/bam/
#  31 TB of files that look local; none is loaded until it is read, so the first read of
#  each is slow and the estimate was made as if they were local                            (L33.19)
#  six samples are re-aligned and written to the prefix while call runs; some tasks read
#  the old BAM, some the new                                                                 (L33.18)
#  call writes VCFs to the file system; export to S3 is asynchronous; the process exits 0,
#  load starts and commits 388 of 412 samples                                               (L33.5)
#  qc: one sample's input is 14 GB; /tmp holds 10 GB; ENOSPC after 9 minutes             (L33.14)

# The next week
$ kill 9233          # a rerun mei abandons
#  31 TB stay on /scratch, and fsx-genomics stays up: 48 TiB of SSD Lustre billed for
#  three weeks with no process using it                                                     (L33.4, L33.7)
#  the patients' reads stay on the university's scratch until the purge, 21 days later  (L33.3)

# Three weeks later · a rerun of 40 samples
#  the receipt says the inputs were staged to /scratch; the plan skips the copy; the purge
#  removed them; align fails 40 minutes in; a student suggests a cron that touches the files (L33.2)

# Meanwhile on dc-sp-01, walkthrough 2 every day
$ cp /s3/raw/2026-11/ /hdfs/staging/2026-11/
#  1.8 TiB copied into HDFS for a Spark job that could read S3 directly                    (L33.15)
```

| Step | Convoy object | Computer's object | Where the state lives |
|---|---|---|---|
| a stage-in to `hpc-sp` | a `cp` by hand | a transfer through the university's transfer nodes | the transfer service |
| a staged copy | none | files on `/scratch` or `/project` | the university's Lustre |
| the purge | none | the HPC team's policy | the university |
| a stage-out | none, or `mv` by hand | multipart uploads to S3 | S3 |
| a linked file system | none | an FSx file system and its data repository association | AWS |
| an export | none | the file system's export to S3 | AWS, asynchronous |
| a function's disk | none | `/tmp` of the invocation | the function's sandbox |
| HDFS staging | a `cp` | files in HDFS | the datanodes |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe: staged artifact, content id, cache | ● | intermediate data between two stages on one computer (L33.13); a staged copy as a cache entry (L33.2) | AC-33.2, AC-33.13 |
| A2 | Command: class, annotation, executable fields | ○ | `mv` across devices (L33.6) | AC-33.6 |
| A3 | Session: mount table, env, cwd, attach and detach | – | | |
| A4 | Backend selection | ● | a stage whose input does not fit the computer's disk (L33.14); reading in place or staging (L33.15) | AC-33.14, AC-33.15 |
| A5 | Completion contract | ● | a stage that exits 0 before its outputs are delivered (L33.5) | AC-33.5 |
| A6 | Device tree | ● | a computer's own file systems (L33.1); files that look local and are not (L33.19) | AC-33.19 |
| A7 | Signals and lifecycle | ○ | `kill` leaves staged copies and a file system behind (L33.7) | AC-33.7 |
| A8 | Secrets as references | – | | |
| A9 | Cost | ● | egress, requests, and a file system billed while idle (L33.4, L33.16) | AC-33.4, AC-33.16 |
| A10 | Identity and authorization | ● | staged copies under a shared account (L33.8); a quota shared with people outside Convoy (L33.12) | AC-33.8, AC-33.12 |
| A11 | Platform defaults and `caps` | ● | `stage_keep`, `stage_share_max`, file systems in the computer contract (L33.1, L33.7, L33.12) | AC-33.7 |
| A12 | Storage | ● | scratch, project space, a linked file system, a function's disk, HDFS | AC-33.2, AC-33.19 |
| A13 | Networking | ● | transfer nodes, a link that drops, egress (L33.9) | AC-33.9 |
| A14 | Time and queueing | ○ | stage-in time on a shared link (L33.17) | |
| A15 | Audit, receipts, retention and `replay` | ● | the snapshot a stage read (L33.18); where patient data stayed and for how long (L33.3) | AC-33.3, AC-33.18 |
| A16 | Durability | ● | a purge (L33.2); a move that loses data (L33.6) | AC-33.2, AC-33.6 |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative verb, declarative owner | a file system created per run through the provider's API | ADR 0024 decision 1: devices are installed by a module | **L33.11** |
| P2 | POSIX semantics that cannot hold | `mv` from a file system to an object store | the command map warns that the S3 case is not atomic | **L33.6** |
| P3 | Orphaned or lost state | staged copies and a file system after `kill` | ADR 0006 keeps what is detached; nothing about staging | **L33.7** |
| P4 | Identity leak or escalation | staged patient reads under a cluster's one service account | L29.5: a computer declares `identity=` | **L33.8** |
| P5 | Invisible cost | egress of 38 TB; a file system billed while idle | the command map prices `cp` across stores; nothing prices a stage's staging | **L33.4** |
| P6 | Lying cache | a receipt says the inputs are staged; the purge removed them | ADR 0001 content ids; nothing about a copy someone else deletes | **L33.2** |
| P7 | Silent failure | a stage exits 0 with its outputs not delivered | ADR 0025 decision 2 maps backend outcomes to exit codes; the copy is not an outcome | **L33.5** |
| P8 | Partial fleet | a transfer link that drops at 61 % | ADR 0011 amendment 4 makes disconnection a state | **L33.9** |
| P9 | Created elsewhere | files someone copied into the output directory by hand | ADR 0016 decision 3: objects record their actor | **L33.10** |
| P10 | Multi-party action | a quota shared with students and owned by an investigator outside Convoy | ADR 0025 decision 9 settles allocations; nothing about quotas | **L33.12** |
| P11 | Data at rest | patient reads on a university's scratch for 21 days | ADR 0011 amendment 3 binds residency to mounts, not to copies on a computer | **L33.3** |
| P12 | Process-tree inheritance | two stages of one tree on one computer, joined through the artifact store | ADR 0001 amendment 2: one store per topology | **L33.13** |

Seven more questions without a probe: does a computer's contract say what file systems it has at all (**L33.1**, open); does placement check that a stage's input fits a computer's disk (**L33.14**, rule); when should a stage read in place rather than copy (**L33.15**, rule); does `explain` warn of millions of small files (**L33.16**, rule); can the time of a copy over a shared link be promised (**L33.17**, accepted); which version of a source does a stage read when the source changes during it (**L33.18**, rule); and do files that are loaded on first read look local (**L33.19**, rule).

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| inputs do not fit the computer's disk | plan (L33.14) | 3 | the input size and the file system's free size | another computer, or a larger file system |
| staged copy purged or changed | the stage-in check (L33.2) | none; restaged | the copy, the content id, the reason | none |
| transfer interrupted | the transfer (L33.9) | none while resumable; 1 past the deadline | files done and left, by content id | none; it resumes |
| stage-out incomplete | the stage (L33.5) | 1 | outputs verified at the destination, and missing | rerun the stage-out |
| move fails part-way | `mv` (L33.6) | 1 | files moved, files left at the source | `mv` again |
| foreign files in an output directory | the stage-out (L33.10) | none | the files not shipped and their owners | none |
| process ends with copies staged | the computer driver (L33.7) | none | copies, sizes, removal time | `hold` to keep them |
| shared quota above the limit | plan (L33.12) | 3 until approved | the quota, its owner, the approval asked for | `approve` |

## 8. Acceptance criteria

- **AC-33.2** Given a staged copy, then it is a cache entry keyed by content id on the target file system, verified against its checksums before a stage uses it; a copy that is missing, changed or past the file system's declared purge is staged again, never trusted; Convoy never touches files to avoid a purge.
- **AC-33.3** Given data with a residency, then every file system a stage copies it to carries a residency and is checked against it, and staged copies are removed when the process tree ends unless a hold covers them; the receipt records each copy, where it was and when it was removed.
- **AC-33.4** Given a stage that needs staging, then `explain` prices each copy (egress, requests, and the target file system's capacity-hours) inside the stage's band; a file system created for a process is part of the process and removed with it, and one with no process using it is shown in `/sys` with its daily price.
- **AC-33.5** Given a stage with outputs on a computer's own file system, then the stage succeeds only when every output is committed and verified by content id at its destination; an asynchronous export is part of the stage, and a stage whose outputs are not all delivered ends with exit 1, listing what is missing.
- **AC-33.6** Given `mv` across devices, then it copies everything, verifies it at the destination, and only then deletes the sources; a failure part-way deletes nothing and lists what was copied.
- **AC-33.7** Given a process that ends in any way (exit, `kill`, deadline, unreachable computer), then its staged copies and the file systems made for it are removed after the platform default `stage_keep` unless held, and `/sys/proc/<pid>/staged` lists them until then.
- **AC-33.8** Given a computer with `identity=shared`, then staged data goes only into a per-group directory that the computer enforces; a mount marked `class=restricted` is never staged to a computer that cannot enforce one.
- **AC-33.9** Given a transfer, then it is per file by content id and resumable; a stage starts only when its whole manifest is verified at the target, and `ps` shows the process as `staging` with the files done and left.
- **AC-33.10** Given a stage-out, then it ships only files the process wrote, in a directory made for the process; other files found there are listed with their owners and not shipped.
- **AC-33.11** Given a file system that outlives one process, then it is a device installed by a module; a file system for one process is made by the computer's driver through the mechanism the computer declares, never by Convoy calling the provider directly.
- **AC-33.12** Given a file system whose quota is shared with people outside Convoy, then the computer declares it, `explain` prints the quota left, and staging above the platform default `stage_share_max` of it needs an approval from the group that owns the allocation.
- **AC-33.13** Given two stages of one process tree on the same computer, then the intermediate data stays on that computer's file system by content id; only the tree's declared outputs and what the contract needs (checkpoints, receipts) go to the topology's store.
- **AC-33.14** Given a stage, then placement compares its staged input and expected output with each computer's file system capacity per task, and a computer that cannot hold them is removed before ranking.
- **AC-33.15** Given a stage whose input the computer can reach, then it reads in place unless the plan shows staging is cheaper or faster for the reads the stage will make; `explain` prints the choice and why.
- **AC-33.16** Given a copy, then `explain` prints its file count, the request cost, and the target file system's declared metadata limit, and warns above it.
- **AC-33.18** Given a stage-in, then it reads a pinned snapshot of the source (object versions, a table snapshot), and the receipt records it; a change to the source during the process does not reach it.
- **AC-33.19** Given a file system linked to an object store, then `ls -l` and `/sys` show each file's state (`stub`, `loaded`, `modified`, `exported`), `du` reports loaded and total bytes, and `explain` counts unloaded bytes as a stage-in.

## 9. Persona lenses

- **DE**: wants staging to be part of the plan, with a time, a price and a verified result, and wants to stop copying data that can be read where it is.
- **SEC**: wants patient reads to go only where residency and group isolation hold, and to be gone when the run is.
- **FIN**: wants egress, requests and a file system's idle days on the receipt of the process that caused them.
- **PE**: wants file systems in the computer contract, made by the driver or a module, never by a job definition.
- **DS**: wants a shared reference genome staged once and reused, not copied by every run.
- **SRE**: wants a transfer that resumes, and a stage that fails when its outputs did not arrive.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L33.1 | none | A computer's own file systems are not in the design. ADR 0025 decision 2's contract has no field for them, placement reads data locality only by topology, and moving inputs to a computer before a stage and outputs back after it is left to the user. Nothing says what a computer can reach directly, what it must stage, or what a staged copy is. | **open** (drafted 2026-11-14 in [ADR 0029](../adr/0029-a-computers-file-systems-are-in-its-contract-staging-is-a-planned-copy.md): every computer reports its file systems and what its tasks can reach in `/sys/compute/<id>/fs/`; the planner reads in place or stages and says why; stage-in and stage-out are steps of the stage compiled per kind, from a pinned source by content id and resumable; staged copies are verified cache entries that keep their residency and group; a stage succeeds only when its outputs arrive; what staging creates ends with the process; staging is priced and shared quotas are asked for). The record asked for: a `fs` field in the computer contract (each file system's kind, capacity per task, quota and who shares it, purge policy, throughput and metadata limits, residency, and whether it is linked to an object store), staging as planned steps of a stage (stage-in and stage-out) compiled per kind (a transfer service, a burst buffer, a linked file system, a function's download), staged copies keyed by content id and reused, and placement that chooses between reading in place and staging | ADR 0025 d2, d4; ADR 0001 a2; ADR 0003 a1; ADR 0011 a1 |
| L33.2 | P6 | A receipt said the inputs were staged; the university's purge had removed them, and the rerun failed 40 minutes in. | **rule** (applied: ADR 0029 rules; ADR 0001 rules): a staged copy is a cache entry keyed by content id, verified against its checksums before use; a copy that is missing, changed or past the file system's declared purge is staged again; Convoy never touches files to avoid a purge | ADR 0001; ADR 0025 d2 |
| L33.3 | P11 | Patient reads stayed on a university scratch for 21 days after the run. | **rule** (applied: ADR 0029 rules; ADR 0011 rules; ADR 0006 rules; ADR 0009 rules): every file system a stage copies to carries a residency and is checked like a mount (ADR 0011 amendment 3); staged copies are removed when the process tree ends unless held, and the receipt records each copy, its place and its removal | ADR 0011 a3; ADR 0006; ADR 0009 |
| L33.4 | P5 | 38 TB of egress was priced nowhere, and a 48 TiB file system billed for three weeks with no process using it. | **rule** (applied: ADR 0029 rules; ADR 0010 rules; ADR 0025 rules; ADR 0009 rules): `explain` prices each staging copy (egress, requests, the target's capacity-hours) inside the stage's band; a file system made for a process is part of it and is removed with it; one no process uses is shown in `/sys` with its daily price | ADR 0010; ADR 0025 d9; ADR 0009 |
| L33.5 | P7 | Slurm jobs ended COMPLETED with their copy to S3 failed at 80 %, and an asynchronous export let load commit 388 of 412 samples, both with exit 0. | **rule** (applied: ADR 0029 rules; ADR 0004 rules): a stage with outputs on a computer's own file system succeeds only when every output is committed and verified by content id at its destination; an asynchronous export is part of the stage; otherwise exit 1 with the missing outputs listed | ADR 0004; ADR 0025 d2 |
| L33.6 | P2 | `mv` to an object store deleted each source as its copy returned, and 212 BAMs whose uploads never completed were lost. | **rule** (applied: ADR 0029 rules; ADR 0015 rules; command map): `mv` across devices copies everything, verifies it at the destination, and only then deletes the sources; a failure part-way deletes nothing and lists what was copied | command map `mv`; ADR 0015 |
| L33.7 | P3 | A killed run left 31 TB on scratch and a file system running. | **rule** (applied: ADR 0029 rules; ADR 0006 rules; ADR 0004 rules): when a process ends in any way, its staged copies and the file systems made for it are removed after the platform default `stage_keep` unless held; `/sys/proc/<pid>/staged` lists them until then | ADR 0006; ADR 0004 a3 |
| L33.8 | P4 | Staged patient reads under the cluster's one service account were readable by another group's jobs. | **rule** (applied: ADR 0029 rules; ADR 0016 rules): on a computer with `identity=shared`, staged data goes only into a per-group directory the computer enforces; a mount marked `class=restricted` is never staged to a computer that cannot enforce one | ADR 0016 d4; ADR 0016 rules (L29.5); ADR 0025 d7 |
| L33.9 | P8 | The transfer link dropped at 61 % and the copy started again from nothing, and align started on whatever had arrived. | **rule** (applied: ADR 0029 rules; ADR 0011 rules; ADR 0001 rules; command map): transfers are per file by content id and resumable; a stage starts only when its whole manifest is verified at the target; `ps` shows `staging` with files done and left | ADR 0011 a4; ADR 0001; command map `cp` (L27.5) |
| L33.10 | P9 | Files a postdoc had copied into the output directory were shipped to the patients' bucket. | **rule** (applied: ADR 0029 rules; ADR 0016 rules; ADR 0001 rules): a stage-out ships only files the process wrote, in a directory made for the process; other files there are listed with their owners and not shipped | ADR 0016 d3; ADR 0001 |
| L33.11 | P1 | A job definition created a file system through the provider's API, outside any module. | **rule** (applied: ADR 0029 rules; ADR 0024 rules; ADR 0010 rules): a file system that outlives one process is a device installed by a module (ADR 0024); one for a single process is made by the computer's driver through the mechanism the computer declares, never by Convoy calling the provider | ADR 0024 d1, d8; ADR 0010 a1 |
| L33.12 | P10 | Staging took 38 TB of a 60 TB quota shared with students and owned by an investigator outside Convoy. | **rule** (applied: ADR 0029 rules; ADR 0017 rules; ADR 0025 rules): a computer declares which file systems have a quota shared outside Convoy; `explain` prints what is left; staging above `stage_share_max` of it needs an approval from the group that owns the allocation | ADR 0017 d2; ADR 0025 d9; ADR 0016 d7 |
| L33.13 | P12 | Two stages of one tree on the same computer were joined through the topology's store, moving 31 TB out and back. | **amend** (applied: ADR 0001 a2 revised) ADR 0001 amendment 2: a computer's own file system may hold the artifacts passed between stages of one process tree on that computer, keyed by content id and bounded by its purge; only the tree's declared outputs and what the contract needs (checkpoints, receipts) go to the topology's store | ADR 0001 a2; ADR 0004 a3 |
| L33.14 | none | A 14 GB input was sent to a function whose disk holds 10 GB, and failed after 9 minutes. | **rule** (applied: ADR 0029 rules; ADR 0025 rules; ADR 0003 rules): placement compares a stage's staged input and expected output with each computer's file system capacity per task and removes those that cannot hold them before ranking | ADR 0025 d2, d4; ADR 0003 a1 |
| L33.15 | none | Walkthrough 2 copies 1.8 TiB from S3 into HDFS every day for a job that can read S3. | **rule** (applied: ADR 0029 rules; ADR 0003 rules; walkthrough 2): a stage reads its input in place when the computer can reach it, unless the plan shows staging is cheaper or faster for the reads the stage will make; `explain` prints the choice and why | ADR 0003 a1; walkthrough 2 |
| L33.16 | none | 2.1 million small files slowed the university's metadata server, with no warning and no request cost shown. | **rule** (applied: ADR 0029 rules; ADR 0010 rules; ADR 0025 rules; command map): `explain` of a copy prints its file count, its request cost, and the target file system's declared metadata limit, and warns above it | ADR 0010; ADR 0025 d2 |
| L33.17 | none | The same stage-in took 31 hours one week and 9 the next. | **accepted**: a copy over a link or a file system shared with others has no promised time; `explain` gives the estimate from earlier receipts and marks it as such | ADR 0025 d2 |
| L33.18 | none | Six BAMs changed in the source while call ran, and tasks read different versions. | **rule** (applied: ADR 0029 rules; ADR 0001 rules; ADR 0015 rules): a stage-in reads a pinned snapshot of the source (object versions, a table snapshot), recorded in the receipt; changes to the source during the process do not reach it | ADR 0001; ADR 0015 |
| L33.19 | none | Files on a linked file system looked local and were loaded on first read, so the estimate was wrong. | **rule** (applied: ADR 0029 rules; ADR 0011 rules; command map): on a file system linked to an object store, `ls -l` and `/sys` show each file's state (`stub`, `loaded`, `modified`, `exported`), `du` reports loaded and total bytes, and `explain` counts unloaded bytes as a stage-in | ADR 0011 d9; command map `ls`, `du` |
