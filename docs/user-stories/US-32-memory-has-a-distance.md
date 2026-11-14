# US-32 · Memory has a distance: one NVLink domain for a training run, a cluster's memory for a Spark job, and a cache that is not a disk

| Field | Value |
|---|---|
| Primary persona | ML |
| Secondary personas | DE, DS, PE, SRE, FIN, SEC |
| Workload class | memory beyond one node: a tensor-parallel training run that must sit inside one 72-GPU NVLink domain, a Spark job sized against the cluster's aggregate memory, a shared in-memory cache used by two groups, and a node pool with CXL memory; one Kubernetes cluster in us-east-1 with two GB200 NVL72 racks, and one on-premises cluster |
| Walkthrough | written against [ADR 0025](../adr/0025-a-computer-is-a-device-kubernetes-is-one-driver.md) decisions 2, 4, 5 and 6, [ADR 0004](../adr/0004-completion-contract-and-exit-codes.md) amendments 1 and 3, [ADR 0008](../adr/0008-signal-routing.md) amendments 1 and 4, [ADR 0021](../adr/0021-a-node-is-tested-before-it-is-trusted.md) decisions 2, 3, 4 and 9, [ADR 0011](../adr/0011-device-tree-and-mount-table.md) decisions 7 and 9, [ADR 0001](../adr/0001-pipe-is-a-staged-artifact.md) amendment 2, [ADR 0003](../adr/0003-three-backends-local-first.md) amendment 1, [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md) decisions 2, 3 and 8, [ADR 0017](../adr/0017-approvals-are-objects-on-the-process.md) decision 2, [ADR 0024](../adr/0024-a-device-is-installed-by-terraform.md) decision 8, walkthroughs 2 and 5, and the command map row for `ipcs`; it follows US-31 |
| Gaps already closed there | G2 (coordinated groups), G4 (gang contract) |
| Milestones needed | M3, M5, M6, M8, M10 |
| Findings in this document | L32.1 – L32.19 |

## 1. Story

**As an** ML engineer, **I want** to ask for "one NVLink domain" the way I ask for a GPU, and to be told before anything runs whether a domain that large exists, is healthy and is free, **so that** a run that needs memory shared across 72 GPUs never starts split across two racks at a third of its speed with every exit code saying 0, a rack with two bad GPUs does not hold my run in a queue with no estimate, and the people next to me (a data engineer who wants "the cluster's memory" for a Spark job, a data scientist who puts features in a shared cache) know which memory is shared, which is only near, and which is gone when a node goes.

**Context.** After US-31 the platform runs ADRs 0001 to 0027. The ML computer is `k8s-eks-123456789012-us-east-1-ml-gb200`, mounted as `/clusters/gb200`. Its GPU pool is two GB200 NVL72 racks, `nvl-a` and `nvl-b`. Each rack is one NVLink domain of 72 GPUs on 18 nodes, four GPUs and two Grace CPUs per node, in which a GPU can read and write the memory of every other GPU in the rack. Between racks, and to the cluster's other pools, traffic goes over the RDMA network, not NVLink. Kubernetes sees the domain through NVIDIA's DRA driver: a `ComputeDomain` object, with IMEX daemons on its nodes, gives the pods of one workload the channels they need to share memory. Kueue admits gangs; its topology-aware scheduling is installed but not configured with the rack label.

The data team's computer is the same EKS cluster plus `k8s-capi-dc-sp-01`, an on-premises cluster that holds the `events` table and has a node pool with CXL memory expanders. The platform runs a remote shuffle service for Spark and a shared in-memory cache, `shm-alluxio-use1`, which nobody has yet described as a device. The research group runs its own Ray cluster through the KubeRay operator.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Trains the model | `yuki` | user `yuki`, group `llm` | needs 72 GPUs in one domain |
| Runs the Spark job | `rafael` | user `rafael`, group `analytics` | 2 TB of Parquet |
| Shares features | `noor` | user `noor`, group `research` | a KubeRay cluster, the shared cache |
| Owns GPU capacity | `capacity` | group `capacity` | decides who holds a whole rack |
| Health controller | the agent session of ADR 0021 | `svc:platform` | excludes GPUs |
| Platform | platform team | `svc:platform` | installs the DRA driver, the shuffle service and the cache |
| Pays | FinOps | group `finops` | |

