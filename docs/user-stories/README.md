# User stories as design probes

The nine workloads in [walkthroughs.md](../walkthroughs.md) showed that the design carries them. This directory asks a harder question of the same nine, plus four written against [ADR 0012](../adr/0012-logs-and-queues-are-devices.md) to reach the edge and the Kafka primitive, and one against [ADR 0013](../adr/0013-native-verbs-have-one-shape.md) and [ADR 0014](../adr/0014-assisted-completion-the-model-proposes.md) for the assisted shell, and one against [ADR 0015](../adr/0015-tables-and-catalogs-are-devices.md) for tables and catalogs, and one fleet operation (a faulty node) that found the node layer missing: for each persona who would run them, does every aspect of the abstraction hold, and where does a verb route to the wrong place? Each story follows [TEMPLATE.md](TEMPLATE.md), which fixes sixteen aspects to cover and twelve loophole probes to run, so that a reader can tell "checked and fine" from "never checked". Findings are collected below with a proposed disposition. **All twenty-nine `amend` findings were applied on 2026-11-09**: as dated amendments to ADR 0001 to 0011 and as inline revisions of the drafts ADR 0012 and 0014, each citing its finding. The sixty `rule` findings were written the same day into a dated Rules section of the ADR each touches, a rules table in the command map, a section of the compiler document and a per-milestone section of the roadmap. The `open` and `accepted` findings stand as written in each story.

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

## Personas

The first five were set by the owner; the other five were added because the walkthroughs kept needing them: someone owns the on-call, someone owns the data pipelines as code, someone must be able to defend the evidence, someone reads the bill, and someone owns the stateful services whose data a careless verb can lose.

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

## Coverage: aspects × stories

`●` the story stresses the aspect, `○` uses it, `–` does not touch it. Read the rows: an aspect with few `●` has few tests.

| Aspect | 01 | 02 | 03 | 04 | 05 | 06 | 07 | 08 | 09 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | Stress |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| A1 Pipe, artifact, cache | ● | ○ | ○ | ● | ○ | – | ● | ● | ○ | ● | ● | ● | – | ○ | ● | ○ | 8 |
| A2 Command, annotation | ● | ● | ○ | ● | ● | ○ | ● | ○ | ● | ○ | ○ | ● | ○ | ● | ○ | – | 8 |
| A3 Session, mount table, attach | ○ | ○ | ● | ○ | ● | ● | ○ | ○ | ● | ● | ○ | ○ | ○ | ● | ○ | ○ | 6 |
| A4 Backend selection | ● | ● | ○ | ● | ● | – | ○ | ○ | – | ● | ● | ● | ○ | ○ | ○ | ○ | 7 |
| A5 Completion contract | ● | ● | ● | ● | ● | – | ● | ● | ● | ● | ● | ○ | ● | ○ | ● | ● | 13 |
| A6 Device tree, mounts, `/sys` | ○ | ● | ● | ○ | ● | ● | ○ | ● | ● | ● | ● | ○ | ● | ● | ● | ● | 12 |
| A7 Signals, groups | ○ | ● | ○ | ○ | ● | ● | ○ | – | ● | ○ | – | ● | – | – | – | ● | 6 |
| A8 Secrets | – | – | ○ | – | – | ● | ● | – | ● | ● | ○ | ○ | ● | ● | ○ | – | 6 |
| A9 Cost | ● | ● | – | ● | ● | ○ | ● | ○ | – | ● | ● | ○ | ● | ● | ● | ● | 11 |
| A10 Identity, approvals | ○ | ○ | ● | ○ | ○ | ● | ○ | ● | ● | ○ | ○ | ○ | ● | ● | ○ | ○ | 6 |
| A11 Defaults and `caps` | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | 16 |
| A12 Storage | – | ● | – | ○ | ● | ● | – | ○ | – | ○ | ○ | – | ○ | – | ● | ● | 5 |
| A13 Networking | – | ○ | ● | ● | ○ | ○ | ● | – | – | ● | ○ | ○ | – | – | – | – | 4 |
| A14 Time, queueing | ● | ○ | ○ | ○ | ● | – | ○ | ● | ○ | ● | ○ | ○ | – | ○ | ○ | ○ | 4 |
| A15 Audit, receipts, retention | ● | ○ | ● | ● | ○ | ○ | ● | ● | ● | ● | ● | ○ | ○ | ● | ● | ● | 11 |
| A16 Durability | ○ | ○ | ● | ○ | ● | ● | ○ | ● | ● | ● | ○ | ● | ○ | ○ | ○ | ● | 8 |

