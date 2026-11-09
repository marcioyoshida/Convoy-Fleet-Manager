# US-16 · Find the node that is hanging my jobs, take it out of service without losing anyone's data, and get a healthy one back

| Field | Value |
|---|---|
| Primary persona | SRE |
| Secondary personas | PE, ML, DBA, FIN, SEC |
| Workload class | fleet operation: node fault isolation, cordon, drain, fence, replace |
| Walkthrough | none; written against [ADR 0011](../adr/0011-device-tree-and-mount-table.md) decision 7 (`/sys` metadata) and amendment 4, [ADR 0008](../adr/0008-signal-routing.md) and its amendments, [ADR 0004](../adr/0004-completion-contract-and-exit-codes.md) amendment 1, [ADR 0010](../adr/0010-platform-defaults-and-two-package-tiers.md) amendment 1; it also takes the "debugging one failed partition of a thousand" gap from the register |
| Gaps already closed there | none |
| Milestones needed | M3, M8, M10 |
| Findings in this document | L16.1 – L16.7 |

## 1. Story

**As an** SRE, **I want** to find which machine is behind a pipeline that has stopped making progress, read what is wrong with it in one place, take it out of service in a way that respects every workload on it, and have the pool give me a healthy replacement, **so that** a bad disk, a dead GPU or a memory module throwing errors costs the fleet one node for an hour rather than a night of stuck jobs and a corrupted volume.

**Context.** Three faults recur on the GPU pool: an NVMe that stops completing I/O while the node still reports Ready, a GPU that falls off the bus (XID 79) and leaves training ranks hanging, and a DIMM whose correctable error count climbs until an uncorrectable error kills a pod. Today each is a different runbook across `kubectl describe node`, `nvidia-smi` over SSH, `dmesg`, the cloud console and a ticket. The last incident force-deleted a pod stuck in disk wait, which let a second node mount the same volume while the first still held it.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | the on-call SRE | `sudo --as platform` on the cluster device | cordon, drain and node deletion are platform verbs |
| Approves | a second SRE for fencing while the node is powered on | | L16.7 |
| Pays | the platform cost centre for the cordoned hours; the workloads for their lost time | | L16.6 |
| Audits | the incident review; the hardware vendor for the RMA | the evidence artifact the cordon receipt pins | |
| Operates the platform | the same team | node agent, DCGM, EDAC and hung-task collectors, the node pool manager (Karpenter, Cluster API or a managed node group) | |

## 3. Preconditions

- **Capabilities**: `node-agent` (dmesg, hung-task and EDAC counters, GPU reset), `dcgm` (GPU health and XID events), `node-pool:karpenter` (or `cluster-api`, `managed-node-group`; the manager of the pool), `pdb` (eviction API), `non-graceful-shutdown` (the out-of-service taint, Kubernetes 1.28+).
- **Packages**: `cnpg` (US-06) for the database unit on the node; `torchrun` (US-05).
- **Mounts**: `/clusters/prod-gpu` under the platform identity.
- **Annotations**: `torchrun: gang, group torchrun, checkpoint_seconds 600, freeze_max_seconds 540`; `cnpg/cluster: scale delegate, member_signals none`.
- **Platform defaults** `explain` must print: the health thresholds that mark a node suspect (correctable ECC rate, hung-task seconds), the drain grace per workload class, who may fence a powered-on node, the pool manager per node pool, the evidence retention class.

## 4. Main path

