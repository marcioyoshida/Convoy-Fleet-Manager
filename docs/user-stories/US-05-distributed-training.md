# US-05 · Train on 64 GPUs for 36 hours and survive a spot reclaim

| Field | Value |
|---|---|
| Primary persona | ML (ML Engineer) |
| Secondary personas | PE, FIN, SRE |
| Workload class | coordinated group, gang |
| Walkthrough | [walkthroughs.md §5](../walkthroughs.md) |
| Gaps already closed there | G2, G4, G5 |
| Milestones needed | M1, M2, M3, M6, M10 |
| Findings in this document | L5.1 – L5.6 |

## 1. Story

**As an** ML engineer, **I want** to launch torchrun on eight nodes with eight GPUs each, with a checkpoint cadence, a deadline and a budget, **so that** a preempted node costs ten minutes rather than a day and I can freeze, inspect and resume the whole run like one process.

**Context.** A run costs several thousand dollars and spot capacity is a third of the price of on-demand. Today the team uses a PyTorchJob CRD and a cron that watches for stuck ranks.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | the ML engineer | personal identity on `/clusters/ml-training`, `ns=llm` | detached for 36 hours |
| Approves | none; the budget is the gate | | |
| Pays | team llm | | the largest single receipt in the fleet |
| Audits | FinOps, weekly | preemption overhead versus band | L5.3 |
| Operates the platform | platform team | Kueue, JobSet, node agents, spot node groups, checkpoint store | decides where checkpoints may live (L5.1) |

## 3. Preconditions

- **Capabilities**: `jobset`, `kueue`, `gpu`, `node-agent`, `objstore` for checkpoints, `opencost`.
- **Packages**: `torchrun` with its annotation (`group torchrun, gang required, checkpoint_seconds 600, freeze_max_seconds` L5.2).
- **Mounts**: `/ckpt` on a zone-independent device; `/data` read-only.
- **Annotations**: as above, plus `member_signals: none` (US-02 L2.3).
- **Platform defaults** `explain` must print: spot versus on-demand placement, preemption rate used in the band, checkpoint device requirement, the collective timeout the freeze bound derives from.

## 4. Main path

