# User stories as design probes

The nine workloads in [walkthroughs.md](../walkthroughs.md) showed that the design carries them. This directory asks a harder question of the same nine, plus four written against [ADR 0012](../adr/0012-logs-and-queues-are-devices.md) to reach the edge and the Kafka primitive, and one against [ADR 0013](../adr/0013-native-verbs-have-one-shape.md) and [ADR 0014](../adr/0014-assisted-completion-the-model-proposes.md) for the assisted shell, and one against [ADR 0015](../adr/0015-tables-and-catalogs-are-devices.md) for tables and catalogs, and one fleet operation (a faulty node) that found the node layer missing, and one against [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md) for identity (an offboarding, a one-member approving group, a root assumption, a group that vanishes), and one against [ADR 0017](../adr/0017-approvals-are-objects-on-the-process.md) for approvals (a release answered after the author's session ended, a policy edit under a pending gate, a denial, an agent at a gate, a cron entry gated nightly), and one against [ADR 0018](../adr/0018-labels-are-the-signal-admission-is-the-decision.md) for labels (cost by group across two clouds, a backup of one process, adoption of a foreign Deployment, a cluster without admission, a package without a mapping), and one against [ADR 0019](../adr/0019-a-unit-has-an-address-reach-is-a-grant.md) for networking (a cross-team grant, a gated public address, a span refused, a notebook behind the login, a replica debugged from the laptop, a cluster that enforces nothing, a canary under open streams), and one against [ADR 0020](../adr/0020-a-session-is-a-journal-not-a-pod.md) for the session (a drained session host, a second terminal and a watch, an overnight token expiry, a secret typed into env, `newgrp` across residencies, an edge site without its link), and one against ADR 0012 decision 10 and ADR 0007 amendment 5 for typed topics and secret scrub (two pinned reader schemas over one topic, a copy into a held lake table, a leaked password scrubbed across artifacts, logs, a topic and a table), and one against [ADR 0021](../adr/0021-a-node-is-tested-before-it-is-trusted.md) for node health (a night with nobody awake: a GPU fault under a training rank, a shared filesystem failing in one zone, a collector rollout, the platform's own repairer, an edge site without its link), and one from an application developer's laptop to test and production (an arm64 image on amd64 nodes, session env inherited by a unit, a toolbox that is not the service's image, a dependency one major version apart, undeclared egress, a namespace's defaulted limit, a laptop build offered to production, a profile difference found by a crash), and one against [ADR 0022](../adr/0022-the-laptop-lends-a-tree-the-cluster-builds-it.md) for the inner loop (a monorepo lend, a generated directory and a production sample in a working tree, a build holding the group's push token, a dev unit on a shared unit's name, a forgotten breakpoint, a laptop asleep mid-save, a commit with no link to the tree that passed), and one against [ADR 0023](../adr/0023-telemetry-is-a-device-the-shell-reads.md) for telemetry (a canary condition diluted by the stable version, an on-call group that owns nothing, an estimate that is a heavy query, a full-text store under `grep -F`, a trace split across residencies, a token in a span, a replay across downsampling), and one fleet operation that moves production from one cloud provider to another (Terraform state and its translation, a table copied by bytes, a database cutover with no lag source and no way back, holds under a lock that cannot move, tokens with no groups, Spot notices of 15 seconds, a forecast with no end date), and one against [ADR 0024](../adr/0024-a-device-is-installed-by-terraform.md) for self-service infrastructure (two approval gates for one change, a re-plan after approval, a lock mode a module only claimed, an unrelated plan that replaces a topic, a partial apply, an adoption that deletes settings, `--like` dropping compaction), and one fleet operation on compute without a Kubernetes API (a pipeline across functions, an external Slurm cluster, a batch service and a VM pool in two regions: a function that cannot be frozen, partitions longer than a function's bound, a queue measured in hours, a job under a shared uid, a requeue that truncates logs, a controller out of reach, a node on a cluster the platform does not run), and one fleet operation across the platform's OCI registries (a critical CVE in a base image: a base resolved from a stale cache, a scan older than the CVE, a promotion that drops signatures, a cluster whose admission only audits, a tag moved behind the reconciler, an edge mirror without the digest, an exception agreed in chat, a lifecycle policy that expires running digests), and one weekend of events nobody heard (an approval request with no route, a health route failing for three weeks, an exception that lapsed unannounced, a fan-out of 1 900 messages, an approval from a chat button, stderr in a mail, a timeout counted before anyone knew), and one week of memory beyond a node (a gang split across two NVLink racks, a rack two GPUs short, a fabric fault read as GPU faults, a Spark job that spilled 4.8 TB, a cache that lost a third of its data, a request to pool two clusters' memory), and one genomics pipeline whose data had to be somewhere first (38 TB staged by hand over a link that dropped, a purge under a cached copy, a stage that exited 0 before its outputs arrived, a `mv` that lost 212 files, a file system billed for three weeks after its run): for each persona who would run them, does every aspect of the abstraction hold, and where does a verb route to the wrong place? Each story follows [TEMPLATE.md](TEMPLATE.md), which fixes sixteen aspects to cover and twelve loophole probes to run, so that a reader can tell "checked and fine" from "never checked". Findings are collected below with a proposed disposition. **All twenty-nine `amend` findings were applied on 2026-11-09**: as dated amendments to ADR 0001 to 0011 and as inline revisions of the drafts ADR 0012 and 0014, each citing its finding. The sixty `rule` findings were written the same day into a dated Rules section of the ADR each touches, a rules table in the command map, a section of the compiler document and a per-milestone section of the roadmap. US-17's three revisions to ADR 0016 and five rules, US-18's two revisions to ADR 0017 and four rules, US-19's two revisions to ADR 0018 and four rules, US-20's two revisions to ADR 0019 and eight rules, US-21's two revisions to ADR 0020 and eight rules, US-22's revision to ADR 0007 amendment 5 and seven rules, US-23's three revisions to ADR 0021 and eight rules, and US-24's two revisions to ADR 0013 and seven rules, were applied the same way on 2026-11-10. The `open` and `accepted` findings stand as written in each story.

| Story | Title | Primary | Secondary | Findings |
|---|---|---|---|---|
| [US-01](US-01-dag-pipeline.md) | Run the nightly DAG as a shell script and trust the rerun | DE | DS, SRE, FIN | 6 |
| [US-02](US-02-spark-hadoop.md) | Submit a Spark job over HDFS from the shell and keep control of it | DE | PE, FIN, SRE | 6 |
| [US-03](US-03-cicd-release.md) | Release through staging and two regions with a canary and a second approver | DO | SRE, SEC, PE | 5 |
| [US-04](US-04-ml-pipeline.md) | Go from notebook to a sweep to a served model without leaving the shell | DS | ML, FIN, PE | 6 |
| [US-05](US-05-distributed-training.md) | Train on 64 GPUs for 36 hours and survive a spot reclaim | ML | PE, FIN, SRE | 6 |
| [US-06](US-06-multitenant-database.md) | Give a tenant a database as a directory and never lose a replica's data | DBA | PE, SEC, SRE | 7 |
| [US-07](US-07-agent-orchestration.md) | Run a triage agent over a ticket queue with a token budget and no secret in sight | AI | SEC, FIN, SRE | 7 |
| [US-08](US-08-regulated-month-end.md) | Close the month in-region, on time, and prove it two years later | SEC | DE, FIN, SRE | 7 |
| [US-09](US-09-secret-rotation.md) | Rotate a leaked database password across the fleet and prove the old one is dead | SEC, PE | SRE, DO, DBA | 8 |
| [US-10](US-10-edge-telemetry.md) | Aggregate telemetry at forty remote sites over links that drop daily | DE | SRE, PE, FIN, SEC | 9 |
| [US-11](US-11-topic-reprocess.md) | Reprocess thirty days of orders from a topic after a bug fix, without touching production consumers | DE | SEC, FIN, SRE | 6 |
| [US-12](US-12-continuous-consumer-unit.md) | Run a shell pipeline as a continuous consumer and know when the shell is the wrong tool | DE | SRE, PE, ML | 6 |
| [US-13](US-13-tenant-topics-and-work-queues.md) | Hand a tenant its topics as directories and give agents a work queue they cannot poison | PE | SEC, FIN, AI | 6 |
| [US-14](US-14-assisted-shell.md) | Let the shell finish my line in a regulated namespace and at a cut-off site, without it leaking, paying or acting for me | AI | SEC, FIN, PE, DE | 6 |
| [US-15](US-15-lakehouse-backfill.md) | Backfill a month into the lake table from the topic, publish it when audited, and query the warehouse mirror within a budget | DE | SEC, FIN, DS, SRE | 7 |
| [US-16](US-16-node-fault.md) | Find the node that is hanging my jobs, take it out of service without losing anyone's data, and get a healthy one back | SRE | PE, ML, DBA, FIN, SEC | 7 |
| [US-17](US-17-offboarding.md) | Offboard an engineer on Friday without stranding their jobs, and know on Monday who could still act as them | SEC | PE, DE, DO, SRE, FIN | 8 |
| [US-18](US-18-approval-under-pressure.md) | Get a release approved after I have gone home, be told why it was denied, and never let an agent approve an agent | DO | SEC, AI, SRE, FIN, PE | 7 |
| [US-19](US-19-label-the-fleet.md) | Bill every team across two clouds from one label, back up one process, and adopt what we did not create | PE | FIN, SEC, SRE, DE, DBA | 7 |
| [US-20](US-20-expose-and-reach.md) | Put an API on the internet, let one other team call it, and reach a broken pod from my laptop without opening anything else | SRE | DO, DS, SEC, PE, FIN | 11 |
| [US-21](US-21-session-survives.md) | Keep my shell through a sleeping laptop, a drained node, an expired token and a dead link, and always know which cluster I am typing into | DE | SRE, SEC, PE, DS, FIN | 11 |
| [US-22](US-22-leak-in-the-stream.md) | Find every copy of a password a producer wrote into an event stream, from the topic to the lake, while the stream's schema keeps changing | SEC | DE, DBA, PE, FIN | 9 |
| [US-23](US-23-the-night-nobody-was-paged.md) | Let the fleet take its own faulty nodes and components out of service overnight, and find in the morning only what needed me | SRE | ML, PE, FIN, SEC, DBA | 12 |
| [US-24](US-24-but-it-runs-on-my-machine.md) | But it runs on my machine: take a service from a green laptop to test and production, and learn every difference before it bites | DEV | DO, SRE, PE, SEC, FIN | 11 |
| [US-25](US-25-edit-here-run-there.md) | Edit here, run there: lend my monorepo to the cluster, reload and debug a dev unit, and ship the exact tree I tested | DEV | SEC, PE, FIN, SRE, DO | 16 |
| [US-26](US-26-the-dashboard-said-green.md) | The dashboard said green: find out at 2 a.m. why the service is slow, with metrics, logs and traces from the shell, without paying for the investigation twice or leaking what I read | SRE | DO, SEC, FIN, PE, DEV | 16 |
| [US-27](US-27-move-production-to-another-cloud.md) | Move production from AWS to Google Cloud: translate the infrastructure, replicate the data, cut over and back, and know the bill before we start | PE | FIN, DBA, SRE, SEC, DE, DO | 17 |
| [US-28](US-28-install-my-own-devices.md) | Install my team's own devices from the shell, adopt the ones we made by hand, and never let a plan destroy data nobody asked to touch | DE | PE, SEC, FIN, DO, SRE | 15 |
| [US-29](US-29-one-pipeline-many-computers.md) | One pipeline, many computers: preprocess on functions, train on the university's Slurm cluster, evaluate on Batch, serve from a VM pool, in two regions, from one shell | ML | PE, FIN, SEC, SRE, DE | 18 |
| [US-30](US-30-the-base-image-has-a-cve.md) | The base image has a critical CVE: find every image that carries it, rebuild and promote by digest across registries and regions, mirror to the edge, and know what still runs it | SEC | PE, DEV, DO, SRE, FIN, ML | 17 |
| [US-31](US-31-nobody-was-told.md) | Nobody was told: a weekend of approvals, finished jobs, expiries and node faults, and who hears about each, where, and how fast | DO | SRE, SEC, ML, FIN, PE, DE | 17 |
| [US-32](US-32-memory-has-a-distance.md) | Memory has a distance: one NVLink domain for a training run, a cluster's memory for a Spark job, and a cache that is not a disk | ML | DE, DS, PE, SRE, FIN, SEC | 19 |
| [US-33](US-33-the-data-has-to-be-there-first.md) | The data has to be there first: staging a genomics pipeline into a university's scratch, a linked Lustre file system and a function's disk, and back | DE | SEC, FIN, PE, DS, SRE | 19 |

