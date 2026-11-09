# US-01 · Run the nightly DAG as a shell script and trust the rerun

| Field | Value |
|---|---|
| Primary persona | DE (Data Engineer) |
| Secondary personas | DS, SRE, FIN |
| Workload class | batch DAG |
| Walkthrough | [walkthroughs.md §1](../walkthroughs.md) |
| Gaps already closed there | G1 |
| Milestones needed | M1, M2, M3, M4 |
| Findings in this document | L1.1 – L1.6 |

## 1. Story

**As a** data engineer, **I want** the nightly events pipeline to be one shell script with a diamond in it (extract, two cleaning branches, join, load), **so that** a rerun after a bad night re-executes only what changed and nobody rewrites the script in a workflow DSL.

**Context.** The events table feeds the morning dashboards; a late or doubled load is noticed by finance at 08:00. Today the pipeline is an Airflow DAG whose Python wraps the same five binaries, and a rerun means clearing tasks by hand.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | the team's cron | `etl-nightly` service account via the `/clusters/prod-data/teams/etl` mount | the script never runs under a person |
| Approves | none | | |
| Pays | team etl | cost centre label on the mount | receipt per night |
| Audits | the on-call data engineer, next morning | reads the receipt and the cache hits | needs "what ran, what was reused, what failed, which partition" |
| Operates the platform | platform team | installed Argo, the artifact store, OpenCost | sets `concurrencyPolicy: Forbid` as the CronJob default |

## 3. Preconditions

- **Capabilities**: `argo`, `artifact-store`, `indexed-job`, `cronjob`, `opencost`.
- **Packages**: none beyond the toolbox; `extract`, `clean`, `enrich`, `load` are user tools added with `tool add`, so they carry annotations.
- **Mounts**: `/clusters/prod-data` with `ns=teams-etl`; `/s3/raw` read-only; `/s3/ref` read-only; `/artifacts` on the in-region store.
- **Annotations**: `extract: fan-out`, `clean: stateless`, `enrich: pattern` with a declared side input, `join: fan-in`, `load: non-parallel, side_effects: true`.
- **Platform defaults** `explain` must print: CronJob concurrency, deadline action for cron (`cancel, keep partial`), the artifact store chosen, the env subset hashed per command.

## 4. Main path

