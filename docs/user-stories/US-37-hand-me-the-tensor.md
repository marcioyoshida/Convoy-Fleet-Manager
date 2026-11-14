# US-37 · Hand me the tensor: an ETL job and a training job that want to share GPU memory instead of an artifact

| Field | Value |
|---|---|
| Primary persona | DE |
| Secondary personas | ML, SEC, FIN, PE, SRE |
| Workload class | zero-copy handoff in one NVLink domain: a Spark job with GPU acceleration cleans a 6 TB table into the memory of 24 GPUs on one GB200 rack and hands it, as memory handles, to a separate training job of another group on the same rack; then the same pattern inside one Ray job; ADR 0001 says stages exchange artifacts, ADR 0028 says memory is shared only inside a domain the hardware provides |
| Walkthrough | written against [ADR 0001](../adr/0001-pipe-is-a-staged-artifact.md) and its amendment 2, [ADR 0028](../adr/0028-memory-has-a-distance-and-a-domain-is-claimed-whole.md) decisions 1 to 9, [ADR 0021](../adr/0021-a-node-is-tested-before-it-is-trusted.md) decision 4, [ADR 0004](../adr/0004-completion-contract-and-exit-codes.md) amendments 1 and 2, [ADR 0008](../adr/0008-signal-routing.md), [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md) decisions 2 and 7, [ADR 0017](../adr/0017-approvals-are-objects-on-the-process.md) decision 2, [ADR 0010](../adr/0010-platform-defaults-and-two-package-tiers.md) amendment 1, the pipeline compiler §8.5, and the command map rows for `cat` and `ls`; it follows US-36 |
| Gaps already closed there | G2 (coordinated groups), G4 (gang contract) |
| Milestones needed | M3, M4, M6, M7, M8 |
| Findings in this document | L37.1 – L37.19 |

## 1. Story

**As a** data engineer, **I want** the table my GPU job just cleaned to be read by the training job on the same rack straight from GPU memory, without writing 6 TB out and reading it back, **so that** the handoff costs seconds, not an hour, and so that whatever makes that possible also says who owns the memory, how long it is kept, what happens when my job ends or a GPU fails, and who else may read it.

**Context.** After US-36 the platform runs ADRs 0001 to 0032. Stages exchange staged artifacts (ADR 0001), with `|=` as the opt-in live channel through a relay. ADR 0028 lets a process tree claim an NVLink domain whole (`--domain=nvlink`, compiled to a DRA claim and a `ComputeDomain` with IMEX channels), lets children join the claim with `--domain=@parent`, gives one claim to one process tree and one group, and clears GPU memory before a domain changes groups. A proposal circulating in the platform team describes a "unified" pattern: an ETL job exports IMEX memory handles for its GPU buffers; a training job on the same domain imports them and reads the table as tensors, with no copy. Nothing in Convoy describes an artifact that lives in GPU memory.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs the ETL | `rafael` | user `rafael`, group `analytics` | Spark with the RAPIDS accelerator |
| Trains on the result | `yuki` | user `yuki`, group `llm` | |
| Owns the rack's capacity | `capacity` | group `capacity` | reclaims nodes by preemption |
| Owns the data's class | `amara` | user `amara`, group `security` | the table has customer rows, `class=restricted` |
| Manages the `etl` namespace | Argo CD | `svc:platform` | prunes what is not in Git |
| Pays | FinOps | group `finops` | |

## 3. Preconditions

- **Capabilities** (`caps`): on `gb200`: `dra`, `imex`, `domain:nvlink`, `mem_scrub=reset`; racks `nvl-a` and `nvl-b`, each about 13 TB of HBM across 72 GPUs.
- **Packages**: `spark-operator` with the RAPIDS accelerator; `kuberay`; the training image reads Arrow buffers through DLPack.
- **Platform defaults** `explain` must print: for an artifact that is meant to stay in GPU memory, which domain holds it, which process holds the claim, who may import it, how long it is held and at what price, what happens to it when its holder ends, is preempted or loses a GPU, and how it is read from outside the domain.

## 4. Main path

