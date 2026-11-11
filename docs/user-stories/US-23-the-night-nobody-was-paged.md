# US-23 · Let the fleet take its own faulty nodes and components out of service overnight, and find in the morning only what needed me

| Field | Value |
|---|---|
| Primary persona | SRE |
| Secondary personas | ML, PE, FIN, SEC, DBA |
| Workload class | fleet operation without a person present: GPU, shared-filesystem and memory faults detected by probes, attributed, excluded and cordoned by the controller; a gang drained automatically; a platform repairer; an edge site without its link |
| Walkthrough | written against [ADR 0021](../adr/0021-a-node-is-tested-before-it-is-trusted.md), with [ADR 0011](../adr/0011-device-tree-and-mount-table.md) amendments 4 and 5 (disconnected devices, the node tree), [ADR 0008](../adr/0008-signal-routing.md) amendments 1 and 4 (group stop, unterminatable processes), [ADR 0004](../adr/0004-completion-contract-and-exit-codes.md) amendment 1 (retries), [ADR 0017](../adr/0017-approvals-are-objects-on-the-process.md) (approval requests), [ADR 0014](../adr/0014-assisted-completion-the-model-proposes.md) (the model as a device, the outbound scrub) and [ADR 0010](../adr/0010-platform-defaults-and-two-package-tiers.md) amendment 1 (`rm` as intent to the pool manager); it reuses the GPU pools of US-16 |
| Gaps already closed there | none |
| Milestones needed | M2, M8, M10 |
| Findings in this document | L23.1 – L23.12 |

## 1. Story

**As an** SRE on call for the GPU fleet, **I want** the fleet to take faulty GPUs, filesystem clients and nodes out of service at night without waking me, to stop and page me when what it sees looks like something other than a sick node, and to leave me in the morning a short list of what it did and what only I can do, **so that** a dead GPU costs the training team minutes rather than a night, and an automation bug cannot empty a pool.

**Context.** US-16 was handled by hand at 09:16. Since then the platform has installed the probe package and switched pool `gpu-c` (Karpenter, 22 nodes of 8 GPUs) from `node_health_mode=shadow` to `act`, with `auto_drain: gang` and a budget of 2 controller cordons per pool and 1 per zone per hour. Pool `gpu-d` is a managed node group with the cloud's auto-repair, reported as `caps: node-repair`. The training team's run, pid 9700, is a 64-rank JobSet over 8 nodes with `--gang`, a checkpoint every 30 minutes to the shared filesystem `efs-train`, and `--retries=1`, which the team set for its own code. A cnpg replica (`db-2`, created elsewhere) runs on `gpu-c-03`. The edge site `factory-3` runs a two-node k3s cluster with one GPU each and an inference unit; its link is down from 01:00 to 06:00. Nobody is awake between 01:00 and 07:30.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | the controller's health agent | `svc:platform-health` | ADR 0021 decision 6 |
| On call | `sre-ana` | user `sre-ana`, group `platform` | paged; answers approval requests |
| Owns the training run | `ml-team` | group | pid 9700, `--gang`, `--retries=1` |
| Owns the replica | `?` | created elsewhere | cnpg operator, PDB `db` |
| Pays | the platform cost centre for probes, burn-in, inferences and cordoned hours; `ml-team` for lost GPU hours | | |
| Audits | the incident review; security for what reached the model | evidence artifacts, the model mount's audit record | |
| Operates the platform | platform team | probe package, node agent v2.3 then v2.4, DCGM, Karpenter, the cloud's auto-repair on `gpu-d`, the health model device `/dev/model/by-id/health-eu` | |

## 3. Preconditions

- **Capabilities**: `node-agent`, `dcgm`, `node-pool:karpenter` (`gpu-c`), `node-pool:managed-node-group` with `node-repair: cloud-auto-repair` (`gpu-d`), `fs-probe:efs`, `dra-device-taints` absent (the device plugin marks GPUs), `non-graceful-shutdown`.
- **Mounts**: `/clusters/train` (the cluster device, `self` in this story), `/fs/train` on `efs-train` (one mount target per zone); the model device `health-eu` with `residency=eu-central-1`, `scope=judge`, `retention=0`.
- **Platform defaults** `explain` must print: `node_health_policy` v3 and its rules, `fault_spread` 25%, `exclusion_max` 15 min, `verdict_min_confidence` 0.8, `node_health_budget` per pool and zone, `collector_stale_max`, `auto_drain` per pool, `burn_in` per pool (`gpu-c`: full, 25 min), `health_suppress` 4 h.