## 3. Preconditions

- **Capabilities** (`caps`): on `gb200`, `dra`, `kueue`, `jobset`, `imex`; on `dc-sp-01`, `kueue`, `spark-operator`.
- **Packages**: `nvidia-dra-driver`, `kueue`, `jobset`, `spark-operator`, `celeborn` (the remote shuffle service), `alluxio`, `kuberay` (installed by `research`, not by the platform).
- **Platform defaults** `explain` must print: the memory domains of every computer a plan can use, their health and free size, the distance between the members a plan places, where a stage spills, and whether a store a plan writes to keeps what it holds.

## 4. Main path

```bash
# Monday 09:00 · yuki, group llm
$ cd /clusters/gb200/llm
$ batch --gang --replicas=18 --gpu=4 --checkpoint=/ckpt --deadline=96h --budget=60000 \
    torchrun train.py --tensor-parallel=72
pid 9400  JobSet llm/train-9400  18 replicas × 4 GPU  gang (Kueue, all-or-nothing)
#  Kueue admits 18 replicas: 10 nodes on nvl-a, 8 on nvl-b. NCCL finds no NVLink path between
#  the halves and falls back to the RDMA network; a step takes 3.1 times the benchmark.
#  Every rank is healthy, ps says running, and nothing will ever exit non-zero          (L32.1, L32.2)
$ ls /sys/compute/k8s-eks-123456789012-us-east-1-ml-gb200/
#  allocatable GPUs per pool; no racks, no NVLink domains, no word of which GPUs share memory (L32.1)

# Monday 11:00 · yuki kills 9400 and asks for "one rack"; there is no flag for it, so a node
#  selector on nvl-a's label goes into a pod template
#  nvl-b has 70 healthy GPUs: the health controller excluded gpu/1 on two nodes (ADR 0021)
#  nvl-a has 8 GPUs taken by 2-GPU notebook pods that the scheduler spread there        (L32.5)
#  Kueue keeps the gang pending; explain says "estimated start: unknown"                 (L32.3)
#  meanwhile 6 nodes of nvl-b run Spark executors that wanted their 480 GB of Grace memory (L32.5)

# Tuesday 02:00 · the notebooks end; 9401 is admitted on nvl-a
#  9401's evaluation step launches six child jobs; each inherits --gang and the node selector,
#  and each waits for a whole rack that its own parent holds                            (L32.15)

# Thursday 14:10 · an NVSwitch tray in nvl-a logs link errors
#  the health controller reads them as GPU faults and excludes gpu/1 on two nodes        (L32.6)
#  the gang restarts from /ckpt/step-41000 at 72 GPUs; no rack has 72 healthy; it waits,
#  and the 96-hour deadline keeps counting                                                (L32.4)
$ kill -9 9401
#  the ComputeDomain is deleted, but IMEX channels on four nodes are not released;
#  nvl-a shows 72 free and refuses the next claim for 40 minutes                          (L32.7)
#  the next claim on nvl-a is from research; nothing says the GPUs' memory was cleared   (L32.8)
#  a platform engineer relabels nvl-a's nodes with kubectl to "fix" the domain;
#  the DRA driver writes the labels back an hour later                                    (L32.12)
#  someone asks in capacity's channel who reserved nvl-b for nine days                   (L32.13)

# The same week · rafael, group analytics, on the same cluster
$ batch spark-submit --executors=40 --executor-memory=64g sessionize.py /tables/events/2026-11
#  2.5 TiB of executor memory for 2 TB of Parquet, which is about 7 TB once decoded in memory;
#  4.8 TB spills to the nodes' root disks, two nodes reach disk pressure and evict other pods (L32.16)
#  1.2 TB of shuffle files from the payments columns stay in the shared shuffle service
#  after the job, readable by the service's other tenants                                 (L32.14)
#  the retry is placed on dc-sp-01, where the table lives (ADR 0003 amendment 1); its pool
#  has CXL memory that the node shows as a NUMA node with no CPUs; executors are given it as
#  ordinary memory and run 40 % slower, and nothing says why                              (L32.1, L32.19)

# The same week · noor, group research
#  the KubeRay cluster keeps features in Ray's object store, which lives in the pods' /dev/shm;
#  no bound counts it; an out-of-memory kill takes out the head pod and the features     (L32.11)
$ mount /dev/shm/by-id/alluxio-use1 /shm         # what noor expects to exist; nothing does
#  noor writes features.parquet to the platform cache instead; rafael's job reads it there;
#  a node holding a third of the cache is replaced overnight and the next read returns 0 rows (L32.10)
#  noor maps /shm/weights.bin MAP_SHARED with write from pods on two nodes, expecting one memory (L32.9)

# Friday · "can we pool the memory of the us-east-1 cluster and the EU cluster for one query?" (L32.17)
#  yuki tries a 64-GPU run split over both racks on purpose; the plan's estimate is a guess  (L32.18)
```

