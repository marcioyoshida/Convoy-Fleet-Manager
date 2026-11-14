# ADR 0025: A computer is a device, and Kubernetes is one driver of it

**Status:** Draft · 2026-11-12 · rules from user story US-29 added 2026-11-13
**Supersedes / relates to:** open finding L29.1 (compute is a Kubernetes device class only); ADR 0003 and its amendment 1 (three backends, placement by locality); ADR 0011 decisions 1, 2, 3, 6, 7 and 9 and amendments 1, 3, 4 and 5 (clusters as devices, `by-id` and views, mounts, `all`, `/sys`, the same rules for every class, drivers from packages, `residency`, `disconnected`, the node tree); ADR 0004 and its amendments (the completion contract); ADR 0008 (signals); ADR 0009 and ADR 0023 decision 7 (stderr as one path); ADR 0013 decisions 4, 7, 8 and 11 (contract flags, unit verbs, device verbs and `caps`, no flag outside the record); ADR 0016 decisions 4 and 5 (open is a credential exchange, `svc:<group>`); ADR 0019 decisions 5 and 6 (the platform's boundary, unenforced reach reported); ADR 0021 decision 7 (what the health controller may do); ADR 0024 (installed by Terraform); US-29 findings L29.2 to L29.18

## Context

Everything in this design runs on Kubernetes. A cluster is a character device under `/dev/k8s/` (ADR 0011 decision 1), a process is a pod or a Job, exit codes come from `successPolicy` and `backoffLimitPerIndex` (ADR 0004), signals are pod deletions and cgroup freezes (ADR 0008), stderr is the container log (ADR 0023 decision 7), metadata is `/sys/k8s/<id>/` and the planner's distributed tier is an Indexed Job or an Argo Workflow (ADR 0003). The rest of the design (devices, mounts, residency, approvals, receipts, cost) never needed Kubernetes and does not mention it.

US-29 ran one pipeline across the computers an ML team actually uses: functions on Lambda and Cloud Run for preprocessing, a Slurm cluster at a partner university for training, AWS Batch spot capacity for evaluation and a VM pool for a legacy inference server, with shards resident in two regions. Driver packages could register the four as devices, but nothing could run a stage on them, under classes (`/dev/fn`, `/dev/slurm`, `/dev/batch`, `/dev/vm`) the design does not name. The findings that followed are each one field of a contract nobody wrote: which signals a computer delivers (L29.2), how long a partition may run (L29.3), what a requeue is (L29.4), whose identity a job runs as (L29.5), what it costs and in which unit (L29.6), when it will start (L29.7), where its logs are written (L29.8), who operates it (L29.14).

Facts that constrain the design:

- The computers differ in kind, not only in vendor. A function has no signals and a hard maximum run time; Slurm has its own queue, accounting, requeue and time limits and runs Apptainer images; a managed batch service is a queue in front of instances; a VM pool has no scheduler at all. An abstraction at the intersection of their features would be too weak for any of them.
- Kubernetes already reaches some of them through bridges (Virtual Kubelet providers, Slurm operators and the Slinky project, Interlink). Where a site runs one, the computer is a Kubernetes cluster and needs nothing new; most sites do not.
- Multi-cloud and multi-region are already answered for clusters by topology in the path (ADR 0011 decision 2): the planner reads a region from a device path, prices the egress line and places by locality. Federation was declined (ADR 0011 decision 6, KubeFed is archived).
- Every computer must be installed, opened and paid for the same way every other device is (ADR 0024, ADR 0016, ADR 0010).

## Decision

1. **Compute is one device class, and a Kubernetes cluster is one kind of computer.** Every computer is a device under `/dev/compute/`, keyed in `by-id` by the provider's own identifier with a kind prefix, and shown through the same generated views as every class (ADR 0011 decisions 2 and 9). `/dev/k8s` is kept as the view `by-kind/k8s`, and `/sys/k8s/<id>/` as the same directory as `/sys/compute/<id>/`, so no path in any earlier record changes.

   ```
   /dev/compute/
   ├── by-id/
   │   ├── k8s-eks-123456789012-us-east-1-prod-main
   │   ├── slurm-hpc-sp
   │   ├── fn-lambda-123456789012-us-east-1
   │   ├── fn-cloudrun-acme-ml-europe-west4
   │   ├── batch-awsbatch-123456789012-us-east-1-spot-gpu
   │   └── vm-mig-acme-ml-europe-west4-infer
   ├── by-kind/{k8s,slurm,fn,batch,vm}/…
   ├── by-provider/<provider>/<region>/…
   ├── by-topology/…      by-label/…
   └── self
   /dev/k8s -> compute/by-kind/k8s
   ```

2. **Each kind has a driver, and every driver fills one contract.** A compute driver comes from a package (ADR 0011 amendment 1) and must supply every field below; a driver with an empty field is not installable, as a native verb with an empty field is not finished (ADR 0013 decision 2). The fields are the computer's `caps` and `/sys/compute/<id>/` entries, and the planner reads nothing else.

   | Field | What the driver supplies | Read by |
   |---|---|---|
   | `stage` | how a partitioned stage, a gang and a unit map to backend objects, or `none` for each | ADR 0003, ADR 0013 decision 7 |
   | `status` | the mapping from backend states to process states, `unknown` included | `ps`, `wait` |
   | `exit` | the mapping from backend outcomes to contract codes 0 to 3 and the degraded channel | ADR 0004 |
   | `signals=` | the signals it delivers and what each does | `kill`, `pause` (ADR 0008) |
   | `stderr` | where each attempt's stderr is, and how attempts are numbered | ADR 0009, ADR 0023 decision 7 |
   | `image=` | the formats it runs (`oci`, `sif`, `zip`) and the conversion from an OCI digest | ADR 0022, ADR 0001 |
   | `bounds` | `max_duration`, `max_partitions`, concurrency, memory and accelerator limits, account-wide limits with what others hold | the planner |
   | `preemption` | the notice in seconds, and whether the backend requeues | ADR 0008, `--retries` |
   | `identity=` | `per-user`, `per-group` or `shared`, and the exchange that opens it | ADR 0016 decision 4 |
   | `price` | the price model (pod-hour, request and GB-second, instance-second with a minimum, allocation units with a balance) | ADR 0010, receipts |
   | `queue` | an estimated start for a stage of a given shape, or `unknown` | placement (decision 4) |
   | `netpol=` | the enforcement reach compiles to (`networkpolicy`, `firewall`, `security-group`, `none`) | ADR 0019 |
   | `operator=` | `platform` or `external`, and the external operator's channel | ADR 0021, ADR 0017 |
   | `sinks` | the log and metric stores the computer writes to, with their topology | ADR 0023, residency |
   | `backend` | the job id it writes to `/sys/proc/<pid>/backend` at submission | `ps`, reattach |

3. **Five kinds ship first, and the table above says what each can do.** The first drivers are `k8s` (unchanged), `slurm`, `fn` (Lambda, Cloud Run, Azure Functions), `batch` (AWS Batch, Google Cloud Batch, Azure Batch) and `vm` (auto scaling groups, managed instance groups, scale sets). Other schedulers (PBS, LSF, HTCondor, Nomad, Ray) are further driver packages under the same contract.

   | Contract item | k8s | slurm | fn | batch | vm |
   |---|---|---|---|---|---|
   | partitioned stage | Indexed Job | job array | one invocation per partition | array job | one process per instance slot |
   | gang | JobSet | one job, `srun` steps per rank | `none` | multi-node job where offered | `none` |
   | unit | Deployment | `none` | service (an HTTP-triggered function, a Cloud Run service) | `none` | instance group with rolling update |
   | signals | TERM, KILL, STOP, CONT | through `scancel --signal` | `none` (the bound ends it) | TERM, KILL | TERM, KILL, STOP, CONT through the node agent |
   | image | oci | sif, converted from the digest | oci or zip | oci | oci |
   | preemption | the pool's notice | the partition's preempt mode, requeue | none | the spot notice | the spot notice |
   | operator | platform or external | usually external | platform | platform | platform |

   A VM pool has no scheduler: its driver places one partition per instance slot through the node agent (ADR 0008) that its module installs, and Convoy schedules nothing finer than that.

4. **Placement is one ordered filter over every computer the session has mounted.** The backend tier is still chosen from input size (ADR 0003); the distributed tier now targets any computer whose `stage` field supports the stage's shape. Among those, the planner removes computers outside the residency of any input or whose `sinks` lie outside it, then those whose `caps` or `bounds` cannot run the stage (partitions are sized to fit `max_duration` before a computer is refused), then prefers the computer where the largest input is local (ADR 0003 amendment 1), and ranks what is left by price band and estimated start. `explain` prints every candidate with the step that kept or removed it, and `--deadline` removes a candidate whose estimated start cannot meet it. `--on=<path>` pins a stage to a mounted computer and is added to the contract flags of ADR 0013 decision 4; a pin that fails a filter is a refusal, never a silent move.

5. **There is no federated computer, and regions stay in the path.** A computer is one provider account or project, one region or site, one scheduler. Multi-region work is several computers, and the planner splits a stage over inputs in several topologies into one part per topology, each placed inside its residency, with only the outputs crossing as a priced egress line (ADR 0012 decision 5). A unit that serves from several regions is one unit per computer under one address (ADR 0019, `--global`). `/dev/compute/all` is the read-only union of ADR 0011 decision 6, so `ps` and `top` list processes on every mounted computer, and an unreachable computer is listed as `disconnected`, never omitted.

6. **A process is the same object on every computer.** `/proc/<pid>` and `/sys/proc/<pid>/` are unchanged; `/sys/proc/<pid>/backend` adds the kind, the computer and the backend's job id, written at submission. Process states come from the driver's `status` field. A process on an unreachable computer is `unknown`, never `failed`, until the scheduler's accounting resolves it when the link returns (ADR 0011 amendment 4), and a `kill` in the meantime is queued. stderr is one path across attempts on every kind (ADR 0023 decision 7), and a backend's requeue or preemption is a new attempt that never counts against `--retries`, as `DisruptionTarget` does not on Kubernetes (ADR 0004).

7. **Opening a computer is a credential exchange per kind.** `mount` of a computer exchanges the session's login token for that kind's credential (ADR 0016 decision 4): a Kubernetes token, a per-user `slurmrestd` token mapped to the user's uid at the site, an IAM role or service account for the owning group for `fn`, `batch` and `vm`. Launched processes run as `svc:<group>` (ADR 0016 decision 5) where the computer has groups, and as the mapped user where it has only users. A computer that accepts work only from one shared account reports `identity=shared`; `explain` says that files the job writes are owned by it, and a mount with `identity=required` refuses the placement.

8. **A computer is installed like every device, and may be run by someone else.** Computers the platform runs are installed by a module (ADR 0024): a cluster, a function runtime's account settings, a batch compute environment and queue, an instance group with its template. Writes to what a module owns are intent: `release` and `scale` on a `vm` unit write the image and the size to the instance's module inputs, and the provider's rolling update applies them. A computer the platform does not run (a university cluster, a partner's batch queue) is `iac: none` and `operator=external`: every verb of use works, node verbs from people and from the health controller become requests on the operator's channel, and the controller only reports and excludes the node from placement.

