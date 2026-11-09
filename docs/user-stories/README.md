# User stories as design probes

The nine workloads in [walkthroughs.md](../walkthroughs.md) showed that the design carries them. This directory asks a harder question of the same nine, plus a tenth written against [ADR 0012](../adr/0012-logs-and-queues-are-devices.md) to reach the edge: for each persona who would run them, does every aspect of the abstraction hold, and where does a verb route to the wrong place? Each story follows [TEMPLATE.md](TEMPLATE.md), which fixes sixteen aspects to cover and twelve loophole probes to run, so that a reader can tell "checked and fine" from "never checked". Findings are collected below with a proposed disposition. **Nothing here has been applied to an ADR yet**; the register is the input to that decision.

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

| Aspect | 01 | 02 | 03 | 04 | 05 | 06 | 07 | 08 | 09 | 10 | Stress |
|---|---|---|---|---|---|---|---|---|---|---|---|
| A1 Pipe, artifact, cache | ● | ○ | ○ | ● | ○ | – | ● | ● | ○ | ● | 5 |
| A2 Command, annotation | ● | ● | ○ | ● | ● | ○ | ● | ○ | ● | ○ | 6 |
| A3 Session, mount table, attach | ○ | ○ | ● | ○ | ● | ● | ○ | ○ | ● | ● | 5 |
| A4 Backend selection | ● | ● | ○ | ● | ● | – | ○ | ○ | – | ● | 5 |
| A5 Completion contract | ● | ● | ● | ● | ● | – | ● | ● | ● | ● | 9 |
| A6 Device tree, mounts, `/sys` | ○ | ● | ● | ○ | ● | ● | ○ | ● | ● | ● | 7 |
| A7 Signals, groups | ○ | ● | ○ | ○ | ● | ● | ○ | – | ● | ○ | 4 |
| A8 Secrets | – | – | ○ | – | – | ● | ● | – | ● | ● | 4 |
| A9 Cost | ● | ● | – | ● | ● | ○ | ● | ○ | – | ● | 6 |
| A10 Identity, approvals | ○ | ○ | ● | ○ | ○ | ● | ○ | ● | ● | ○ | 4 |
| A11 Defaults and `caps` | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | 10 |
| A12 Storage | – | ● | – | ○ | ● | ● | – | ○ | – | ○ | 3 |
| A13 Networking | – | ○ | ● | ● | ○ | ○ | ● | – | – | ● | 4 |
| A14 Time, queueing | ● | ○ | ○ | ○ | ● | – | ○ | ● | ○ | ● | 4 |
| A15 Audit, receipts, retention | ● | ○ | ● | ● | ○ | ○ | ● | ● | ● | ● | 7 |
| A16 Durability | ○ | ○ | ● | ○ | ● | ● | ○ | ● | ● | ● | 6 |

**What no story reaches.** The session attach protocol itself (reconnect mid-command, two terminals on one session), `expose --public` and anything else in the undesigned networking abstraction, `pkg install` with a failing `requires`, `nice`/`renice` under contention, debugging a failing partition (`strace`, ephemeral containers), and locks. US-10 closed the edge and disconnected-operation row. Two candidate stories would close most of the rest: a platform engineer installing a package that a cluster cannot satisfy, and an engineer debugging one failed partition of a thousand.

## Findings register

Sixty-seven findings. Disposition counts: **19 amend** (an ADR amendment is proposed), **36 rule** (a sentence in an existing document), **4 open** (design that does not exist yet), **7 accepted** (a surfaced limitation), 1 cross-reference. Full text is in each story's section 10.

