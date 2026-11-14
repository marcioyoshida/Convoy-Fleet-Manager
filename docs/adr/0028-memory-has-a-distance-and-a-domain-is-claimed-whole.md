# ADR 0028: Memory has a distance, and a domain is claimed whole

**Status:** Draft · 2026-11-14 · rules from user story US-32 added 2026-11-14
**Supersedes / relates to:** open finding L32.1 (memory domains, distances and tiers are not modelled; shared in-memory stores are not devices); ADR 0025 decisions 2, 4, 5 and 9 (the computer contract, placement as one ordered filter, no federated computer, cost in the computer's unit); ADR 0004 amendments 1 and 3 (the gang contract, the inherited contract); ADR 0008 amendments 1 and 4 (coordinated groups, fencing); ADR 0021 decisions 2, 3, 4 and 9 (component classes, attribution, exclusion, receipts); ADR 0011 decisions 7 and 9 and amendments 1 and 4 (`/sys`, the same rules for every class with `/dev/accel` as DRA claims, drivers from packages, `disconnected`); ADR 0001 amendment 2 (the artifact store per topology); ADR 0003 amendment 1 (placement by locality); ADR 0013 decision 4 (contract flags); ADR 0016 decisions 2, 3 and 8 (rights to groups, objects belong to a group, the group is the cost centre); ADR 0017 decision 2 (gates from policy); ADR 0024 decision 8 (a device not installed by a module says so); walkthroughs 2 and 5; the command map row for `ipcs`

## Context

Memory is not one pool. Inside a node, Linux already says so: NUMA nodes with a distance table, and CXL memory expanders shown as NUMA nodes with no CPUs. Inside a rack, an NVLink domain (a GB200 NVL72 is one domain of 72 GPUs on 18 nodes) lets every GPU read and write every other GPU's memory, which no network gives. Between racks, the fabric carries messages and one-sided RDMA reads, not loads and stores. Between computers there is a network measured in milliseconds, and between regions an egress line. Frameworks that look like "the cluster's memory" are partitioned memory with messages between the parts: Spark's executors and its shuffle, Ray's object store, a cache such as Alluxio over an object store.

The design names none of this. A computer's `/sys` lists allocatable GPUs and memory per pool (ADR 0011 decision 7, ADR 0025 decision 2); `--gang` admits all or nothing (ADR 0004 amendment 1) but not where; `/dev/accel` is named as DRA claims (ADR 0011 decision 9) and never described. US-32 spent a week in that gap: a 72-GPU gang admitted across two racks at a third of its speed with every exit code 0, a rack two GPUs short and a gang queued with no estimate, an NVSwitch fault read as two GPU faults, channels left after `kill -9`, a rack handed to another group with nothing said about its memory, a Spark job that spilled 4.8 TB onto root disks, a cache that lost a third of its data and returned 0 rows, and a request to pool two clusters' memory for one query.

Facts that constrain the design:

- Shared memory across devices exists only where hardware provides it, and it is exposed by a driver: on Kubernetes, NVIDIA's DRA driver gives a workload a `ComputeDomain` and the IMEX channels its pods need; on Slurm, the scheduler's topology constraints place a job inside a block of switches. Software distributed shared memory has been tried many times and loses to its own coherence traffic.
- The size of a domain that can be claimed is the size of its healthy, free part, never the pool's sum; one excluded GPU makes a 72-GPU domain a 71-GPU domain.
- Domain layout, membership and health are written by the driver and the fabric manager; anything else that writes them is overwritten.
- Device memory and host memory are not always cleared between workloads; whether they are depends on the driver and the hardware.
- In-memory stores lose what they hold when a node goes, unless they are a cache over a store that keeps it.

## Decision

1. **Memory is described by distance, per computer, and the driver reports it.** The computer contract (ADR 0025 decision 2) gains a field, `mem`, that every driver fills: the memory tiers of each node pool (`near` for HBM and DRAM, `far` for CXL expanders and other CPU-less NUMA nodes), the domains the computer has, and the distance class between any two places a plan can use. The classes are ordered and are classes, not numbers:

   | Class | Means | Example |
   |---|---|---|
   | `local` | the same device or NUMA node | one GPU's HBM |
   | `coherent` | loads and stores across devices inside a domain | NVLink inside an NVL72 rack; NUMA nodes of one host |
   | `rdma` | one-sided reads and messages over the fabric between domains | InfiniBand, EFA, RoCE |
   | `network` | messages over the cluster network | pods in other pools |
   | `computer` | another computer; data moves only by copy | another cluster, another region |

   `/sys/compute/<id>/mem/` holds them, and `numactl --hardware <computer>` is the view:

   ```
   /sys/compute/k8s-eks-123456789012-us-east-1-ml-gb200/mem/
   ├── tiers          gpu-gb200: hbm near, lpddr near (coherent with the GPUs); cpu-m7: dram near
   ├── distance       local < coherent (nvlink) < rdma (efa) < network
   └── domains/
       ├── nvl-a      kind=nvlink members=72 healthy=72 free=64 stranded=8 state=claimed group=llm until=…
       └── nvl-b      kind=nvlink members=72 healthy=70 free=70 state=free
   ```

   Layout, membership, health and state are read from the driver (its node labels and its claims' status) and never written by Convoy. A memory bound (`--mem=`) means near memory; a stage that can use far memory says `--mem-tier=far|any`, and placement counts a pool's near memory only, unless the stage says otherwise.

2. **Shared memory is a domain the hardware provides, and Convoy never assembles one.** Memory is shared across devices only inside a domain a driver exposes: an NVLink domain, the NUMA nodes of one host, a CXL pool when a driver exposes one as a domain. Convoy offers no distributed shared memory in software, no coherence across nodes outside a domain, and nothing across computers: a request for a domain or a shared-memory device that spans computers is refused, and data moves between computers by `cp` with its digest (ADR 0025 decision 5, ADR 0026). Between domains of one computer, programs use the fabric through their own libraries (NCCL, MPI, Spark's shuffle); Convoy places them and reports the distance, and does not carry their traffic.

3. **`--domain=<kind>` is a contract flag, and a claim is whole.** `--domain=nvlink` (or `numa`, or another kind a driver reports) is a contract flag on every creating verb (ADR 0013 decision 4). The planner compiles it to the driver's claim: a DRA claim and a `ComputeDomain` on Kubernetes, the scheduler's topology constraint on Slurm, a placement group on a VM pool. A gang with `--domain` is admitted only inside one domain of that kind and is never split. Without the flag, `explain` prints the domains the members would span and the distance between them. `/sys/proc/<pid>/placement`, `ps -T` and the receipt record the domain each member got. The members of one claim are one coordinated group (ADR 0008 amendment 1): `STOP` on one member is refused, and a signal goes to the group.

4. **A domain's capacity is its healthy free size, and placement says which one fits.** In the ordered filter of ADR 0025 decision 4, a stage with `--domain` keeps only computers with a domain of that kind whose healthy size can hold it, after residency and caps and before ranking. `explain` prints the largest healthy domain and its free size. When no healthy domain is large enough, the plan is refused with exit 3 and names the excluded components and their repair state; when one is large enough but busy, the estimate comes from the driver's `queue` field. A gang that loses members of its domain during a run restarts from its checkpoint in any healthy domain that can hold it (ADR 0004 amendment 1 as revised by L32.4), or at `--shrink-to=<n>` where the program supports it, and ends with exit 3 at its deadline if none can.

5. **A claim belongs to one process tree and one group, and is not inherited.** A claim is created for one process and owned by its group (ADR 0016 decision 3); two groups never share one. `--domain` and `--gang` are not inherited by children, unlike the contract's budget and deadline (ADR 0004 amendment 3); a child that should run inside its parent's claim says `--domain=@parent` and uses members the parent leaves free. A claim of a whole domain for longer than the platform default `domain_hold_max` needs an approval from the group that owns the capacity (ADR 0017 decision 2), and every hold is shown in `/sys/compute/<id>/mem/domains/` with its group and end time.

6. **A claim ends by draining, and memory is cleared before another group.** A domain moves `free`, `claimed`, `draining`, `free`. It leaves `claimed` only when every member has exited, and leaves `draining` only when the driver confirms every member's channels are released; a domain in `draining` longer than `drain_max` is a health event (ADR 0021). Before a domain or an accelerator serves a different group, its device memory is cleared by a driver reset or the vendor's scrub, and the node receipt records which (ADR 0021 decision 9). Each computer declares `mem_scrub=reset|vendor|none` in `caps`, and a residency or group policy may refuse a computer that declares `none`.

7. **The fabric is a component, and the domain is a scope.** ADR 0021 gains a component class `fabric`: the links and switches that join a domain and the daemons that run it (IMEX on NVLink domains), with the driver's and the fabric manager's counters as passive sources and a bounded all-to-all transfer as the burn-in probe. A fault in the fabric is attributed to the domain, a scope above the node, and excludes the domain rather than the GPUs on either side of it (ADR 0021 decisions 2 and 3 as revised by L32.6).

8. **Domains are kept for work that needs them, and stranding is shown.** Work that claims no domain is placed outside pools whose nodes belong to domains unless the platform default `domain_reserve` allows it; pools with domains are for claims first. Free members of a part-used domain that no waiting claim can use are shown as `stranded` and priced to the platform's overhead (ADR 0010), never to the work beside them, so the cost of fragmentation is visible to whoever sets `domain_reserve`.

9. **Shared-memory stores are devices under `/dev/shm/`, and they are volatile.** In-memory stores the platform runs for several processes are devices under `/dev/shm/by-id/`, installed by a module (ADR 0024), with kinds `cache` (over a backing store, as Alluxio), `store` (keys and objects with no backing, as Redis or a Ray cluster run as a service) and `shuffle` (a remote shuffle service). `/sys/shm/<id>/` says `volatile`, names its `backing=` device or `none`, and carries residency, retention and a price. A file on such a device is written once and read by any member, as on an object store; a writable shared mapping across nodes is refused with exit 2, and a read-only mapping is a local copy. A read of an object the store lost fails with `EIO` and is refetched only from a declared backing store, never returned empty. A checkpoint or a pipe artifact is never written to a volatile device without backing (L5.1, ADR 0001 amendment 2). A store that runs inside a process (Ray's object store in a job's pods, Spark's cache, a framework's use of `/dev/shm` in a pod) is part of that process, counts in its memory bound and ends with it. `ipcs` lists only the node's own objects.

10. **Memory is sized before it is used, and spill has a device.** For a Spark or dataframe stage, `explain` prints the input size on disk, its estimated size in memory from the format and the schema, the stage's aggregate memory, the predicted spill and the device the spill goes to. Spill goes to a bounded device named in the plan (local NVMe or a volume), removed when the pod ends; spill to a node's root disk is refused. A shuffle that leaves the process's pods goes to a `shuffle` device, which keeps a process's files under the process's group and removes them when the process ends.

## Declined

| Proposal | Why not | What replaces it |
|---|---|---|
| distributed shared memory across nodes in software | coherence traffic and false sharing cost more than the copies they hide, and the distance stays, unseen | domains the hardware provides (decision 2) and stores with object semantics (decision 9) |
| one memory pool across clusters or regions | a remote read is about a million times slower than local GPU memory | copies by digest (decision 2; ADR 0025 decision 5) |
| distances as numbers (latencies, bandwidths) | they change with load and hardware generation and would be read as promises | ordered classes (decision 1), and estimates from receipts marked as such (L32.18) |
| Convoy writing domain labels or domain objects | the driver owns them and writes them back | Convoy creates the claim only (decisions 1, 3) |
| a domain's free GPUs counted with the pool's | a gang is then split across domains | the healthy free size of one domain (decision 4) |
| `/dev/shm` with tmpfs semantics across nodes | a writable shared mapping across nodes cannot hold | written-once files and refused shared writes (decision 9) |

## Consequences

- L32.1 is drafted here. [US-32](../user-stories/US-32-memory-has-a-distance.md) was applied on 2026-11-14: L32.4 revised ADR 0004 amendment 1 inline (a gang restarts into any healthy domain, or ends with exit 3) and L32.6 revised ADR 0021 decisions 2 and 3 inline (the `fabric` class and the domain scope), which decisions 4 and 7 assume; its fourteen rules are in the Rules section below, with pointers in ADRs 0001, 0003, 0004, 0007, 0008, 0010, 0011, 0013, 0016, 0017, 0021, 0024 and 0025; L32.18 and L32.19 are accepted.
- ADR 0025 decision 2 gains the `mem` field, decision 4's filter gains the domain fit, and decision 5 now covers memory. ADR 0004 amendment 1's gang may carry `--domain`; ADR 0013 decision 4 gains `--domain` and `--mem-tier`. ADR 0011 decision 9's `/dev/accel` DRA claims are how a domain claim compiles on Kubernetes, and `/dev/shm` is a new class under it. ADR 0021 gains the `fabric` class and the domain scope. ADR 0008 amendment 1's coordinated groups include the members of a claim.
- New paths: `/sys/compute/<id>/mem/` with `tiers`, `distance` and `domains/`, `/sys/proc/<pid>/placement`, `/dev/shm/` and `/sys/shm/<id>/`. New platform defaults: `domain_reserve`, `domain_hold_max`, `drain_max`. New `caps`: `domain:<kind>`, `mem_scrub=`.
- ROADMAP: placement by domain, domain capacity in `explain` and memory sizing for Spark and dataframe stages are M3; the `/dev/shm` drivers and volatile semantics are M5 with mounts; domain claims, holds, the reserve and gang restarts into another domain are M6 with Kueue; draining, scrub and the `fabric` class are M8; the DRA driver mapping and the store modules are M9; the fabric probe on the node is M10.
- Open, to be settled against this record: CXL memory shared between hosts, once a scheduler can place it; memory shared between a CPU and its GPUs (Grace and Hopper or Blackwell) as a tier a stage can ask for; inference servers that spill a model's cache across tiers and nodes; a story that probes this record.

## Rules · 2026-11-14 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L32.2** (US-32) a gang with `--domain` is admitted only inside one domain of that kind and is never split; without it, `explain` prints the domains its members would span; `/sys/proc/<pid>/placement`, `ps -T` and the receipt record the domain each member got (decision 3).
- **L32.3** (US-32) a domain's capacity is its healthy free size, never the pool's sum; `explain` prints the largest healthy domain and refuses with exit 3 when none is large enough; when one is large enough but busy, the driver's `queue` field gives the estimate (decision 4).
- **L32.5** (US-32) work that claims no domain is placed outside pools with domains unless `domain_reserve` allows it; free members of a part-used domain that no waiting claim can use are shown as `stranded` and priced to the platform's overhead, never to the work beside them (decision 8).
- **L32.7** (US-32) a domain claim is released only when every member has exited; the domain shows `draining` until the driver confirms every member's channels are free, and only then is offered again; a domain in `draining` past `drain_max` is a health event (decision 6).
- **L32.8** (US-32) a domain or an accelerator handed to a different group has its device memory cleared first, by a driver reset or the vendor's scrub, recorded in the node receipt; each computer declares `mem_scrub=reset|vendor|none`, and a residency or group policy may refuse `none`; one claim belongs to one process tree and one group (decisions 5, 6).
- **L32.9** (US-32) a file on a `/dev/shm` device is written once and read by any member; a writable shared mapping across nodes is refused with exit 2, and a read-only mapping is a local copy; `ipcs` lists only the node's own objects (decision 9).
- **L32.10** (US-32) a `/dev/shm` device declares `volatile` and its `backing=` device or `none`; a read of a lost object fails with `EIO` and is refetched only from a declared backing store, never returned empty; a checkpoint or pipe artifact on a volatile device without backing is refused (decision 9).
- **L32.11** (US-32) an object store inside a process (Ray's, Spark's cache, a framework's `/dev/shm`) is part of that process, counts in its memory bound and ends with it; a store is a device only when installed as one, and one not installed by a module says so (decision 9).
- **L32.12** (US-32) domain layout, membership and health are read from the driver and never written by Convoy; the only object Convoy creates is the claim; the driver's configuration is part of the computer's install module (decisions 1, 3).
- **L32.13** (US-32) a claim of a whole domain longer than `domain_hold_max` needs an approval from the group that owns the capacity; every hold is shown in `/sys/compute/<id>/mem/domains/` with its group and end time (decision 5).
- **L32.14** (US-32) spill goes to a bounded device named in the plan, removed when the pod ends; a shuffle service is a `/dev/shm` device of kind `shuffle` with residency, retention and an owner per process, and a process's shuffle files are removed when it ends (decision 10).
- **L32.15** (US-32) `--domain` and `--gang` are not inherited; a child may name `--domain=@parent` to run inside the parent's claim on members it leaves free (decision 5).
- **L32.16** (US-32) for a Spark or dataframe stage `explain` prints the input size on disk, its estimated size in memory, the stage's aggregate memory, the predicted spill and the spill device; spill to a node's root disk is refused (decision 10).
- **L32.17** (US-32) no memory is shared across computers; a request for a domain or a `/dev/shm` device spanning computers is refused, and data moves between computers by `cp` with its digest (decision 2).
- **L34.14** (US-34) interactive work is kept out of domain pools by `domain_reserve` unless `--gpu=whole` (full rule in [ADR 0030](0030-an-interactive-process-holds-a-computer-while-someone-is-attached.md)).