## Personas

The first five were set by the owner; the other five were added because the walkthroughs kept needing them: someone owns the on-call, someone owns the data pipelines as code, someone must be able to defend the evidence, someone reads the bill, and someone owns the stateful services whose data a careless verb can lose. The eleventh, the application developer, was added with US-24 at the owner's request: the person whose code every other persona runs.

| Id | Persona | Owns | Fears | Adopts only if |
|---|---|---|---|---|
| PE | Platform Engineer | defaults, packages, quotas, the device tree, `caps` | hidden defaults, policy bypass, support load | every default is visible in `explain` and every bypass is a refusal |
| DO | DevOps Engineer | CI/CD, releases, environments, the GitOps repository | drift, inert changes, fights with the reconciler | every mutation of a managed object is intent |
| SRE | Site Reliability Engineer | on-call, SLOs, incident response, capacity | silent degradation, unbounded blast radius, signals that do not do what they say | degraded is loud, freezes are bounded, failures page |
| DE | Data Engineer | pipelines as code, data quality, lineage | stale cache, double loads, partials read as complete | reruns are provably minimal and side effects are idempotent |
| DS | Data Scientist | experiments, notebooks, features | irreproducible results, waiting on YAML, lost work | the shell history is the experiment log |
| ML | ML Engineer | training, serving, the model registry | gang failures, checkpoint loss, GPU waste | a preemption costs minutes and lineage survives promotion |
| AI | AI Engineer | agents, tools, prompts, LLM spend | runaway spend, leaked keys, non-determinism mistaken for cache | tokens are in the budget and a rerun never replays |
| SEC | Security & Compliance (incl. auditor) | policy, evidence, residency, retention, secrets | bypassable policy, unverifiable evidence, secret sinks | policy lives on the mount and the receipt is the evidence |
| FIN | FinOps | showback, chargeback, budgets | untracked spend, self-reported numbers | every receipt separates declared from settled |
| DBA | Stateful service owner | databases, quorum, backups | scale-down data loss, cascading deletes, freezes on quorum members | detach is never destroy and `rm -r` asks twice |
| DEV | Application Developer | service code, its tests, the pull request | "works on my machine", a slow inner loop, learning the platform's rules one crash at a time | the same service runs the same way on the laptop, in test and in production, and every difference is printed before the release |

## Coverage: aspects × stories

`●` the story stresses the aspect, `○` uses it, `–` does not touch it. Read the rows: an aspect with few `●` has few tests.

| Aspect | 01 | 02 | 03 | 04 | 05 | 06 | 07 | 08 | 09 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 | 24 | 25 | 26 | 27 | 28 | 29 | 30 | 31 | 32 | 33 | Stress |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| A1 Pipe, artifact, cache | ● | ○ | ○ | ● | ○ | – | ● | ● | ○ | ● | ● | ● | – | ○ | ● | ○ | ○ | ○ | ○ | ○ | ○ | ● | ○ | ○ | ● | ● | ● | ○ | ○ | ● | – | ● | ● | 15 |
| A2 Command, annotation | ● | ● | ○ | ● | ● | ○ | ● | ○ | ● | ○ | ○ | ● | ○ | ● | ○ | – | – | – | – | ○ | ○ | ○ | – | ● | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | 9 |
| A3 Session, mount table, attach | ○ | ○ | ● | ○ | ● | ● | ○ | ○ | ● | ● | ○ | ○ | ○ | ● | ○ | ○ | ● | ● | – | ● | ● | – | ○ | ● | ● | ○ | ○ | ○ | ○ | ○ | ● | – | – | 13 |
| A4 Backend selection | ● | ● | ○ | ● | ● | – | ○ | ○ | – | ● | ● | ● | ○ | ○ | ○ | ○ | – | – | ○ | – | ○ | ○ | – | ● | ● | ○ | ● | – | ● | ● | – | ● | ● | 14 |
| A5 Completion contract | ● | ● | ● | ● | ● | – | ● | ● | ● | ● | ● | ○ | ● | ○ | ● | ● | ○ | ● | – | – | – | ● | ● | ○ | ○ | ● | ● | ● | ● | ○ | ● | ● | ● | 23 |
| A6 Device tree, mounts, `/sys` | ○ | ● | ● | ○ | ● | ● | ○ | ● | ● | ● | ● | ○ | ● | ● | ● | ● | ○ | ○ | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | 27 |
| A7 Signals, groups | ○ | ● | ○ | ○ | ● | ● | ○ | – | ● | ○ | – | ● | – | – | – | ● | ○ | ○ | – | – | ● | – | ● | – | ● | – | ● | ● | ● | – | ○ | ● | ○ | 13 |
| A8 Secrets | – | – | ○ | – | – | ● | ● | – | ● | ● | ○ | ○ | ● | ● | ○ | – | ● | – | – | – | ● | ● | – | ○ | ● | ● | ● | ○ | ○ | ● | ● | – | – | 14 |
| A9 Cost | ● | ● | – | ● | ● | ○ | ● | ○ | – | ● | ● | ○ | ● | ● | ● | ● | ○ | ○ | ● | ● | ○ | ○ | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | 24 |
| A10 Identity, approvals | ○ | ○ | ● | ○ | ○ | ● | ○ | ● | ● | ○ | ○ | ○ | ● | ● | ○ | ○ | ● | ● | ● | ● | ● | ● | ● | ○ | ● | ● | ● | ● | ● | ● | ● | ● | ● | 22 |
| A11 Defaults and `caps` | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | 33 |
| A12 Storage | – | ● | – | ○ | ● | ● | – | ○ | – | ○ | ○ | – | ○ | – | ● | ● | – | ○ | ● | – | – | ● | ● | ○ | ○ | ○ | ● | ● | ○ | ● | – | ● | ● | 13 |
| A13 Networking | – | ○ | ● | ● | ○ | ○ | ● | – | – | ● | ○ | ○ | – | – | – | – | – | ○ | – | ● | ○ | – | – | ● | ● | ○ | ● | – | ● | ○ | ○ | ○ | ● | 10 |
| A14 Time, queueing | ● | ○ | ○ | ○ | ● | – | ○ | ● | ○ | ● | ○ | ○ | – | ○ | ○ | ○ | ○ | ● | ○ | ○ | ● | ○ | ● | ○ | ○ | ● | ○ | ● | ● | ○ | ● | ● | ○ | 12 |
| A15 Audit, receipts, retention | ● | ○ | ● | ● | ○ | ○ | ● | ● | ● | ● | ● | ○ | ○ | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ○ | ○ | ● | ● | ● | ● | ● | ● | ○ | ● | 25 |
| A16 Durability | ○ | ○ | ● | ○ | ● | ● | ○ | ● | ● | ● | ○ | ● | ○ | ○ | ○ | ● | ● | ○ | ○ | ○ | ● | ● | ● | – | ● | ○ | ● | ○ | ○ | ● | ○ | ● | ● | 17 |

**What no story reaches.** The session attach protocol itself (probed by US-21 against ADR 0020; an SSH bridge and session limits remain), `pkg install` with a failing `requires`, `nice`/`renice` under contention, debugging a failing partition beyond locating it (`strace`, ephemeral containers; US-16 covers the locating), and locks. US-23 probes the node health agent of [ADR 0021](../adr/0021-a-node-is-tested-before-it-is-trusted.md), which automates US-16's cordon. US-10 closed the edge and disconnected-operation row; US-14 covers interactive assistance and the model as a device. US-15 covers tables and catalogs: a backfill from a topic into an Iceberg table with a hold, a branch write published by `release`, and a priced warehouse read. Two candidate stories would close most of the rest: a platform engineer installing a package that a cluster cannot satisfy, and an engineer debugging one failed partition of a thousand.

## Findings register

Three hundred and thirty-five findings. Disposition counts: **70 amend** (29 applied to the ADRs on 2026-11-09; 17 on 2026-11-10: 11 inline to the drafts ADR 0016 to 0020, then L22.1 to ADR 0007 amendment 5, L23.1 to L23.3 inline to ADR 0021, and L24.3 and L24.4 inline to ADR 0013; 4 on 2026-11-11: L25.1, L25.4, L25.7 and L25.11 inline to ADR 0022; 11 on 2026-11-12: L26.1, L26.2, L26.5, L26.9 and L26.12 inline to ADR 0023, then L27.4, L27.12 and L27.13 inline to ADRs 0010, 0016 and 0008; then L28.1, L28.12 and L28.14 inline to ADR 0024; 6 on 2026-11-13: L29.3 and L29.7 inline to ADR 0003, then L30.6 inline to ADR 0003 amendment 1 and L30.7 inline to ADR 0022, then L31.5 and L31.9 inline to ADR 0017; L32.4 and L32.6 proposed against ADR 0004 amendment 1 and ADR 0021, and L33.13 against ADR 0001 amendment 2, not yet applied), **225 rule** (60 written into their documents on 2026-11-09, 51 on 2026-11-10, 11 on 2026-11-11, 33 on 2026-11-12, 40 on 2026-11-13; 14 proposed by US-32 and 16 by US-33, not yet written), **12 open** (design that did not exist yet; ten drafted: on 2026-11-10 L6.5 in ADR 0016, L3.1 in ADR 0017, L4.6 in ADR 0019, L12.5 in ADR 0012 decision 10, L9.7 in ADR 0007 amendment 5; on 2026-11-11 L24.1 in ADR 0022; on 2026-11-12 L27.1 in ADR 0024 and L29.1 in ADR 0025; on 2026-11-13 L30.1 in ADR 0026 and L31.1 in ADR 0027; L32.1, memory domains, and L33.1, data staging, not yet drafted), **27 accepted** (a surfaced limitation), 1 cross-reference. Full text is in each story's section 10.