| Step | Convoy object | Kubernetes or system object | Where the state lives |
|---|---|---|---|
| a gang | a process with `--gang` | JobSet and a Kueue workload | Kueue |
| one NVLink domain | none | `ComputeDomain`, IMEX daemons and channels, ResourceClaims (DRA) | the DRA driver |
| rack membership | none | node labels set by the driver | the nodes |
| a GPU exclusion | `/sys/k8s/<id>/nodes/<node>/gpu/<i>/online` | a device taint (ADR 0021) | the controller |
| Spark executor memory and spill | a process with a coordinated group | executor pods, `emptyDir` on the root disk | the nodes |
| shuffle | none | Celeborn workers | the shuffle service |
| a shared cache | none | Alluxio workers | the cache |
| a Ray object store | none | `/dev/shm` inside Ray pods | the pods |
| CXL memory | none | a CPU-less NUMA node | the node's kernel |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe: staged artifact, content id, cache | ● | a pipe artifact or checkpoint on a store that loses what it holds (L32.10) | AC-32.10 |
| A2 | Command: class, annotation, executable fields | ○ | `spark-submit` and `torchrun` as coordinated groups; Spark's memory estimate (L32.16) | AC-32.16 |
| A3 | Session: mount table, env, cwd, attach and detach | – | | |
| A4 | Backend selection | ● | placement by memory domain (L32.1, L32.3) | AC-32.3 |
| A5 | Completion contract | ● | exit 0 at a third of the speed (L32.2); a gang restart into a degraded domain (L32.4) | AC-32.2, AC-32.4 |
| A6 | Device tree | ● | no memory domains in `/sys` (L32.1); a shared-memory device (L32.9) | AC-32.9 |
| A7 | Signals and lifecycle | ● | `kill -9` and the domain's channels (L32.7) | AC-32.7 |
| A8 | Secrets as references | – | | |
| A9 | Cost | ● | GPUs stranded in a part-used domain; CPU work on GPU nodes (L32.5) | AC-32.5 |
| A10 | Identity and authorization | ● | a domain handed to another group (L32.8); who may hold a rack (L32.13) | AC-32.8, AC-32.13 |
| A11 | Platform defaults and `caps` | ● | `domain_reserve`, `domain_hold_max`, `mem_scrub` (L32.5, L32.8, L32.13) | AC-32.5 |
| A12 | Storage | ● | spill and shuffle as devices with bounds and residency (L32.14, L32.16) | AC-32.14, AC-32.16 |
| A13 | Networking | ○ | NVLink inside the rack, RDMA between racks (L32.2, L32.18) | |
| A14 | Time and queueing | ● | an estimated start for a domain (L32.3); a deadline that runs while a gang waits (L32.4) | AC-32.3 |
| A15 | Audit, receipts, retention and `replay` | ○ | placement and scrub in the receipts (L32.2, L32.8) | |
| A16 | Durability | ● | a volatile memory store (L32.10) | AC-32.10 |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative verb, declarative owner | rack labels and `ComputeDomain` objects belong to the DRA driver | ADR 0010 amendment 1 writes managed objects as intent; the driver is not GitOps | **L32.12** |
| P2 | POSIX semantics that cannot hold | `mmap` with `MAP_SHARED` across nodes; `ipcs` | the command map maps `ipcs` to Leases and named queues | **L32.9** |
| P3 | Orphaned or lost state | IMEX channels left after `kill -9` | ADR 0008 amendment 4 fences nodes; nothing about channels | **L32.7** |
| P4 | Identity leak or escalation | GPU memory handed to the next group | ADR 0016 decision 2 binds rights to groups; nothing about memory | **L32.8** |
| P5 | Invisible cost | eight GPUs stranded in a rack; CPU work on GPU nodes | ADR 0025 decision 9 prices the computer, not its fragments | **L32.5** |
| P6 | Lying cache | a read from a cache that lost a third of its data returns 0 rows | none | **L32.10** |
| P7 | Silent failure | a gang split across racks runs at a third of the speed with exit 0 | ADR 0004 amendment 1 admits all or nothing, not where | **L32.2** |
| P8 | Partial fleet | a rack with 70 healthy GPUs and a run that needs 72 | ADR 0021 decision 4 excludes the component; placement does not see the rack | **L32.3** |
| P9 | Created elsewhere | Ray's object store inside pods that KubeRay created | ADR 0024 decision 8: a device not installed by a module says so | **L32.11** |
| P10 | Multi-party action | one group holds a whole rack for nine days | ADR 0017 decision 2 gates come from policy | **L32.13** |
| P11 | Data at rest | spill on root disks; shuffle files in a shared service after the job | ADR 0001 amendment 2 places artifacts, not intermediate files of a package | **L32.14** |
| P12 | Process-tree inheritance | evaluation jobs inherit `--gang` and the rack selector | ADR 0004 amendment 3: the contract is inherited | **L32.15** |