```bash
# Monday 07:00 · rafael, group analytics, on rack nvl-a
$ batch --gang --domain=nvlink --replicas=6 --gpu=4 spark-submit --rapids clean.py /tables/events/2026-11
#  6 TB of cleaned rows end as Arrow buffers in the memory of 24 GPUs
#  there is no artifact that lives in GPU memory, so clean.py writes the IMEX handles into a
#  ConfigMap etl/handles for the training job to find                                         (L37.1)
#  etl is managed by Argo CD; the ConfigMap is pruned three minutes later                       (L37.2)

# 08:00 · yuki, group llm
$ batch --gang --domain=nvlink --replicas=12 --gpu=4 torchrun train.py --from-handles etl/handles
#  nvl-a has no 48 free GPUs until notebooks leave; Kueue places the gang on nvl-b, where the
#  handles mean nothing; it fails at step 0                                                       (L37.9)
#  rerun on nvl-a: llm's processes import analytics' GPU memory; nothing checked the group       (L37.5)
#  the trainer expects float16 columns; the buffers are float32; the first batch is garbage    (L37.15)
# 09:30 · the Spark job finishes and exits; its memory is freed; 31 ranks hit illegal-address
#  faults reading it                                                                              (L37.4)

# rafael restarts the ETL with a sleep loop at the end of clean.py to keep the memory alive
#  24 GPUs held idle for three days, about $14 000, to avoid writing 6 TB once                  (L37.6)
#  analytics never agreed to hold its claim for llm                                             (L37.11)
# Tuesday · the sleep loop runs a second cleaning pass that updates the buffers in place; the
#  trainer, mid-epoch, reads half old rows and half new                                         (L37.7)
# Tuesday 15:00 · GPU 3 on node a-07 has an XID and is excluded (ADR 0021); its part of the table
#  is gone; the loader reads zeros from that range and the loss curve looks normal             (L37.8)
# Wednesday · capacity preempts two ETL nodes with two minutes' notice; their part of the table
#  cannot be written anywhere in time; 1 TB is lost                                              (L37.16)
$ cat /artifacts/etl-clean | head        # rafael, from the session
#  hangs: nothing connects the session to GPU memory on nvl-a                                  (L37.3)
#  the table has customer rows (class=restricted); analytics releases nvl-a on Thursday and
#  research claims it; nothing ties the scrub to the data that was there                       (L37.12)

# The next week · yuki tries the pattern inside one Ray job: ETL actors and trainers under one
#  claim (--domain=@parent)
#  Ray keeps the GPU objects in its own store; Convoy sees none of them                         (L37.10, L37.19)
#  a trainer actor runs out of memory and restarts; the driver's reference count drops, Ray
#  frees the ETL output, and the restarted actor silently recomputes 6 TB                       (L37.13)
#  a team on nvl-b asks to read the table; nothing says it could be copied over RDMA          (L37.14)
#  the next month's table is 15 TB; a rack holds about 13 TB                                   (L37.18)
#  the platform team asks why the table is not simply an artifact that stays where it is      (L37.17)
```

| Step | Convoy object | Kubernetes or framework object | Where the state lives |
|---|---|---|---|
| the cleaned table | none | Arrow buffers in 24 GPUs' memory | the Spark executors' processes |
| the handles | none | a ConfigMap in a managed namespace | the API server, until pruned |
| the import | none | IMEX channels of the `ComputeDomain` | the DRA driver |
| the Ray variant | none | Ray's GPU objects and their reference counts | the Ray driver |
| the class and residency | the input table's | none | lost at the handoff |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe: staged artifact, content id, cache | ● | an artifact held in GPU memory (L37.1, L37.17); mutated after publication (L37.7) | AC-37.7, AC-37.17 |
| A2 | Command: class, annotation, executable fields | ○ | the layout of the buffers (L37.15) | AC-37.15 |
| A3 | Session: mount table, env, cwd, attach and detach | ○ | `cat` from the session (L37.3) | AC-37.3 |
| A4 | Backend selection | ● | an importer placed in another domain (L37.9) | AC-37.9 |
| A5 | Completion contract | ● | a lost part read as zeros (L37.8) | AC-37.8 |
| A6 | Device tree | ● | where a resident artifact lives and is listed (L37.2, L37.3) | AC-37.2 |
| A7 | Signals and lifecycle | ● | the holder exits (L37.4); preemption (L37.16) | AC-37.4, AC-37.16 |
| A8 | Secrets as references | – | | |
| A9 | Cost | ● | 24 GPUs held to avoid a write (L37.6) | AC-37.6 |
| A10 | Identity and authorization | ● | another group imports (L37.5); a hold for another group (L37.11) | AC-37.5, AC-37.11 |
| A11 | Platform defaults and `caps` | ● | `resident_max`, `hold_vs_stage`, `resident_hold_max` | AC-37.6 |
| A12 | Storage | ● | GPU memory as an artifact tier (L37.17) | AC-37.17 |
| A13 | Networking | ○ | an RDMA copy from another rack (L37.14) | AC-37.14 |
| A14 | Time and queueing | ○ | a holding period (L37.11) | |
| A15 | Audit, receipts, retention and `replay` | ● | lineage and recomputation (L37.13); class at rest (L37.12) | AC-37.12, AC-37.13 |
| A16 | Durability | ● | GPU memory is volatile (L37.4, L37.8, L37.16) | AC-37.16 |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative verb, declarative owner | handles in a ConfigMap of a namespace Argo CD prunes | ADR 0010 amendment 1: managed objects are intent | **L37.2** |
| P2 | POSIX semantics that cannot hold | `cat` of an artifact that lives in GPU memory on a rack | ADR 0001: an artifact is in a store | **L37.3** |
| P3 | Orphaned or lost state | the exporter exits while importers read its memory | ADR 0028 decision 6: a claim drains when its members exit | **L37.4** |
| P4 | Identity leak or escalation | another group imports GPU memory | ADR 0028 decision 5: one claim, one group | **L37.5** |
| P5 | Invisible cost | 24 GPUs held three days to avoid one write | ADR 0028 decision 8 prices stranded members | **L37.6** |
| P6 | Lying cache | buffers updated in place after they were handed over | ADR 0001: artifacts are immutable by content id | **L37.7** |
| P7 | Silent failure | a lost part of the table read as zeros | ADR 0021 decision 4 excludes the GPU | **L37.8** |
| P8 | Partial fleet | the importer placed on the other rack | ADR 0028 decision 4: placement by domain | **L37.9** |
| P9 | Created elsewhere | Ray's GPU objects inside one job | L32.11: stores inside a process count in its bound | **L37.10** |
| P10 | Multi-party action | one group holds its claim for another's training | ADR 0028 decision 5: whole-domain holds need approval | **L37.11** |
| P11 | Data at rest | restricted rows in GPU memory when the rack changes hands | ADR 0028 decision 6: memory cleared between groups | **L37.12** |
| P12 | Process-tree inheritance | a framework's reference count frees what a restarted member needs | ADR 0004 amendment 3 | **L37.13** |