```bash
$ batch --replicas=8 --gpu=8 --disk=per-replica:2Ti --gang --checkpoint=/ckpt \
    --deadline=36h --budget=$9000 torchrun --nnodes=8 --nproc-per-node=8 … train.py
$ top                                   # GPU-UTIL, COST/h, step, last checkpoint
$ kill -STOP --group 8120 && kill -CONT --group 8120
(spot reclaim on replica 5 → gang restart from the last checkpoint)
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `batch --gang` | process 8120, group | JobSet, one replicated Job × 8, headless Service, `startupPolicy` | JobSet status, receipt |
| `--disk=per-replica` | 8 volumes | 8 PVCs named by index, pre-created (L5.6) | PVCs |
| `--checkpoint=/ckpt` | checkpoint path | object-store mount in every pod | object store |
| `kill -STOP --group` | group freeze | node agents, Lease barrier | node agents |
| preemption | stop with checkpoint, group restart | `DisruptionTarget` ignored; JobSet restart-all | checkpoint |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ○ | the checkpoint is not an artifact; the final weights are | the final artifact id appears in the receipt |
| A2 | Command | ● | the richest executable annotation in the seed | `type torchrun` prints gang, group, checkpoint cadence, freeze bound |
| A3 | Session | ● | detached for 36 hours; `top` on reattach | the process is unaffected by session restarts |
| A4 | Backend | ● | JobSet (G5) | `explain` says JobSet and why |
| A5 | Contract | ● | gang (G4), retries as group restarts, deadline and budget | a preemption restarts the group and does not burn retries |
| A6 | Device tree | ● | `/ckpt` device must be zone-independent; `/sys` preemption rate | the planner refuses a zonal checkpoint path (L5.1) |
| A7 | Signals | ● | group freeze with a bound (L5.2); member signals refused | freezing past the bound converts to stop-with-checkpoint |
| A8 | Secrets | – | none (dataset is internal) | |
| A9 | Cost | ● | COST/h, band includes preemption overhead (L5.3) | receipt separates productive time from restart loss |
| A10 | Identity | ○ | team service account | |
| A11 | Defaults | ● | spot placement, checkpoint device rule, freeze bound | all in `explain` |
| A12 | Storage | ● | per-replica PVCs across a gang restart | replica 5's new pod reattaches PVC 5 (L5.6) |
| A13 | Networking | ○ | headless Service for the rendezvous | `$CONVOY_PID-0` resolves before rank 0's peers start |
| A14 | Time | ● | queue wait versus deadline (L5.5) | `explain` prints what the deadline counts from |
| A15 | Audit | ○ | receipt with restart events | each restart is a line with the cause |
| A16 | Durability | ● | the whole story is about surviving loss | AC-5.2 |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | Convoy owns the JobSet | no contention | ok |
| P2 | POSIX cannot hold | `kill -STOP` on a collective | ADR 0008 amendment 1 freezes all members; but for how long? | **L5.2** |
| P3 | Orphaned state | checkpoints on a per-replica zonal disk after a cross-zone restart | unreachable from the new zone | **L5.1** |
| P4 | Identity leak | none | | ok |
| P5 | Invisible cost | the nine minutes lost per preemption | billed, not in the band | **L5.3** |
| P6 | Lying cache | training is never cached | executable | ok |
| P7 | Silent failure | `--disk=per-replica` on a JobSet: no `volumeClaimTemplates` exist on Jobs | the flag could emit nothing | **L5.6** |
| P8 | Partial fleet | one cluster | n/a | ok |
| P9 | Created elsewhere | none | | ok |
| P10 | Multi-party | none | | ok |
| P11 | Data at rest | checkpoints and final weights in the object store | retention class | ok |
| P12 | Inheritance | deadline across pause and re-queue | ADR 0004 does not say what the deadline counts from | **L5.5** |

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| spot reclaim on one replica | `DisruptionTarget` | continues | restart from step N, minutes lost | automatic |
| rank 3 OOM | container exit | group restart, counted | retry 1 of N, rank 3 | automatic, then exit 1 after `--retries` |
| budget hit | cost watcher | 2 | paused after checkpoint, GPUs released | `kill -CONT 8120` re-queues the gang |
| freeze past the collective timeout | router bound (L5.2) | continues | converted to stop-with-checkpoint | `kill -CONT` |
| checkpoint path zonal | plan time | 3 | reason, allowed devices | change `--checkpoint` |
| 64 GPUs unavailable | Kueue | queued | holder, expected free time | wait, `nice` |

## 8. Acceptance criteria

- **AC-5.1** Given `--gang` with `--replicas=8`, when Kueue admits, then all eight pods are admitted together or none, and `ps` shows `queued` with the holder meanwhile.
- **AC-5.2** Given a preemption of one replica, then every replica restarts from the latest checkpoint within one `checkpoint_seconds` of loss, the retry counter is unchanged, and the receipt records the lost minutes.
- **AC-5.3** Given `kill -STOP --group`, then all 64 ranks are frozen within one barrier period; given the freeze reaches `freeze_max_seconds`, then the group is checkpointed and stopped instead of resumed into a dead collective.
- **AC-5.4** Given `--checkpoint` on a zonal per-replica disk, then the plan is refused with the allowed device types.
- **AC-5.5** Given `--disk=per-replica:2Ti`, then eight PVCs named by index exist before the first pod starts and replica 5's restarted pod mounts PVC 5.
- **AC-5.6** Given `--deadline=36h`, then `explain` states whether queue time counts, and the receipt shows submitted, started and finished timestamps.

## 9. Persona lenses

- **ML**: wants to think in ranks and checkpoints, not pods; refuses if a freeze can kill the run on resume.
- **PE**: owns the checkpoint device policy and the spot node groups; wants the freeze bound derived from the package, not guessed.
- **FIN**: wants preemption loss as its own line in the receipt and in the band.
- **SRE**: wants `top` to show the projected finish time against the deadline.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L5.1 | P3 | Per-replica disks are zonal; a replica restarted in another zone cannot attach its disk, so a checkpoint kept there is unreachable and the gang restart stalls. | **rule**: `--gang` requires `--checkpoint` on a zone-independent device (object store or RWX); the planner refuses a per-replica PVC path for checkpoints; `explain` prints the rule | ADR 0004 amendment 1, ROADMAP M3 |
| L5.2 | P2 | A group freeze longer than the collective's timeout (NCCL, c10d) kills the job on CONT. The amendment gives no bound. | **amend** ADR 0008 amendment 1: annotation `freeze_max_seconds` from the package; `kill -STOP --group` warns with the bound, and at the bound converts the freeze to stop-with-checkpoint | ADR 0008, annotations.yaml |
| L5.3 | P5 | Preemption does not burn retries but does burn money; the band ignores it. | **rule**: `/sys/k8s/<id>/preemption_rate` per node group (ADR 0011 decision 7); the band widens by expected restart loss on spot; the receipt separates productive from lost time | ADR 0011, pipeline-compiler.md §cost |
| L5.4 | P2 | Rank-0 hostname resolution at start depends on headless Service DNS propagation. | **accepted**: JobSet `startupPolicy` plus torchrun's own rendezvous retries; note it | walkthroughs.md |
| L5.5 | P12 | ADR 0004 does not say what the deadline counts from; `activeDeadlineSeconds` counts from Job start, users expect submission, and a budget pause re-queues the gang for hours. | **rule**: the deadline counts from submission by default, queue and pause time included; `--deadline-from=start` opts out; `explain` prints the choice and the queue estimate | ADR 0004, ROADMAP M6 |
| L5.6 | P7 | Jobs and JobSets have no `volumeClaimTemplates`; `--disk=per-replica` has no native object to compile to. | **rule**: the session pre-creates PVCs named `<pid>-<index>` and mounts them by index through the downward API; they are retained on restart and on `kill` (ADR 0006); M3 exit criterion gains this case | ROADMAP M3, ADR 0006 |
