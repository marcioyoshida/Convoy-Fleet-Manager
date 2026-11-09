# US-12 · Run a shell pipeline as a continuous consumer and know when the shell is the wrong tool

| Field | Value |
|---|---|
| Primary persona | DE (Data Engineer) |
| Secondary personas | SRE, PE, ML |
| Workload class | continuous unit, follow and write faces, engine as a package |
| Walkthrough | none; written against [ADR 0012](../adr/0012-logs-and-queues-are-devices.md) decisions 2, 4 and 7 |
| Gaps already closed there | none |
| Milestones needed | M2, M5, M9, M11 |
| Findings in this document | L12.1 – L12.6 |

## 1. Story

**As a** data engineer, **I want** a stateless enrichment to run continuously from one topic to another as a unit I can scale, reload and roll, and **I want** the planner to refuse a windowed aggregation on a stream and point me at the Flink package, **so that** the easy ninety percent stays a one-line pipeline and the hard ten percent is an explicit engine job with the same process verbs.

**Context.** Today both cases are Flink jobs, and the stateless ones are the majority of the on-call pages. The team wants the stateless ones to be a shell line it can read, and wants to stop pretending `uniq -c` over an infinite stream means anything.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | the data engineer | a read-write identity on `/queues/orders`, group prefix `enrich-` | the unit owns a consumer group |
| Approves | none | | |
| Pays | team data | pods through OpenCost; broker shared | |
| Audits | the on-call | unit receipts: rollouts, rebalances, lag | |
| Operates the platform | platform team | Strimzi, the Flink operator package, schema registry (L12.5) | |

## 3. Preconditions

- **Capabilities**: `queue:kafka`, `service-unit`, `operator:flink`, optionally `schema-registry`.
- **Packages**: `enrich` user tool (`stateless`); `flink` executable (`non-parallel`, `compiles_to: flinkdeployment`, `group: flink`, `checkpoint: savepoint`).
- **Mounts**: `/queues/orders` with `topics=orders.,group=enrich-`, optionally `schema=https://registry…` (L12.5).
- **Annotations**: `enrich: stateless`; `uniq`, `sort`, `join`: aggregate and fan-in classes, which are refused on a follow (L12.2).
- **Platform defaults** `explain` must print: exactly-once mode for follow-to-write units, the assignor and surge settings for rollouts, the group prefix, where lag comes from.

## 4. Main path