9. **Cost is settled in the computer's own unit.** Each computer's `price` field is its model; `explain` prints the band in it and the receipt settles in it. An allocation is shown as units used and the balance read from the scheduler, never as money unless the platform sets a rate, and a pipeline's receipt sums only lines that share a unit.

## Declined

| Proposal | Why not | What replaces it |
|---|---|---|
| a generic job API at the intersection of every backend | too weak for any of them; hides what each computer cannot do | one contract whose fields each driver fills, and refusals that name the field (decisions 2, 4) |
| a class per kind (`/dev/slurm`, `/dev/fn`, as US-29 found them) | the planner must compare a function, a queue and a cluster for the same stage | one class with `by-kind` views (decision 1) |
| Kubernetes in front of everything through bridges as the only path | most sites run no bridge; a bridge hides the scheduler's own limits | a bridged site is a `k8s` computer; others get a driver |
| one federated computer across regions or providers | partial failures, no placement for writes, the reason ADR 0011 declined federation | one computer per region or site, and per-topology splits (decision 5) |
| Convoy scheduling inside VM pools | a second scheduler to operate | one partition per instance slot through the node agent (decision 3) |
| converting allocations to money by default | a price nobody pays; FinOps and the PI read different numbers | settlement in the computer's unit (decision 9) |

