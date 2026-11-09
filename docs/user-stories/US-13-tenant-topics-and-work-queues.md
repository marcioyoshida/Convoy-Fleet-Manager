# US-13 · Hand a tenant its topics as directories and give agents a work queue they cannot poison

| Field | Value |
|---|---|
| Primary persona | PE (Platform Engineer) |
| Secondary personas | SEC, FIN, AI |
| Workload class | platform provisioning, work queue |
| Walkthrough | none; written against [ADR 0012](../adr/0012-logs-and-queues-are-devices.md) decisions 1, 6, 7 and 8 and US-07's `/queue/human-review` |
| Gaps already closed there | none |
| Milestones needed | M5, M7, M9 |
| Findings in this document | L13.1 – L13.6 |

## 1. Story

**As a** platform engineer, **I want** a tenant's topics, identities and quotas to be created with `mkdir` and `chmod` under the tenant's directory, and **I want** a named work queue that agents drain with `xargs` and that sends poison items to a dead-letter topic, **so that** topic sprawl, partition exhaustion and stuck queues become visible in `ls` and `df` instead of in an incident.

**Context.** Forty tenants share two Strimzi clusters and one MSK cluster. Topics are created by ticket, ACLs by hand, and twice a quarter a broker runs out of partitions. The support team's agents (US-07) pull tickets from a queue that has stalled on one malformed message three times.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | the platform engineer | `sudo --as platform` on the queue device | |
| Approves | a second engineer for `rm` of a topic with data | | L13.1 |
| Pays | tenant acme | partition-hours on MSK, pods on Strimzi | L13.3 |
| Audits | security, quarterly | who may read and write which prefix | L13.2 |
| Operates the platform | the same team | topic templates, dead-letter defaults | |

## 3. Preconditions

- **Capabilities**: `queue:kafka`, `gitops` (KafkaTopic and KafkaUser are reconciled), `keda` for queue-depth scaling of work-queue Jobs.
- **Packages**: `agent-run` (US-07).
- **Mounts**: `/queues/acme` on the Strimzi device with `topics=acme.,group=acme-,quota=partitions:200,bytes-in:50MB/s`; `/queue/tickets` from `mkfifo`.
- **Annotations**: none beyond US-07.
- **Platform defaults** `explain` must print: the topic template (partitions, replication, retention), the ACL set a mode maps to, the dead-letter topic and max deliveries, the partition quota remaining.

## 4. Main path