| Id | Short name | Probe | Disposition | Touches |
|---|---|---|---|---|
| L1.1 | env subset in the artifact id is undefined | P6 | rule, applied | ADR 0001, compiler §3 |
| L1.2 | side-effecting stage reruns are not idempotent | P7 | amend, applied | ADR 0002; ADR 0002 a1 |
| L1.3 | user floor contradicts a downstream fan-in | P12 | rule, applied | ADR 0004 |
| L1.4 | process substitution is a bash dialect | P2 | accepted | command map |
| L1.5 | partial artifacts look complete, have no class | P3, P5 | rule, applied | ADR 0006 a1 |
| L1.6 | a skipped CronJob is silent | P7 | rule, applied | command map, M4 |
| L2.1 | `/dev/fs` devices need a fileserver driver | P7 | amend, applied | ADR 0011 d9; ADR 0011 a1 |
| L2.2 | executables control their own footprint; band is declared | P5 | rule, applied | ADR 0010 |
| L2.3 | signals to one group member are undefined | P12 | amend, applied | ADR 0008 a1; ADR 0008 a3 |
| L2.4 | executors are created by the operator, seen by label only | P9 | accepted | annotations |
| L2.5 | copy into a replicated filesystem needs capacity × replication | P3 | rule, applied | ADR 0011 d7 |
| L2.6 | pause-with-checkpoint is impossible without a checkpoint | P2 | rule, applied | ADR 0004 |
| L3.1 | approval through the author's own stdin | P10 | **open** (drafted: ADR 0017; approver ADR 0016 d7) | ADR 0005, ADR 0016, ADR 0017 |
| L3.2 | canary weight patched on a GitOps-managed route | P1 | amend, applied | ADR 0010; ADR 0010 a1 |
| L3.3 | the GitOps repository is not a device | P4, P11 | amend, applied | ADR 0011; ADR 0011 a2 |
| L3.4 | no transaction across clusters | P3, P8 | accepted | M11 |
| L3.5 | metric conditions with no metrics package | P7 | rule, applied | M11 |
| L4.1 | `cp` out of `/artifacts` drops the content id | P11 | rule, applied | ADR 0011 d9, M5 |
| L4.2 | no channel for a command to write receipt fields | P7 | rule, applied | ADR 0009, M1 |
| L4.3 | idle GPU units have no policy | P5 | rule, applied | ADR 0010, M11 |
| L4.4 | fail-fast cancels independent siblings | P2 | rule, applied | ADR 0004 |
| L4.5 | content id of `/s3` objects undefined | P6 | rule, applied | ADR 0001, M5 |
| L4.6 | `expose` depends on undesigned networking | P9 | **open** (drafted: ADR 0019) | stories.md, ADR 0019 |
| L5.1 | zonal checkpoint disk unreachable after cross-zone restart | P3 | rule, applied | ADR 0004 a1 |
| L5.2 | group freeze has no bound; collectives time out | P2 | amend, applied | ADR 0008 a1; ADR 0008 a3 |
| L5.3 | preemption loss is billed but not in the band | P5 | rule, applied | ADR 0011 d7 |
| L5.4 | rank-0 DNS at start | P2 | accepted | walkthroughs |
| L5.5 | what the deadline counts from | P12 | rule, applied | ADR 0004 |
| L5.6 | Jobs have no per-index volume claims | P7 | rule, applied | M3, ADR 0006 |
| L6.1 | hierarchical paths over flat namespaces | P2 | amend, applied | ADR 0011; ADR 0011 a3 |
| L6.2 | namespace deletion cascades to volumes | P2, P3 | rule, applied | command map, ADR 0006 |
| L6.3 | a policy as a `fuser` holder | P7 | accepted | command map |
| L6.4 | backups copy Secrets | P4 | rule, applied | ADR 0007, M5 |
| L6.5 | tenancy depends on the identity design | P9 | **open** (drafted: ADR 0016) | ADR 0016, stories.md |
| L6.6 | namespaces created outside Convoy have no owner | P9 | rule, applied | command map |
| L6.7 | prompts in non-interactive sessions | P10 | rule, applied | ADR 0005 |
| L7.1 | the artifact store is a secret sink | P4, P11 | amend, applied | ADR 0007, ADR 0009; ADR 0007 a2 |
| L7.2 | metered spend is self-reported | P5 | rule, applied | M2, ADR 0010 |
| L7.3 | budget and deadline do not propagate to children | P12 | amend, applied | ADR 0004, ADR 0005; ADR 0004 a3 |
| L7.4 | external transient failures are unclassified | P2 | rule, applied | ADR 0004 |
| L7.5 | degraded flag lost in a pipeline | P8 | see L9.5 | ADR 0004 |
| L7.6 | hostname egress needs FQDN-aware policy | P7 | rule, applied | ADR 0010, M7 |
| L7.7 | queue items are data at rest | P11 | rule, applied | ADR 0011 |
| L8.1 | the artifact store was never placed; residency of pipes | P11 | amend, applied | ADR 0001, ADR 0011; ADR 0001 a2, ADR 0011 a2 |
| L8.2 | residency as a flag is bypassable | P12, P4 | amend, applied | ADR 0011, ADR 0010; ADR 0011 a3 |
| L8.3 | a hold must cover external inputs | P9 | amend, applied | ADR 0006 a1; ADR 0006 a2 |
| L8.4 | deadline time zone | P2 | rule, applied | ADR 0004 |
| L8.5 | who replays, under what rights | P4 | rule, applied | ADR 0009 |
| L8.6 | held images in an external registry | P7 | rule, applied | ADR 0006 a1, M9 |
| L8.7 | `page:` with no alerting package | P7 | rule, applied | ADR 0004, ADR 0010 |
| L9.1 | a secret has a producer; rotation is two-sided | P10 | amend, applied | ADR 0007 a1; ADR 0007 a3 |
| L9.2 | dual validity versus cutover | P2 | rule, applied | ADR 0007, ADR 0008 a1 |
| L9.3 | `reload: restart` fights GitOps | P1 | amend, applied | ADR 0010; ADR 0010 a1 |
| L9.4 | the reference index cannot see every consumer | P9 | rule, applied | ADR 0007 a1 |
| L9.5 | degraded has no POSIX channel | P8 | amend, applied | ADR 0004, ADR 0005; ADR 0004 a2 |
| L9.6 | HUP swallowed by a wrapper entrypoint | P7 | rule, applied | ADR 0008 a2 |
| L9.7 | the leaked value stays in logs and artifacts | P4 | **open** (drafted: ADR 0007 a5) | ADR 0007, ADR 0009 |
| L9.8 | forcing a deferred retirement is two-person | P3, P10 | rule, applied | ADR 0007 |
| L10.1 | a disconnected device is not a removed device | P8 | amend, applied | ADR 0011 d8; ADR 0011 a4 |
| L10.2 | receipts lost when an outage outlasts the pod TTL | P7 | amend, applied | ADR 0009; ADR 0009 a1 |
| L10.3 | reads before the earliest offset must fail, not shrink | P2 | rule, applied | ADR 0012 |
| L10.4 | deadline expires while queued for a down site | P12 | accepted | ADR 0004 via L5.5 |
| L10.5 | links have a price per gigabyte and a state | P5 | rule, applied | ADR 0011 d7 |
| L10.6 | secret references need a site-local source | P4, P8 | amend, applied | ADR 0007; ADR 0007 a4 |
| L10.7 | stages must run where their input device is | P11, P5 | amend, applied | ADR 0003; ADR 0003 a1 |
| L10.8 | a replaced broker is a new by-id | P9 | accepted | ADR 0011 d2 |
| L10.9 | `caps` is per device, not per fleet | P7 | rule, applied | ADR 0003, ADR 0010 |
| L11.1 | time cuts resolve to offsets at plan time; CreateTime late events | P6 | rule, applied | ADR 0012 d2 |
| L11.2 | a rerun appends; a log cannot be replaced | P3 | amend, applied | ADR 0012 d2; ADR 0012 d2 revised |
| L11.3 | topic reads are ordered per partition only | P2 | rule, applied | ADR 0012, compiler §4 |
| L11.4 | the hold copy is the recurring cost | P5 | rule, applied | ADR 0006 a1, ADR 0012 d3 |
| L11.5 | topic partitions are the granularity, not the count | P12 | amend, applied | ADR 0012 d2; ADR 0012 d2 revised |
| L11.6 | bounded reads must not commit group offsets | P7 | rule, applied | ADR 0012 d2, d4 |
| L12.1 | a follow has no artifact id to key exactly-once on | P7 | amend, applied | ADR 0012 d2; ADR 0012 d2 revised |
| L12.2 | aggregate and fan-in never terminate on a follow | P2 | rule, applied | ADR 0002, ADR 0012 d4 |
| L12.3 | rolling restarts rebalance the group | P12 | rule, applied | ADR 0010, M11 |
| L12.4 | lag comes from the device, not the metrics package | P7 | rule, applied | ADR 0012 d7 |
| L12.5 | schema-encoded topics are opaque to text tools | P11 | **open** (drafted: ADR 0012 d10) | ADR 0012, M5 |
| L12.6 | a unit joining a foreign group steals partitions | P9 | rule, applied | ADR 0012 d1, d7 |
| L13.1 | `rm` on a topic has no detach; backup and approval | P3 | rule, applied | ADR 0012 d7, ADR 0006 |
| L13.2 | POSIX modes cannot express Kafka ACLs | P2 | rule, applied | ADR 0012 d7, command map |
| L13.3 | partitions are a quota, not only a price | P5 | amend, applied | ADR 0012 d1; ADR 0012 d1 revised |
| L13.4 | a queue without a dead letter stalls on poison | P7 | rule, applied | ADR 0012 d6 |
| L13.5 | `xargs < queue` ends; `tail -f queue \| xargs` does not | P12 | rule, applied | ADR 0012 d2, d6 |
| L13.6 | `ls` on ten thousand topics | P12 | accepted | ADR 0012 d7 |
| L14.1 | a fallback to grammar order must say so | P2 | rule, applied | ADR 0014 d6, d7 |
| L14.2 | the translation comment is unscrubbed free text | P4 | amend, applied | ADR 0014 d5; ADR 0014 d5 revised |
| L14.3 | a gloss per candidate is a plan per candidate | P5 | rule, applied | ADR 0013 d3, ADR 0014 d10 |
| L14.4 | residency says where, not how long the provider keeps it | P11 | amend, applied | ADR 0014 d3; ADR 0014 d3 revised |
| L14.5 | identity and residency at the keystroke are undefined | P12 | rule, applied | ADR 0014 d4, d5 |
| L14.6 | ranking is non-deterministic | none | accepted | ADR 0014 d2, d9 |
| L15.1 | `--replace` removes rows the input never contained | P7 | amend, applied | ADR 0015 d4; ADR 0015 d4 revised |
| L15.2 | `release` on a table cannot fast-forward a moving main | P1 | rule, applied | ADR 0015 d4 |
| L15.3 | the warehouse estimate assumes on-demand billing | P5 | rule, applied | ADR 0015 d6, d8 |
| L15.4 | an aborted write leaves staged files | P3 | rule, applied | ADR 0015 d4, d5 |
| L15.5 | a hold tag is removable by any table writer | P11 | amend, applied | ADR 0015 d5, ADR 0006 a2; ADR 0015 d5 revised |
| L15.6 | the text serialization is lossy for types | P2 | accepted | ADR 0015 d2 |
| L15.7 | warehouse residency is per dataset, not per device | P12 | rule, applied | ADR 0015 d1 |
| L16.1 | nodes exist nowhere in the design | none | amend, applied | ADR 0011 d7; ADR 0011 a5 |
| L16.2 | receipts and `ps` do not say where a partition ran | P12 | rule, applied | ADR 0009, ADR 0013 d5 |
| L16.3 | a drain is a signal routed per backing object | P12 | rule, applied | ADR 0008, ADR 0004 a1 |
| L16.4 | a node has a pool manager or nobody | P1 | rule, applied | ADR 0010 a1, ADR 0011 |
| L16.5 | an absent health source must read unknown | P7 | rule, applied | ADR 0010, ADR 0011 d7 |
| L16.6 | a cordoned node bills and counts as capacity | P5 | rule, applied | ADR 0003, ADR 0011 d7 |
| L16.7 | KILL cannot end a process in kernel wait; fencing a live kernel corrupts | P2, P3 | amend, applied | ADR 0008, ADR 0006; ADR 0008 a4 |
| L17.1 | a launched process cannot hold a person's token | P12 | amend, applied | ADR 0016 d4, d5 |
| L17.2 | removal at the provider lands at token refresh | P6, P7 | amend, applied | ADR 0016 d6, d9; ADR 0010 |
| L17.3 | a refused approval needs a fallback group | P10 | rule, applied | ADR 0016 d7, ADR 0013 |
| L17.4 | `sudo` with a shell redirect runs as the user | P2 | rule, applied | ADR 0013 d10, command map |
| L17.5 | a group can vanish at the provider | P1, P3 | amend, applied | ADR 0016 d9, ADR 0011 |
| L17.6 | `ps -A` shows actor names fleet-wide | P4 | rule, applied | ADR 0013 d5, ADR 0009 |
| L17.7 | a cron entry whose author left is not flagged | P12 | rule, applied | ADR 0013, ADR 0011 d7 |
| L17.8 | a prompt-set secret is known to its actor | P11 | rule, applied | ADR 0007 a3, ADR 0013 |
| L18.1 | the plan hash must be a shape or no reuse window matches | P12 | amend, applied | ADR 0017 d6, d10 |
| L18.2 | the controller resumes a gate, not a session | P3 | amend, applied | ADR 0017 d4; ADR 0016 d5, d9 |
| L18.3 | void reasons enumerated; re-price at resume | P1, P6, P8 | rule, applied | ADR 0017 d6; ADR 0006 a2; ADR 0011 a4 |
| L18.4 | identical pending requests share one object | P5, P10 | rule, applied | ADR 0017 d1, d4 |
| L18.5 | answers are visible to the requesting actor | P4 | rule, applied | ADR 0017 d8; L17.6 |
| L18.6 | every policy has a bounded timeout; expiry releases holds | P5 | rule, applied | ADR 0017 d2, d7; ADR 0010 |
| L18.7 | a canary and its promotion are two gates | none | accepted | ADR 0017 d6 |
| L19.1 | cost by group needs declared, settled, idle, `?` and unverified rows | P5 | rule, applied | ADR 0010 L7.2, ADR 0018 d6, ADR 0011 d7 |
| L19.2 | ownership walks ownerReferences; the controller labels children | P2, P3, P12 | rule, applied | ADR 0018 d1, d5, d6; ADR 0010 |
| L19.3 | adoption writes labels only; a unit record is separate | P9 | rule, applied | ADR 0018 d5, ADR 0013 |
| L19.4 | a shell-only cluster marks cost and holds unverified; chown root-only | P7 | rule, applied | ADR 0018 d3, d5; ADR 0011 d7 |
| L19.5 | the actor is an annotation, never a label | P4 | amend, applied | ADR 0018 d1, d2; L17.6 |
| L19.6 | a recreated group is a new group with a new slug | none | accepted | ADR 0018 d7, ADR 0016 d9 |
| L19.7 | managed-by is left to any existing manager | P1 | amend, applied | ADR 0018 d1, ADR 0010 a1 |
| L20.1 | `expose` on a managed unit writes intent | P1 | rule, applied | ADR 0019 d2, d3; ADR 0010 a1 |
| L20.2 | session reach: owned units by port-forward, granted units by internal route | P2 | amend, applied | ADR 0019 d3, d7; ADR 0016 d8 |
| L20.3 | a withdrawn hostname stays with its group for a hold period | P3 | rule, applied | ADR 0019 d8, ADR 0018 d7 |
| L20.4 | internal certificates come from a private issuer only | P4 | rule, applied | ADR 0019 d2, d8 |
| L20.5 | cross-zone and cross-region calls are priced; Services prefer the same zone | P5 | rule, applied | ADR 0019 d10, ADR 0011 L10.5 |
| L20.6 | revoking a grant lists running callers and tells them | P7 | rule, applied | ADR 0019 d3, ADR 0017 |
| L20.7 | hostname and path pre-flight against every route on the Gateway | P9 | rule, applied | ADR 0019 d5, d9; ADR 0018 d5 |
| L20.8 | TLS termination and access logs are residency facts | P11 | amend, applied | ADR 0019 d2, d5; ADR 0011 a3 |
| L20.9 | promotion to a Job re-plans reach as `--connect` | P12 | rule, applied | ADR 0019 d3, d7; ADR 0005 |
| L20.10 | weights move new connections only; report open streams and drain | none | rule, applied | ADR 0019 d9, ADR 0013 |
| L20.11 | an internal notebook is the group's, not the person's | none | accepted | ADR 0019 d2, d3; ADR 0016 d3 |
| L21.1 | any eviction of a session pod is a drain with notice; no spot, do-not-disrupt while local work runs | P1, P7 | rule, applied | ADR 0020 d3, d8, d12; ADR 0008 |
| L21.2 | the journal is encrypted; redacted env values are memory-only | P11 | amend, applied | ADR 0020 d2; ADR 0007 |
| L21.3 | the generated kubeconfig holds no token | P4 | rule, applied | ADR 0020 d10; ADR 0016 d4 |
| L21.4 | `--share` is same-user; another member gets a consented read-only watch | P10 | amend, applied | ADR 0020 d6; ADR 0016 d2 |
| L21.5 | `newgrp` across residencies is refused | P11 | rule, applied | ADR 0020 d8; ADR 0016 d1, d11 |
| L21.6 | scrollback is bounded and memory-only; resume says what was dropped | P2 | rule, applied | ADR 0020 d5; ADR 0009 |
| L21.7 | prompt marks carry their age; stale shows `?` | P6 | rule, applied | ADR 0020 d11; ADR 0011 a4 |
| L21.8 | dormancy stops local commands and resumes them on wake | P3 | rule, applied | ADR 0020 d4, d7; ADR 0008 |
| L21.9 | no exec into session pods; break-glass by approval, shown by `who` | P9 | rule, applied | ADR 0020 d8; ADR 0016 d6; ADR 0017 |
| L21.10 | a local command past `local_max` warns and is receipted | P7 | rule, applied | ADR 0020 d3; ADR 0013 |
| L21.11 | a one-host residency waits in `restoring` | P8 | accepted | ADR 0020 d8; ADR 0016 d11 |
| L22.1 | scrub scope follows device writes and reads forward from them | P11, P12 | amend, applied | ADR 0007 a5; ADR 0012 d2; ADR 0015 d4 |
| L22.2 | each device class reports what scrub it can do; retention dates for the rest | P11 | rule, applied | ADR 0007 a5; ADR 0012 d3, d10; ADR 0015 d4, d5 |
| L22.3 | schema cache keyed by registry and id, with fingerprints | P6 | rule, applied | ADR 0012 d10 |
| L22.4 | consumer groups past matched offsets listed and paged | P9 | rule, applied | ADR 0007 a5; ADR 0012 L12.6 |
| L22.5 | purge needs an approver from each object's owning group | P10 | rule, applied | ADR 0007 a5; ADR 0017 d2 |
| L22.6 | scrub match locations visible to security and owners only | P4 | rule, applied | ADR 0007 a5; ADR 0016 L17.6 |
| L22.7 | fixed text forms for decimals, timestamps, bytes, enums, unions | P2 | rule, applied | ADR 0012 d10; ADR 0015 d2 |
| L22.8 | schema-aware scan of string fields; budget binds in order of exposure | P5 | rule, applied | ADR 0007 a5; ADR 0013 d3 |
| L22.9 | a held snapshot keeps leaked rows readable until the hold expires | none | accepted | ADR 0007 a5; ADR 0015 d5, L15.5 |
| L23.1 | an empty node the controller drained is replaced without a person | P10, P5 | amend, applied | ADR 0021 d7, d8; ADR 0010 a1 |
| L23.2 | a GPU reset waits until no process holds the GPU | P2 | amend, applied | ADR 0021 d4; ADR 0011 a5 |
| L23.3 | fault spread computed along the path to the device (mount target, zone) | P8 | amend, applied | ADR 0021 d3; ADR 0011 a1, a4 |
| L23.4 | a drain caused by a component the checkpoint needs skips the checkpoint | P7 | rule, applied | ADR 0021 d7; ADR 0008 a1; ADR 0004 a1 |
| L23.5 | the health model gets named kernel facilities only, never audit records | P4 | rule, applied | ADR 0021 d6; ADR 0014 d5 |
| L23.6 | readings carry the collector version; a step with a rollout is the rollout's | P6 | rule, applied | ADR 0021 d5, d8 |
| L23.7 | `caps` says whether the platform repairer honours a drain | P1 | rule, applied | ADR 0021 d10; ADR 0008 a1 |
| L23.8 | burn-in capacity is shown as joining, with an estimated start | P5 | rule, applied | ADR 0021 d11; ADR 0003 L16.6; ADR 0013 d5 |
| L23.9 | at most one abandoned probe per component | P3 | rule, applied | ADR 0021 d2; ADR 0008 a4 |
| L23.10 | the policy runs at a cut-off site; verdicts queue for the link | P8 | rule, applied | ADR 0021 d5, d6; ADR 0011 a4; ADR 0009 a1 |
| L23.11 | infrastructure retries are counted apart from `--retries` | P12 | rule, applied | ADR 0004 a1; ADR 0021 d7 |
| L23.12 | a component with no probe class degrades with no verdict | none | accepted | ADR 0021 consequences |
| L24.1 | the laptop's working tree and local images have no path into the session | P11 | **open** (drafted: ADR 0022) | ADR 0005; ADR 0011 a2; ADR 0020 d5 |
| L24.2 | the plan checks the digest's platforms against the target nodes | P7 | rule, applied | ADR 0013 d3, d4; ADR 0011 d7 |
| L24.3 | a unit's env is its declaration; session env only when named | P12 | amend, applied | ADR 0013 d4, d7; ADR 0005; ADR 0001 |
| L24.4 | `explain --at` and the release diff against where the digest last passed | P6, P8 | amend, applied | ADR 0013 d3, d7; ADR 0010; ADR 0017 |
| L24.5 | local stages under a source mount run in its `image=` | P2 | rule, applied | ADR 0003; ADR 0011 a2; ADR 0013 d3 |
| L24.6 | `--connect` carries a version range checked against `/sys` | P9 | rule, applied | ADR 0019 d3; ADR 0011 d7 |
| L24.7 | `explain` prints the namespace's defaulted limits, quota and Pod Security | P9 | rule, applied | ADR 0013 d3; ADR 0010 |
| L24.8 | branch units carry a TTL and a daily cost | P3, P5 | rule, applied | ADR 0010; ADR 0013 d7; ADR 0006 |
| L24.9 | provenance is a mount requirement checked at plan time; secret-shaped files in layers refused | P4 | rule, applied | ADR 0010; ADR 0013 d7; ADR 0007 a2 |
| L24.10 | undeclared egress is printed; units declare `--egress`; drops counted | P7 | rule, applied | ADR 0019 d4; ADR 0002; ADR 0013 d4 |
| L24.11 | no interactive debugging of production, no callbacks to the laptop | none | accepted | ADR 0005; ADR 0019 d7; ADR 0016 d6 |
| L25.1 | a sparse lend of one root as one tree; build context outside the snapshot refused at plan time | P7 | amend, applied | ADR 0022 d1, d4 |
| L25.2 | `lend --dry-run` lists sent, excluded and withheld files; a build needing one is refused | P7 | rule, applied | ADR 0022 d2, d4; ADR 0007 a2 |
| L25.3 | data files refused by `lend_exclude`, naming the residency | P11 | rule, applied | ADR 0022 d2, d3; ADR 0010; ADR 0016 d11 |
| L25.4 | a snapshot is current only when complete; request by path is authorization, not lazy fetch | P8 | amend, applied | ADR 0022 d1, d2, d3; ADR 0011 a4; ADR 0020 d5 |
| L25.5 | a followed unit is digest and tree; a digest passes only for the tree it was built from | P6 | rule, applied | ADR 0022 d7; ADR 0013 d7 |
| L25.6 | a debugger pause is shown and bounded by `debug_pause_max` | P10 | rule, applied | ADR 0022 d8; ADR 0008; ADR 0010 |
| L25.7 | dev units are `<name>-<user>` with their own address; a shared unit's name is refused | P10 | amend, applied | ADR 0022 d6; ADR 0019 d2, d3; ADR 0013 d4 |
| L25.8 | `--follow` coalesces builds, caps them per hour and shows their cost | P5 | rule, applied | ADR 0022 d7; ADR 0010 |
| L25.9 | build steps run without registry credentials, with `build_egress`; the builder pushes | P4 | rule, applied | ADR 0022 d4; ADR 0016 d5; ADR 0019 d4 |
| L25.10 | the worktree id carries the user; snapshots are the lender's unless shared | P4 | rule, applied | ADR 0022 d1, d3; ADR 0016 d3; ADR 0011 a2 |
| L25.11 | a snapshot's tree id is the git tree hash; a CI digest names the snapshot that passed | P6 | amend, applied | ADR 0022 d1, d5; ADR 0001; ADR 0013 d7 |
| L25.12 | dev units refused in mounts with `release_path=git` | P1 | rule, applied | ADR 0022 d6; ADR 0010 a1 |
| L25.13 | local stages under a worktree get a discarded writable overlay | P2 | rule, applied | ADR 0003; ADR 0022 d1 |
| L25.14 | secret-shaped plain env values refused; `--env-of` writes none | P12 | rule, applied | ADR 0022 d9; ADR 0013 d4; ADR 0007 a2 |
| L25.15 | `--like` lists the target's policies it does not reproduce | P9 | rule, applied | ADR 0022 d6; ADR 0013 d3 |
| L25.16 | no save-to-reload latency target; the measured time is printed | none | accepted | ADR 0022 d7; ADR 0020 d5 |
| L26.1 | a canary condition reads the canary version only, gateway responses included | P7 | amend, applied | ADR 0023 d5; ADR 0013 d7; ADR 0019 |
| L26.2 | the settle state uses the tenant's out-of-order window | P6 | amend, applied | ADR 0023 d3, d12; ADR 0001 a1 |
| L26.3 | receipts record source and resolution; `replay --verify` fails across downsampling | P6 | rule, applied | ADR 0023 d3; ADR 0006 a2 |
| L26.4 | residency binds processing and storage, not where the reader sits | P11 | accepted | ADR 0016 d11; ADR 0020; ADR 0005 |
| L26.5 | `telemetry_readers` groups read without owning, audited | P10 | amend, applied | ADR 0023 d8; ADR 0016 d3, d6; ADR 0017 |
| L26.6 | artifacts from telemetry reads keep the source's retention and its scrub | P11, P3 | rule, applied | ADR 0023 d3; ADR 0007 a2, a5; ADR 0006 |
| L26.7 | an unestimable read is bounded by its upper bound | P5 | rule, applied | ADR 0023 d9; ADR 0014 d6 |
| L26.8 | estimates use statistics endpoints only, never a data query | P5 | rule, applied | ADR 0023 d9 |
| L26.9 | only substring stores narrow by `grep -F` | P6 | amend, applied | ADR 0023 d10; ADR 0001; ADR 0002 |
| L26.10 | a read across stores names the missing ones and is degraded | P8 | rule, applied | ADR 0023 d8, d12; ADR 0004 a2 |
| L26.11 | `caps: label-keys`; joins fall back to pod names | P9 | rule, applied | ADR 0023 d6; ADR 0018 |
| L26.12 | stderr of a multi-pod process is prefixed by partition and attempt | P2 | amend, applied | ADR 0023 d7; ADR 0009; ADR 0013 d5 |
| L26.13 | `slo:` names platform SLO objects; shadowing refused | P1 | rule, applied | ADR 0023 d5, d6; ADR 0010 a1 |
| L26.14 | telemetry reads pass the redactor before terminal, journal and artifacts | P4 | rule, applied | ADR 0023 d2; ADR 0020 L21.2; ADR 0007 a5 |
| L26.15 | live columns from one controller cache per store, cost shared | P5 | rule, applied | ADR 0023 d4, d9; ADR 0013 d5 |
| L26.16 | unattended conditions read as `svc:<group>`; long unknown rolls a canary back | P12 | rule, applied | ADR 0023 d4, d5; ADR 0016 d5 |
| L27.1 | Terraform state, plans and cross-provider translation have no place in the shell | P9 | **open** (drafted: ADR 0024) | debate-review; ADR 0010; ADR 0011; ADR 0014 |
| L27.2 | Terraform state and plans are a secret sink | P4 | rule, applied | ADR 0007 a2; ADR 0020 L21.2 |
| L27.3 | account resources in no state are listed before translation | P9 | rule, applied | ADR 0011 d7; ADR 0018 |
| L27.4 | writes to Terraform-managed resources are intent to the IaC repository | P1 | amend, applied | ADR 0010 a1 |
| L27.5 | a table copy rewrites its metadata; byte copies of tables refused | P5, P7 | rule, applied | ADR 0015 d2, d4; ADR 0011 |
| L27.6 | large cross-provider copies use the provider's transfer service, verified | P5 | rule, applied | ADR 0003 a1; ADR 0011 |
| L27.7 | replication packages provide lag; the cutover waits on it | P3, P10 | rule, applied | ADR 0012; ADR 0013 d5 |
| L27.8 | rollback after moving writes needs reverse replication | P8 | rule, applied | ADR 0013 d7; ADR 0006 |
| L27.9 | decommission refused while holds; `hold move` keeps expiry and chain of custody | P11 | rule, applied | ADR 0006 a1, a2; ADR 0001 a2 |
| L27.10 | receipts and holds naming a broker's offsets are listed and copied before it goes | P6 | rule, applied | ADR 0012 d2; ADR 0006 a2 |
| L27.11 | `secret move` rotates on the target; values copied only controller-side | P4 | rule, applied | ADR 0007 a3, a4; ADR 0017 |
| L27.12 | groups resolved from the directory when the token carries none | P12, P10 | amend, applied | ADR 0016 d1, d2 |
| L27.13 | preemption grace bounded by the pool's notice | P2 | amend, applied | ADR 0008; ADR 0004 a1 |
| L27.14 | the release diff covers the clusters' `caps` and the verbs they disable | P8 | rule, applied | ADR 0013 d3; ADR 0011 |
| L27.15 | script bands include units over a horizon; `cost --reprice-at` | P5, P10 | rule, applied | ADR 0013 d7; ADR 0010; ADR 0017 |
| L27.16 | commitments, credits and discounts are not in the forecast | P5 | accepted | ADR 0010 |
| L27.17 | a canary across providers writes DNS weights as intent | P1 | rule, applied | ADR 0019 d2; ADR 0013 d7 |
| L28.1 | one approval record for the repository review and the Convoy gate | P10 | amend, applied | ADR 0024 d4; ADR 0017 d1, d5 |
| L28.2 | `mknod` shows its stage; `kill` closes the PR before apply, defers during it | P2 | rule, applied | ADR 0024 d3, d5; ADR 0004; ADR 0013 d5 |
| L28.3 | a re-plan voids the approval and re-asks with the diff | P6 | rule, applied | ADR 0024 d4; ADR 0017; L18.3 |
| L28.4 | claimed `caps` are verified against the provider before registration | P6 | rule, applied | ADR 0024 d5; ADR 0011 d7 |
| L28.5 | destroys of data resources a change did not name are refused; a workspace per instance | P1 | rule, applied | ADR 0024 d4, d6; ADR 0006 |
| L28.6 | a partial apply registers `iac: partial`, priced and completable | P3 | rule, applied | ADR 0024 d5; ADR 0006 |
| L28.7 | delegated paths take package instances only; principals limited to the group | P4 | rule, applied | ADR 0024 d2, d10; ADR 0016 d5 |
| L28.8 | usage-priced bands from contract inputs; plans reused per input hash | P5 | rule, applied | ADR 0024 d4; ADR 0010 |
| L28.9 | an unreachable runner is reported; out-of-band devices marked for adoption | P8 | rule, applied | ADR 0024 d8, d10 |
| L28.10 | adoption needs a no-change plan; unmodelled attributes listed or passed through | P9 | rule, applied | ADR 0024 d8 |
| L28.11 | where the runner publishes plans is declared; sensitive inputs are references | P11 | rule, applied | ADR 0024 d7, d10; ADR 0007 a2 |
| L28.12 | platform class defaults fill contract inputs before module defaults | P12 | amend, applied | ADR 0024 d2; ADR 0010 |
| L28.13 | package upgrades never re-plan instances; each instance upgrades itself | P1 | rule, applied | ADR 0024 d2, d6; ADR 0010 |
| L28.14 | `--like` refused on unsupported inputs unless `--drop` names them | P7 | amend, applied | ADR 0024 d9; ADR 0012 |
| L28.15 | a clean plan does not guarantee a clean apply | none | accepted | ADR 0024 d4 |
| L29.1 | compute is a Kubernetes device class only; no class or driver contract for other computers | none | **open** (drafted: ADR 0025) | ADR 0003; ADR 0011 d1, d2, d9; ADR 0013 d8; ADR 0004; ADR 0008 |
| L29.2 | each computer lists the signals it delivers; others refused; `TERM` on functions stops dispatch | P2 | rule, applied | ADR 0008; ADR 0013 d8, d10 |
| L29.3 | partitions sized to the target's `max_duration`, or refused at plan time | none | amend, applied | ADR 0003; ADR 0002; ADR 0004 |
| L29.4 | a scheduler's requeue is a new attempt, appended and prefixed, never a retry | P12 | rule, applied | ADR 0004; ADR 0009; ADR 0023 d7 |
| L29.5 | computers declare `identity=per-user\|per-group\|shared`; shared is printed and refusable | P4 | rule, applied | ADR 0016 d4, d5; ADR 0011 d4 |
| L29.6 | each computer declares its price model; allocations shown in their own units | P5 | rule, applied | ADR 0010; ADR 0009 |
| L29.7 | placement: residency, then `caps` and bounds, then locality, ranked by cost and estimated start | P8 | amend, applied | ADR 0003 a1; ADR 0013 d3; ADR 0004 |
| L29.8 | input residency binds placement, and a computer's log sink counts | P11 | rule, applied | ADR 0011 a3; ADR 0016 d11; ADR 0023 d8 |
| L29.9 | `release` and `scale` on a module-installed VM pool write module inputs as intent | P1 | rule, applied | ADR 0010 a1; ADR 0024 d6; ADR 0013 d7 |
| L29.10 | the remaining deadline compiles into the scheduler's time limit; steps are members | P12 | rule, applied | ADR 0004 a3; ADR 0008 a1 |
| L29.11 | image conversion starts from the digest, cached, both hashes in the receipt | P6 | rule, applied | ADR 0022; ADR 0001; ADR 0009 |
| L29.12 | function stages invoke synchronously or with a read failure destination; throttles counted | P7 | rule, applied | ADR 0004; ADR 0004 a2 |
| L29.13 | backend job id in `/sys/proc/<pid>/backend`; unreachable computer means `unknown` | P3 | rule, applied | ADR 0011 a4; ADR 0004; ADR 0006 |
| L29.14 | `operator: external` turns node verbs into requests on the operator's channel | P10 | rule, applied | ADR 0021 d7; ADR 0017; ADR 0011 a5 |
| L29.15 | account-wide limits shown with what others hold; the quota check uses what remains | P9 | rule, applied | ADR 0003; ADR 0011 d7 |
| L29.16 | `expose` compiles to the computer's own enforcement as module intent | none | rule, applied | ADR 0019 d5, d6; ADR 0024 |
| L29.17 | results differ across hardware; the receipt names the computer and versions | none | accepted | ADR 0009 |
| L29.18 | a stage over several topologies is split per topology; only outputs cross | P11 | rule, applied | ADR 0003 a1; ADR 0011 a3; ADR 0012 d5 |
| L30.1 | registries are not a device class: no repositories, tags, digests, referrers, residency or price | none | **open** (drafted: ADR 0026) | ADR 0011 d9, a1, a2; ADR 0006; ADR 0022 d4; ADR 0013 d8 |
| L30.2 | a tag is a name for a digest; moving one owned by GitOps, CI or a replication rule is intent or refused | P1 | rule, applied | ADR 0010 a1; ADR 0013 d7; ADR 0022 d5 |
| L30.3 | `rm` of a pinned digest refused; `du` reports unique and shared layer bytes | P2 | rule, applied | ADR 0006; ADR 0009; ADR 0013 d6 |
| L30.4 | lifecycle policies are device facts; digests they would expire that something pins are protected first | P3 | rule, applied | ADR 0006; L8.6; ADR 0024 d6 |
| L30.5 | repositories are owned by groups; pulls scoped to the pod's group, not the node's role | P4 | rule, applied | ADR 0016 d2, d5; ADR 0018 |
| L30.6 | the image is an input for locality and egress; pull from the nearest replica | P5 | amend, applied | ADR 0003 a1; ADR 0025 d4; ADR 0010 |
| L30.7 | `image-build` resolves every `FROM` to a digest, keys the cache by it; `--refresh-base` | P6 | amend, applied | ADR 0022 d4; ADR 0001; ADR 0009 |
| L30.8 | scan results carry database version and time; stale scans rescanned or refused | P6 | rule, applied | ADR 0013 d3; ADR 0017 |
| L30.9 | `cp` of an image copies its referrers or is refused; promotion is a copy by digest | P7 | rule, applied | ADR 0001; ADR 0013 d8 |
| L30.10 | `caps` reports `verify=enforce\|audit\|none`; `verify=required` refuses the rest | P7 | rule, applied | ADR 0018 d6; ADR 0019 d6 |
| L30.11 | `release` checks the cluster's mirror holds the digest before replacing any replica | P8 | rule, applied | ADR 0011 a4; ADR 0013 d7; ADR 0025 |
| L30.12 | `fuser` on a digest lists every pod running it; images listed by base digest | P9 | rule, applied | ADR 0011 d7; L9.4 |
| L30.13 | a vulnerability exception is an approval object pinned to a digest and a CVE, with an expiry | P10 | rule, applied | ADR 0017 d1; ADR 0016 d7 |
| L30.14 | registries and repositories carry residency; replication or `cp` outside it refused | P11 | rule, applied | ADR 0011 a3; ADR 0007 a2; L24.9 |
| L30.15 | receipts record every container's digest from pod status, injected ones included | P12 | rule, applied | ADR 0009; ADR 0018 |
| L30.16 | scanners disagree; a clean scan is not proof | none | accepted | ADR 0017 |
| L30.17 | production uses public images only by digest from a platform mirror | none | rule, applied | ADR 0022 d5; ADR 0010; ADR 0006 |
| L31.1 | Convoy's own events have no channel, route, subscription or event model | none | **open** (drafted: ADR 0027) | ADR 0017 d2, d4; ADR 0021 d5, d8; ADR 0023 d6; ADR 0011 d9; command map `mailx` |
| L31.2 | routes name what the platform has; Convoy never edits schedules, escalations or alert rules | P1 | rule, applied | ADR 0023 d6; ADR 0010 a1 |
| L31.3 | `write` to a detached user is journaled; notify verbs exit on acceptance, delivery is a state | P2 | rule, applied | ADR 0020 d2; ADR 0004 |
| L31.4 | events go to the owning group's route, never only to a person; a default route catches the rest | P3 | rule, applied | ADR 0016 d5, d9 |
| L31.5 | an approval from a channel counts only with a mapped, re-authenticated account | P4 | amend, applied | ADR 0017 d5; ADR 0024 d4; ADR 0016 d4 |
| L31.6 | events rolled up per process; routes rate limited; messages priced to the owning group | P5 | rule, applied | ADR 0010; ADR 0009 |
| L31.7 | messages carry the object path and state time; later states update or supersede them | P6 | rule, applied | ADR 0017 d8; ADR 0013 d5 |
| L31.8 | a delivery failure is an event to the fallback route; failing routes shown in `explain` | P7 | rule, applied | ADR 0021 d5, d8; ADR 0011 a4 |
| L31.9 | an approval's timeout counts from the first successful delivery of its request | none | amend, applied | ADR 0017 d2, d7; ADR 0004 |
| L31.10 | events at a disconnected site are spooled with receipts and delivered with their raise time | P8 | rule, applied | ADR 0009 a1; ADR 0011 a4 |
| L31.11 | events carry a deduplication key from the object path; Convoy never alerts on telemetry | P9 | rule, applied | ADR 0023 d6; ADR 0018 |
| L31.12 | approval requests go to the on-call of the approving group; further answers asked one at a time | P10 | rule, applied | ADR 0017 d2, d4, d7; ADR 0016 d7 |
| L31.13 | message bodies are path, class, status and scrubbed summary; channels carry residency | P11 | rule, applied | ADR 0007 a2; ADR 0011 a3; ADR 0016 d11 |
| L31.14 | `--notify` applies to a process tree; children and partitions roll up into the root | P12 | rule, applied | ADR 0004 a2, a3 |
| L31.15 | every event class has an urgency: `page`, `notify` or `digest` | none | rule, applied | ADR 0017 d4; ADR 0010 |
| L31.16 | delivery is at least once and best effort; the object is the record | none | accepted | ADR 0009 |
| L31.17 | every object with an expiry warns its owning group at `expiry_warn` | none | rule, applied | ADR 0006; ADR 0026 d8; ADR 0022 d3 |
| L32.1 | memory domains, distances and tiers are not modelled; shared in-memory stores are not devices | none | **open** | ADR 0025 d2, d4; ADR 0011 d7, d9; ADR 0004 a1; ADR 0003 a1 |
| L32.2 | a gang with `--domain` is never split; the domain each member got is recorded | P7 | rule | ADR 0004 a1; ADR 0025 d6; ADR 0003 a1 |
| L32.3 | a domain's capacity is its healthy free size; `explain` refuses when no domain is large enough | P8 | rule | ADR 0025 d2, d4; ADR 0021 d4 |
| L32.4 | a gang restart goes to any healthy domain that can hold it, or ends with exit 3 at the deadline | none | amend | ADR 0004 a1; ADR 0021 d4 |
| L32.5 | work with no domain claim kept out of domain pools; stranded members shown and priced to the platform | P5 | rule | ADR 0010; ADR 0025 d9; ADR 0016 d8 |
| L32.6 | a `fabric` component class, and the domain as a scope above the node | none | amend | ADR 0021 d2, d3 |
| L32.7 | a domain is `draining` until every member's channels are free | P3 | rule | ADR 0008 a1, a4; ADR 0021 d9 |
| L32.8 | device memory cleared between groups and recorded; `mem_scrub=`; one claim, one group | P4 | rule | ADR 0016 d2, d3; ADR 0021 d9 |
| L32.9 | a shared-memory device holds files written once; no writable mapping across nodes | P2 | rule | ADR 0011 d9; command map `ipcs` |
| L32.10 | shared-memory devices are `volatile` with a declared `backing=`; a lost object is `EIO` | P6 | rule | ADR 0001 a2; ADR 0004 rules; ADR 0011 a4 |
| L32.11 | a store inside a process counts in its memory bound; a store is a device only when installed | P9 | rule | ADR 0024 d8; ADR 0016 d3; ADR 0025 d2 |
| L32.12 | domain layout is read from the driver; Convoy creates only the claim | P1 | rule | ADR 0010 a1; ADR 0024 d6; ADR 0021 d10 |
| L32.13 | holding a whole domain past `domain_hold_max` needs the capacity group's approval | P10 | rule | ADR 0017 d2; ADR 0016 d7 |
| L32.14 | spill to a bounded device named in the plan; shuffle service a device; files removed at the end | P11 | rule | ADR 0001 a2; ADR 0011 a3; ADR 0007 a2 |
| L32.15 | `--domain` and `--gang` are not inherited; `--domain=@parent` | P12 | rule | ADR 0004 a3 |
| L32.16 | `explain` sizes a Spark or dataframe stage's memory and spill | none | rule | ADR 0003; pipeline compiler |
| L32.17 | no memory shared across computers; data moves by copy | none | rule | ADR 0025 d5; ADR 0011 d6 |
| L32.18 | distance class is reported, not achieved bandwidth | none | accepted | ADR 0003 a1 |
| L32.19 | CXL memory shown only as the tiers and NUMA nodes the node reports | none | accepted | ADR 0025 d2 |
| L33.1 | a computer's own file systems and staging into and out of them are not in the design | none | **open** | ADR 0025 d2, d4; ADR 0001 a2; ADR 0003 a1; ADR 0011 a1 |
| L33.2 | a staged copy is a cache entry verified before use; purged or changed copies are restaged | P6 | rule | ADR 0001; ADR 0025 d2 |
| L33.3 | staging targets carry residency; staged copies removed when the tree ends unless held | P11 | rule | ADR 0011 a3; ADR 0006; ADR 0009 |
| L33.4 | staging priced in the band; a file system made for a process is removed with it | P5 | rule | ADR 0010; ADR 0025 d9; ADR 0009 |
| L33.5 | a stage succeeds only when its outputs are verified at their destination | P7 | rule | ADR 0004; ADR 0025 d2 |
| L33.6 | `mv` across devices deletes sources only after the whole destination is verified | P2 | rule | command map `mv`; ADR 0015 |
| L33.7 | staged copies and per-process file systems removed after `stage_keep` on any end | P3 | rule | ADR 0006; ADR 0004 a3 |
| L33.8 | on `identity=shared` computers, staging only into per-group directories the computer enforces | P4 | rule | ADR 0016 d4; ADR 0025 d7 |
| L33.9 | transfers resumable per file; a stage starts only on a verified manifest | P8 | rule | ADR 0011 a4; ADR 0001 |
| L33.10 | a stage-out ships only files the process wrote | P9 | rule | ADR 0016 d3; ADR 0001 |
| L33.11 | lasting file systems installed by modules; per-process ones made by the driver | P1 | rule | ADR 0024 d1, d8; ADR 0010 a1 |
| L33.12 | staging into a quota shared outside Convoy above `stage_share_max` needs approval | P10 | rule | ADR 0017 d2; ADR 0025 d9; ADR 0016 d7 |
| L33.13 | stages of one tree on one computer pass artifacts through its file system | P12 | amend | ADR 0001 a2; ADR 0004 a3 |
| L33.14 | placement removes computers whose file systems cannot hold a stage's data | none | rule | ADR 0025 d2, d4; ADR 0003 a1 |
| L33.15 | read in place unless staging is cheaper or faster; `explain` says which | none | rule | ADR 0003 a1; walkthrough 2 |
| L33.16 | `explain` of a copy prints file count, request cost and metadata limit | none | rule | ADR 0010; ADR 0025 d2 |
| L33.17 | copy time over a shared link is estimated, not promised | none | accepted | ADR 0025 d2 |
| L33.18 | a stage-in reads a pinned snapshot of its source | none | rule | ADR 0001; ADR 0015 |
| L33.19 | linked file systems show each file's load state; unloaded bytes count as stage-in | none | rule | ADR 0011 d9; command map `ls`, `du` |