```bash
$ cd /queues/orders
$ run --unit enrich-orders 'tail -f orders.placed | enrich --geo /s3/ref/geo.parquet | tee --key=order_id orders.enriched'
unit enrich-orders  Deployment 1/1  consumer group enrich-orders (12 partitions, max scale 12)
                    mode: consume-transform-produce, exactly-once (offsets committed in the producer transaction)
                    lag source: broker group offsets via the queue device
$ scale enrich-orders 6 && wait --lag-below=1000 enrich-orders --timeout=10m
$ top -p enrich-orders
UNIT           MEMBERS  PARTS  LAG   RATE      COST/h
enrich-orders  6/6      12     412   8.1k/s    $0.44

$ run --unit top-skus 'tail -f orders.placed | cut -f3 | sort | uniq -c | sort -rn | head -20'
convoy: `sort`, `uniq -c` are aggregate and fan-in classes; on a follow they never terminate.
        Either a bounded read on a schedule (crontab: cat --from=@-1h … | sort | uniq -c), or a windowed engine:
        flink run --window=1h top-skus.jar (package flink, group flink, savepoint on TERM). Nothing was created.

$ batch --budget=$300/day flink run --parallelism=8 --window=1h top-skus.jar orders.placed sku.top
pid 8410  FlinkDeployment data/top-skus-8410  group flink  band (declared) $6–$9/h  savepoint on TERM

$ kill -HUP enrich-orders           # reload: rolling restart, cooperative assignor, maxSurge 0
$ kill -TERM 8410                    # stop with savepoint; receipt names the savepoint path
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `run --unit … tail -f … tee` | service unit, follow to write | Deployment of the toolbox, consumer group, transactional producer | offsets and records in the broker |
| `scale`, `wait --lag-below` | unit verbs | replicas; lag read from the device | |
| refused `top-skus` | plan-time refusal | nothing | |
| `batch flink run` | package executable, group | FlinkDeployment | operator; savepoint path in the receipt |
| `kill -HUP` | reload per declaration | rolling restart with `maxSurge: 0` | |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ● | a follow has no artifact; exactly-once needs a different key (L12.1) | no artifact ids appear in the unit receipt; records are not duplicated across a restart |
| A2 | Command | ● | classes decide what a stream may do (L12.2) | `explain` refuses aggregate and fan-in on a follow |
| A3 | Session | ○ | the unit outlives the session | |
| A4 | Backend | ● | a unit, not a backend; an engine as a package | `explain` names neither Indexed Job nor Argo |
| A5 | Contract | ○ | lag is a condition, budget per day on the engine | `wait --lag-below` is not an exit code |
| A6 | Device tree | ○ | one queue mount | |
| A7 | Signals | ● | TERM commits and stops; HUP is a rolling restart; TERM on Flink is a savepoint | `kill -TERM` on the unit leaves no uncommitted transaction |
| A8 | Secrets | ○ | identity reference | |
| A9 | Cost | ○ | pods through OpenCost; declared band for the engine (US-02 L2.2) | |
| A10 | Identity | ○ | group prefix from the mount | |
| A11 | Defaults | ● | exactly-once mode, assignor, surge, lag source | all in `explain` |
| A12 | Storage | – | | |
| A13 | Networking | ○ | broker reachable from the namespace | |
| A14 | Time | ○ | a window is an engine's concept, not the shell's | |
| A15 | Audit | ○ | rollouts and rebalances as receipt lines | |
| A16 | Durability | ● | a pod loss mid-batch | the partition is reassigned and the aborted transaction is invisible to readers |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | the unit's Deployment under GitOps | US-03 L3.2 | ok |
| P2 | POSIX cannot hold | `sort`, `uniq`, `join`, `wc` on an infinite input | the shell never terminates; nothing in the compiler refuses it | **L12.2** |
| P3 | Orphaned state | an aborted transaction on pod loss | the broker discards it | ok with L12.1 |
| P4 | Identity leak | none | | ok |
| P5 | Invisible cost | the engine's real parallelism | declared band, US-02 L2.2 | ok |
| P6 | Lying cache | none; follows are never cached | ADR 0012 d2 | ok |
| P7 | Silent failure | exactly-once keyed by the input artifact id, but a follow has none | ADR 0012 d2 has no key for this case | **L12.1** |
| P8 | Partial fleet | broker unreachable | the unit's probe fails; rollout stalls; receipt | ok |
| P9 | Created elsewhere | a consumer group with the unit's name already exists | the unit would join it and steal partitions | **L12.6** |
| P10 | Multi-party | none | | ok |
| P11 | Data at rest | Avro records are opaque bytes to `cut -f3` | no schema handling | **L12.5** |
| P12 | Inheritance | rolling restart causes a rebalance storm | assignor and surge are undefined | **L12.3** |

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| pod lost mid-transaction | broker aborts the transaction | unit continues | member lost, partitions reassigned | automatic |
| aggregate on a follow | plan time | 3 | the classes and the two alternatives | bounded cron or engine |
| group name taken by a non-Convoy consumer | plan time (L12.6) | 3 | members not labelled by Convoy | pick another unit name |
| lag never drops | `wait --timeout` | 2 | lag at timeout | `scale` up to the partition count |
| Flink budget hit | cost watcher | 2 | stopped with savepoint path | `kill -CONT 8410` resumes from it |
| schema changes under `cut -f3` | silent today | | | L12.5 |

## 8. Acceptance criteria

- **AC-12.1** Given a follow-to-write unit, when a member dies after producing but before committing, then readers of the output topic never see the aborted records and the input offsets are re-consumed exactly once.
- **AC-12.2** Given a pipeline with an aggregate or fan-in class on a follow, then `explain` refuses with the two alternatives and creates nothing.
- **AC-12.3** Given `kill -HUP` on a consumer unit, then the rollout proceeds one member at a time with no surge and partitions move cooperatively, and the receipt lists the rebalances.
- **AC-12.4** Given `wait --lag-below=N`, then lag is read from the broker's group offsets through the device and no metrics package is required.
- **AC-12.5** Given an existing consumer group with the unit's name whose members are not Convoy's, then the unit is refused at plan time.
- **AC-12.6** Given `kill -TERM` on a `flink` process, then the operator takes a savepoint, the receipt names its path, and `kill -CONT` resumes from it.

## 9. Persona lenses

- **DE**: wants the stateless case to be a readable line and the stateful case to be refused loudly, not to hang.
- **SRE**: wants exactly-once to be a mode printed by `explain`, and rollouts that do not storm the group.
- **PE**: owns the Flink package and the schema registry; wants the group prefix to be enforced by the mount.
- **ML**: wants the same unit shape for feature streams feeding online inference.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L12.1 | P7 | ADR 0012 keys the idempotent producer by the input artifact id. A follow produces no artifact, so a follow-to-write unit has no key and falls to at-least-once. | **amend** (applied: ADR 0012 d2 revised) ADR 0012 decision 2: for follows, the unit runs consume-transform-produce with the input offsets committed inside the producer transaction (Kafka's exactly-once), keyed by group, partition and batch; `explain` prints the mode; a sink device without transactions makes the unit at-least-once and says so | ADR 0012 |
| L12.2 | P2 | Aggregate, fan-in and global-order classes never terminate on an unbounded input; nothing in the compiler refuses them on a follow. | **rule** (applied: ADR 0002 rules; compiler): on a follow the planner refuses aggregate and fan-in classes and `order: global`, and prints the two alternatives: a bounded read on a schedule, or an engine package with a window; `limit` (`head`) stays allowed and closes the follow | ADR 0002, ADR 0012 decision 4, pipeline-compiler.md §8.5 |
| L12.3 | P12 | A rolling restart of a consumer unit rebalances the group on every pod change; with an eager assignor and a surge the group stalls repeatedly. | **rule** (applied: ADR 0010 rules; roadmap M11): consumer units default to `maxSurge: 0`, the cooperative-sticky assignor and a drain on TERM that commits before leaving; printed by `explain` | ADR 0010, ROADMAP M11 |
| L12.4 | P7 | Where lag comes from decides whether `wait --lag-below` needs the metrics package (US-03 L3.5). | **rule** (applied: ADR 0012 rules): lag is computed from the broker's group offsets through the queue device, under `/sys/queue/<id>/groups/`; no metrics capability required | ADR 0012 decision 7 |
| L12.5 | P11 | Avro and Protobuf topics are opaque bytes to text tools; `cut -f3` on them is silently wrong, and a schema change is invisible. | **open**: a mount option `schema=<registry>` makes reads emit decoded rows for the `typed` class and `cat` print one JSON object per record, and writes encode against the latest compatible schema; the M5 design decides whether text tools see JSON or are refused | ADR 0012 (open item), ROADMAP M5 |
| L12.6 | P9 | A consumer group created outside Convoy with the unit's name would be joined by the unit, which steals its partitions. | **rule** (applied: ADR 0012 rules): unit groups are prefixed by the mount's `group=` option; a group that exists with members not labelled by Convoy is a plan-time refusal; `ps` shows foreign groups with owner `?` | ADR 0012 decisions 1 and 7 |
