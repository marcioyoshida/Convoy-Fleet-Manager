# US-29 · One pipeline, many computers: preprocess on functions, train on the university's Slurm cluster, evaluate on Batch, serve from a VM pool, in two regions, from one shell

| Field | Value |
|---|---|
| Primary persona | ML |
| Secondary personas | PE, FIN, SEC, SRE, DE |
| Workload class | fleet operation: one pipeline whose stages run on compute that has no Kubernetes API (a function runtime in two providers, an external Slurm cluster, a managed batch service, a VM pool), with data resident in two regions |
| Walkthrough | written against [ADR 0003](../adr/0003-three-backends-local-first.md) and its amendment 1, [ADR 0011](../adr/0011-device-tree-and-mount-table.md) decisions 1 to 3 and amendments 1, 3 and 4, [ADR 0004](../adr/0004-completion-contract-and-exit-codes.md) and its amendments, [ADR 0008](../adr/0008-signal-routing.md), [ADR 0009](../adr/0009-stdout-artifact-stderr-log.md), [ADR 0013](../adr/0013-native-verbs-have-one-shape.md) decisions 4, 7 and 8, [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md), [ADR 0019](../adr/0019-a-unit-has-an-address-reach-is-a-grant.md), [ADR 0021](../adr/0021-a-node-is-tested-before-it-is-trusted.md), [ADR 0022](../adr/0022-the-laptop-lends-a-tree-the-cluster-builds-it.md), [ADR 0023](../adr/0023-telemetry-is-a-device-the-shell-reads.md) and [ADR 0024](../adr/0024-a-device-is-installed-by-terraform.md); it follows US-27 and US-28 |
| Gaps already closed there | none |
| Milestones needed | M4, M5, M7, M8, M9 |
| Findings in this document | L29.1 – L29.18 |

## 1. Story