## Structural misroutes

Reading the register across stories, the findings cluster into fifteen patterns. These are the places where the abstraction routes a verb to the wrong layer, not bugs in one story. Each should be settled once, as a principle, before the stories that depend on it are built.

| # | Misroute | Pattern | Findings | Settle in |
|---|---|---|---|---|
| S1 | **Degraded has no channel** | ADR 0004 defines "exit 0 with a degraded flag", but POSIX composition only carries `$?`. Pipes, `wc -l` and scripts read a partial fleet as a complete one. | L9.5, L7.5, L26.10, L31.14, L32.2 | ADR 0004 |
| S2 | **The process tree does not inherit the contract** | Budgets, deadlines, residency and signals are per process; children, group members and retries fall outside them. | L7.3, L2.3, L5.5, L1.3, L23.11, L24.3, L26.16, L29.4, L29.10, L30.15, L32.15, L33.13 | ADR 0004, ADR 0005, ADR 0008 |
| S3 | **Policy on the verb, not on the mount** | Residency, egress and retention as flags are forgotten; as mount options they bind every command. | L8.2, L7.6, L7.7, L24.9 | ADR 0011, ADR 0010 |
| S4 | **Imperative verbs on reconciled objects** | Anything GitOps owns must be written as intent: image, route weight, scale, restart. G3 closed one case of a general rule. | L3.2, L9.3, L3.3, L20.1, L23.7, L25.12, L27.4, L27.17, L28.5, L28.13, L29.9, L30.2, L31.2, L32.12, L33.11 | ADR 0010 |
| S5 | **Hierarchical paths over flat names** | `cd`, `mkdir`, `rmdir` imply a tree; namespaces are a flat list with cascading deletion. | L6.1, L6.2, L6.6 | ADR 0011 |
| S6 | **The artifact store is data at rest** | Pipes, receipts and logs have a location, a retention and a leak surface; the design treated them as plumbing. | L8.1, L8.3, L8.6, L7.1, L4.1, L1.5, L21.2, L22.1, L24.1, L25.3, L25.4, L26.6, L27.9, L30.3, L30.14, L32.14, L33.3, L33.7 | ADR 0001, ADR 0006, ADR 0007, ADR 0011, ADR 0020 |
| S7 | **Self-reported numbers in the ledger** | Metered spend, receipt metrics and executable bands come from the thing being measured. | L7.2, L4.2, L2.2 | ADR 0009, ADR 0010 |
| S8 | **Multi-party actions in a single-actor shell** | Approvals, producer/consumer rotation and forced retirements need a second identity; the shell has one stdin. | L3.1, L9.1, L9.8, L6.7, L25.6, L25.7, L26.5, L28.1, L30.13, L31.5, L31.12, L32.13, L33.12 | ADR 0005, identity design |
| S9 | **The created-elsewhere blind spot** | Executors, namespaces, consumers and inputs the planner did not create are seen by label, index or not at all. | L9.4, L2.4, L6.6, L8.3, L20.7, L22.4, L24.6, L24.7, L25.15, L26.11, L26.13, L27.1, L27.3, L28.10, L29.15, L30.4, L30.12, L30.17, L31.11, L32.11, L33.10 | ADR 0007, ADR 0011, ADR 0019 |
| S10 | **Impossible pause, unbounded freeze** | Pause-with-checkpoint and group freeze are the default lifecycle verbs; some workloads have no checkpoint and collectives time out. | L2.6, L5.2, L4.3, L27.13 | ADR 0004, ADR 0008 |
| S11 | **Disconnected is a state, not a failure** | Device state, receipts, secrets and placement all assume the session can reach the cluster. At the edge the link is part of the system: a down device must queue, receipts must be spooled where the work ran, secrets must resolve locally, and compute must go to the data. | L10.1, L10.2, L10.6, L10.7, L10.5, L23.10, L25.4, L29.13, L30.11, L31.10, L33.9 | ADR 0003, ADR 0007, ADR 0009, ADR 0011 |
| S12 | **A log is append-only; reruns, replacements and deletes do not mean what they mean on a file** | A rerun of a write appends beside the old records, a replacement is impossible, a delete is immediate and total, and an unbounded read never ends. Every verb that assumes file semantics must say what it does on a log. | L11.2, L11.3, L12.2, L13.1, L13.5, L22.2, L27.10, L33.6 | ADR 0012, ADR 0007 |
| S13 | **Context that leaves the session is data at rest somewhere else** | Anything sent to a model, a provider or a remote service for the user's convenience (a comment to translate, a candidate list, a plan) is retained there under rules the session does not control; residency names a place, not a duration, and free text typed by a user is not covered by "secrets are references". | L14.2, L14.4, L14.5, L20.4, L20.8, L21.5, L23.5, L25.14, L26.4, L26.14, L28.11, L29.8, L29.18, L31.13 | ADR 0014 decisions 3 and 5, ADR 0019 decision 2, ADR 0020 decision 8 |
| S14 | **The node is not in the tree** | Clusters are devices and pods are processes, so the layer where disks, GPUs and memory fail has no path, no health file, no verb and no column in `ps`; every hardware incident starts outside the shell, and the one verb that exists for a hung process (`kill -KILL`) cannot work there. | L16.1, L16.2, L16.7, L23.2, L23.9, L29.14, L32.1, L32.6 | ADR 0011 decision 7, ADR 0008, ADR 0021 |
| S15 | **Identity changes outside the shell** | Users and groups are born and die at the identity provider, and a token is a cache of membership for its lifetime; Convoy can only observe a removal, at the event where the provider pushes one and at token expiry otherwise, and it can refuse nothing the provider does. Every rule about removal, orphaned groups and root assumption must say what the shell sees and when, never pretend to a veto. | L17.1, L17.2, L17.5, L27.12, L31.4 | ADR 0016 decisions 5, 6 and 9 |