## Consequences

- L29.1 is drafted here. [US-29](../user-stories/US-29-one-pipeline-many-computers.md) was applied on 2026-11-13: L29.3 and L29.7 revised ADR 0003 inline (the decision and amendment 1), which decision 4 assumes; its fourteen rules are in the Rules section below, with pointers in ADRs 0003, 0004, 0008, 0009, 0010, 0011, 0013, 0016, 0017, 0019, 0021, 0022, 0023 and 0024; L29.17 is accepted.
- ADR 0011 decision 1 reads as one kind of decision 1 here; `/dev/k8s` and `/sys/k8s/<id>/` remain as views. ADR 0003's distributed tier is no longer Indexed Job or Argo only; on Kubernetes it still is. ADR 0013 decision 4 gains `--on`. The node tree of ADR 0011 amendment 5 exists only where the driver exposes nodes (`k8s`, `slurm`, `vm`).
- New `caps` keys: `stage`, `signals=`, `image=`, `bounds`, `preemption`, `identity=`, `price`, `queue`, `netpol=`, `operator=`, `sinks`. New paths: `/dev/compute/`, `/sys/compute/<id>/queue`, `/sys/proc/<pid>/backend`.
- ROADMAP: the compute class, its views and `/sys/compute/` are M8 with the fleet; the five driver packages are M9; placement by the ordered filter and `--on` are M3 with the distributed backend; identity modes and the per-kind exchange are M7; price models are M4 with receipts.
- Open, to be settled against this record: moving data to a computer's own file system before a stage (a Slurm cluster's Lustre, a function's ephemeral disk) and back; interactive work on a scheduler (`salloc`, a notebook on a GPU node); GPU clouds that are neither functions nor batch (per-second GPU containers); computers at the edge with no provider API; reconciling an allocation's accounting with the platform's receipts; a story that probes this record.