```bash
$ sudo --as platform
$ top
PID   TYPE         PROGRESS         STUCK            NODE
7810  indexed-job  998/1000 done    part 17 D 9 min  gpu-b-14
7790  jobset       rank 5 no heartbeat 4 min          gpu-b-14

$ status 7810 | grep -v done
part 17   running 41 min   attempt 1   node gpu-b-14   state D (blocked in kernel I/O on nvme1n1)

$ cat /sys/k8s/self/nodes/gpu-b-14/health
disk    hung tasks 3 on nvme1n1, oldest 540 s         (node-agent)
gpu     3/4 healthy; gpu3 XID 79 fallen off the bus 09:12   (dcgm)
memory  EDAC ce 1 842 (+310/h, threshold 100/h), ue 0  (node-agent)
k8s     Ready=True  DiskPressure=False                  (conditions say nothing is wrong)
verdict suspect: disk, gpu, memory

$ dmesg gpu-b-14 | tail -3
[09:12:04] NVRM: Xid (PCI:0000:41:00): 79, GPU has fallen off the bus.
[09:14:11] INFO: task postgres:2211 blocked for more than 120 seconds.
[09:15:30] EDAC MC1: 310 CE memory read error on CPU_SrcID#1_MC#1_Chan#0_DIMM#0

$ echo 0 > /sys/k8s/self/nodes/gpu-b-14/online            # cordon
reason: nvme hung tasks, gpu3 XID 79, ECC CE storm
node gpu-b-14 cordoned   evidence: health + dmesg excerpt pinned as artifact 3f1c…   receipt 7823

$ ps --node gpu-b-14
PID   TYPE         OWNER     ON THIS NODE
7810  indexed-job  dana      part 17 (D state)
7790  jobset       ml-team   rank 5 of 64 (--gang)
7311  unit db-2    ?         cnpg replica 2 of 3 (created elsewhere)

$ kill -TERM --node gpu-b-14                               # drain
7790 rank 5: group stop-with-checkpoint, restart from checkpoint 09:10 on other nodes (10 min lost)   done 2 min
7311 db-2: switchover delegated to the cnpg operator, PDB respected; replica 2 decommissioned     done 40 s
7810 part 17: cannot terminate, blocked in kernel I/O; pod remains until the node is fenced       pending
node gpu-b-14 drained except 1 process   receipt 7824

$ echo 1 > /sys/k8s/self/nodes/gpu-b-14/fenced
refused (exit 3): node is powered on. Fencing a running kernel lets a second node attach pvc-7810-17 while this one still holds it.
  alternatives: rm the node (the pool manager powers it off and fencing follows), or power it off at the cloud and retry.

$ rm /sys/k8s/self/nodes/gpu-b-14
node gpu-b-14 is managed by karpenter (NodeClaim gpu-b-14-xk2): deleting the claim, Karpenter replaces it in pool gpu-b
instance i-0abc… terminating … off 09:31   fenced: out-of-service taint set; pvc-7810-17 detached
7810 part 17: attempt 2 scheduled on gpu-b-07                                                     receipt 7825

$ wait 7810
exit 0            # 1000/1000, part 17 attempt 2, 999 partitions were already done and are not rerun

$ ls /sys/k8s/self/nodes/ | grep gpu-b-
gpu-b-01 … gpu-b-13  gpu-b-15 … gpu-b-21 (joined 09:34, hotplug node-added)

$ cat /sys/k8s/self/nodes/gpu-b-14/receipt
cordoned 09:16 by sre-ana reason … evidence 3f1c… · drained 09:18 (3 processes, 1 unterminatable) · fenced 09:31 after power off · replaced by gpu-b-21 09:34 · cordoned hours billed 0.25 ($0.98)
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `top`, `status` | partitions with node and kernel state | pod status plus node agent process state | receipts |
| `cat …/nodes/<n>/health`, `dmesg <n>` | node metadata files | node conditions, DCGM, EDAC and hung-task counters through the node agent | `/sys` cache with age |
| `echo 0 > …/online` | cordon with reason and evidence | `spec.unschedulable`, an event, an evidence artifact | Node object, artifact store, receipt |
| `kill -TERM --node` | drain as signals per backing object | eviction API with PDBs; gang stop-with-checkpoint; operator switchover | receipts per process |
| `echo 1 > …/fenced` | fence | `node.kubernetes.io/out-of-service` taint | refused until powered off |
| `rm …/nodes/<n>` | replace as intent to the pool manager | NodeClaim or Machine deletion; cloud instance termination; a new node | pool manager, cloud, hotplug event |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ○ | 999 finished partitions are cache hits; only part 17 reruns | the receipt shows one attempt 2 |
| A2 | Command | – | | |
| A3 | Session | ○ | `sudo --as platform` for node verbs | |
| A4 | Backend | ○ | the rescheduled partition lands on another node | `status` shows the new node |
| A5 | Contract | ● | attempt 2 under `--retries`; a gang restart counts as one retry | AC-16.5 |
| A6 | Device tree | ● | nodes under `/sys/k8s/<id>/nodes/<name>/` with writable `online` and `fenced` (L16.1) | the files exist and `ls` lists nodes with state |
| A7 | Signals | ● | drain as `kill -TERM --node`: gang, delegated unit, unterminatable process (L16.3, L16.7) | each process's receipt names its path |
| A8 | Secrets | – | | |
| A9 | Cost | ● | cordoned hours still bill; planner capacity excludes the node (L16.6) | the node receipt prices the cordon; `explain` quota excludes it |
| A10 | Identity | ○ | platform identity for node verbs; a second person for fencing while powered on | |
| A11 | Defaults | ● | thresholds, grace per class, fence policy, pool manager, `caps` for health sources (L16.5) | all in `explain`; `health` says `unknown` without `dcgm` |
| A12 | Storage | ● | a volume held by a hung node; RWO re-attach only after fencing | AC-16.4 |
| A13 | Networking | – | | |
| A14 | Time | ○ | hung-task seconds, drain grace | |
| A15 | Audit | ● | the node receipt and the pinned evidence artifact | AC-16.7 |
| A16 | Durability | ● | the pipeline, the training run and the database survive one node | AC-16.5 |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | the node belongs to Karpenter or Cluster API; deleting the Node object alone is reverted or leaves an instance | `rm` is intent to the pool manager (ADR 0010 a1), the manager must be discoverable | **L16.4** |
| P2 | POSIX cannot hold | `kill -KILL` on a process blocked in kernel I/O does not return, on Linux or here | the design says KILL is grace 1 s | **L16.7** |
| P3 | Orphaned state | a force-deleted pod's volume stays attached to the dead node | fencing before re-attach (L16.7) | ok with L16.7 |
| P4 | Identity leak | `dmesg` of a node shows other tenants' process names | platform identity only; the evidence artifact carries class `sensitive` | ok |
| P5 | Invisible cost | a cordoned node bills until replaced; capacity shown to the planner includes it | nothing in ADR 0003 excludes cordoned allocatable | **L16.6** |
| P6 | Lying cache | `health` from a `/sys` cache older than the fault | age shown; `cat --fresh` forces the agent | ok |
| P7 | Silent failure | without DCGM the GPU is "healthy" because nothing says otherwise; `Ready=True` throughout | a missing source must read `unknown` | **L16.5** |
| P8 | Partial fleet | the node agent on the faulty node is itself hung | `health` marks the agent's fields `stale` with age; conditions still come from the API | ok |
| P9 | Created elsewhere | the cnpg replica was not started through Convoy | `ps --node` shows owner `?`; drain uses the eviction API, which respects its PDB (S9) | ok |
| P10 | Multi-party | a hardware RMA is a vendor ticket | `page:` with the evidence artifact (`caps: alerting`) | ok |
| P11 | Data at rest | the evidence artifact (dmesg excerpt) | retention class from the platform; pinned by the node receipt | ok |
| P12 | Inheritance | where did partition 17 run, which attempt, on which node? | receipts record no node today | **L16.2**; also the gap "debugging one failed partition" |

A thirteenth question for this story: nothing in the design names a node at all. Clusters are devices and pods are processes; the layer where hardware fails is invisible. **L16.1**.

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| partition blocked in kernel I/O | `top`, `status` (L16.2) | running | node, state D, seconds | cordon, drain, fence, replace |
| GPU fallen off the bus | `health` from DCGM | n/a | XID, time, index | `echo 1 > gpu/3/reset` first; replace if it recurs |
| correctable ECC above threshold | `health` from EDAC | n/a | rate and threshold | cordon and replace at the next window |
| uncorrectable ECC | pod killed, MCE in dmesg | 1 for the partition | node, attempt | retry lands elsewhere; cordon |
| fence requested while powered on | pre-flight (L16.7) | 3 | the volume at risk | `rm` the node or power off |
| pool is static (bare metal, k3s site) | `rm` pre-flight (L16.4) | 3 | no pool manager | `page:` the hardware owner; node stays cordoned |
| node agent hung | `health` field age | n/a | `stale` per field | the API-side conditions still read |
| drain blocked by a PDB for longer than the grace | eviction API | 1 after grace | the unit and its PDB | operator switchover, or `--force` with a second identity |

## 8. Acceptance criteria

- **AC-16.1** Given `/sys/k8s/<id>/nodes/<name>/`, then `health`, `dmesg`, `online`, `fenced`, `managed-by`, `price` and `receipt` exist; `health` reads `unknown` for every source whose capability is absent, never `ok`.
- **AC-16.2** Given a running partition, then `status <pid>` and `top` name its node, attempt and kernel state, and the receipt records the node of every attempt.
- **AC-16.3** Given `echo 0 > online`, then the node is unschedulable, a reason was prompted for, an evidence artifact is pinned by the node receipt, and `explain` of any new plan excludes the node's capacity.
- **AC-16.4** Given `echo 1 > fenced` on a powered-on node, then it is refused naming the volumes at risk; given the instance is off, then the out-of-service taint is set and the volumes detach.
- **AC-16.5** Given `kill -TERM --node`, then a `--gang` process restarts from its checkpoint counting one retry, a delegated unit switches over through its operator, a process in kernel wait is reported as unterminatable and the drain completes without it, and every process's receipt names its path.
- **AC-16.6** Given `rm` on a node whose pool has a manager, then the manager's object is deleted as intent, the instance is terminated, fencing follows power-off, and a replacement joins with a hotplug event; given a static pool, then `rm` is refused and names the `page:` route.
- **AC-16.7** Given the node receipt, then it lists cordon, drain, fence and replacement with identities and times, prices the cordoned hours, and pins the evidence.

## 9. Persona lenses

- **SRE**: wants one file that says what is wrong and one verb per step, each refusing when it would corrupt data; refuses if fencing can run against a live kernel.
- **PE**: owns the pool managers and the thresholds; wants replace to be intent to Karpenter or Cluster API, never a hand-deleted Node object.
- **ML**: wants a dead GPU to cost ten minutes and one retry, not a hung run until morning.
- **DBA**: wants the drain to go through the operator's switchover and the PDB, and the volume never attached twice.
- **FIN**: wants cordoned hours on the node's receipt and out of the planner's capacity.
- **SEC**: wants the dmesg excerpt classified and the cordon reason and identity in the receipt.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L16.1 | none | Nodes exist nowhere in the design: a cluster is a character device and a pod is a process, so there is no path for the layer where disks, GPUs and memory fail, and no verb for cordon, fence or replace. | **amend** ADR 0011 decision 7: `/sys/k8s/<id>/nodes/<name>/` holds per-node files fed by the API and the node agent: `health` (verdict and per-source lines), `dmesg`, `conditions`, `gpu/<i>/health` and `gpu/<i>/reset`, `edac`, `price`, `managed-by`, `receipt`; two files are writable under the platform identity, `online` (cordon, with a prompted reason and pinned evidence, the sysfs convention for taking a CPU offline) and `fenced` (the out-of-service taint, decision L16.7); `rm` on the node directory is replacement; `ls` lists nodes with state | ADR 0011, ROADMAP M8, M10 |
| L16.2 | P12 | Receipts record partitions and attempts but not where they ran; `ps`, `top` and `status` have no node column, so the first step of every hardware incident is guesswork. This is also the register's open gap "debugging one failed partition of a thousand". | **rule**: the receipt records node and attempt per partition and per group member; `status <pid>` lists partitions with node, attempt and kernel state (from the node agent) and `top` surfaces partitions stuck in D state; `ps --node <name>` and `ps -o node` exist | ADR 0009, ADR 0013 decision 5 |
| L16.3 | P12 | Draining a node is a signal to every process on it, and each has a different right answer: a gang restarts from checkpoint, a delegated unit switches over, a bare pod is evicted under its PDB, a process created elsewhere is still evicted. | **rule**: `kill -TERM --node <name>` routes per backing object as ADR 0008 does for a pid: group stop-with-checkpoint counting one retry (ADR 0004 a1), operator switchover for `scale: delegate`, the eviction API for everything else so PDBs hold; owner `?` processes are included and listed; the drain receipt names each process's path and the ones it could not terminate | ADR 0008, ADR 0004 a1 |
| L16.4 | P1 | A node is owned by a pool manager (Karpenter, Cluster API, a managed node group) or by nobody; deleting the Node object directly either gets reverted or leaves a billed instance behind. | **rule**: `managed-by` under the node names the manager and its object; `rm` on a node is written as intent to that object (ADR 0010 a1), fencing follows the instance's power-off, and the replacement is the manager's; with no manager `rm` is refused and names the `page:` route for the hardware owner | ADR 0010 a1, ADR 0011 |
| L16.5 | P7 | GPU health comes from DCGM and memory health from EDAC through the node agent; without them a faulty GPU reads as healthy because nothing says otherwise, and `Ready=True` all along. | **rule**: every `health` source is a capability (`dcgm`, `node-agent`); an absent source reads `unknown`, never `ok`, and `explain` on a GPU plan prints `gpu health: unknown` when `dcgm` is missing | ADR 0010, ADR 0011 d7 |
| L16.6 | P5 | A cordoned node bills until it is replaced, and its allocatable still counts in the planner's plan-time quota check, so `explain` promises capacity that cannot schedule. | **rule**: cordoned and fenced nodes are excluded from allocatable in `/sys/k8s/<id>/` and from the plan-time check (ADR 0003); the node receipt prices cordoned hours and `df`-style output for the pool shows them | ADR 0003, ADR 0011 d7 |
| L16.7 | P2, P3 | A process blocked in kernel I/O cannot be terminated by TERM or KILL, on Linux or here, so "KILL uses grace 1 s" is false for exactly the case this story is about. Force-deleting its pod lets the volume attach to a second node while the first still holds it, which is how the last incident corrupted a database. | **amend** ADR 0008: `kill` on a process in uninterruptible wait reports `unterminatable` and does not force-delete; the honest verb is at the node: `fenced` sets the out-of-service taint and is refused while the node is powered on unless a second identity approves, because fencing a live kernel permits two writers; after power-off (by the pool manager on `rm`, or by hand) fencing proceeds, volumes detach and the partition is rescheduled | ADR 0008, ADR 0006, ADR 0011 (L16.1) |