## Amendments by ADR

The first table lists every finding that touches an ADR, whatever its disposition. The second lists the amendments actually written on 2026-11-09 and the findings each applies.

| ADR | Findings that touch it |
|---|---|
| 0001 pipe is a staged artifact | L1.1, L4.5, L8.1 |
| 0003 three backends, local first | L10.7, L10.9 |
| 0002 annotation classes | L1.2, L12.2 |
| 0004 completion contract | L1.3, L2.6, L4.4, L5.1, L5.5, L7.3, L7.4, L8.4, L8.7, L9.5, L23.11 |
| 0005 session is server-side | L3.1, L6.7, L7.3, L9.5, L24.1, L24.3, L24.11 |
| 0006 detach is not destroy | L1.5, L5.6, L6.2, L8.3, L8.6, L11.4, L13.1 |
| 0007 secrets are references | L3.3, L6.4, L7.1, L9.1, L9.2, L9.4, L9.7, L9.8, L10.6, L22.1, L22.2, L22.4, L22.5, L22.6, L22.8, L22.9 |
| 0008 signal routing | L2.3, L5.2, L9.2, L9.3, L9.6, L23.4, L23.7 |
| 0009 stdout artifact, stderr log | L4.2, L7.1, L8.5, L9.7, L10.2 |
| 0010 platform defaults | L2.2, L3.2, L4.3, L6.2, L7.2, L7.6, L8.2, L8.7, L9.3, L10.9, L12.3, L13.3 |
| 0011 device tree | L2.1, L2.5, L3.3, L4.1, L5.3, L6.1, L7.7, L8.1, L8.2, L10.1, L10.5, L10.8, L24.1, L24.2, L24.5 |
| 0012 logs and queues are devices | L10.3, L11.1, L11.2, L11.3, L11.5, L11.6, L12.1, L12.2, L12.4, L12.5, L12.6, L13.1, L13.2, L13.3, L13.4, L13.5, L13.6, L22.2, L22.3, L22.7 |
| 0013 native verbs have one shape (draft) | closes as verb entries: L3.5, L4.4, L5.5, L8.7, L10.9, L11.4, L12.2, L13.5; references without deciding: L3.1, L7.3, L9.5; L14.3; L24.2, L24.3, L24.4, L24.5, L24.7, L24.8, L24.9, L24.10 |
| 0014 assisted completion (draft) | L14.1, L14.2, L14.3, L14.4, L14.5, L14.6, L23.5 |
| 0015 tables and catalogs are devices (draft) | L15.1, L15.2, L15.3, L15.4, L15.5, L15.6, L15.7 |
| 0008 signal routing (node fault) and 0011 device tree (node tree) | L16.1, L16.3, L16.4, L16.5, L16.6, L16.7 |
| 0016 identity is a user, rights belong to groups (draft) | L17.1, L17.2, L17.3, L17.5, L17.6, L17.7; references: L3.1, L6.5 |
| 0017 approvals are objects on the process (draft) | closes L3.1; gives L6.2, L9.8, L13.1 and L17.3 their object; L18.1, L18.2, L18.3, L18.4, L18.5, L18.6, L18.7 |
| 0018 labels are the signal, admission is the decision (draft) | gives L6.6, L7.2, L12.6 and ADR 0016 decision 8 their mechanism; L19.1, L19.2, L19.3, L19.4, L19.5, L19.6, L19.7 |
| 0019 a unit has an address, reach is a grant (draft) | drafts L4.6; relies on L7.6, L10.5, L10.9 and L19.4; L20.1, L20.2, L20.3, L20.4, L20.5, L20.6, L20.7, L20.8, L20.9, L20.10, L20.11; L24.6, L24.10, L24.11 |
| 0020 a session is a journal, not a pod (draft) | the design behind ADR 0005; relies on L6.7, L18.2 and L20.9; L21.1, L21.2, L21.3, L21.4, L21.5, L21.6, L21.7, L21.8, L21.9, L21.10, L21.11 |
| 0021 a node is tested before it is trusted (draft) | automates the cordon of L16.1 and relies on L16.6 and L16.7; L23.1, L23.2, L23.3, L23.4, L23.5, L23.6, L23.7, L23.8, L23.9, L23.10, L23.11, L23.12 |
| 0022 the laptop lends a tree, the cluster builds it (draft) | drafts L24.1; relies on L24.2, L24.3, L24.4, L24.5, L24.8, L24.9, L24.10 and L24.11; L25.1, L25.2, L25.3, L25.4, L25.5, L25.6, L25.7, L25.8, L25.9, L25.10, L25.11, L25.12, L25.13, L25.14, L25.15, L25.16 |
| 0023 telemetry is a device the shell reads (draft) | replaces L3.5 with decision 5; L26.1, L26.2, L26.3, L26.5, L26.6, L26.7, L26.8, L26.9, L26.10, L26.11, L26.12, L26.13, L26.14, L26.15, L26.16 |
| fleet operation across providers (US-27) | L27.1 (Terraform, open); ADR 0010 a1: L27.4; ADR 0016: L27.12; ADR 0008: L27.13; ADR 0007: L27.2, L27.11; ADR 0006: L27.9; ADR 0011: L27.3; ADR 0012: L27.7, L27.10; ADR 0013: L27.8, L27.14, L27.15; ADR 0015: L27.5; ADR 0019: L27.17; ADR 0003: L27.6; L27.16 accepted |
| 0024 a device is installed by Terraform (draft) | drafts L27.1; realises L27.2, L27.3, L27.4 and L27.17 in decisions 6 and 7; L28.1, L28.2, L28.3, L28.4, L28.5, L28.6, L28.7, L28.8, L28.9, L28.10, L28.11, L28.12, L28.13, L28.14, L28.15 |
| compute beyond Kubernetes (US-29) | L29.1 (a compute class, open, drafted in ADR 0025); ADR 0003: L29.3, L29.7 (amend), L29.15, L29.18; ADR 0008: L29.2; ADR 0004: L29.4, L29.10, L29.12; ADR 0016: L29.5; ADR 0010: L29.6, L29.9; ADR 0011: L29.8, L29.13; ADR 0022: L29.11; ADR 0021: L29.14; ADR 0019: L29.16; L29.17 accepted |
| 0025 a computer is a device, Kubernetes is one driver (draft) | drafts L29.1; gives L29.2, L29.4, L29.5, L29.6, L29.8, L29.9, L29.10, L29.11, L29.12, L29.13, L29.14, L29.15, L29.16 and L29.18 their fields; assumes L29.3 and L29.7 in decision 4; the fourteen rules written in its Rules section on 2026-11-13 |
| OCI registries (US-30) | L30.1 (a registry class, open, drafted in ADR 0026); ADR 0003 a1: L30.6 (amend); ADR 0022 d4: L30.7 (amend); ADR 0006: L30.3, L30.4; ADR 0010: L30.2, L30.17; ADR 0016: L30.5; ADR 0013: L30.8, L30.9; ADR 0018: L30.10, L30.15; ADR 0011: L30.11, L30.12, L30.14; ADR 0017: L30.13; L30.16 accepted |
| 0026 a registry is a device, a tag is a name for a digest (draft) | drafts L30.1; makes the held registry of L8.6 a device of the class; gives L30.2, L30.3, L30.4, L30.5, L30.8, L30.9, L30.10, L30.11, L30.12, L30.13, L30.14, L30.15 and L30.17 their decisions; assumes L30.6 and L30.7; the thirteen rules written in its Rules section on 2026-11-13 |
| notification channels (US-31) | L31.1 (channels and events, open, drafted in ADR 0027); ADR 0017: L31.5, L31.9 (amend), L31.7, L31.12, L31.15; ADR 0023: L31.2, L31.11; ADR 0020: L31.3; ADR 0016: L31.4; ADR 0010: L31.6; ADR 0021: L31.8; ADR 0009: L31.10; ADR 0007: L31.13; ADR 0004: L31.14; ADR 0006: L31.17; L31.16 accepted |
| memory beyond a node (US-32) | L32.1 (memory domains and shared-memory devices, open); ADR 0004: L32.4 (amend), L32.2, L32.10, L32.15; ADR 0021: L32.6 (amend), L32.7, L32.8; ADR 0025: L32.3, L32.11, L32.17; ADR 0010: L32.5; ADR 0011: L32.9; ADR 0001: L32.14; ADR 0024: L32.12; ADR 0017: L32.13; ADR 0003 and the pipeline compiler: L32.16; L32.18 and L32.19 accepted |
| data staging (US-33) | L33.1 (a computer's file systems and staging, open); ADR 0001: L33.13 (amend), L33.2, L33.9, L33.10, L33.18; ADR 0025: L33.5, L33.14, L33.16; ADR 0011: L33.3, L33.19; ADR 0010: L33.4; ADR 0006: L33.7; ADR 0016: L33.8; ADR 0024: L33.11; ADR 0017: L33.12; command map: L33.6; ADR 0003 and walkthrough 2: L33.15; L33.17 accepted |
| 0027 Convoy tells a group about its own objects (draft) | drafts L31.1; gives `page:<route>` (ADR 0017) and the health controller's pages (ADR 0021) their channel; gives L31.2, L31.3, L31.4, L31.6, L31.7, L31.8, L31.10, L31.11, L31.12, L31.13, L31.14, L31.15 and L31.17 their decisions; assumes L31.5 and L31.9; the thirteen rules written in its Rules section on 2026-11-13 |

| ADR | Amendment | Applies |
|---|---|---|
| 0001 | a2 the artifact store is a device per topology | L8.1 |
| 0002 | a1 side effects and idempotency | L1.2 |
| 0003 | a1 placement by device locality | L10.7 |
| 0004 | a2 the degraded channel; a3 the contract is inherited | L9.5; L7.3 |
| 0006 | a2 a hold extends to inputs | L8.3 |
| 0007 | a2 the artifact store is a secret sink; a3 a secret has a producer; a4 a mount declares its secret source | L7.1; L9.1; L10.6 |
| 0008 | a3 member signals and the freeze bound | L2.3, L5.2 |
| 0009 | a1 receipts survive a lost link | L10.2 |
| 0010 | a1 managed objects are written as intent | L3.2, L9.3 |
| 0011 | a1 drivers from packages; a2 source repositories and artifact stores; a3 `nsmap` and `residency`; a4 disconnected is a state | L2.1; L3.3, L8.1; L6.1, L8.2; L10.1 |
| 0012 | decisions 1 and 2 revised inline | L13.3; L11.2, L11.5, L12.1 |
| 0014 | decisions 3 and 5 revised inline | L14.4; L14.2 |
| 0015 | decisions 4 and 5 revised inline | L15.1; L15.5 |
| 0011 | a5 nodes are a tree under `/sys` | L16.1 |
| 0008 | a4 kernel wait is unterminatable; fencing is a node verb | L16.7 |
| 0016 | decisions 4, 5, 6 and 9 revised inline | L17.1; L17.2; L17.5 |
| 0017 | decisions 4 and 6 revised inline | L18.2; L18.1 |
| 0018 | decisions 1, 2, 3, 5 and 7 revised inline | L19.5 (all five); L19.7 (decision 1) |
| 0019 | decisions 2 and 7 revised inline | L20.8; L20.2 |
| 0020 | decisions 2 and 6 revised inline | L21.2; L21.4 |
| 0007 | amendment 5 decision 2 revised inline | L22.1 |
| 0021 | decisions 3, 4 and 7 revised inline | L23.3; L23.2; L23.1 |
| 0013 | decisions 3, 4 and 7 revised inline | L24.4 (decisions 3 and 7); L24.3 (decision 4) |
| 0022 | decisions 1, 2, 4, 5 and 6 revised inline; decision 3 by rule L25.10 | L25.1 (decisions 1 and 4); L25.4 (decisions 1 and 2); L25.7 (decision 6); L25.11 (decisions 1 and 5) |
| 0023 | decisions 3, 5, 7, 8 and 10 revised inline | L26.2 (decision 3); L26.1 (decision 5); L26.12 (decision 7); L26.5 (decision 8); L26.9 (decision 10) |
| 0010 | amendment 1 revised inline | L27.4 |
| 0016 | decision 1 revised inline | L27.12 |
| 0008 | preemption grace revised inline | L27.13 |
| 0024 | decisions 2, 4 and 9 revised inline | L28.12 (decision 2); L28.1 (decision 4); L28.14 (decision 9) |
| 0003 | the decision and amendment 1 revised inline | L29.3 (decision); L29.7 (amendment 1) |
| 0003, 0022 | amendment 1 and decision 4 revised inline | L30.6 (ADR 0003 amendment 1); L30.7 (ADR 0022 decision 4) |
| 0017 | decisions 5 and 7 revised inline | L31.5 (decision 5); L31.9 (decision 7) |

## Writing a new story

Copy [TEMPLATE.md](TEMPLATE.md). Keep the persona's words in section 1 and Convoy's words everywhere else. Fill all sixteen aspects even when the answer is `–`, and run all twelve probes even when the answer is `ok`: the register's value is that absence of a finding means it was checked. Number findings `L<story>.<n>`, propose a disposition, name the document it touches, and add the row here. If a finding repeats a pattern above, cite the S-id rather than inventing a new principle.