**What no story reaches.** The session attach protocol itself (reconnect mid-command, two terminals on one session), `expose --public` and anything else in the undesigned networking abstraction, `pkg install` with a failing `requires`, `nice`/`renice` under contention, debugging a failing partition beyond locating it (`strace`, ephemeral containers; US-16 covers the locating), and locks. US-10 closed the edge and disconnected-operation row; US-14 covers interactive assistance and the model as a device. US-15 covers tables and catalogs: a backfill from a topic into an Iceberg table with a hold, a branch write published by `release`, and a priced warehouse read. Two candidate stories would close most of the rest: a platform engineer installing a package that a cluster cannot satisfy, and an engineer debugging one failed partition of a thousand.

## Findings register

One hundred and five findings. Disposition counts: **29 amend** (all applied to the ADRs on 2026-11-09), **60 rule** (written into their documents on 2026-11-09), **5 open** (design that does not exist yet), **10 accepted** (a surfaced limitation), 1 cross-reference. Full text is in each story's section 10.

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
| L3.1 | approval through the author's own stdin | P10 | **open** | ADR 0005, identity |
| L3.2 | canary weight patched on a GitOps-managed route | P1 | amend, applied | ADR 0010; ADR 0010 a1 |
| L3.3 | the GitOps repository is not a device | P4, P11 | amend, applied | ADR 0011; ADR 0011 a2 |
| L3.4 | no transaction across clusters | P3, P8 | accepted | M11 |
| L3.5 | metric conditions with no metrics package | P7 | rule, applied | M11 |
| L4.1 | `cp` out of `/artifacts` drops the content id | P11 | rule, applied | ADR 0011 d9, M5 |
| L4.2 | no channel for a command to write receipt fields | P7 | rule, applied | ADR 0009, M1 |
| L4.3 | idle GPU units have no policy | P5 | rule, applied | ADR 0010, M11 |
| L4.4 | fail-fast cancels independent siblings | P2 | rule, applied | ADR 0004 |
| L4.5 | content id of `/s3` objects undefined | P6 | rule, applied | ADR 0001, M5 |
| L4.6 | `expose` depends on undesigned networking | P9 | **open** | stories.md |
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
| L6.5 | tenancy depends on the identity design | P9 | **open** | stories.md |
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
| L9.7 | the leaked value stays in logs and artifacts | P4 | **open** | ADR 0007, ADR 0009 |
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
| L12.5 | schema-encoded topics are opaque to text tools | P11 | **open** | ADR 0012, M5 |
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

## Structural misroutes

Reading the register across stories, the findings cluster into fourteen patterns. These are the places where the abstraction routes a verb to the wrong layer, not bugs in one story. Each should be settled once, as a principle, before the stories that depend on it are built.

| # | Misroute | Pattern | Findings | Settle in |
|---|---|---|---|---|
| S1 | **Degraded has no channel** | ADR 0004 defines "exit 0 with a degraded flag", but POSIX composition only carries `$?`. Pipes, `wc -l` and scripts read a partial fleet as a complete one. | L9.5, L7.5 | ADR 0004 |
| S2 | **The process tree does not inherit the contract** | Budgets, deadlines, residency and signals are per process; children, group members and retries fall outside them. | L7.3, L2.3, L5.5, L1.3 | ADR 0004, ADR 0005, ADR 0008 |
| S3 | **Policy on the verb, not on the mount** | Residency, egress and retention as flags are forgotten; as mount options they bind every command. | L8.2, L7.6, L7.7 | ADR 0011, ADR 0010 |
| S4 | **Imperative verbs on reconciled objects** | Anything GitOps owns must be written as intent: image, route weight, scale, restart. G3 closed one case of a general rule. | L3.2, L9.3, L3.3 | ADR 0010 |
| S5 | **Hierarchical paths over flat names** | `cd`, `mkdir`, `rmdir` imply a tree; namespaces are a flat list with cascading deletion. | L6.1, L6.2, L6.6 | ADR 0011 |
| S6 | **The artifact store is data at rest** | Pipes, receipts and logs have a location, a retention and a leak surface; the design treated them as plumbing. | L8.1, L8.3, L8.6, L7.1, L4.1, L1.5 | ADR 0001, ADR 0006, ADR 0011 |
| S7 | **Self-reported numbers in the ledger** | Metered spend, receipt metrics and executable bands come from the thing being measured. | L7.2, L4.2, L2.2 | ADR 0009, ADR 0010 |
| S8 | **Multi-party actions in a single-actor shell** | Approvals, producer/consumer rotation and forced retirements need a second identity; the shell has one stdin. | L3.1, L9.1, L9.8, L6.7 | ADR 0005, identity design |
| S9 | **The created-elsewhere blind spot** | Executors, namespaces, consumers and inputs the planner did not create are seen by label, index or not at all. | L9.4, L2.4, L6.6, L8.3 | ADR 0007, ADR 0011 |
| S10 | **Impossible pause, unbounded freeze** | Pause-with-checkpoint and group freeze are the default lifecycle verbs; some workloads have no checkpoint and collectives time out. | L2.6, L5.2, L4.3 | ADR 0004, ADR 0008 |
| S11 | **Disconnected is a state, not a failure** | Device state, receipts, secrets and placement all assume the session can reach the cluster. At the edge the link is part of the system: a down device must queue, receipts must be spooled where the work ran, secrets must resolve locally, and compute must go to the data. | L10.1, L10.2, L10.6, L10.7, L10.5 | ADR 0003, ADR 0007, ADR 0009, ADR 0011 |
| S12 | **A log is append-only; reruns, replacements and deletes do not mean what they mean on a file** | A rerun of a write appends beside the old records, a replacement is impossible, a delete is immediate and total, and an unbounded read never ends. Every verb that assumes file semantics must say what it does on a log. | L11.2, L11.3, L12.2, L13.1, L13.5 | ADR 0012 |
| S13 | **Context that leaves the session is data at rest somewhere else** | Anything sent to a model, a provider or a remote service for the user's convenience (a comment to translate, a candidate list, a plan) is retained there under rules the session does not control; residency names a place, not a duration, and free text typed by a user is not covered by "secrets are references". | L14.2, L14.4, L14.5 | ADR 0014 decisions 3 and 5 |
| S14 | **The node is not in the tree** | Clusters are devices and pods are processes, so the layer where disks, GPUs and memory fail has no path, no health file, no verb and no column in `ps`; every hardware incident starts outside the shell, and the one verb that exists for a hung process (`kill -KILL`) cannot work there. | L16.1, L16.2, L16.7 | ADR 0011 decision 7, ADR 0008 |