| Id | Short name | Probe | Disposition | Touches |
|---|---|---|---|---|
| L1.1 | env subset in the artifact id is undefined | P6 | rule | ADR 0001, compiler §3 |
| L1.2 | side-effecting stage reruns are not idempotent | P7 | amend | ADR 0002 |
| L1.3 | user floor contradicts a downstream fan-in | P12 | rule | ADR 0004 |
| L1.4 | process substitution is a bash dialect | P2 | accepted | command map |
| L1.5 | partial artifacts look complete, have no class | P3, P5 | rule | ADR 0006 a1 |
| L1.6 | a skipped CronJob is silent | P7 | rule | command map, M4 |
| L2.1 | `/dev/fs` devices need a fileserver driver | P7 | amend | ADR 0011 d9 |
| L2.2 | executables control their own footprint; band is declared | P5 | rule | ADR 0010 |
| L2.3 | signals to one group member are undefined | P12 | amend | ADR 0008 a1 |
| L2.4 | executors are created by the operator, seen by label only | P9 | accepted | annotations |
| L2.5 | copy into a replicated filesystem needs capacity × replication | P3 | rule | ADR 0011 d7 |
| L2.6 | pause-with-checkpoint is impossible without a checkpoint | P2 | rule | ADR 0004 |
| L3.1 | approval through the author's own stdin | P10 | **open** | ADR 0005, identity |
| L3.2 | canary weight patched on a GitOps-managed route | P1 | amend | ADR 0010 |
| L3.3 | the GitOps repository is not a device | P4, P11 | amend | ADR 0011 |
| L3.4 | no transaction across clusters | P3, P8 | accepted | M11 |
| L3.5 | metric conditions with no metrics package | P7 | rule | M11 |
| L4.1 | `cp` out of `/artifacts` drops the content id | P11 | rule | ADR 0011 d9, M5 |
| L4.2 | no channel for a command to write receipt fields | P7 | rule | ADR 0009, M1 |
| L4.3 | idle GPU units have no policy | P5 | rule | ADR 0010, M11 |
| L4.4 | fail-fast cancels independent siblings | P2 | rule | ADR 0004 |
| L4.5 | content id of `/s3` objects undefined | P6 | rule | ADR 0001, M5 |
| L4.6 | `expose` depends on undesigned networking | P9 | **open** | stories.md |
| L5.1 | zonal checkpoint disk unreachable after cross-zone restart | P3 | rule | ADR 0004 a1 |
| L5.2 | group freeze has no bound; collectives time out | P2 | amend | ADR 0008 a1 |
| L5.3 | preemption loss is billed but not in the band | P5 | rule | ADR 0011 d7 |
| L5.4 | rank-0 DNS at start | P2 | accepted | walkthroughs |
| L5.5 | what the deadline counts from | P12 | rule | ADR 0004 |
| L5.6 | Jobs have no per-index volume claims | P7 | rule | M3, ADR 0006 |
| L6.1 | hierarchical paths over flat namespaces | P2 | amend | ADR 0011 |
| L6.2 | namespace deletion cascades to volumes | P2, P3 | rule | command map, ADR 0006 |
| L6.3 | a policy as a `fuser` holder | P7 | accepted | command map |
| L6.4 | backups copy Secrets | P4 | rule | ADR 0007, M5 |
| L6.5 | tenancy depends on the identity design | P9 | **open** | stories.md |
| L6.6 | namespaces created outside Convoy have no owner | P9 | rule | command map |
| L6.7 | prompts in non-interactive sessions | P10 | rule | ADR 0005 |
| L7.1 | the artifact store is a secret sink | P4, P11 | amend | ADR 0007, ADR 0009 |
| L7.2 | metered spend is self-reported | P5 | rule | M2, ADR 0010 |
| L7.3 | budget and deadline do not propagate to children | P12 | amend | ADR 0004, ADR 0005 |
| L7.4 | external transient failures are unclassified | P2 | rule | ADR 0004 |
| L7.5 | degraded flag lost in a pipeline | P8 | see L9.5 | ADR 0004 |
| L7.6 | hostname egress needs FQDN-aware policy | P7 | rule | ADR 0010, M7 |
| L7.7 | queue items are data at rest | P11 | rule | ADR 0011 |
| L8.1 | the artifact store was never placed; residency of pipes | P11 | amend | ADR 0001, ADR 0011 |
| L8.2 | residency as a flag is bypassable | P12, P4 | amend | ADR 0011, ADR 0010 |
| L8.3 | a hold must cover external inputs | P9 | amend | ADR 0006 a1 |
| L8.4 | deadline time zone | P2 | rule | ADR 0004 |
| L8.5 | who replays, under what rights | P4 | rule | ADR 0009 |
| L8.6 | held images in an external registry | P7 | rule | ADR 0006 a1, M9 |
| L8.7 | `page:` with no alerting package | P7 | rule | ADR 0004, ADR 0010 |
| L9.1 | a secret has a producer; rotation is two-sided | P10 | amend | ADR 0007 a1 |
| L9.2 | dual validity versus cutover | P2 | rule | ADR 0007, ADR 0008 a1 |
| L9.3 | `reload: restart` fights GitOps | P1 | amend | ADR 0010 |
| L9.4 | the reference index cannot see every consumer | P9 | rule | ADR 0007 a1 |
| L9.5 | degraded has no POSIX channel | P8 | amend | ADR 0004, ADR 0005 |
| L9.6 | HUP swallowed by a wrapper entrypoint | P7 | rule | ADR 0008 a2 |
| L9.7 | the leaked value stays in logs and artifacts | P4 | **open** | ADR 0007, ADR 0009 |
| L9.8 | forcing a deferred retirement is two-person | P3, P10 | rule | ADR 0007 |
| L10.1 | a disconnected device is not a removed device | P8 | amend | ADR 0011 d8 |
| L10.2 | receipts lost when an outage outlasts the pod TTL | P7 | amend | ADR 0009 |
| L10.3 | reads before the earliest offset must fail, not shrink | P2 | rule | ADR 0012 |
| L10.4 | deadline expires while queued for a down site | P12 | accepted | ADR 0004 via L5.5 |
| L10.5 | links have a price per gigabyte and a state | P5 | rule | ADR 0011 d7 |
| L10.6 | secret references need a site-local source | P4, P8 | amend | ADR 0007 |
| L10.7 | stages must run where their input device is | P11, P5 | amend | ADR 0003 |
| L10.8 | a replaced broker is a new by-id | P9 | accepted | ADR 0011 d2 |
| L10.9 | `caps` is per device, not per fleet | P7 | rule | ADR 0003, ADR 0010 |