**As an** ML engineer, **I want** to run my nightly pipeline on whatever computer suits each step (cheap functions for preprocessing, the university's GPU cluster for training, spot batch capacity for evaluation, the VM pool our legacy inference server needs), with the European shards processed in Europe and the American ones in America, **so that** I write the pipeline once, the shell picks where each step runs and tells me why, and I never learn four schedulers, four log formats and four ways to cancel a job.

**Context.** After US-28 the platform runs ADR 0024 for installs. The `vision` group trains a detection model on two shard sets: `s3://acme-frames-us` (us-east-1) and `gs://acme-frames-eu` (europe-west4, `residency=eu`). Training runs on `hpc-sp`, a Slurm cluster at a partner university that the platform does not operate; `vision` holds an allocation of 40 000 GPU-hours a year under account `vision-pi`, and the cluster runs containers with Apptainer. The platform team has installed driver packages (ADR 0011 amendment 1) that register four non-Kubernetes computers as devices, under provisional classes because the design has none: `/dev/fn/by-id/lambda-123456789012-us-east-1` and `/dev/fn/by-id/cloudrun-acme-ml-europe-west4`, `/dev/slurm/by-id/hpc-sp`, `/dev/batch/by-id/awsbatch-123456789012-us-east-1-spot-gpu`, and `/dev/vm/by-id/mig-acme-ml-europe-west4-infer`, a managed instance group installed by a `vm-pool` module (ADR 0024) whose instance template pins an NVIDIA driver the inference server needs. `priya` (user `priya`, group `vision`) owns the pipeline.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | `priya` | user `priya`, group `vision` | |
| Owns the allocation | `vision-pi` | the PI, a member of `vision-leads` | the allocation is the university's, not the platform's |
| Operates `hpc-sp` | the university's HPC team | none; a ticket queue and `scontrol` | outside the platform and its identity provider |
| Operates the other devices | platform team | `svc:platform` | packages, modules, quotas |
| Pays | `vision` | functions, Batch, the VM pool; the allocation for Slurm | |
| On call for the inference service | SRE | group `sre` | |

## 3. Preconditions

- **Capabilities** (`caps`): on `lambda-…`: `fn:lambda`, `max_duration=900s`, `signals=none`; on `cloudrun-…`: `fn:cloudrun`; on `hpc-sp`: `slurm:23.11`, `container:apptainer`, `gpu:a100`, `identity` unknown; on `awsbatch-…`: `batch:aws`, `spot`; on `mig-…`: `vm:mig`, `iac:atlantis`.
- **Packages**: driver packages `fn-lambda`, `fn-cloudrun`, `slurm`, `batch-aws`, `vm-pool`; the `vm-pool` device package with modules `gcp-mig`, `aws-asg`.
- **Platform defaults** `explain` must print: the placement order, `residency` per mount, the price model of each computer, the preemption notice of each, the queue estimate of each, the identity each submits as.

## 4. Main path

```bash
# Monday 18:00, the nightly pipeline, written once
[priya] $ cat nightly.sh
cat /data/frames/{us,eu}/*.parquet | preprocess --shard | tee /data/tensors/ \
  && train --gang --ranks=32 --gpus=a100 /data/tensors/ > /models/det-v7 \
  && evaluate --partitions=200 /models/det-v7 /data/holdout/ \
  && release /units/infer --image=det-v7
[priya] $ explain nightly.sh
explain: no placement for stage 1: no Kubernetes cluster is mounted under /data/frames       (L29.1)
#  four computers are in the tree; the planner places only on /dev/k8s, and the four drivers
#  register devices nothing can run a stage on

# the platform team mounts the computers by hand for the trial
[priya] $ explain nightly.sh
stage 1 preprocess    placement: lambda-…-us-east-1 (cheapest)                                 (L29.7)
#  both shard sets go to us-east-1, including the EU shards; and the function logs for the
#  EU stage land in CloudWatch in us-east-1                                                      (L29.8)
stage 1 partitions:  410 by input size; predicted 22 min each on the function                  (L29.3)
stage 2 train         placement: hpc-sp (largest input local? no: cheapest, $0.00)              (L29.6)
#  the allocation is 40 000 GPU-hours a year with 6 100 left; $0.00 is what the bill says and
#  nothing the PI will accept
stage 2 start:       unknown                                                                    (L29.7)
#  squeue --start on hpc-sp says 09:40 tomorrow; Batch spot would start now
stage 3 evaluate      placement: awsbatch-…-spot-gpu
stage 4 release       placement: mig-…-infer   writes: instance template image=det-v7          (L29.9)

# 18:30 run, after splitting partitions by hand
[priya] $ ./nightly.sh &
[priya] $ ps -T
PID   STAGE        DEVICE       STATE      MEMBERS
9101  preprocess   lambda-…     running    410 invocations, 37 throttled, 0 failed?           (L29.12)
#  the account's 1 000 concurrent executions are shared: another team reserved 600            (L29.15)
#  async invocations whose retries ran out were dropped; the stage reports success
[priya] $ kill -STOP 9101
#  accepted; nothing happens; a function cannot be frozen                                       (L29.2)

# 21:10 train is submitted to hpc-sp
#  slurmrestd accepted the job from the driver's service account; the job runs as uid
#  convoy-svc; checkpoints on the university's Lustre are owned by convoy-svc, readable by
#  every Convoy tenant on that cluster                                                           (L29.5)
#  the SIF image was built on the login node from the tag det-train:latest                       (L29.11)
#  --time is the partition's 48 h; the pipeline's --deadline=08:00 is not passed               (L29.10)

# 02:30 the HPC team drains two nodes for a firmware fix; Slurm requeues the job
#  the rank logs slurm-%A_%a.err are truncated by the requeue; the [index.attempt] prefix
#  restarts at .0 and the first attempt's errors are gone                                       (L29.4)
#  the requeue counts against --retries as a failure

# 03:00 the node-health controller sees a GPU fault on hpc-sp-g17 in the telemetry
#  and has no verb: it cannot cordon a node of a cluster it does not operate                    (L29.14)

# 04:15 slurmrestd is unreachable for 50 minutes (the university's VPN)
[priya] $ ps 9102
9102  train  hpc-sp  failed                                                                     (L29.13)
#  the job is still running on hpc-sp and finishes at 06:40; nothing records its job id

# 07:00 evaluate on Batch spot; 200 partitions; the holdout set lives in both regions
#  the planner pulls the EU holdout to us-east-1 to evaluate it                                  (L29.18)

# 07:50 release to the VM pool
[priya] $ release /units/infer --image=det-v7
#  the driver patched the MIG's instance template in place; the next Atlantis plan reverts it   (L29.9)
[sre] $ expose infer --internal --allow=apps
#  refused: no NetworkPolicy; the VM pool's firewall rules are not in the model                  (L29.16)

# the receipt: evaluate's mAP is 0.4 points below last week's; last week's ran on GKE A100s
#  with another driver and cuDNN; the difference is not a regression                             (L29.17)
```

| Step | Convoy object | Backend object | Where the state lives |
|---|---|---|---|
| preprocess | a stage of N partitions | N function invocations | the function runtime, its log service |
| train | a gang process | one Slurm job (`sbatch`), steps per rank (`srun`) | `slurmctld`, the university's file system |
| evaluate | a stage of 200 partitions | an AWS Batch array job | Batch, CloudWatch |
| release | a unit release | an instance template change and a rolling update | the IaC repository (ADR 0024), the MIG |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe: staged artifact, content id, cache | ○ | an OCI image converted to SIF (L29.11) | AC-29.11 |
| A2 | Command: class, annotation, executable fields | ○ | executables declare a run-time bound per partition | |
| A3 | Session: mount table, env, cwd, attach and detach | ○ | mounts of four computers | |
| A4 | Backend selection | ● | placement beyond Kubernetes; partitions sized to a function's bound; queue wait (L29.1, L29.3, L29.7) | AC-29.3, AC-29.7 |
| A5 | Completion contract | ● | async invocations, throttles, requeues and a lost controller (L29.4, L29.12, L29.13) | AC-29.4, AC-29.12, AC-29.13 |
| A6 | Device tree | ● | provisional classes per kind (L29.1); account-wide limits (L29.15) | AC-29.15 |
| A7 | Signals and lifecycle | ● | `kill -STOP` on a function; preemption on Slurm (L29.2, L29.4) | AC-29.2 |
| A8 | Secrets as references | ○ | the Slurm token of the exchange (L29.5) | |
| A9 | Cost | ● | allocation units, per-request prices, per-second VMs (L29.6) | AC-29.6 |
| A10 | Identity and authorization | ● | jobs under a shared uid (L29.5); an operator outside the platform (L29.14) | AC-29.5, AC-29.14 |
| A11 | Platform defaults and `caps` | ● | `max_duration`, `signals`, `identity`, `operator` per device | AC-29.2 |
| A12 | Storage | ○ | checkpoints on a file system the platform does not own | |
| A13 | Networking | ● | reach on a VM pool (L29.16); a fan-in across regions (L29.18) | AC-29.16, AC-29.18 |
| A14 | Time and queueing | ● | a Slurm queue measured in hours; deadlines compiled to `--time` (L29.7, L29.10) | AC-29.10 |
| A15 | Audit, receipts, retention and `replay` | ● | receipts that name the computer and its hardware (L29.17) | AC-29.17 |
| A16 | Durability | ○ | a job that outlives its controller's link (L29.13) | |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative verb, declarative owner | `release` to a VM pool whose instance template is a module input | ADR 0010 amendment 1 covers GitOps objects; ADR 0024 makes the template intent; the driver does not know | **L29.9** |
| P2 | POSIX semantics that cannot hold | `kill -STOP` and `pause` on a function invocation | ADR 0008 freezes a cgroup through a node agent | **L29.2** |
| P3 | Orphaned or lost state | the controller of an external scheduler is unreachable while the job runs | ADR 0011 amendment 4 makes disconnected a state for devices, not for processes on them | **L29.13** |
| P4 | Identity leak or escalation | a Slurm cluster that sees one service account for every Convoy user | ADR 0016 assumes a credential exchange per user per device | **L29.5** |
| P5 | Invisible cost | a cluster billed as an allocation, functions billed per request | ADR 0010 prices pod-hours | **L29.6** |
| P6 | Lying cache | an image converted on the login node from a tag | ADR 0022 builds by digest on the cluster; conversion is not a build | **L29.11** |
| P7 | Silent failure | throttled and dropped asynchronous invocations | ADR 0004 counts partition failures the backend reports | **L29.12** |
| P8 | Partial fleet | one computer's queue is hours long | ADR 0003 amendment 1 places by locality then `caps`; it has no notion of start time | **L29.7** |
| P9 | Created elsewhere | account-wide limits consumed by other teams' functions | quota is checked from partitions × requests of this plan | **L29.15** |
| P10 | Multi-party action | a faulty node on a cluster the platform does not operate | ADR 0021 decision 7 lists what the controller may do on clusters the platform runs | **L29.14** |
| P11 | Data at rest | EU shards sent to a US function; logs in the function's region; a fan-in across regions | residency is a mount option (ADR 0011 amendment 3) and binds devices, not the computer a stage is placed on | **L29.8**, **L29.18** |
| P12 | Process-tree inheritance | a deadline and retries inside a scheduler with its own time limit and requeue | ADR 0004 amendment 3 inherits into child processes Convoy creates | **L29.10**, **L29.4** |

Two more questions without a probe: is there a device class for compute at all (A4 and A6), and does a partition fit on the computer it is placed on (A4)? **L29.1**, open; **L29.3**, amend. A fourth answer: the same model evaluated on different hardware does not give the same numbers. **L29.17**, accepted. A thirteenth question about a unit's address on a VM pool, outside every probe, is **L29.16**.

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| no computer satisfies a stage | plan (L29.1, L29.3) | 3 | each candidate and the `caps` or bound it lacks | `--engine`, smaller partitions |
| signal the computer cannot deliver | `kill` (L29.2) | 3 | the signal and the computer's `signals` | `kill -TERM` |
| invocations throttled or dropped | the driver (L29.12) | 1 or 0 degraded | throttles and failures per partition | `replay` of failed partitions |
| requeue by the external scheduler | the driver (L29.4) | none | the attempt and the reason | none; it does not count |
| controller unreachable | the driver (L29.13) | none while within `--deadline` | `unknown` with the backend job id | `wait`, `kill` queued |
| job runs as a shared identity | plan (L29.5) | 3 under `identity=required` | the computer's identity mode | per-user submission |
| fault on a node the platform does not operate | the health controller (L29.14) | none | the evidence and the operator's channel | a request to the operator |
| a release written to a template the IaC owns | plan (L29.9) | 3 | the module input to write | `release` as intent |

## 8. Acceptance criteria

- **AC-29.2** Given a computer whose `caps` list `signals=none` or a subset, then `kill` with a signal outside it is refused with exit 3 naming the computer and the signal; `kill -TERM` on a function stage stops new invocations and lets the running ones end within their bound; `pause` is refused.
- **AC-29.3** Given a stage placed on a computer with `max_duration`, then the planner sizes partitions so each fits the bound with the executable's declared throughput, or refuses the placement at plan time and names the next candidate.
- **AC-29.4** Given a requeue or preemption by an external scheduler, then the attempt number increases, stderr for each attempt is kept and prefixed `[index.attempt]`, and the requeue does not count against `--retries`.
- **AC-29.5** Given a computer that accepts work only from a shared identity, then `caps` shows `identity=shared`, `explain` says that files the job writes are owned by that identity, and a mount with `identity=required` refuses the placement.
- **AC-29.6** Given each computer's price model, then `explain` prints a band in that model (an allocation in its own units with the balance, a per-request and per-second price, an instance-second price with its minimum) and the receipt settles in the same units.
- **AC-29.7** Given candidates for a stage, then placement filters by residency, then `caps` and bounds, then locality, then ranks by cost and estimated start; `explain` prints the candidates with each one's reason and estimated start, and `--deadline` excludes a candidate whose estimated start cannot meet it.
- **AC-29.8** Given a stage over resident data, then the placement and the computer's log sink are both inside the residency, and a stage whose computer writes logs outside it is refused.
- **AC-29.9** Given a VM pool installed by a module, then `release` and `scale` write the image and the size as intent to the instance's module inputs, and the provider's own rolling update applies them.
- **AC-29.10** Given a process with a deadline and a budget on an external scheduler, then the driver submits with a time limit no later than the remaining deadline and a step count no greater than the plan's, and members started inside the job are shown in `ps -T`.
- **AC-29.11** Given an image for a computer that runs another format, then the conversion starts from the digest, is a cached stage keyed by it, and the receipt records the digest and the converted image's hash.
- **AC-29.12** Given a function stage, then invocations are synchronous or have a failure destination the driver reads; throttles and failures are counted per partition and drive the contract.
- **AC-29.13** Given an unreachable controller, then the process is `unknown`, never `failed`, with the backend job id in `/sys/proc/<pid>/backend`, and it is resolved from the scheduler's accounting when the link returns.
- **AC-29.14** Given a computer with `operator: external`, then node verbs from the health controller and from users become requests on the operator's declared channel, and the controller only reports.
- **AC-29.15** Given an account-wide limit, then `/sys/<class>/<id>/` shows the limit and what others hold, and the plan-time quota check uses what remains.
- **AC-29.16** Given a unit on a VM pool, then `expose` compiles to the pool's own enforcement (firewall rules, security groups) as module intent, and a pool with none reports `netpol: none` as ADR 0019 decision 6 does.
- **AC-29.17** Given a receipt, then it names the computer, its hardware model and the driver and library versions, and `diff` of two receipts shows them.
- **AC-29.18** Given a stage over inputs in two regions, then it is split per region, each part placed inside its residency, and only the stage's outputs cross, priced as an egress line.

## 9. Persona lenses

- **ML**: wants to write the pipeline once, see why each stage runs where it does and when it will start, and lose neither a checkpoint nor a log to another scheduler's requeue.
- **PE**: wants each computer to state what it can do (signals, bounds, identity, operator) so the planner refuses rather than pretends, and wants no class of compute the platform cannot see.
- **FIN**: wants an allocation shown as an allocation, functions and VMs priced in their own units, and one receipt per pipeline that adds them up.
- **SEC**: wants EU shards and their logs to stay in the EU, and no job on a shared cluster to run as an identity every tenant shares.
- **SRE**: wants the inference service's reach enforced on VMs as it is on clusters, and node faults on clusters the platform does not run to reach the people who do.
- **DE**: wants the preprocessing stage's partial failures counted, not dropped by an asynchronous retry policy.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L29.1 | none | Compute is a Kubernetes device class only: the planner places stages on `/dev/k8s`, and processes, exit codes, signals, logs and `/sys/k8s/<id>` are defined against the Kubernetes API. Functions, external schedulers, managed batch services and VM pools can be registered by driver packages but are devices nothing can run on, under classes the design does not name. | **open** (drafted 2026-11-12 in [ADR 0025](../adr/0025-a-computer-is-a-device-kubernetes-is-one-driver.md): compute is one device class `/dev/compute/` with `k8s`, `slurm`, `fn`, `batch` and `vm` kinds, `/dev/k8s` kept as a view; every driver fills one contract of fifteen fields; placement is one ordered filter over every mounted computer; no federated computer). The record asked for: a compute device class with one driver contract per kind of computer (partitioned stage, status, signals, stderr, image format, bounds, preemption notice, identity, price model, operator), Kubernetes as one driver among them, topology views unchanged | ADR 0003; ADR 0011 d1, d2, d9; ADR 0013 d8; ADR 0004; ADR 0008 |
| L29.2 | P2 | A function cannot be frozen, signalled or checkpointed; only its timeout ends it. `kill -STOP` was accepted and did nothing. | **rule** (applied: ADR 0008 rules; ADR 0013 rules): each computer lists the signals it delivers in `caps` (`signals=`); a signal outside the list is refused with exit 3 naming the computer; `TERM` on a function stage stops new invocations and lets running ones end within their bound; `pause` is refused where no checkpoint exists | ADR 0008; ADR 0013 d8, d10 |
| L29.3 | none | Partition count derives from input size only; on a computer with a maximum run time (900 s on Lambda), partitions sized for a pod run past it and die at the bound. | **amend** (applied: ADR 0003 decision revised) ADR 0003 decision: partition count derives from input size and from the target's `max_duration` with the executable's declared throughput; a partition that cannot fit is a plan-time refusal naming the bound and the next candidate | ADR 0003; ADR 0002; ADR 0004 |
| L29.4 | P12 | An external scheduler requeues preempted or drained jobs under the same job id: the rank logs are truncated, the attempt prefix restarts and the requeue counts as a failure. | **rule** (applied: ADR 0004 rules; ADR 0009 rules; ADR 0023 rules): the driver maps each requeue to a new attempt, opens stderr in append mode and prefixes `[index.attempt]` as ADR 0023 decision 7 does; a requeue by the scheduler is a preemption and never counts against `--retries`, as `DisruptionTarget` does not (ADR 0004) | ADR 0004; ADR 0009; ADR 0023 d7 |
| L29.5 | P4 | The Slurm driver submits through `slurmrestd` with one service account, so every Convoy user's job runs as one uid and writes files every tenant on the cluster can read. | **rule** (applied: ADR 0016 rules): each computer declares `identity=per-user\|per-group\|shared` in `caps`; per-user submission uses the exchange of ADR 0016 (a per-user token for `slurmrestd`, the user's mapped uid); under `shared` `explain` says that the job's files are owned by the shared identity, and a mount with `identity=required` refuses the placement | ADR 0016 d4, d5; ADR 0011 d4 |
| L29.6 | P5 | Cost bands assume pod-hours; an external cluster bills against an allocation in GPU-hours and prints $0.00, functions bill per request and GB-second, VMs per second with a minimum. | **rule** (applied: ADR 0010 rules; ADR 0009 rules): each computer declares its price model; an allocation is shown in its own units with the balance read from the scheduler (`sshare`), never converted to money without a rate the platform sets; the receipt settles each stage in its model and sums only what shares a unit | ADR 0010; ADR 0009 |
| L29.7 | P8 | Placement chooses by locality, then `caps`; it has no residency filter of its own and no notion of when a computer will start the work, so a stage waits 15 hours in a Slurm queue while a spot pool is idle. | **amend** (applied: ADR 0003 amendment 1 revised) ADR 0003 amendment 1: placement filters by residency, then `caps` and bounds, then locality, and ranks what is left by cost and estimated start; each computer reports an estimated start (`squeue --start`, queue depth) in `/sys/<class>/<id>/queue`; `explain` prints every candidate with its reason and estimate, and `--deadline` excludes a candidate that cannot start in time | ADR 0003 a1; ADR 0013 d3; ADR 0004 |
| L29.8 | P11 | The EU shards went to a function in us-east-1, and the function's logs are written in the region of the function, not of the data. | **rule** (applied: ADR 0011 rules; ADR 0023 rules): the residency of every mounted input binds the placement of the stage that reads it; a computer's log and metric sinks carry a topology, and a stage whose sink is outside the residency is refused | ADR 0011 a3; ADR 0016 d11; ADR 0023 d8 |
| L29.9 | P1 | The VM pool's instance template is a module input (ADR 0024); the driver patched it in place for `release`, and the next plan reverted the image. | **rule** (applied: ADR 0010 rules; ADR 0024 rules): on a computer installed by a module, `release` and `scale` write the image and the size as intent to the instance's module inputs (`/sys/<class>/<id>/iac/vars/`), and the provider's own rolling update applies them | ADR 0010 a1; ADR 0024 d6; ADR 0013 d7 |
| L29.10 | P12 | Inside an external scheduler, the pipeline's deadline is not passed to the job (the partition's 48 h limit applies), and steps started inside the job are invisible. | **rule** (applied: ADR 0004 rules): the driver compiles the remaining deadline into the scheduler's time limit (`--time`, `--deadline`) and the plan's members into its step count; steps inside the job are shown as members in `ps -T` | ADR 0004 a3; ADR 0008 a1 |
| L29.11 | P6 | The cluster runs Apptainer; the SIF was built on the login node from a tag, so the receipt names a digest that is not what ran. | **rule** (applied: ADR 0022 rules; ADR 0009 rules): a conversion to another image format starts from the digest, is a cached stage keyed by it (ADR 0001), and the receipt records the digest and the converted image's hash | ADR 0022; ADR 0001; ADR 0009 |
| L29.12 | P7 | Asynchronous invocations that were throttled or whose retries ran out were dropped by the runtime, and the stage reported success. | **rule** (applied: ADR 0004 rules): function stages invoke synchronously or with a failure destination the driver reads; throttles and failures are counted per partition, shown in `ps -T` and drive the contract | ADR 0004; ADR 0004 a2 |
| L29.13 | P3 | When the scheduler's controller was unreachable, `ps` showed the process failed while the job kept running, and nothing recorded the job's id. | **rule** (applied: ADR 0011 rules; ADR 0004 rules): the backend job id is written to `/sys/proc/<pid>/backend` at submission; an unreachable computer makes its processes `unknown`, never `failed` (ADR 0011 amendment 4); they are resolved from the scheduler's accounting when the link returns, and a `kill` meanwhile is queued | ADR 0011 a4; ADR 0004; ADR 0006 |
| L29.14 | P10 | The health controller found a GPU fault on a node of a cluster the platform does not operate and had no verb to act on it. | **rule** (applied: ADR 0021 rules; ADR 0017 rules): a computer declares `operator: platform\|external` and, if external, the operator's channel; on an external computer node verbs from the controller and from users become requests on that channel, and the controller only reports and excludes the node from placement | ADR 0021 d7; ADR 0017; ADR 0011 a5 |
| L29.15 | P9 | Function concurrency is an account-wide limit per region, and another team's reservation consumed most of it; the plan-time quota check saw only this plan. | **rule** (applied: ADR 0003 rules; ADR 0011 rules): account-wide limits are shown in `/sys/<class>/<id>/` with what others hold, and the plan-time check uses what remains | ADR 0003; ADR 0011 d7 |
| L29.16 | none | A unit on a VM pool has no NetworkPolicy and no cluster DNS; `expose` was refused rather than compiled to what the pool enforces. | **rule** (applied: ADR 0019 rules): each computer declares its enforcement (`netpol: networkpolicy\|firewall\|security-group\|none`); `expose` on a VM pool compiles to a load balancer and firewall rules written as module intent (ADR 0024); names come from the platform's DNS zone (decision 5) | ADR 0019 d5, d6; ADR 0024 |
| L29.17 | none | The same model evaluated on other GPUs, drivers and libraries gives slightly different numbers. | **accepted**: results are not promised to be identical across computers; the receipt names the computer, its hardware model, and the driver and library versions, and `diff` of two receipts shows them | ADR 0009 |
| L29.18 | P11 | A stage over inputs in two regions was placed in one, pulling the EU holdout to us-east-1. | **rule** (applied: ADR 0003 rules; ADR 0011 rules): a stage over inputs in several topologies is split per topology, each part placed inside its residency, and only the stage's outputs cross, priced as an egress line | ADR 0003 a1; ADR 0011 a3; ADR 0012 d5 |