```bash
$ sudo --as platform
$ mkdir /queues/acme/acme.events
topic acme.events  12 partitions, rf 3, retention 7d (template "tenant-m")  written as intent (KafkaTopic, commit 9c1e…)
quota: partitions 188/200 remaining

$ chmod g+rw /queues/acme/acme.events && ls -l /queues/acme/
drw-r-----  acme-writers  acme-readers  acme.events   12p  0 B   7d   lag -
→ KafkaUser acme-writers: Read, Write, Describe on prefix acme.; acme-readers: Read, Describe; groups acme-*

$ mkfifo /queue/tickets
queue tickets on fifo_device kafka-strimzi-prod  12 partitions  retention 3d  dead-letter tickets.dlq after 3 deliveries

$ xargs -P 8 -n 1 agent-run --policy=triage-v3 < /queue/tickets          # drain what is there now
pid 7901  work-queue Job, parallelism 8, 1 204 items, max 3 deliveries, band $14–$40 (metered)
exit 0 (degraded): 1 201 done, 3 items to tickets.dlq

$ run --unit triage 'tail -f /queue/tickets | xargs -P 8 -n 1 agent-run --policy=triage-v3'   # keep draining
unit triage  Deployment 0..12 (KEDA on queue depth)  dead-letter after 3 deliveries

$ rm /queues/acme/acme.events
convoy: acme.events holds 38 GiB with 7d retention. Refusing without --with-data and a second approval.
        `backup /queues/acme/acme.events --to /s3/acme/archive/` copies the retained range first.

$ df -h /queues/acme
PARTITIONS  USED/QUOTA  BROKER-LIMIT  BYTES   COST/mo
acme        188/200     4 000/4 000   2.1 TiB $212  (MSK: partition-hours $0.0015, storage $0.10/GB)
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `mkdir topic` | intent write | KafkaTopic in the GitOps repository, reconciled | repository, broker |
| `chmod`, `ls -l` | mode to ACL mapping | KafkaUser ACLs | broker |
| `mkfifo` | named queue | topic plus dead-letter topic | broker |
| `xargs < /queue/tickets` | bounded drain | work-queue Job, each pod pulls until empty | offsets per item |
| `run --unit … tail -f` | continuous drain | Deployment scaled by KEDA on depth | |
| `rm topic` | destructive verb | refused without `--with-data` | |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | – | | |
| A2 | Command | ○ | `agent-run` from US-07 | |
| A3 | Session | ○ | `sudo --as platform` on a queue mount | |
| A4 | Backend | ○ | work-queue Job versus unit (L13.5) | `explain` names which and why |
| A5 | Contract | ● | per-item deliveries, degraded exit with a dead-letter count | AC-13.4 |
| A6 | Device tree | ● | mount options carry quotas (L13.3) | `df` shows partitions against quota and broker limit |
| A7 | Signals | – | | |
| A8 | Secrets | ● | KafkaUser credentials as references created by `chmod` | `ls -l /secrets` shows them |
| A9 | Cost | ● | partition-hours as a price shape | `df` prices partitions, not only bytes |
| A10 | Identity | ● | POSIX modes onto Kafka ACLs (L13.2) | `getfacl` shows the full set |
| A11 | Defaults | ● | template, dead-letter, quota | all in `explain` |
| A12 | Storage | ○ | retained bytes per topic | |
| A13 | Networking | – | | |
| A14 | Time | – | | |
| A15 | Audit | ○ | every `mkdir`, `chmod`, `rm` a receipt | |
| A16 | Durability | ○ | a work-queue Job survives the session | |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | KafkaTopic and KafkaUser are GitOps-managed | intent write (S4) | ok |
| P2 | POSIX cannot hold | `rwx` onto Read, Write, Create, Describe, Delete, Alter | lossy mapping | **L13.2** |
| P3 | Orphaned state | `rm` on a topic deletes data at once; there is no detach | ADR 0006 | **L13.1** |
| P4 | Identity leak | a KafkaUser credential | reference | ok |
| P5 | Invisible cost | partitions are billed and limited, not bytes alone | mount quota and `df` | **L13.3** |
| P6 | Lying cache | none | | ok |
| P7 | Silent failure | a poison item redelivered forever stalls the queue | no dead-letter rule in `mkfifo` | **L13.4** |
| P8 | Partial fleet | none | | ok |
| P9 | Created elsewhere | topics created by ticket before Convoy | `ls -l` shows owner `?`; `chown` adopts (S9) | ok |
| P10 | Multi-party | second approval on `rm` | US-03 L3.1 | ok once L3.1 exists |
| P11 | Data at rest | retention on the queue and the dead-letter topic | from the template | ok |
| P12 | Inheritance | `xargs < queue` versus `tail -f queue \| xargs`: when does the process end? | undefined | **L13.5** |

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| partition quota exhausted | `mkdir` pre-flight | 3 | used, quota, broker limit | raise quota or delete topics |
| poison item | max deliveries | continues, degraded | items moved to the dead-letter topic | `cat /queue/tickets.dlq` |
| `rm` on a topic with data | pre-flight | 3 | bytes, retention, approval needed | `backup` then approved `rm` |
| ACL mode cannot express a request | `chmod` | 3 | the ACL set and `setfacl` | `setfacl` |
| broker partition limit reached fleet-wide | `df` (L13.3) | | | platform adds brokers |

## 8. Acceptance criteria

- **AC-13.1** Given `mkdir` under a queue mount, then a KafkaTopic commit exists before the topic does, `explain` printed the template, and the mount's partition quota decreased.
- **AC-13.2** Given `chmod g+rw` on a topic, then `ls -l` shows the mode and `getfacl` lists the exact Kafka ACL operations granted, and the KafkaUser credential appears under `/secrets` as a reference.
- **AC-13.3** Given `df` on a queue mount, then it shows partitions used against the mount quota and the broker limit, and prices partition-hours where the device bills them.
- **AC-13.4** Given an item that fails three deliveries, then it moves to the dead-letter topic, the drain continues, and the exit is 0 degraded with the count.
- **AC-13.5** Given `xargs < /queue/x`, then the process ends when the queue is empty at plan time; given `tail -f /queue/x | xargs`, then the planner requires `run --unit`.
- **AC-13.6** Given `rm` on a topic holding data, then it is refused without `--with-data` and a second approval, and names the `backup` spelling.

## 9. Persona lenses

- **PE**: wants topic sprawl and partition exhaustion in `df`, and ACLs they can read in `ls -l`.
- **SEC**: wants the mode-to-ACL mapping explicit, because "rw" hides Alter and Delete.
- **FIN**: wants partition-hours as a cost line, since that is what MSK bills.
- **AI**: wants the ticket queue to never stall on one message (US-07).

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L13.1 | P3 | Deleting a topic destroys its data immediately and a topic has no detach; ADR 0012 decision 7 requires `--with-data` but says nothing about backup or approval thresholds. | **rule**: `rm` on a topic with retained bytes requires `--with-data`, a second approval above a platform threshold (US-03 L3.1), and prints the `backup` spelling that copies the retained range into the artifact store first | ADR 0012 decision 7, ADR 0006 |
| L13.2 | P2 | POSIX modes cannot express Kafka's operation set (Read, Write, Create, Describe, Delete, Alter, AlterConfigs). | **rule**: `chmod` maps `r` to Read and Describe, `w` to Write, `x` to Create under the prefix; everything else is `setfacl`/`getfacl`; `ls -l` marks a `+` when ACLs exceed the mode, as POSIX ACLs do | ADR 0012 decision 7, posix-command-map.md (`chmod`, `getfacl`) |
| L13.3 | P5 | Partitions are a scarce, billed resource with a per-broker limit; a tenant's `mkdir` can exhaust the cluster. ADR 0012 prices bytes and partition-hours but has no quota. | **amend** ADR 0012 decision 1: mount options `quota=partitions:<n>,bytes-in:<rate>,bytes-out:<rate>`; `mkdir` pre-flights the partition quota and the broker limit; `df` shows both | ADR 0012, ADR 0010 |
| L13.4 | P7 | A named queue with redelivery and no dead-letter stalls on one poison item. | **rule**: `mkfifo` creates a dead-letter topic by default with `max-deliveries` from the platform template; consumers move an item after the limit and the process exits degraded with the count; `mkfifo --no-dead-letter` is explicit | ADR 0012 decision 6 |
| L13.5 | P12 | `xargs < /queue/x` and `tail -f /queue/x \| xargs` look alike and differ in whether the process ever ends. | **rule**: reading a queue by redirection is a bounded drain to the depth observed at plan time and compiles to a work-queue Job; a follow into `xargs` is continuous and requires `run --unit`, scaled by queue depth when `caps: keda` exists | ADR 0012 decisions 2 and 6, posix-command-map.md (`xargs`) |
| L13.6 | P12 | `ls` on a device with ten thousand topics is ten thousand Describe calls. | **accepted**: `ls` reads the `/sys/queue/<id>/` metadata cache refreshed on the inventory cadence; `ls --fresh` forces a call and says what it costs | ADR 0012 decision 7, ADR 0011 decision 7 |