## 4. Main path

```bash
# 02:58  gpu-c-07: XID 79 on gpu2, rank 18 of pid 9700 holds it
[controller] gpu-c-07 gpu2: XID 79 (dcgm, 0 s)   rule xid-79 -> exclude gpu2, reset
[controller] gpu-c-07 gpu2 excluded (device plugin: unhealthy; allocatable nvidia.com/gpu 8 -> 7)
[controller] gpu-c-07 gpu2 reset: refused by driver, gpu in use by pid 9700 rank 18              (L23.2)
[controller] rule reset-failed -> cordon   budget gpu-c 1/2, zone b 1/1 per hour
[controller] gpu-c-07 cordoned (auto)   evidence 51ae…   receipt 9701
[controller] auto_drain gpu-c: gang -> kill -TERM --node gpu-c-07
[controller] 9700: group stop-with-checkpoint, grace 600 s; rank 18 cannot checkpoint (gpu2 off the bus)
[controller] 9700: checkpoint timed out at 600 s; restart from checkpoint 02:30 (28 min lost)      (L23.4)
[controller] 9700: retry 1 of 1 used                                                            (L23.11)
[controller] gpu-c-07 drained: 1 process, 0 unterminatable, no volumes attached
[controller] rm gpu-c-07 -> approval request 9711 to platform (decision 7)                       (L23.1)
[controller] 9700 restart: 7 nodes free, Karpenter adds gpu-c-22 (untested, burn-in 25 min)
#        status 9700 at 03:05 read "pending: waiting for capacity", with no start time         (L23.8)

# 03:20  efs-train: hung tasks on gpu-c-03, gpu-c-04, gpu-c-05, all in zone a
[controller] fs/efs-train no response on gpu-c-03 (stat 30 s timeout), gpu-c-04, gpu-c-05
[controller] peers: efs-train mounted on 40 nodes, failing on 3 (7.5% < fault_spread 25%) -> node-local
#        zone a has 4 nodes mounting efs-train, through one mount target; 3 of 4 fail          (L23.3)
[controller] fs client excluded on gpu-c-03, -04, -05 (convoy.io/fs-excluded.efs-train)
[controller] 03:35 exclusion_max passed on gpu-c-03 -> cordon   budget gpu-c 2/2
[controller] gpu-c-03: db-2 (owner ?) is not in auto_drain -> drain request 9714 to platform
[controller] gpu-c-04, gpu-c-05: budget reached -> cordon requests 9715, 9716, paged
#        by 07:30 gpu-c-04 carries 17 abandoned probe containers, all in D state on efs-train   (L23.9)

# 03:31  9700 restarts on 8 nodes, including gpu-c-22 after burn-in

# 04:05  node agent v2.4 rolls out; its EDAC collector reports cumulative counts, not a rate
[controller] memory: correctable rate above 100/h on 30 of 38 nodes (node-agent, fresh)
[controller] rule ecc-ce-rate -> cordon x30; budget gpu-c 2/2 -> 28 cordon requests, paged      (L23.6)

# 04:30  gpu-c-11: three weak signals, none past its threshold
[controller] gpu-c-11 ambiguous: EDAC 80/h (100), nvme hung tasks 1 (3), gpu5 probe 1.9 x baseline (2.0)
[controller] health agent: context = probe receipts, node metadata, dmesg excerpt (scrubbed)
#        the excerpt held an audit EXECVE line with the arguments of a training process        (L23.5)
[controller] verdict: node, cordon, confidence 0.74 < 0.8 -> page with verdict 9720

# 05:10  gpu-d-02 (pool with the cloud's auto-repair): XID 79
[controller] gpu-d-02 gpu1 excluded; condition convoy.io/GPUHealthy=False published; node actions deferred to cloud-auto-repair
[platform:cloud-auto-repair] gpu-d-02 replaced at 05:21; pods deleted without Convoy's drain
#        pid 9800 (gang on gpu-d) lost 41 min since its last checkpoint                        (L23.7)

# 05:40  gpu-c-14 XID 79 under the restarted 9700
[controller] 9700: no retry left -> exit 1 (retries exhausted) at 05:52                        (L23.11)

# 02:00 to 06:00  factory-3, link down: gpu0 on node f3-a XID 79; nothing acts until 06:02     (L23.10)

# 07:30  sre-ana logs in
[sre-ana] $ ls /sys/approvals/platform/
9711  rm gpu-c-07            02:59  svc:platform-health  node empty, cordoned 4.5 h, $17.40 so far
9714  drain gpu-c-03         03:35  svc:platform-health  db-2 (owner ?) PDB db
9715  cordon gpu-c-04        03:36  …
9716  cordon gpu-c-05        03:36  …
9721 … 9748  cordon x28      04:06  rule ecc-ce-rate
9720  verdict gpu-c-11       04:30  confidence 0.74
[sre-ana] $ cat /sys/k8s/self/nodes/gpu-c-07/receipt
gpu2 excluded 02:58 by svc:platform-health (node-health v3, rule xid-79) · reset refused (in use) · cordoned 02:58 (rule reset-failed) · drained 03:09 (auto_drain gang) · rm requested 02:59, pending · cordoned hours 4.5 ($17.40)
[sre-ana] $ cat /sys/k8s/self/nodes/gpu-c-22/receipt
joined 03:04 · burn-in 03:04-03:29 passed (gpu diag r3, memory test) · untested taint lifted 03:29
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| probe | a process with a receipt | a pod pinned to the node, tolerating cordon | receipts, `health`, node conditions |
| component exclusion | `gpu/<i>/online`, `fs/<device>/online` | device plugin health, node label, planner anti-affinity | Node object, node receipt |
| cordon | `online 0` by the controller | `spec.unschedulable`, an event | Node object, node receipt, evidence artifact |
| automatic drain | `kill -TERM --node` | group stop-with-checkpoint, eviction API | process receipts |
| request | an approval on the health process | `Approval` resource (ADR 0017) | `/sys/approvals/platform/` |
| verdict | a typed record citing receipts | an inference on the model device | evidence artifact, model mount audit |
| burn-in | the `convoy.io/untested` taint | probe pods, node taint | node receipt |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ○ | evidence artifacts pin probe receipts and the verdict | |
| A2 | Command | – | | |
| A3 | Session | ○ | the health agent is a session under a service identity | |
| A4 | Backend | – | | |
| A5 | Contract | ● | a retry spent by an infrastructure fault; a degraded night (L23.11) | AC-23.6 |
| A6 | Device tree | ● | component `online` files, `degraded` per zone, probes per node (L23.2, L23.3) | AC-23.2, AC-23.3 |
| A7 | Signals | ● | automatic gang drain; checkpoint skipped when the fault is the checkpoint's (L23.4) | AC-23.4 |
| A8 | Secrets | – | | |
| A9 | Cost | ● | empty cordoned node billed overnight; burn-in and untested capacity (L23.1, L23.8) | AC-23.1, AC-23.5 |
| A10 | Identity | ● | `svc:platform-health`, requests answered by people, the model's context (L23.5) | AC-23.7 |
| A11 | Defaults | ● | the policy, the budget, spread, burn-in, the collector version | AC-23.8 |
| A12 | Storage | ● | a shared filesystem through one mount target per zone; hung probes (L23.3, L23.9) | AC-23.3 |
| A13 | Networking | – | | |
| A14 | Time | ● | dwell, `exclusion_max`, burn-in, the hours nobody is awake | |
| A15 | Audit | ● | receipts and evidence for every automatic action; requests in the inbox | AC-23.1 |
| A16 | Durability | ● | the checkpoint a drain cannot take; the repairer that skips the drain (L23.4, L23.7) | AC-23.4 |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | the cloud's auto-repair on `gpu-d` replaces a node on the condition Convoy published, without Convoy's drain | decision 10 defers node actions to the repairer and says nothing about whether the repairer drains | **L23.7** |
| P2 | POSIX cannot hold | `gpu/<i>/reset` is a write to a file; the driver refuses a reset while a process holds the GPU, which is always the case for a training rank | decision 4 tries the reset first, so every XID on a busy GPU escalates to cordon by rule | **L23.2** |
| P3 | Orphaned state | a probe of a hung mount is abandoned in D state by design, and a probe runs every `probe_interval` | nothing bounds the abandoned probes; they pile up as unterminatable pods on the node | **L23.9** |
| P4 | Identity leak | the `dmesg` excerpt sent to the model carries an audit EXECVE record with a process's arguments | the outbound scrub catches known secret shapes, not arguments in general | **L23.5** |
| P5 | Invisible cost | a new node bills during its 25-minute burn-in and its capacity is not usable; `status 9700` says "waiting for capacity" with no time | decision 11 prices burn-in but neither the planner nor `status` knows the node is coming | **L23.8** |
| P6 | Lying cache | a collector rollout changes what `edac` means on 30 nodes; every source is fresh, so `collector_stale_max` does not trip | decision 8 halts on stale sources and on a probe class failing widely, not on a passive source that changed meaning | **L23.6** |
| P7 | Silent failure | the automatic drain waits the full checkpoint grace for a checkpoint that cannot be taken, because the dead GPU is a member's | decision 7 drains by ADR 0008; nothing says the fault can be the checkpoint's own | **L23.4** |
| P8 | Partial fleet | (a) the shared filesystem fails in one zone through its mount target; (b) the edge site's link is down while its GPU fails | (a) spread over all 40 nodes reads a zone fault as three node faults; (b) the agent runs in the controller and cannot reach the site | **L23.3**, **L23.10** |
| P9 | Created elsewhere | the cnpg replica on `gpu-c-03` was not started through Convoy | `auto_drain` lists `gang` only, so the drain is a request, and the PDB holds when it is approved | ok |
| P10 | Multi-party | `rm` of a drained, empty node waits for a person at 03:00 while the node and its replacement both bill | decision 7 makes every replacement a request, even one that can lose nothing | **L23.1** |
| P11 | Data at rest | evidence artifacts and the model's context | class `sensitive`, the cluster's residency, `retention=0` on the model mount (decisions 6 and 9) | ok |
| P12 | Inheritance | the automatic drain restarts pid 9700 and spends the one retry the team kept for its own bugs; the next infrastructure fault fails the job | ADR 0004 amendment 1 counts a gang restart as one retry, whatever caused it | **L23.11** |

A thirteenth question: at 02:10 an InfiniBand link on `gpu-c-09` flapped for two hours and slowed every collective of pid 9700 by 30%. No probe class covers it, `health` said nothing, and only `top` showed the job slow. **L23.12**, accepted.

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| GPU off the bus, process on it | dcgm | n/a | excluded, reset deferred until the GPU is free | none; automatic |
| GPU reset fails on an idle GPU | probe after reset | n/a | reset failed, cordoned | `rm` by the controller or a request |
| shared-filesystem client hung on one node | fs probe, peers pass | n/a | client excluded, then cordoned | none; automatic |
| shared filesystem hung through one mount target | fs probe, zone peers fail | n/a | device `degraded` in that zone | the device owner; no node action |
| collector change reads as a fault on many nodes | version change with the step | n/a | attributed to the collector, controller halted for that source | roll back the collector, or accept the new baseline |
| drain where a member cannot checkpoint | the drain's cause | n/a | checkpoint skipped, last good checkpoint named | none |
| repairer replaces a node without a drain | node removed while processes ran | 1 for the gang member | `platform:<repairer>`, last checkpoint | `replay` or the gang's own restart |
| site cut off from the controller | link state | n/a | site-local rule actions, verdicts queued | none; ambiguous cases wait for the link |
| infrastructure retries exhausted | retry count | 1 | infra retries and the faults that spent them | `replay` |
| a component with no probe degrades | `top` only | none | nothing | a person (accepted, L23.12) |

## 8. Acceptance criteria

- **AC-23.1** Given a node the controller cordoned and drained with no unterminatable process and no attached volume, in a pool with a manager, then the controller writes `rm` itself within the budget and the receipt says so; otherwise `rm` is a request whose entry in the inbox shows the node's hourly price and the cordoned hours so far.
- **AC-23.2** Given a GPU fault on a GPU a process holds, then the GPU is excluded at once, the reset waits until the processes on it have stopped, and the escalation to cordon follows only a failed reset on an idle GPU or the policy's own rule.
- **AC-23.3** Given a shared-filesystem probe failing on several nodes, then the spread is computed per path to the device (mount target or zone) as well as over the device, and a path above `fault_spread` is marked `degraded` with no node action.
- **AC-23.4** Given an automatic drain whose cause is a component a member needs to checkpoint, then the drain skips the checkpoint, restarts from the last good one and names it in the receipt.
- **AC-23.5** Given a node joining under burn-in, then its capacity is excluded from the plan-time check and shown as `joining, burn-in until <t>`, and `status` of a process waiting on it prints the estimated start.
- **AC-23.6** Given a restart caused by a node health action, then the receipt counts it as an infrastructure retry, separate from the process's own `--retries`, within the platform's own bound.
- **AC-23.7** Given a model verdict, then the context it was sent contains only the kernel facilities the policy names and no audit record, and the evidence artifact lists which facilities were sent.
- **AC-23.8** Given a passive source whose collector version changed within the dwell window on more than `fault_spread` of a pool, then the controller attributes the step to the collector, takes no node action on that source and pages.

## 9. Persona lenses

- **SRE**: wants the inbox at 07:30 to hold only what needed a person, and wants the controller to stop by itself when the pattern is the platform's, not the hardware's.
- **ML**: wants a dead GPU to cost the time since the last good checkpoint, not a checkpoint grace on top, and not the retry the team kept for its own code.
- **PE**: wants spread computed along the path the data takes (mount target, zone), the collector's version in every reading, and the repairer's drain behaviour in `caps`.
- **FIN**: wants an empty cordoned node gone before morning and burn-in shown as time, not as missing capacity.
- **SEC**: wants what reaches the model to be a short list of kernel facilities, never audit records, and wants the list in the evidence.
- **DBA**: wants a replica created elsewhere never drained by a policy that only knows gangs; the story shows it is not.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L23.1 | P10, P5 | Decision 7 makes every replacement a request. A node the controller drained, with no unterminatable process and no attached volume, can lose nothing by being replaced, yet it bills all night next to the node that replaced its capacity. | **amend** (applied: ADR 0021 d7 revised) ADR 0021 decision 7: `rm` is automatic for a node the controller cordoned when the drain completed with no unterminatable process, no volume attachment remains, the pool has a manager and the replacement is within the budget; every other replacement is a request, and the request shows the node's hourly price and cordoned hours | ADR 0021 d7, d8; ADR 0010 a1; ADR 0011 a5 |
| L23.2 | P2 | The driver refuses a GPU reset while any process holds the GPU, so decision 4's "reset once" always fails on a training GPU and every XID escalates to cordon. | **amend** (applied: ADR 0021 d4 revised) ADR 0021 decision 4: a GPU fault excludes the GPU at once; the reset is attempted only when no process holds it (after the drain of the processes on it, or at once on an idle GPU), and a failed reset on an idle GPU is what escalates to cordon; the policy may still cordon directly for XID classes it names | ADR 0021 d4; ADR 0011 a5 |
| L23.3 | P8 | Decision 3 computes spread over the nodes mounting a device. A shared filesystem is reached through a path per zone (an EFS mount target), so a zone's fault on 3 of 4 nodes reads as 7.5% of 40 and is attributed to three nodes. | **amend** (applied: ADR 0021 d3 revised) ADR 0021 decision 3: spread is computed at every level of the path to the device that the driver reports in `caps` (node, mount target or zone, device), from the narrowest up; a level above `fault_spread` is marked `degraded` (`fs/efs-train@a degraded`), its owner is alerted and no node below it is acted on | ADR 0021 d3; ADR 0011 a1, a4 |
| L23.4 | P7 | An automatic drain stops a gang with checkpoint (ADR 0008 amendment 1). When the fault is a member's GPU, or the node's client for the checkpoint device, the checkpoint cannot complete and the drain waits the full grace for nothing. | **rule** (applied: ADR 0021 and ADR 0008 rules; roadmap M10; command map): when the drain's cause is a component a member needs to checkpoint (its GPU, or the node's client for the device the checkpoint is written to, from the process's mounts), the drain skips the checkpoint, restarts from the last good one and names it in the receipt | ADR 0021 d7; ADR 0008 a1; ADR 0004 a1 |
| L23.5 | P4 | The `dmesg` excerpt sent to the model can carry audit records (EXECVE with arguments), NFS server paths and other workload detail; the outbound scrub of ADR 0014 decision 5 matches secret shapes, not arguments. | **rule** (applied: ADR 0021 and ADR 0014 rules; roadmap M2): the context sent to the health model includes only kernel lines of the facilities the policy names (`xid`, `edac`, `mce`, `nvme`, hung-task headers, the shared-filesystem client), never an audit record, with task names replaced by pid and pod; the evidence artifact lists the facilities sent | ADR 0021 d6; ADR 0014 d5; S13 |
| L23.6 | P6 | A collector rollout changes what a passive source means on many nodes at once; every reading is fresh, so `collector_stale_max` does not trip, and decision 8's halt covers probe classes only. | **rule** (applied: ADR 0021 rules; roadmap M10): every passive reading carries its collector's version; a step change on more than `fault_spread` of a pool within the dwell window of a collector, driver or kernel version change is attributed to the change, takes no node action on that source and pages; decision 8's halt applies to passive sources as to probes | ADR 0021 d5, d8; ROADMAP M10 |
| L23.7 | P1 | Decision 10 defers node actions to the platform's repairer, which may replace a node without Convoy's drain, so a gang loses its work since the last checkpoint and a delegated unit gets no switchover. | **rule** (applied: ADR 0021 rules; roadmap M8): `caps: node-repair` reports whether the repairer honours a drain (`drain=honoured\|ignored`) and its wait before acting; where it is ignored, the controller cordons and drains on the condition it publishes, within the repairer's wait, and `explain` on a plan for that pool prints `repair: forced after <wait>` | ADR 0021 d10; ADR 0008 a1; S4 |
| L23.8 | P5 | A node under burn-in bills for 25 minutes before it is usable, and the process waiting for it reads "waiting for capacity" with no time. | **rule** (applied: ADR 0021 rules; roadmap M8; command map): capacity under burn-in is excluded from the plan-time check and listed as `joining, burn-in until <t>`; `status` and `wait` of a process waiting on it print the estimated start including the burn-in | ADR 0021 d11; ADR 0003 rule L16.6; ADR 0013 d5 |
| L23.9 | P3 | A probe of a hung mount is abandoned in D state by design, and nothing bounds how many accumulate. | **rule** (applied: ADR 0021 rules; roadmap M10): at most one abandoned probe per component per node; while one is outstanding the probe is not rerun and the result reads `no response since <t>`; abandoned probes are listed under the node's `probe` and leave with the node's fencing or reboot | ADR 0021 d2; ADR 0008 a4 |
| L23.10 | P8 | The health agent runs in the controller; a site cut off from it (ADR 0011 amendment 4) gets no exclusion and no cordon until the link returns. | **rule** (applied: ADR 0021 rules; roadmap M2): the deterministic policy and the probes run in the site's per-cluster part of the controller, beside the receipt spool of ADR 0009 amendment 1, so a cut-off site excludes and cordons by rule within its own budget; verdicts for ambiguous cases are queued and the page is sent when the link returns | ADR 0021 d5, d6; ADR 0011 a4; ADR 0009 a1; S11 |
| L23.11 | P12 | A restart caused by a node health action counts as one of the process's own retries (ADR 0004 amendment 1), so two hardware faults in one night fail a job whose code did nothing wrong. | **rule** (applied: ADR 0021 and ADR 0004 rules; roadmap M2; command map): a retry caused by a node health action, or by the platform's repairer, is counted as an infrastructure retry, separate from `--retries`, within the platform's `infra_retries` bound; the receipt lists the faults that spent them | ADR 0004 a1; ADR 0021 d7; S2 |
| L23.12 | none | A component with no probe class (an InfiniBand link that flaps) slows a gang for hours with no verdict. | **accepted**: probes for NICs, RDMA and InfiniBand are an open item of ADR 0021; until then the gang's own heartbeat in `top` is the only signal | ADR 0021 consequences |