```bash
$ cd /clusters/prod-data/teams/etl
$ explain nightly.sh                       # six stages, floor 100 %, band $4–$7
$ crontab -e
0 3 * * *  batch nightly.sh --deadline=2h --budget=$10
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `extract … \| tee >(…) >(…)` | stages 1–3, fan-out | Argo Workflow with three Indexed Job nodes | artifacts in the store, receipts in the session |
| `wait` | barrier (G1) | Argo dependency edges | plan |
| `join … \| load` | fan-in then side-effect stage | Indexed Job merge tree, then one pod | the table, outside Convoy |
| `crontab` line | scheduled process | CronJob, `concurrencyPolicy: Forbid`, `timeZone` from the session | the CronJob object |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ● | every edge is an artifact; the rerun depends on correct ids | a rerun with `geo.parquet` unchanged re-executes only `clean`, `join`, `load` |
| A2 | Command | ● | four user tools with four different classes | `type clean` prints the class and the env subset it hashes |
| A3 | Session | ○ | cwd sets cluster and namespace; cron runs with no attached terminal | the CronJob runs with the session detached and the receipt is still written |
| A4 | Backend | ● | fan-out to Indexed Jobs under an Argo Workflow | `explain` names the backend per stage and why |
| A5 | Contract | ● | floor 100 % inferred from the join; deadline and budget from cron | a partition failure cancels the sibling branch and exits 1 |
| A6 | Device tree | ○ | `/s3/raw` and `/artifacts` are devices in one region | `explain` shows no egress line |
| A7 | Signals | ○ | `kill` on the Workflow cancels every node | `kill <pid>` leaves no running pods |
| A8 | Secrets | – | none | |
| A9 | Cost | ● | band at `explain`, receipt per night | receipt cost is within the band on a normal night |
| A10 | Identity | ○ | cron runs under the mount's service account | the receipt names `etl-nightly`, not a person |
| A11 | Defaults | ● | `concurrencyPolicy`, deadline action, store | all three appear in `explain` |
| A12 | Storage | – | no volumes | |
| A13 | Networking | – | none | |
| A14 | Time | ● | cron, deadline | a run that cannot start before the next window is a receipt, not a silence (L1.6) |
| A15 | Audit | ● | the morning read is the receipt | receipt lists cache hits with ids |
| A16 | Durability | ○ | session restart mid-run | the Workflow continues; `ps` shows it after the restart |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | `load` writes a table; nothing in the cluster owns it | no controller contends | ok |
| P2 | POSIX cannot hold | `tee >(…)` and `wait` on process substitutions | bash dialect; the compiler defines `wait` as the join (G1) | L1.4 |
| P3 | Orphaned state | partial artifacts kept at deadline | kept, by the cron default | L1.5 |
| P4 | Identity leak | none | | ok |
| P5 | Invisible cost | S3 GET and the store's storage of partials | S3 requests priced in the band ([README](../../README.md) example); partial storage is not | L1.5 |
| P6 | Lying cache | `extract --date=$DAY`: is `DAY` in the hashed env subset? | ADR 0001 hashes "the env subset" without saying who chooses it | **L1.1** |
| P7 | Silent failure | `load` rerun after a partial failure | side effects are not cached; nothing stops a second load | **L1.2** |
| P8 | Partial fleet | one cluster, one region | n/a | ok |
| P9 | Created elsewhere | the target table | outside Convoy by design | accepted |
| P10 | Multi-party | none | | ok |
| P11 | Data at rest | artifacts and receipts in the in-region store | ADR 0006 amendment 1 retention classes | ok, see US-08 for location |
| P12 | Inheritance | the cron budget binds the whole Workflow | one process, one contract | ok; floor conflict L1.3 |

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| one `clean` partition fails | Job `podFailurePolicy` | 1 | partition id, toolbox digest, sibling branch cancelled | `batch nightly.sh` again: cache hits on the rest |
| deadline at 05:00 | session deadline watcher | 2 | stages done, partials kept with `partial` class | rerun after fix |
| `geo.parquet` changed | content id | 0 | stage 3b miss, 3a hit | none |
| `load` fails after writing half | the tool's exit code | 1 | side-effect stage failed; **rerun may double-load** | manual today (L1.2) |
| CronJob skipped (queue full at 03:00) | `startingDeadlineSeconds` | 2 (L1.6) | "not started by 03:30" | `batch nightly.sh` by hand |

## 8. Acceptance criteria

- **AC-1.1** Given `nightly.sh` and an unchanged `geo.parquet`, when it runs on two consecutive nights, then the second receipt lists stage 3b as a cache hit and only stages 3a, 5 and 6 execute.
- **AC-1.2** Given a failing partition in stage 3a, when the Job reports it, then the sibling branch is cancelled within one reconcile period, exit is 1, and no pod of pid remains.
- **AC-1.3** Given `--deadline=2h`, when the deadline passes, then exit is 2, completed artifacts are retained with class `partial`, and `ls -l /artifacts` marks them.
- **AC-1.4** Given `type extract`, then the output names the environment variables included in its artifact id.
- **AC-1.5** Given a `load` stage with `side_effects: true`, when the stage is retried, then the retry is refused unless the annotation declares `idempotent: true`, and the receipt says so.
- **AC-1.6** Given a CronJob that could not start within its window, then a receipt with exit 2 exists for that night.

## 9. Persona lenses

- **DE**: must read one receipt and know what ran, what was reused and what to rerun; refuses if a rerun can double-load.
- **DS**: wants the same `tee >(…)` spelling to work interactively on a sample before it is scheduled.
- **SRE**: wants the 03:00 skip to page, not to be discovered at 08:00.
- **FIN**: wants the nightly cost on one line per night, and partial-artifact storage counted somewhere.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L1.1 | P6 | The env subset in the artifact id is undefined for user tools and executables. An undeclared `DAY` makes every night a cache hit of the first night. | **rule**: annotated commands hash their declared `env` list; `unknown` commands and executables hash the whole environment minus a platform-declared volatile list (`PWD`, `OLDPWD`, `CONVOY_PID`, `_`, `SHLVL`); `type` and `explain --cache` print the subset; a cache hit names the subset in the receipt | ADR 0001, pipeline-compiler.md §3 |
| L1.2 | P7 | Side-effecting stages are not cached, so a rerun after a half-written `load` writes twice. | **amend** (applied: ADR 0002 a1) ADR 0002: `side_effects: true` stages receive their input artifact id as `CONVOY_IDEMPOTENCY_KEY`; automatic retry of such a stage is refused unless `idempotent: true`; `explain` prints the refusal | ADR 0002, annotations.yaml |
| L1.3 | P12 | A user floor below 100 % on a fan-out whose consumer is a fan-in that needs every partition is a contradictory contract. | **rule**: the planner computes the effective floor per stage from downstream classes and refuses a `--min-success` that a fan-in cannot honour (exit 3 at plan time) | ADR 0004, pipeline-compiler.md §7 |
| L1.4 | P2 | `>(…)` is a bash dialect and `wait` on process substitutions is not defined by POSIX. | **accepted**: the parser runs in bash mode, the compiler defines the join (G1); record the deviation | posix-command-map.md |
| L1.5 | P3, P5 | Partial artifacts kept at deadline look like complete ones and have no retention or cost line. | **rule**: retention class `partial`, default seven days, marked in `ls -l /artifacts` and priced in the receipt | ADR 0006 amendment 1 |
| L1.6 | P7 | A CronJob that misses its window is skipped silently by Kubernetes. | **rule**: `crontab` derives `startingDeadlineSeconds` from `--deadline`, and a skipped run writes a receipt with exit 2 and the declared action | posix-command-map.md (`crontab`), ROADMAP M4 |
