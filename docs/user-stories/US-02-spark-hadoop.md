# US-02 · Submit a Spark job over HDFS from the shell and keep control of it

| Field | Value |
|---|---|
| Primary persona | DE (Data Engineer) |
| Secondary personas | PE, FIN, SRE |
| Workload class | coordinated group, package executable |
| Walkthrough | [walkthroughs.md §2](../walkthroughs.md) |
| Gaps already closed there | G2 |
| Milestones needed | M1, M2, M3, M5, M9, M10 |
| Findings in this document | L2.1 – L2.6 |

## 1. Story

**As a** data engineer, **I want** to stage a month of raw events into HDFS and run a sessionization Spark job with a budget and a deadline, **so that** I use the Spark the platform already runs without learning the operator's CRD, and I can see and stop the job like any process.

**Context.** The job runs once a month over about two terabytes and has blown its cloud budget twice when executors were oversized. The team knows Spark; it does not know SparkApplication YAML, and the platform team does not want to write it for them.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | the data engineer | personal identity on `/clusters/prod-data`, `ns=etl` | interactive, then detached |
| Approves | none | | |
| Pays | team etl | | 41 pods, one pid |
| Audits | FinOps, monthly | reads the receipt against the band | needs declared versus actual executors |
| Operates the platform | platform team | runs the Spark operator and HDFS as packages, provides the HDFS filesystem driver | L2.1 |

## 3. Preconditions

- **Capabilities**: `spark-operator`, `fs:hdfs` (fileserver driver), `node-agent` (group freeze), `opencost`.
- **Packages**: `spark-operator 2.1` and `hdfs 3.4` as platform packages; the package for `spark-submit` carries its annotation.
- **Mounts**: `/s3/raw` read-only; `/dev/fs/by-id/hdfs-prod-data` mounted at `/hdfs` with `ns=etl`.
- **Annotations**: `spark-submit: class non-parallel, compiles_to sparkapplication, group spark, signals {STOP: refused-without-group, TERM: graceful-stop}, floor refused, checkpoint none`.
- **Platform defaults** `explain` must print: executor pod template labels (`convoy.io/pid`), the egress line for the `cp`, HDFS replication factor used for the capacity check.

## 4. Main path