Seven more questions without a probe: is there an artifact that lives in GPU memory at all (**L37.1**, open); can a team on another rack read it (**L37.14**, rule); does an importer check the buffers' layout (**L37.15**, rule); what happens to it at a preemption notice (**L37.16**, rule); does ADR 0001's store per topology admit a domain's memory (**L37.17**, amend); what if the table is larger than a domain (**L37.18**, accepted); and is GPU memory shared inside a framework visible at all (**L37.19**, accepted).

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| holder ends with importers attached | the claim (L37.4) | none; the holder moves to `holding` | importers, holding time, price | none; or materialize |
| import from another group | plan (L37.5) | 3 | the groups and the staged spelling | read the staged form |
| importer cannot be placed in the domain | plan (L37.9) | 3 after `--wait`, or the staged form | the domain and the price of the staged form | `--wait`, or stage |
| part of a resident artifact lost | the controller (L37.8) | importers get `EIO` for that part; floor applies | the parts lost and why | rematerialize from the receipt |
| preemption notice on a holder | the computer (L37.16) | none | parts written to the store in the notice, parts lost | rematerialize |
| layout mismatch | the import (L37.15) | 3 | declared and expected layouts | convert, or fix the reader |
| read from outside the domain | `cat`, `cp` (L37.3) | none; a priced copy, or 3 when refused | the copy and its price | none |

## 8. Acceptance criteria