## Structural misroutes

Reading the register across stories, the findings cluster into eleven patterns. These are the places where the abstraction routes a verb to the wrong layer, not bugs in one story. Each should be settled once, as a principle, before the stories that depend on it are built.

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

## Proposed amendments by ADR

| ADR | Findings that touch it |
|---|---|
| 0001 pipe is a staged artifact | L1.1, L4.5, L8.1 |
| 0003 three backends, local first | L10.7, L10.9 |
| 0002 annotation classes | L1.2 |
| 0004 completion contract | L1.3, L2.6, L4.4, L5.1, L5.5, L7.3, L7.4, L8.4, L8.7, L9.5 |
| 0005 session is server-side | L3.1, L6.7, L7.3, L9.5 |
| 0006 detach is not destroy | L1.5, L5.6, L6.2, L8.3, L8.6 |
| 0007 secrets are references | L3.3, L6.4, L7.1, L9.1, L9.2, L9.4, L9.7, L9.8, L10.6 |
| 0008 signal routing | L2.3, L5.2, L9.2, L9.3, L9.6 |
| 0009 stdout artifact, stderr log | L4.2, L7.1, L8.5, L9.7, L10.2 |
| 0010 platform defaults | L2.2, L3.2, L4.3, L6.2, L7.2, L7.6, L8.2, L8.7, L9.3, L10.9 |
| 0011 device tree | L2.1, L2.5, L3.3, L4.1, L5.3, L6.1, L7.7, L8.1, L8.2, L10.1, L10.5, L10.8 |
| 0012 logs and queues are devices | L10.3 |

## Writing a new story

Copy [TEMPLATE.md](TEMPLATE.md). Keep the persona's words in section 1 and Convoy's words everywhere else. Fill all sixteen aspects even when the answer is `–`, and run all twelve probes even when the answer is `ok`: the register's value is that absence of a finding means it was checked. Number findings `L<story>.<n>`, propose a disposition, name the document it touches, and add the row here. If a finding repeats a pattern above, cite the S-id rather than inventing a new principle.