Seven more questions without a probe: is there a memory domain at all (**L32.1**, open); does a gang restart wait for a shape no domain can hold (**L32.4**, amend); has the health controller a class for the fabric that joins GPUs (**L32.6**, amend); does `explain` size a Spark job's memory and say where it spills (**L32.16**, rule); can memory be shared across computers (**L32.17**, rule); is the speed of a stage split across domains known before it runs (**L32.18**, accepted); and can Kubernetes schedule CXL memory shared between hosts (**L32.19**, accepted).

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| gang placed across domains | plan (L32.2) | 3 when `--domain` is declared | the domains each member got | rerun with `--domain` |
| no healthy domain large enough | plan (L32.3) | 3 | the largest healthy domain and its size | a smaller `--replicas`, or wait for repair |
| domain degrades during a run | the gang contract (L32.4) | 3 at the deadline, or restart elsewhere | the domain, the excluded components, the restart target | `--replicas` that fits, from the checkpoint |
| fabric fault read as GPU faults | the health controller (L32.6) | none | the `fabric` scope and the domain | none; the domain is excluded |
| channels left after the domain ends | the domain claim (L32.7) | none | the members whose channels were not freed | the driver's cleanup; `explain` shows the domain `draining` |
| volatile store loses an object | the read (L32.10) | 1, `EIO` | the store, the object, `volatile` | rerun the producer |
| spill exceeds its device | Spark, then the plan on rerun (L32.16) | 1 | the predicted and actual spill and the device | a larger spill device or more executors |
| shared writable mapping across nodes | the device (L32.9) | 2, refused | the mapping asked for | write a new file and read it |

## 8. Acceptance criteria