```bash
$ cp /s3/raw/2026-11/ /hdfs/staging/2026-11/          # copy job, same region
$ batch spark-submit --executors=40 … --deadline=3h --budget=$120 app.jar …
$ ps -T 7301                                            # driver + 40 executors as threads
$ kill -STOP 7301                                       # refused: coordinated group
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `cp` | copy process | copy Job reading S3, writing HDFS through the driver | HDFS blocks |
| `batch spark-submit` | process 7301, group `spark` | SparkApplication; driver pod and executor pods labelled `convoy.io/pid=7301` | operator status, receipt |
| `ps -T` | threads of 7301 | executor pods | API server |
| `kill -STOP --group` | group freeze | node agents freeze 41 cgroups within one Lease barrier | node agents |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ○ | the `cp` is a staged copy; the Spark output is not an artifact (opaque executable) | receipt records the HDFS output path, not an id |
| A2 | Command | ● | an executable with `group`, `compiles_to`, refused floor | `type spark-submit` prints all of it |
| A3 | Session | ○ | detach while the job runs | reattach shows the same pid |
| A4 | Backend | ● | `compiles_to: sparkapplication` is a fourth kind of target: a package CRD | `explain` says "package target", not a backend name |
| A5 | Contract | ● | deadline and budget bind an opaque group; floor refused | `--min-success` exits 3 at plan time with the reason |
| A6 | Device tree | ● | `/dev/fs` device, mount with `ns`; two devices decide egress | `mount` lists `/hdfs` with its type and options |
| A7 | Signals | ● | STOP refused on a member; TERM is graceful stop; member signals (L2.3) | `kill -STOP 7301` is refused with the two alternatives |
| A8 | Secrets | – | none | |
| A9 | Cost | ● | band from flags, actual from 41 pods | receipt shows declared executors versus observed (L2.2) |
| A10 | Identity | ○ | the Spark pods run under the mount's service account | SparkApplication `serviceAccount` equals the mount identity |
| A11 | Defaults | ● | label propagation, replication factor | both in `explain` |
| A12 | Storage | ● | HDFS as a filesystem device | `df /hdfs` shows capacity after replication |
| A13 | Networking | ○ | driver to executors inside the cluster | no policy needed; default allow within namespace |
| A14 | Time | ○ | deadline only | `activeDeadlineSeconds` on the driver |
| A15 | Audit | ○ | receipt per run | 41 pods attributed to one pid in OpenCost |
| A16 | Durability | ○ | driver loss is Spark's failure, not a Convoy retry | exit 1 with the operator's reason |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | the SparkApplication is created by Convoy, owned by Convoy | no GitOps contention | ok |
| P2 | POSIX cannot hold | `--on-budget=pause` on a job with no checkpoint | ADR 0004 says pause-with-checkpoint is the default action | **L2.6** |
| P3 | Orphaned state | the staging copy triples on HDFS; a failed job leaves staging | staging is user data; `du /hdfs/staging` shows it | L2.5 (capacity) |
| P4 | Identity leak | none | | ok |
| P5 | Invisible cost | the jar can override executor count and size | band computed from flags only | **L2.2** |
| P6 | Lying cache | opaque executable, never cached | ADR 0001: executables are not stages | ok |
| P7 | Silent failure | `mount /dev/fs/by-id/hdfs-…` with no HDFS client in the fileserver | ADR 0011 decision 9 names the device, not who reads it | **L2.1** |
| P8 | Partial fleet | one cluster | n/a | ok |
| P9 | Created elsewhere | executors are created by the Spark operator, not by the planner | labels propagate through the pod template; `ps -T` reads them | L2.4 |
| P10 | Multi-party | none | | ok |
| P11 | Data at rest | HDFS blocks and the S3 source are in one region | device topology | ok |
| P12 | Inheritance | signals to one executor (`kill -TERM 7301.7`) | undefined in ADR 0008 amendment 1 | **L2.3** |

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| driver OOM | operator status `FAILED` | 1 | operator reason, last executor count | `batch` again with more memory |
| budget hit at $120 | session cost watcher | 2 | action `cancel` (pause impossible, L2.6), spend so far | none; rerun with a larger budget |
| deadline 3h | `activeDeadlineSeconds` | 2 | same | same |
| HDFS full during `cp` | copy Job error | 1 | bytes written, capacity at start (L2.5) | `rm` staging, rerun |
| node agent missing on one node | `caps` at plan time | 3 | `--group` unavailable, STOP refused | platform installs the agent |

## 8. Acceptance criteria

- **AC-2.1** Given `spark-submit` installed by a package, when `type spark-submit` runs, then it prints the class, the compile target, the group name, the signal table and that the floor is refused.
- **AC-2.2** Given a running SparkApplication under pid 7301, when `kill -STOP 7301` runs, then it is refused with the `--group` and `-TERM` alternatives and nothing is frozen.
- **AC-2.3** Given `kill -STOP --group 7301`, then all 41 cgroups are frozen within one barrier period and `ps` shows `frozen` for every thread.
- **AC-2.4** Given a jar that requests 400 executors while the flags say 40, then the receipt shows both numbers and the budget still binds on the observed spend.
- **AC-2.5** Given `--on-budget=pause` on an executable annotated `checkpoint: none`, then the plan is refused with the reason at `explain` time.
- **AC-2.6** Given a filesystem device whose driver package is not installed, when `mount` is attempted, then it fails at pre-flight with the capability name.

## 9. Persona lenses

- **DE**: wants `spark-submit` to behave as it does on a laptop plus a budget; refuses if a refused floor looks like a Convoy bug rather than a stated rule.
- **PE**: owns the two packages and the HDFS driver; needs the driver contract written down (L2.1).
- **FIN**: wants declared versus observed resources on the receipt, because the jar decides the real size.
- **SRE**: wants a freeze that cannot stall a driver and a member-signal rule that cannot silently kill an executor.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L2.1 | P7 | A `/dev/fs` device is only usable if the session fileserver has a driver for its type; ADR 0011 names the device, not the driver. | **amend** ADR 0011 decision 9: a filesystem device type is mountable only when a package provides its fileserver driver, discovered by `caps` as `fs:<type>`; `mount` refuses otherwise with the capability name | ADR 0011, ROADMAP M5 |
| L2.2 | P5 | For executables the band is computed from flags, but the executable decides its real footprint (dynamic allocation, config in the jar). | **rule**: bands for executables are tagged `declared` in `explain`; the receipt records declared versus observed; a package may provide an estimator hook that reads the job config | ADR 0010, pipeline-compiler.md §cost |
| L2.3 | P12 | Signals addressed to one member of a coordinated group (`kill -TERM 7301.7`) are undefined. | **amend** ADR 0008 amendment 1: member signals are refused unless the group annotation lists `member_signals`; Spark lists `TERM` (executor loss is handled by the driver), torchrun and MPI list none | ADR 0008, annotations.yaml |
| L2.4 | P9 | Executors are created by the operator, so the planner never sees them; attribution relies on label propagation through the pod template. | **accepted**: the receipt counts what OpenCost observed under the label; `ps -T` reads the API server, not the plan; state this in the executables section | annotations.yaml, walkthroughs.md |
| L2.5 | P3 | A copy into a replicated filesystem needs replication × size of free capacity; a half-copied staging area is worse than a refusal. | **rule**: copy Jobs pre-flight the destination's free capacity × replication factor from `/sys`; `explain cp` prints it | ADR 0011 decision 7, posix-command-map.md (`cp`) |
| L2.6 | P2 | ADR 0004's default budget action, pause with checkpoint, is impossible for an executable with no checkpoint. | **rule**: annotation `checkpoint: none` makes `--on-budget=pause` and `--on-deadline=pause` a plan-time refusal; the default for such executables is `cancel` and `explain` says so | ADR 0004, annotations.yaml |
