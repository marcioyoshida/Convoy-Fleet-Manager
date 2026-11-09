# Walkthroughs: six workloads against the design

The first six workloads were set by the owner; the last three were chosen by the builder to stress parts of the design the first six do not reach (external spend and non-determinism, residency and evidence, fleet-wide references). Each walkthrough runs a workload through the design as it stands: the session as the user sees it, what the planner emits, what happens on failure, and where the design strains. A strain is annotated as a **GAP** with the amendment that closes it. The gaps found here are collected at the end and are applied as amendments to [ADR 0004](adr/0004-completion-contract-and-exit-codes.md) and [ADR 0008](adr/0008-signal-routing.md), plus roadmap and annotation-database changes.

Verbs and flags used below that are not yet specified elsewhere are introduced here and marked as such. Nothing in this document runs yet. The same nine workloads are probed persona by persona, aspect by aspect, in [user-stories/](user-stories/README.md); the ninety-one findings there go beyond the ten gaps below; the twenty-five that amend ADRs were applied on 2026-11-09, the rest are not yet applied.

| # | Workload | Runs on the design as written? | Gaps |
|---|---|---|---|
| 1 | DAG pipeline | yes, once M4 ships | G1 (barrier spelling) |
| 2 | Spark job with Hadoop across nodes | yes, with one refusal | G2 (coordinated groups) |
| 3 | CI/CD release across pods and environments | needs M11 | G3 (release writes intent) |
| 4 | Machine learning pipeline | yes, once M4 and M11 ship | none |
| 5 | Distributed model training | needs two additions | G4 (gang contract), G5 (JobSet backend), G2 |
| 6 | Distributed database, multi-tenant | yes, with one refusal | G6 (stateful scale-down), G2 |
| 7 | Agent orchestration (builder's pick) | yes, with two additions | G7 (non-deterministic cache), G8 (metered spend) |
| 8 | Regulated month-end close, data residency (builder's pick) | yes, with one addition | G9 (retention and legal hold) |
| 9 | Fleet-wide secret rotation after a leak (builder's pick) | needs one signal and one index | G10 (HUP, secret reference index) |

---

## 1. DAG pipeline

**Session**

```bash
$ cd /clusters/prod-data/teams/etl
$ cat nightly.sh
extract --date=$DAY /s3/raw/events/ \
  | tee >(clean | put /artifacts/clean) >(enrich --geo /s3/ref/geo.parquet | put /artifacts/geo) >/dev/null
wait
join -t, -j1 /artifacts/clean /artifacts/geo | load --table=events_enriched

$ explain nightly.sh
stage 1  extract        fan-out     8 partitions (2.1 GiB known)   parallel-local
stage 2  tee            fan-out     2 branches                     argo
stage 3a clean          stateless   8 partitions                   indexed job
stage 3b enrich --geo   pattern     8 partitions, side input 40 MiB indexed job
stage 4  wait           barrier
stage 5  join           fan-in      merge tree (sort -m, 8 → 1)    indexed job
stage 6  load           side-effect non-parallel, 1 pod
contract: floor 100 % (join needs every partition)  deadline none  budget none
cost band: $4–$7

$ crontab -e
0 3 * * *  batch nightly.sh --deadline=2h --budget=$10
```

**What is emitted.** One Argo Workflow. Each stage node is an Indexed Job; each edge is an artifact in the content-addressed store, named by the hash of command, toolbox version, env subset, input ids and partition. `tee >(…) >(…)` is the fan-out spelling the compiler already accepts (pipeline-compiler.md §1). The crontab line is a CronJob with `concurrencyPolicy: Forbid`, which is a platform default made visible by `explain`.

**On a rerun.** If `geo.parquet` is unchanged, stage 3b is a cache hit. Only `clean`, `join` and `load` execute. The receipt lists the hits.

**On failure.** `pipefail` is on. A failing partition in `clean` fails the node, the session cancels the sibling branch (fail-fast, ADR 0004), the Workflow stops, exit code 1, and the receipt names the partition and the toolbox image. A deadline or budget hit returns exit code 2 after the declared action, which for a CronJob defaults to "cancel and keep partial artifacts".

**GAP G1, barrier spelling.** The compiler accepts `>(…)` as fan-out, but nothing says what the fan-in barrier is. The diamond above only works because `wait` with no arguments is defined as "wait for every background branch of the previous command, then continue". That is the POSIX `wait` with no operands, so no new syntax, but it must be stated in the compiler document: `wait` is the DAG join, and `wait $!` joins one branch. *Closed by: a line in pipeline-compiler.md §1 and the roadmap M4 exit criterion.*

---

## 2. Spark job with Hadoop across nodes

**Session**

```bash
$ pkg list | grep -e spark -e hdfs
spark-operator   2.1   gitops   platform
hdfs             3.4   gitops   platform   (StatefulSet, 12 datanodes, 3 zones)

$ ls /dev/fs/by-id/
hdfs-prod-data
$ mount | grep hdfs
/dev/fs/by-id/hdfs-prod-data on /hdfs type hdfs (rw,ns=etl)

$ cp /s3/raw/2026-11/ /hdfs/staging/2026-11/
copy job 7290: 1.8 TiB, same region, no egress; band $2–$3; eta 14m

$ type spark-submit
spark-submit: executable, class non-parallel, compiles to SparkApplication (spark-operator 2.1),
              group: spark (driver + executors), signals: STOP refused, TERM = graceful stop

$ batch spark-submit --executors=40 --executor-cores=4 --executor-memory=16g \
    --deadline=3h --budget=$120 --class etl.Sessionize app.jar /hdfs/staging/2026-11 /hdfs/out/sessions
pid 7301  SparkApplication etl/sessionize-7301  band $60–$110  deadline 3h

$ ps -T 7301
PID       ROLE      STATE    NODE            AGE
7301      driver    running  ip-10-0-3-41    2m
7301.1    executor  running  ip-10-0-3-41    2m
…
7301.40   executor  running  ip-10-0-7-12    1m
(40 executors on 10 nodes)

$ kill -STOP 7301
convoy: 7301 is a coordinated group (spark). Freezing the driver alone times out its executors.
        Use `kill -STOP --group 7301` to freeze all 41 members, or `kill -TERM 7301` for a graceful stop.
```

**What is emitted.** Spark is a package, never a translation target: no text tool is ever turned into a Spark job (ADR 0001). `spark-submit` is an executable whose annotation says `class: non-parallel`, `compiles_to: sparkapplication`. The driver is the process; executors are its threads in `ps -T`, each labelled `convoy.io/pid=7301`, so OpenCost and the receipt attribute all 41 pods to one process. Hadoop is a StatefulSet the platform runs, exposed as a filesystem device under `/dev/fs` and mounted (ADR 0011, decision 9). The S3 to HDFS staging is `cp`, which becomes a copy Job with egress priced from the two device paths.

**The contract.** Deadline and budget bind the whole SparkApplication (`activeDeadlineSeconds` on the driver, budget watched by the session from the cost column). The floor does not apply: Convoy cannot see Spark's partitions, so `--min-success` on a non-parallel executable is refused at plan time with the reason.

**GAP G2, coordinated groups.** ADR 0008 freezes "the pod cgroup". A Spark driver frozen alone stops heart-beating and Spark kills its own executors; the same is true of a torchrun rank (story 5) and a database quorum member (story 6). The signal router must know that a process is a coordinated group and either freeze every member in the same instant through the node agents, or refuse with the reason. *Closed by: ADR 0008 amendment 1 (`group` annotation, `--group` flag, refusal default).*

---

## 3. CI/CD release across pods and environments

**Session**

```bash
$ cd /clusters/ci/teams/payments
$ batch ci.sh --deadline=20m          # build, unit, integration: a DAG, walkthrough 1
pid 7410  exit 0  receipt /artifacts/receipts/7410

$ release payments 1.4.2 --to staging \
  && wait --healthy payments --timeout=10m \
  && read -p "promote 1.4.2 to prod? [y/N] " ok && test "$ok" = y \
  && for env in prod-east-1 prod-eu; do
       release payments 1.4.2 --to $env --canary=10% \
       && wait --healthy payments --timeout=15m --error-rate-below=0.5% \
       && release payments --promote --to $env
     done

$ explain …
release --to staging    writes desired version to gitops repo (platform default release_path=git),
                        commit 3f9c…; waits for Argo CD sync of /clusters/staging/payments
wait --healthy          Deployment rollout status + readiness + Gateway error rate from /sys
read                    Argo suspend node; resumes on session input, times out after 24h (exit 2)
release --to prod-*     policy: approver must differ from author (exit 3 if not)
release --canary=10%    Gateway HTTPRoute weight 10/90 on a second Deployment
release --promote       weight 100/0, old ReplicaSet retained for --rollback
```

**What is emitted.** Environments are mounts in the session's mount table: `/clusters/staging/payments`, `/clusters/prod-east-1/payments`, `/clusters/prod-eu/payments`, each opened under its own identity (ADR 0011). `release` is a verb of the service-unit family (ROADMAP M11). The canary is a Gateway route weight. `read` becomes an Argo suspend node, so the approval, the approver's identity and the elapsed time are in the audit stream (ADR 0009). The loop over environments is explicit because writes to the `all` union are refused (ADR 0011, decision 6). Rollback is `release payments --rollback --to prod-eu`, which is a revert commit plus a wait.

**On failure.** A failed `wait --healthy` in staging stops the chain at `&&`. A canary that breaches the error rate exits 1, the route weight returns to 0/100, and nothing was promoted. A policy failure is exit 3 before any write.

**GAP G3, release writes intent.** The shell is imperative and the platform is declarative. If `release` patched the Deployment image directly, GitOps would revert it within minutes, which is the inert-feature failure mode. `release` must write the intended version where the platform says (the GitOps repository by default, the API server only when the platform sets `release_path=direct` for development namespaces), then wait for reconciliation. The receipt carries both the commit hash and the ReplicaSet hash so a reader can tie the audit line to the object. *Closed by: ROADMAP M11 scope and exit criterion; ADR 0010 already gives the platform the `release_path` default.*

---

## 4. Machine learning pipeline

**Session**

```bash
$ cd /clusters/ml-training/teams/churn
$ run --gpu=1 --unit notebook jupyter && expose notebook --internal
unit notebook  Deployment 1/1  https://notebook.churn.ml-training.internal

$ batch features.sh --deadline=1h       # select, filter, group, tjoin on parquet: typed stages
pid 7520  band $3–$5

$ ls -l /artifacts/ | head -3
9f3a…  412 MiB  features.parquet   from 7520 stage 4   $0.02/mo
71c0…   38 MiB  labels.parquet     from 7520 stage 5   $0.00/mo

$ ls /s3/datasets/churn/configs/*.yaml | xargs -P 16 -n 1 train --features /artifacts/9f3a… --config
pid 7531  Indexed Job 16 indexes, 1 GPU each  band $40–$70  budget none

$ history --receipts | sort -t, -k cost -n | tail -3
7531.11  train  $4.10  auc=0.912  /artifacts/c4e1…
7531.03  train  $4.30  auc=0.918  /artifacts/d02b…

$ cp /artifacts/d02b… /models/churn/1.3            # the registry is an object-store mount
$ run --unit churn-api --gpu=1 --image=serve:1.3 --model=/models/churn/1.3 && scale churn-api 3 && expose churn-api
```

**What is emitted.** Feature engineering is walkthrough 1 with typed verbs that compile to dataframe operators (ADR 0002, `class: typed`). The sweep is `xargs -P`, the fan-out primitive from the command map, which is an Indexed Job with sixteen indexes; a budget on it binds across all indexes. Experiment tracking needs no component: every artifact is content-addressed and every receipt names its inputs, so lineage from a model back to the raw data is `history --receipts` following ids. The registry is a mount of an object-store device. Serving is a service unit (M11) with `scale` and `expose`.

**On failure.** A failing config in the sweep fails its index only; the default floor for `xargs -P` is 100 %, so the user passes `--min-success=12` to accept a degraded sweep (exit 0 with the degraded flag) or lets it fail. The receipts of the twelve that passed are kept either way.

**No gaps.** This is the workload the design was built for. The only missing verbs are `expose` and service units, which are M11.

---

## 5. Distributed model training

**Session**

```bash
$ cd /clusters/ml-training/teams/llm
$ batch --replicas=8 --gpu=8 --disk=per-replica:2Ti --gang --checkpoint=/ckpt \
    --deadline=36h --budget=$9000 \
    torchrun --nnodes=8 --nproc-per-node=8 --rdzv-backend=c10d --rdzv-endpoint=$CONVOY_PID-0:29400 train.py
pid 8120  JobSet llm/train-8120  8 replicas × 8 GPU  gang (Kueue, all-or-nothing)
checkpoint_seconds=600  band $6400–$8900  deadline 36h

$ top
PID   STATE    REPL  GPU-UTIL  NET      COST/h   PROGRESS
8120  running  8/8   94 %      38 Gb/s  $246     step 11200/40000, ckpt 4m ago

$ kill -STOP --group 8120      # all 64 ranks frozen in one instant through the node agents
$ kill -CONT --group 8120

(spot reclaim on replica 5)
convoy: 8120 replica 5 preempted (DisruptionTarget, not counted against retries);
        gang restart from /ckpt/step-11000 on 8 replicas; 9m lost; budget remaining $7,120
```

**What is emitted.** Training is a JobSet (see G5) of one replicated Job with eight indexes, a headless Service so `$CONVOY_PID-0` resolves to rank 0, and `startupPolicy` so rank 0 is up before the rendezvous. Kueue admits the whole set or nothing (M6). `--disk=per-replica` is the stateful scaling rule from the storage review. Preemption is a stop with checkpoint (ADR 0008), so a spot reclaim costs at most `checkpoint_seconds`. `top` reads GPU utilization and the cost rate from `/sys` (ADR 0011, decision 7). The budget binds to the upper bound of the band (README principle 9).

**GAP G4, gang contract.** ADR 0004's contract assumes independent partitions: a floor, retries per index. Training is all-or-nothing, and when one rank dies every rank must restart from the last checkpoint. A `--gang` flag on the contract sets the floor to 100 %, replaces per-index retry with restart-the-group-from-checkpoint, and makes queue admission all-or-nothing. *Closed by: ADR 0004 amendment 1.*

**GAP G5, JobSet backend.** Indexed Job alone gives stable hostnames but no headless Service, no restart-all policy and no startup ordering. JobSet provides all three and is the upstream answer for multi-pod training. It joins Indexed Job and Argo as a backend target. *Closed by: ROADMAP M3 scope.*

**GAP G2 again.** `kill -STOP` on one rank stalls the collective and the other 63 ranks time out. Same amendment as walkthrough 2.

---

## 6. Distributed database, multi-tenant

**Session**

```bash
$ cd /clusters/prod-east-1/platform
$ mkdir tenants/acme
namespace prod-east-1/tenant-acme  quota from template "tenant-m"  network default-deny  owner acme-admins

$ cd tenants/acme
$ run --unit db --package=cnpg/cluster --replicas=3 --disk=per-replica:200Gi --zone-spread
unit db  cnpg Cluster acme/db  3/3  PDB maxUnavailable=1  do-not-disrupt

$ df -h .
VOLUME     SIZE   USED  ZONE        COST
/pvc/db-0  200Gi  41Gi  us-east-1a  $16/mo
/pvc/db-1  200Gi  41Gi  us-east-1b  $16/mo
/pvc/db-2  200Gi  40Gi  us-east-1c  $16/mo

$ scale db 2
convoy: db is stateful (cnpg). Scale-down is delegated to the operator, which decommissions db-2 first.
        Proceed? [y/N] y
unit db  3/3 → 2/2  db-2 decommissioned; /pvc/db-2 retained (detach is not destroy)

$ rmvol /pvc/db-2
convoy: /pvc/db-2 is referenced by the cnpg Cluster's retention policy (fuser). Refusing.
        snapshot first: `backup db` or `snapshot /pvc/db-2`.

$ backup db --to /s3/backups/acme/
VolumeSnapshot × 2, then copy job 8201: 82 GiB, same region; band $1

$ sudo --as platform ps -A --cluster=prod-east-1 | grep ' db '
tenant-acme   db   cnpg   2/2   healthy
tenant-globex db   cnpg   3/3   healthy
```

**What is emitted.** A tenant is a namespace, which is a directory created from the platform's quota template with a default-deny NetworkPolicy and owner labels (ROADMAP M7). The database is a service unit backed by an operator the platform installed as a package (ADR 0010). Per-replica disks, zone spread, a PodDisruptionBudget and do-not-disrupt are platform defaults that `run` prints rather than hides. `df` reads capacity, zone and cost from the volume devices (ADR 0011, decision 9). `rmvol` refuses while in use and `umount` only detaches (ADR 0006). Owner labels and CEL admission keep tenants from touching each other; the operator sees across them with `sudo --as`.

**GAP G6, stateful scale-down.** `scale` on a unit with per-replica disks must delegate to the operator, which knows how to decommission, and must refuse scale-down when there is no operator, because removing a replica of a quorum without decommissioning loses quorum or data. Scale-up delegates too but may proceed without one. *Closed by: a rule in the M11 service-unit design and the ADR 0006 consequence that scale-down never deletes the volume.*

**GAP G2 again.** `kill -STOP` on one quorum member is a lost heartbeat to the others. Same amendment as walkthrough 2.

---

## 7. Agent orchestration (chosen by the builder)

A support-triage agent reads tickets, calls an LLM, runs tools, and escalates. Picked because it stresses secrets as references, non-deterministic commands, external spend that no Kubernetes cost tool can see, and egress policy.

**Session**

```bash
$ cd /clusters/prod-east-1/teams/support
$ ls /secrets/
anthropic-api-key   zendesk-token          # references; `cat` is refused, `ls -l` shows who may mount
$ type agent-run
agent-run: executable, class non-parallel, deterministic: false, side_effects: true,
           egress: api.anthropic.com, *.zendesk.com   (NetworkPolicy allow-list from the package)

$ ls /s3/tickets/open/ | xargs -P 8 -n 1 \
    batch --budget=$40 --deadline=30m --secret=anthropic-api-key --secret=zendesk-token \
    agent-run --policy=triage-v3 --max-turns=12 --escalate-to=/queue/human-review
pid 7610  Indexed Job 8 indexes  band $12–$38 (metered: Anthropic tokens $10–$35, compute $2–$3)

$ top -p 7610
PID    REPL  TURNS  TOKENS-IN  TOKENS-OUT  SPEND   STATE
7610   8/8   4.2    212k       38k         $9.40   running
$ wait 7610
exit 0 (degraded): 7/8 converged; index 5 hit --max-turns and was escalated to /queue/human-review
```

**What is emitted.** The agent is a package-installed executable (ADR 0010), run as an Indexed Job over tickets through `xargs -P`. Secrets are references (ADR 0007): `--secret` mounts them through the Secrets Store CSI into the pod's tmpfs, the session never sees a value, and `ls -l /secrets` shows the reference-only RBAC. The package declares its egress hosts and the planner emits the NetworkPolicy; a tool call to any other host is a connection refused, not a policy surprise. `/queue/human-review` is a named queue (stories.md, `mkfifo`) that a human-in-the-loop session drains with `read`.

**On failure.** `--max-turns` is the agent's own deadline. Hitting it would be a convergence failure for that index, and under the default floor of 100 % it would fail the Job. Because `--escalate-to` declares the action for that case, the index hands the ticket to the queue and exits 0 with the degraded flag, so the overall result is degraded rather than failed. Budget is watched from the metered spend column, not from OpenCost.

**GAP G7, non-deterministic commands and the cache.** The artifact id is `sha256(cmd, toolbox, env subset, input ids, partition)` (ADR 0001). An LLM call with the same inputs does not produce the same output, so a cache hit on a rerun would silently replay yesterday's triage. The annotation database needs `deterministic: false`; the planner then salts the id with the pid and never serves it as a cache hit, while still storing the artifact and receipt for lineage. *Closed by: ADR 0001 amendment 1; annotation field.*

**GAP G8, metered external spend in the band.** OpenCost sees pods, not API tokens. The cost band and the budget must take a second source: a metering file the package writes under `/sys/cost/<pid>/metered` (tokens × price list), summed with compute. The budget binds on the sum. *Closed by: ROADMAP M2 cost sources; ADR 0010 consequence.*

---

## 8. Regulated month-end close with data residency (chosen by the builder)

A finance team closes the month for its EU entity: the pipeline must run in-region, be reproducible on demand for an auditor two years later, and finish before 06:00 with escalation if it will not. Picked because it stresses the device tree's topology, receipts as evidence, and artifact retention.

**Session**

```bash
$ cd /clusters/prod-eu/teams/finance
$ explain close.sh --residency=eu --deadline=06:00 --on-deadline=page:finance-oncall
stage 1  extract /s3/eu/ledger/2026-10/     parallel-local   /dev/obj/by-id/s3-eu-ledger   eu-west-1 ✓
stage 2  fx-normalize /s3/ref/fx-rates       pattern          /dev/obj/by-id/s3-us-ref      us-east-1 ✗
convoy: stage 2 reads a device outside residency eu (exit 3, nothing ran)
        either mount an eu copy or add --allow-cross-region (recorded in the audit stream)

$ cp /s3/ref/fx-rates /s3/eu/ref/fx-rates && sed -i 's#/s3/ref/#/s3/eu/ref/#' close.sh
$ batch close.sh --residency=eu --deadline=06:00 --on-deadline=page:finance-oncall --retain=7y
pid 7702  band €9–€14  all devices eu-west-1  deadline 06:00 (4h12m away)  retention 7y (legal hold)

$ wait 7702 && ls -l /artifacts/receipts/7702
7702  exit 0  2026-11-01T04:41Z  toolbox 1.4 sha256:…  inputs 3 ids  outputs 2 ids  identity finance-batch  retain 7y

(two years later, from any session)
$ replay 7702 --verify
replay 9910: same toolbox digest, same input ids, same env subset → output ids identical  ✓
```

**What is emitted.** `--residency=eu` is a policy check at plan time: every device a stage touches must sit under an `eu-*` topology in the device tree (ADR 0011). A violation is exit 3 before anything runs, which is what an auditor wants to read. The receipt is the evidence: toolbox digest, input content ids, env subset hash, identity and timestamps (ADR 0009). `replay` re-executes from the content ids with the pinned toolbox and reports whether the output ids match, which is reproducibility as a one-line check. `--on-deadline=page:` is the declared action of ADR 0004 bound to the platform's alerting package.

**On failure.** A stage that will miss the deadline is visible in `top` as a projected finish time; at the deadline the declared action runs, exit code 2, partial artifacts kept. A later rerun after the fix is a cache hit on every stage that already succeeded.

**GAP G9, artifact retention and legal hold.** The artifact store needs garbage collection or it fills; nothing in the design says what GC may delete. Receipts and the artifacts they reference must carry a retention class, and `--retain=7y` must set a hold that GC, `rm` and `rmvol` all refuse to break. The toolbox image referenced by a held receipt must be held too, or `replay` fails in year three. *Closed by: ADR 0006 amendment 1 (retention classes and hold); ROADMAP M4.*

---

## 9. Fleet-wide secret rotation after a leak (chosen by the builder)

A database password appeared in a log. The platform team must rotate it, roll every workload in every cluster that references it, and prove nothing still uses the old version. Picked because it stresses secrets as references across the fleet, the read-only union, and a signal the design never defined.

**Session**

```bash
$ sudo --as platform
$ fuser -v /secrets/payments-db-password --cluster=all
CLUSTER        NAMESPACE  PID   UNIT            VERSION  RELOAD
prod-east-1    payments   412   payments-api    v7       hup
prod-east-1    payments   415   payments-worker v7       restart
prod-eu        payments   118   payments-api    v7       hup
staging        payments   77    payments-api    v6       hup
convoy: 1 cluster unreachable (dev-sandbox); list is partial (exit 0, degraded)

$ secret rotate payments-db-password          # writes v8 to the vault; no value shown
payments-db-password v8 written; v7 retained until `secret retire`

$ for c in prod-east-1 prod-eu staging; do
    kill -HUP --cluster=$c $(fuser /secrets/payments-db-password --cluster=$c)
  done
prod-east-1  412 reloaded (SIGHUP delivered, probe healthy)   415 rollout restarted 3/3
prod-eu      118 reloaded
staging      77  reloaded

$ fuser -v /secrets/payments-db-password --version=v7 --cluster=all
(none reachable)   1 cluster unreachable (dev-sandbox)
$ secret retire payments-db-password v7 --when="fuser --version=v7 is empty on all clusters"
convoy: deferred; dev-sandbox unreachable. Will retire when the condition holds; audit line written.
```

**What is emitted.** Secrets are references (ADR 0007), so rotation is a new version in the vault behind the Secrets Store CSI; pods pick it up on their next mount sync, and the session never handles a value. `fuser` over the `all` union lists every referencing process across the fleet, partial on an unreachable cluster, exactly as ADR 0011 decision 6 prescribes. The loop over clusters is explicit because the union is read-only. `kill -HUP` is the reload signal: on a unit whose package declares `reload: hup` the signal is delivered to the process; on `reload: restart` it becomes a rolling restart of the Deployment. `secret retire` is the destructive step and is deferred until the condition is provably true.

**On failure.** A unit that fails its probe after reload is rolled back by the rollout and the loop stops at `&&`. The unreachable cluster is the real risk: retirement is deferred, not forced, and the audit stream records who deferred it and why.

**GAP G10, HUP and the secret reference index.** ADR 0008 defines TERM, KILL, STOP and CONT, not HUP. `fuser` on a secret needs a reference index: which processes mount which secret at which version, per cluster, maintained by the session from the mounts it created plus the inventory for processes created elsewhere. *Closed by: ADR 0008 amendment 2 (HUP = reload per the unit's declaration, else rolling restart); ADR 0007 amendment 1 (secret versions, `fuser` reference index, `rotate` and `retire` verbs with deferred retirement).*

---

## Gaps and amendments

| Gap | Found in | Change | Where |
|---|---|---|---|
| **G1** barrier spelling | 1 | `wait` with no operands is the DAG join; `wait $!` joins one branch | pipeline-compiler.md §1; M4 exit criterion |
| **G2** coordinated groups | 2, 5, 6 | annotation field `group`; `kill -STOP` on a group member is refused unless `--group`; `--group` freezes every member in one instant; TERM on a group is the group's own graceful stop | **ADR 0008 amendment 1**; `toolbox/annotations.yaml` executables section |
| **G3** release writes intent | 3 | `release` writes the desired version to the GitOps repository by default, waits for reconciliation; receipt carries commit and ReplicaSet hashes | ROADMAP M11 |
| **G4** gang contract | 5 | `--gang`: floor 100 %, restart-the-group-from-checkpoint instead of per-index retry, all-or-nothing admission | **ADR 0004 amendment 1** |
| **G5** JobSet backend | 5 | JobSet joins Indexed Job and Argo as a backend target for multi-pod coordinated work | ROADMAP M3 |
| **G6** stateful scale-down | 6 | `scale` down on a unit with per-replica disks delegates to the operator or refuses; volumes are retained | ROADMAP M11; ADR 0006 consequence |
| **G7** non-deterministic cache | 7 | annotation `deterministic: false`; id salted with the pid; never a cache hit; artifact and receipt still stored | **ADR 0001 amendment 1**; `toolbox/annotations.yaml` |
| **G8** metered external spend | 7 | packages write `/sys/cost/<pid>/metered`; band and budget sum it with compute | ROADMAP M2; ADR 0010 consequence |
| **G9** retention and legal hold | 8 | retention classes on receipts, artifacts and the toolbox images they pin; `--retain` sets a hold that GC, `rm` and `rmvol` refuse to break; `replay --verify` | **ADR 0006 amendment 1**; ROADMAP M4 |
| **G10** HUP and secret references | 9 | `kill -HUP` = the unit's declared reload or a rolling restart; secret versions; `fuser` reference index per cluster; `secret rotate` / `secret retire` with deferred retirement | **ADR 0008 amendment 2**; **ADR 0007 amendment 1** |

Ten gaps from nine workloads, and every one is a flag, a field, an index or an amendment: none required a new abstraction. What did not need changing: the three definitions, the artifact store as the pipe, local-first backends, the device tree, detach is not destroy, secrets as references, platform defaults made visible, and cost as part of the contract. Every workload above used all of them unmodified.