- **AC-32.2** Given a gang with `--domain`, then it is admitted only inside one domain of that kind; given a gang without it, then `explain` prints the domains its members would span and `ps -T` and the receipt show the domains each member got.
- **AC-32.3** Given a plan that needs a domain, then `explain` prints the largest healthy domain and its free size, and refuses with exit 3 when no healthy domain is large enough; when one is large enough but busy, the estimated start comes from the domain's queue.
- **AC-32.4** Given a gang in a domain that loses members, then its restart from the checkpoint goes to any healthy domain that can hold it; if none can before the deadline, the process ends with exit 3 and the receipt names the domain and the excluded components.
- **AC-32.5** Given work that claims no domain, then it is placed outside pools with domains unless `domain_reserve` allows; GPUs stranded in a part-used domain are shown in `/sys` and priced to the platform, never to the work beside them.
- **AC-32.6** Given faults in an NVLink switch, a link or a domain daemon, then the health controller attributes them to the `fabric` class with the domain as scope, and excludes the domain rather than the GPUs.
- **AC-32.7** Given a domain whose claim ends, then it shows `draining` until every member's channels are verified free, and only then is it offered again.
- **AC-32.8** Given a domain or an accelerator handed to a different group, then its device memory is cleared first and the node receipt records it; a computer declares `mem_scrub=`, and a policy may refuse a computer with `mem_scrub=none`; one domain claim belongs to one process tree and one group.
- **AC-32.9** Given a file on a shared-memory device, then it is written once and read by any member; a writable shared mapping across nodes is refused with exit 2, and `ipcs` lists only the node's own objects.
- **AC-32.10** Given a shared-memory device, then `/sys` says `volatile` and names its `backing=` store or `none`; a read of a lost object fails with `EIO`, never an empty read; a checkpoint or a pipe artifact on a volatile device without backing is refused.
- **AC-32.11** Given an object store inside a process (Ray's, Spark's cache), then it is part of that process and counts in its memory bound; a store is a device only when it is installed as one.
- **AC-32.12** Given domain layout and membership, then Convoy reads them from the driver and never writes them; the only object Convoy creates is the claim.
- **AC-32.13** Given a claim of a whole domain longer than `domain_hold_max`, then it needs an approval from the group that owns the capacity.
- **AC-32.14** Given spill and shuffle, then spill goes to a bounded device named in the plan and is removed when the pod ends; a shuffle service is a device with residency and retention, and shuffle files are removed when the process ends.
- **AC-32.15** Given a child of a process with `--domain`, then the child does not inherit the domain; it may name `--domain=@parent` to run inside the parent's claim on its free members.
- **AC-32.16** Given a Spark or dataframe stage, then `explain` prints the input size on disk, its estimated size in memory, the stage's aggregate memory, the predicted spill and the spill device; spill to a node's root disk is refused.
- **AC-32.17** Given a request for a domain or a shared-memory device that spans computers, then it is refused; data moves between computers by copy with its digest.

## 9. Persona lenses