- **AC-37.2** Given a resident artifact, then it is registered in the artifact store's index with its domain, holder and importers, never as a Kubernetes object in a namespace someone else manages.
- **AC-37.3** Given a read of a resident artifact from outside its domain, then it is a priced copy through the holder into the store or the reader's own memory, or it is refused with exit 3; `ls -l` shows `resident in <domain>`.
- **AC-37.4** Given a holder whose resident artifact has live importers, then its claim does not drain when it exits: it moves to `holding`, charged to its group, until the importers release or `resident_max` passes, after which the artifact is written to the store and only then released.
- **AC-37.5** Given an import, then the importer is in the holder's process tree under `--domain=@parent`, or in the same group with the holder's grant; another group reads only the staged form, under the usual grants and residency.
- **AC-37.6** Given a resident artifact, then holding it is charged to the holder's group and shown; `explain` compares the price of holding with the price of staging, and above `hold_vs_stage` recommends staging.
- **AC-37.7** Given a resident artifact, then it is immutable once published; importers map it read-only; a holder that changes its buffers publishes a new artifact with a new id.
- **AC-37.8** Given a resident artifact that loses any part, then it is `lost` for that part; importers get `EIO` for it and the stage's floor applies; where the receipt can recompute it or a staged copy exists, it is rematerialized and the receipt says so.
- **AC-37.9** Given an importer of a resident artifact, then its placement is bound to the artifact's domain; if it cannot be placed there within `--wait`, the plan uses the staged form at its price, or is refused.
- **AC-37.10** Given GPU objects a framework keeps inside one process tree, then they are part of that process, count in its memory bound, and are not artifacts; only published ones are.
- **AC-37.11** Given an import across processes, then it declares a holding period, and holding beyond `resident_hold_max` needs the approval of the holder's group.
- **AC-37.12** Given a resident artifact, then it carries the residency and class of its inputs; a domain that holds a restricted one is not offered to another group until the artifact is released and the domain scrubbed; it is never held past the retention of its class.
- **AC-37.13** Given a resident artifact, then it is kept alive by the import records Convoy holds, not by a framework's references; a restarted member re-imports it if it is still resident, and any recomputation is in the receipt.
- **AC-37.14** Given an importer in another domain of the same computer, then it reads a resident artifact over RDMA as a copy into its own memory, priced and shown at distance `rdma`, never as a shared mapping.
- **AC-37.15** Given a resident artifact, then its layout (schema, data types, memory format) is declared, and an import checks it before any read, refusing a mismatch with exit 3.
- **AC-37.16** Given a preemption notice for a holder, then the parts that can be written to the store within the notice are written; `explain` says beforehand how much of the artifact that is; the rest is `lost` under AC-37.8.
- **AC-37.17** Given artifacts passed between stages, then the store may be a domain's GPU memory, with the same content id, as a volatile tier; the artifact is written to the topology's store when it leaves the domain, when it is at risk, or when it is read from outside.

## 9. Persona lenses