Note · 2026-11-14: the draft [ADR 0028](0028-memory-has-a-distance-and-a-domain-is-claimed-whole.md) adds a field to decision 2's contract, `mem` (memory tiers per pool, domains, and the distance class between places), adds the domain fit to decision 4's filter after caps and bounds, and extends decision 5 to memory: nothing is shared across computers.

Note · 2026-11-14: the draft [ADR 0029](0029-a-computers-file-systems-are-in-its-contract-staging-is-a-planned-copy.md) settles the open item on moving data to a computer's own file system: decision 2's contract gains the `fs` field (file systems, quotas, purge, isolation, residency and reach), and decision 4's filter removes computers that cannot hold a stage's staged data and ranks by the staging cost.

Note · 2026-11-14: the draft [ADR 0030](0030-an-interactive-process-holds-a-computer-while-someone-is-attached.md) settles the open item on interactive work on a scheduler: decision 2's `stage` field gains an `interactive` shape per kind (a pod with a terminal or a browser application, `salloc` and `srun --pty`, a VM, `none` on functions and batch), and allocation use outside Convoy is shown in `cost`.
## Rules · 2026-11-13 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L29.2** (US-29) each computer lists the signals it delivers in `caps` (`signals=`); a signal outside the list is refused with exit 3 naming the computer and the signal; `kill -TERM` on a function stage stops new invocations and lets the running ones end within their bound; `pause` is refused where the computer has no checkpoint.
- **L29.4** (US-29) a requeue or preemption by the computer's own scheduler is a new attempt: stderr is opened in append mode and prefixed `[index.attempt]` (ADR 0023 decision 7), and the requeue never counts against `--retries`, as `DisruptionTarget` does not on Kubernetes (ADR 0004).
- **L29.5** (US-29) each computer declares `identity=per-user|per-group|shared`; per-user submission uses the exchange of ADR 0016 decision 4 (a per-user `slurmrestd` token, the user's mapped uid); under `shared`, `explain` says that files the job writes are owned by the shared identity, and a mount with `identity=required` refuses the placement.
- **L29.6** (US-29) each computer declares its price model; an allocation is shown in its own units with the balance read from the scheduler, never converted to money without a rate the platform sets; the receipt settles each stage in its model and sums only lines that share a unit.
- **L29.8** (US-29) the residency of every mounted input binds the placement of the stage that reads it; a computer's log and metric sinks carry a topology, and a stage whose computer writes them outside the residency is refused.
- **L29.9** (US-29) on a computer installed by a module, `release` and `scale` write the image and the size as intent to the instance's module inputs (`/sys/<class>/<id>/iac/vars/`), and the provider's own rolling update applies them.
- **L29.10** (US-29) on an external scheduler the driver compiles the remaining deadline into the job's time limit and the plan's members into its step count; steps inside the job are shown as members in `ps -T`.
- **L29.11** (US-29) a conversion to another image format (SIF, a function's archive) starts from the OCI digest and is a cached stage keyed by it (ADR 0001); the receipt records the digest and the converted image's hash.
- **L29.12** (US-29) function stages invoke synchronously or with a failure destination the driver reads; throttles and failures are counted per partition, shown in `ps -T` and drive the contract.
- **L29.13** (US-29) the backend job id is written to `/sys/proc/<pid>/backend` at submission; an unreachable computer makes its processes `unknown`, never `failed` (ADR 0011 amendment 4); they are resolved from the scheduler's accounting when the link returns, and a `kill` meanwhile is queued.
- **L29.14** (US-29) a computer declares `operator=platform|external` and, if external, the operator's channel; on an external computer node verbs from people and from the health controller become requests on that channel, and the controller only reports and excludes the node from placement.
- **L29.15** (US-29) account-wide limits (function concurrency, batch vCPU quotas, a scheduler's per-account limits) are shown in `/sys/<class>/<id>/` with what others hold, and the plan-time quota check uses what remains.
- **L29.16** (US-29) each computer declares its enforcement (`netpol=networkpolicy|firewall|security-group|none`); `expose` on a VM pool compiles to a load balancer and firewall rules written as module intent (ADR 0024); names come from the platform's DNS zone (ADR 0019 decision 5).
- **L29.18** (US-29) a stage over inputs in several topologies is split into one part per topology, each placed inside its residency, and only the stage's outputs cross, priced as an egress line.
- **L30.10, L30.11** (US-30) `verify=` and the registry or mirror a computer pulls from are fields of `/sys/compute/<id>/` (full rules in [ADR 0026](0026-a-registry-is-a-device-a-tag-is-a-name-for-a-digest.md)).
- **L32.3, L32.11, L32.17** (US-32) the driver's `mem` field gives each domain's healthy free size, which placement uses; stores inside a process count in its `bounds`; no memory is shared across computers (full rules in [ADR 0028](0028-memory-has-a-distance-and-a-domain-is-claimed-whole.md)).
- **L33.4, L33.12, L33.14, L33.16** (US-33) the `fs` field gives capacity per task, shared quotas and metadata limits, which placement and `explain` use; staging is priced in the computer's unit (full rules in [ADR 0029](0029-a-computers-file-systems-are-in-its-contract-staging-is-a-planned-copy.md)).
- **L34.2, L34.9, L34.10, L34.13, L34.15** (US-34) each kind's `interactive` shape, its start estimate and its wall time are in the contract; allocation use outside Convoy appears in `cost` (full rules in [ADR 0030](0030-an-interactive-process-holds-a-computer-while-someone-is-attached.md)).