- **ML**: wants "one rack" to be a word the plan understands, a refusal when it cannot be had, and a restart that goes wherever a healthy rack is.
- **DE**: wants `explain` to say a job will spill 4.8 TB before it fills two nodes' disks.
- **DS**: wants to know which store keeps its data when a node goes, and which does not.
- **PE**: wants rack labels and channels owned by the driver, a scrub between groups, and small work kept out of big domains.
- **SRE**: wants a switch fault to be a fabric fault, not two GPU faults, and a domain that is not offered back until it is clean.
- **FIN**: wants stranded GPUs to show up somewhere and not on the bill of the job next to them.
- **SEC**: wants device memory cleared between groups and shuffle files gone when the job is.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L32.1 | none | Memory has distances the design never names. A computer's `/sys` lists allocatable GPUs and memory per pool, but not the NVLink domains in which GPUs share memory, nor the CPU-less NUMA nodes a CXL expander adds, nor the distance between any two members a plan places. Placement cannot ask for a domain, `explain` cannot refuse one, and the shared in-memory stores the platform runs are not devices. | **open**. The record asked for: memory domains as part of a computer (`/sys/compute/<id>/mem/`, with kind, members, health and free size), a distance class for every pair of places (`local`, `coherent`, `rdma`, `network`, other computer), memory tiers per node pool (near and far), a `--domain=<kind>` contract flag compiled to the driver's claim (a DRA `ComputeDomain` on Kubernetes), `numactl --hardware` as the view, and volatile shared-memory stores as a device class under `/dev/shm/` | ADR 0025 d2, d4; ADR 0011 d7, d9; ADR 0004 a1; ADR 0003 a1 |
| L32.2 | P7 | A gang of 72 GPUs was admitted across two racks and ran at a third of the speed with every exit code 0. | **rule**: a gang with `--domain` is admitted only inside one domain of that kind and is never split; without it, `explain` prints the domains its members would span; `/sys/proc/<pid>/placement`, `ps -T` and the receipt record the domain each member got | ADR 0004 a1; ADR 0025 d6; ADR 0003 a1 |
| L32.3 | P8 | A rack with 70 healthy GPUs and a run that needs 72: the gang waited with "estimated start: unknown". | **rule**: the capacity of a domain is its healthy free size, never the pool's sum; `explain` prints the largest healthy domain and refuses with exit 3 when none is large enough; when one is large enough but busy, the driver's `queue` field gives the estimate | ADR 0025 d2, d4; ADR 0021 d4 |
| L32.4 | none | The gang restarted from its checkpoint at 72 GPUs into a rack that had lost two, and waited while its deadline ran. | **amend** ADR 0004 amendment 1: restart-the-group-from-checkpoint goes to any healthy domain that can hold the gang; if none can before the deadline, the process ends with exit 3, and the receipt names the domain and the excluded components; a gang may declare `--shrink-to=<n>` to restart smaller where the program supports it | ADR 0004 a1; ADR 0021 d4 |
| L32.5 | P5 | Eight GPUs in a rack were stranded by 2-GPU notebooks, and Spark executors took six GB200 nodes for their CPU memory. | **rule**: work that claims no domain is placed outside pools with domains unless the platform default `domain_reserve` allows it; free members of a part-used domain that no waiting claim can use are shown as `stranded` in `/sys/compute/<id>/mem/` and priced to the platform's overhead, never to the work beside them | ADR 0010; ADR 0025 d9; ADR 0016 d8 |
| L32.6 | none | A fault in an NVLink switch tray was read as two GPU faults; the health controller has no class for the fabric that joins GPUs. | **amend** ADR 0021 decisions 2 and 3: a `fabric` component class (NVLink links and switches, the domain's daemons such as IMEX), with its passive sources and a bounded all-to-all probe during burn-in; the domain is a scope above the node, and a fabric fault excludes the domain rather than the GPUs | ADR 0021 d2, d3 |
| L32.7 | P3 | After `kill -9`, the domain object was gone but channels on four nodes were not released, and the rack refused the next claim for 40 minutes while showing 72 free. | **rule**: a domain claim is released only when every member has exited; the domain shows `draining` until the driver confirms every member's channels are free, and only then is it offered again; a domain stuck in `draining` past `drain_max` is a health event (ADR 0021) | ADR 0008 a1, a4; ADR 0021 d9 |
| L32.8 | P4 | A rack used by `llm` was handed to `research` with nothing saying its memory had been cleared. | **rule**: a domain or an accelerator handed to a different group has its device memory cleared first (a driver reset or the vendor's scrub), recorded in the node receipt; each computer declares `mem_scrub=reset\|vendor\|none`, and a residency or group policy may refuse a computer with `none`; one domain claim belongs to one process tree and one group | ADR 0016 d2, d3; ADR 0021 d9 |
| L32.9 | P2 | A writable `MAP_SHARED` mapping of the same file from pods on two nodes was expected to be one memory. | **rule**: a shared-memory device holds files written once and read by any member, like an object store; a writable shared mapping across nodes is refused with exit 2, and a read-only mapping is a local copy; `ipcs` lists only the node's own objects. Convoy never offers memory shared across nodes except through a domain the hardware provides | ADR 0011 d9; command map `ipcs` |
| L32.10 | P6 | The cache lost a third of its data when a node was replaced, and the next read returned 0 rows. | **rule**: a shared-memory device declares `volatile` and its `backing=` store or `none`; a read of an object it lost fails with `EIO` and is refetched only from a declared backing store, never returned empty; a checkpoint or a pipe artifact on a volatile device without backing is refused (extending L5.1) | ADR 0001 a2; ADR 0004 rules (L5.1); ADR 0011 a4 |
| L32.11 | P9 | Ray's object store lived in the pods' `/dev/shm`, counted in no bound, and went with the head pod. | **rule**: an object store inside a process (Ray's, Spark's cache, a framework's shared memory) is part of that process, counts in its memory bound, and goes with it; a store is a device only when it is installed as one, and a store not installed by a module is shown as such (ADR 0024 decision 8) | ADR 0024 d8; ADR 0016 d3; ADR 0025 d2 |
| L32.12 | P1 | A platform engineer relabelled a rack's nodes to "fix" the domain, and the driver wrote the labels back. | **rule**: domain layout, membership and health are read from the driver (its node labels and the claim's status) and never written by Convoy; the only object Convoy creates is the claim; the driver's configuration is part of the computer's install module | ADR 0010 a1; ADR 0024 d6; ADR 0021 d10 |
| L32.13 | P10 | One group held a whole rack for nine days and nobody had agreed to it. | **rule**: a claim of a whole domain for longer than the platform default `domain_hold_max` needs an approval from the group that owns the capacity (ADR 0017), and the hold is shown in `/sys/compute/<id>/mem/` with its group and end time | ADR 0017 d2; ADR 0016 d7 |
| L32.14 | P11 | Spark spilled to root disks, and 1.2 TB of shuffle files from payments columns stayed in a shared service after the job. | **rule**: spill goes to a bounded device named in the plan (local NVMe or a volume), removed when the pod ends; a shuffle service is a device with residency, retention and an owner per process, and a process's shuffle files are removed when it ends | ADR 0001 a2; ADR 0011 a3; ADR 0007 a2 |
| L32.15 | P12 | Six evaluation jobs inherited `--gang` and the rack selector and waited for a rack their parent held. | **rule**: `--domain` and `--gang` are not inherited (the contract's budget and deadline are, under ADR 0004 amendment 3); a child may name `--domain=@parent` to run inside the parent's claim on members it leaves free | ADR 0004 a3 |
| L32.16 | none | `explain` said nothing about a Spark job whose 2 TB of Parquet would be 7 TB in memory against 2.5 TiB of executors. | **rule**: for a Spark or dataframe stage `explain` prints the input size on disk, its estimated in-memory size from the format and schema, the stage's aggregate memory, the predicted spill and the spill device; spill to a node's root disk is refused | ADR 0003; pipeline compiler; walkthrough 2 |
| L32.17 | none | "Can we pool the memory of two clusters for one query?" | **rule** (cross-reference to ADR 0025 decision 5): no memory is shared across computers; a request for a domain or a shared-memory device spanning computers is refused, and data moves between computers by `cp` with its digest | ADR 0025 d5; ADR 0011 d6 |
| L32.18 | none | A run split across two domains on purpose: the plan cannot know its step time. | **accepted**: Convoy reports the distance class between members, not the bandwidth a program will reach; the estimate for a stage at `rdma` distance comes from the package's annotation or from earlier receipts, and is marked as such | ADR 0003 a1 |
| L32.19 | none | Executors were given CXL memory as ordinary memory, and memory shared between hosts over CXL is not something Kubernetes can schedule. | **accepted**: Convoy shows CXL memory only as the tiers and NUMA nodes the node reports, places by tier when the driver reports one, and offers no memory shared between hosts beyond what a driver exposes as a domain | ADR 0025 d2 |