## Amendments by ADR

The first table lists every finding that touches an ADR, whatever its disposition. The second lists the amendments actually written on 2026-11-09 and the findings each applies.

| ADR | Findings that touch it |
|---|---|
| 0001 pipe is a staged artifact | L1.1, L4.5, L8.1 |
| 0003 three backends, local first | L10.7, L10.9 |
| 0002 annotation classes | L1.2, L12.2 |
| 0004 completion contract | L1.3, L2.6, L4.4, L5.1, L5.5, L7.3, L7.4, L8.4, L8.7, L9.5 |
| 0005 session is server-side | L3.1, L6.7, L7.3, L9.5 |
| 0006 detach is not destroy | L1.5, L5.6, L6.2, L8.3, L8.6, L11.4, L13.1 |
| 0007 secrets are references | L3.3, L6.4, L7.1, L9.1, L9.2, L9.4, L9.7, L9.8, L10.6 |
| 0008 signal routing | L2.3, L5.2, L9.2, L9.3, L9.6 |
| 0009 stdout artifact, stderr log | L4.2, L7.1, L8.5, L9.7, L10.2 |
| 0010 platform defaults | L2.2, L3.2, L4.3, L6.2, L7.2, L7.6, L8.2, L8.7, L9.3, L10.9, L12.3, L13.3 |
| 0011 device tree | L2.1, L2.5, L3.3, L4.1, L5.3, L6.1, L7.7, L8.1, L8.2, L10.1, L10.5, L10.8 |
| 0012 logs and queues are devices | L10.3, L11.1, L11.2, L11.3, L11.5, L11.6, L12.1, L12.2, L12.4, L12.5, L12.6, L13.1, L13.2, L13.3, L13.4, L13.5, L13.6 |
| 0013 native verbs have one shape (draft) | closes as verb entries: L3.5, L4.4, L5.5, L8.7, L10.9, L11.4, L12.2, L13.5; references without deciding: L3.1, L7.3, L9.5; L14.3 |
| 0014 assisted completion (draft) | L14.1, L14.2, L14.3, L14.4, L14.5, L14.6 |
| 0015 tables and catalogs are devices (draft) | L15.1, L15.2, L15.3, L15.4, L15.5, L15.6, L15.7 |
| 0008 signal routing (node fault) and 0011 device tree (node tree) | L16.1, L16.3, L16.4, L16.5, L16.6, L16.7 |

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

## Writing a new story

Copy [TEMPLATE.md](TEMPLATE.md). Keep the persona's words in section 1 and Convoy's words everywhere else. Fill all sixteen aspects even when the answer is `–`, and run all twelve probes even when the answer is `ok`: the register's value is that absence of a finding means it was checked. Number findings `L<story>.<n>`, propose a disposition, name the document it touches, and add the row here. If a finding repeats a pattern above, cite the S-id rather than inventing a new principle.