- **DE**: wants to hand a table over in seconds, and wants the platform, not a sleep loop, to say how long it is kept.
- **ML**: wants an import that checks the layout, fails loudly on a lost part, and is placed where the data is.
- **SEC**: wants no other group reading GPU memory, and restricted rows tied to the scrub of the rack.
- **FIN**: wants three days of 24 idle GPUs compared with one write before it happens.
- **PE**: wants no artifacts in ConfigMaps, and one place that knows what lives in which rack's memory.
- **SRE**: wants a GPU fault or a preemption to say which part of which table was lost.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L37.1 | none | An artifact that lives in GPU memory has no design. ADR 0001 stores artifacts in a store per topology; ADR 0028 lets a process tree claim a domain and keep its own memory there, but nothing lets one process publish a buffer for another to import, says who holds it and for how long, what an importer may do, or what happens when the holder ends, is preempted or loses a GPU. The handoff was done with handles in a ConfigMap and a sleep loop. | **open** (drafted 2026-11-14 in [ADR 0033](../adr/0033-a-resident-artifact-is-held-in-a-domain-and-written-out-before-it-goes.md): a domain's memory is a volatile tier of the artifact store, under the content id of the staged form, and an artifact fits in one domain or is not resident; a holder publishes it through the toolbox and registers it in the artifact index, and it is immutable; its layout is declared and checked at import; importers are in the holder's tree or group with a grant, and Convoy's import records keep it alive; importers are placed in its domain or read a priced copy; a holder that ends is `holding` until release, its period or `resident_max`, then writes it out; holding is priced against staging; a lost part is `EIO` under the floor; a preemption notice writes out what fits; it carries its class, and the scrub is tied to it). The record asked for: resident artifacts, held in a domain's GPU memory with the content id of their staged form; publication by a holder process, import by declared importers in the same domain and group, read-only; holding as a priced state with a period and an approval; materialization to the topology's store on exit, risk, loss or a read from outside; layout declared and checked; lineage and class carried | ADR 0001 a2; ADR 0028 d2, d3, d5, d9; compiler §8.5 |
| L37.2 | P1 | Memory handles were published in a ConfigMap that Argo CD pruned three minutes later. | **rule**: a resident artifact is registered in the artifact store's index with its domain, holder and importers, never as a Kubernetes object in a namespace someone else manages | ADR 0010 a1; ADR 0001 a2 |
| L37.3 | P2 | `cat` of an artifact in GPU memory on a rack hung the session. | **rule**: reading a resident artifact from outside its domain is a priced copy through the holder into the store or the reader's memory, or is refused with exit 3; `ls -l` shows `resident in <domain>` | ADR 0001; command map `cat`, `ls` |
| L37.4 | P3 | The ETL job exited, its memory was freed, and 31 importing ranks faulted. | **rule**: a holder whose resident artifact has live importers moves to `holding` when it exits, charged to its group, until the importers release or `resident_max` passes; then the artifact is written to the store and only then released | ADR 0028 d6; ADR 0006; ADR 0004 |
| L37.5 | P4 | `llm`'s processes imported `analytics`' GPU memory, and nothing checked the group. | **rule**: an importer is in the holder's process tree under `--domain=@parent`, or in the same group with the holder's grant; another group reads only the staged form, under the usual grants and residency | ADR 0028 d5; ADR 0016 d2 |
| L37.6 | P5 | 24 GPUs were held idle for three days, about $14 000, to avoid writing 6 TB once. | **rule**: holding a resident artifact is charged to the holder's group and shown; `explain` compares holding with staging, and above `hold_vs_stage` recommends staging | ADR 0028 d8; ADR 0010; ADR 0025 d9 |
| L37.7 | P6 | A second pass updated the buffers in place, and the trainer read half old rows and half new. | **rule**: a resident artifact is immutable once published; importers map it read-only; changed buffers are a new artifact with a new id | ADR 0001; ADR 0026 |
| L37.8 | P7 | A GPU's part of the table was lost to an XID, and the loader read zeros with a normal loss curve. | **rule**: a resident artifact that loses any part is `lost` for that part; importers get `EIO` and the stage's floor applies; where the receipt can recompute it or a staged copy exists it is rematerialized, and the receipt says so | ADR 0021 d4; ADR 0004 a2 |
| L37.9 | P8 | The importing gang was placed on the other rack, where the handles meant nothing. | **rule**: an importer's placement is bound to the resident artifact's domain; if it cannot be placed there within `--wait`, the plan uses the staged form at its price, or is refused | ADR 0028 d4; ADR 0003 a1 |
| L37.10 | P9 | Ray kept the GPU objects in its own store, and Convoy saw none of them. | **rule**: GPU objects a framework keeps inside one process tree are part of that process and count in its memory bound; only published ones are artifacts | ADR 0028 d9 (L32.11); ADR 0024 d8 |
| L37.11 | P10 | `analytics` held its claim for three days for `llm`, and nobody agreed to it. | **rule**: an import across processes declares a holding period, and holding beyond `resident_hold_max` needs the approval of the holder's group | ADR 0017 d2; ADR 0016 d7 |
| L37.12 | P11 | Restricted rows stayed in GPU memory as the rack changed hands, and nothing tied the scrub to them. | **rule**: a resident artifact carries the residency and class of its inputs; a domain holding a restricted one is not offered to another group until it is released and the domain scrubbed; it is never held past the retention of its class | ADR 0028 d6; ADR 0011 a3; ADR 0007 a2 |
| L37.13 | P12 | A framework's reference count freed the table when a trainer restarted, and 6 TB were recomputed silently. | **rule**: a resident artifact is kept alive by Convoy's import records, not by a framework's references; a restarted member re-imports it while it is resident, and any recomputation is in the receipt | ADR 0001; ADR 0006 |
| L37.14 | none | A team on the other rack could not read the table, and nothing offered an RDMA copy. | **rule**: an importer in another domain of the same computer reads a resident artifact over RDMA as a copy into its own memory, priced and shown at distance `rdma`, never as a shared mapping | ADR 0028 d1, d2 |
| L37.15 | none | The trainer expected float16 columns and the buffers were float32. | **rule**: a resident artifact declares its layout (schema, data types, memory format), and an import checks it before any read, refusing a mismatch with exit 3 | ADR 0002; ADR 0015 |
| L37.16 | none | A two-minute preemption notice lost 1 TB of the table. | **rule**: at a preemption notice for a holder, the parts that can be written to the store within the notice are written; `explain` says beforehand how much that is; the rest is `lost` under L37.8 | ADR 0008; ADR 0004 a1; ADR 0021 |
| L37.17 | none | The table could not simply be an artifact that stays where it is. | **amend** ADR 0001 amendment 2: an artifact's store may be a domain's GPU memory, as a volatile tier with the same content id as its staged form; the planner may keep an artifact resident between stages of one process tree or one group in one domain, and writes it to the topology's store when it leaves the domain, when it is at risk, or when it is read from outside | ADR 0001 a2 |
| L37.18 | none | Next month's table is 15 TB, and a rack holds about 13 TB. | **accepted**: a resident artifact fits in one domain or is not resident; Convoy does not page resident artifacts to host memory or across domains, and the staged form serves anything larger | ADR 0028 d2 |
| L37.19 | none | GPU memory shared inside a framework (collective buffers, a Ray job's GPU objects) is invisible to Convoy. | **accepted**: Convoy sees the claim, the bound and published artifacts, not what a framework does inside its own processes | ADR 0028 d2; L32.11 |
