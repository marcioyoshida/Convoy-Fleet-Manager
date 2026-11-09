# US-11 · Reprocess thirty days of orders from a topic after a bug fix, without touching production consumers

| Field | Value |
|---|---|
| Primary persona | DE (Data Engineer) |
| Secondary personas | SEC, FIN, SRE |
| Workload class | bounded topic reads, batch, cross-region |
| Walkthrough | none; written against [ADR 0012](../adr/0012-logs-and-queues-are-devices.md) decisions 2, 3, 5 and 8 |
| Gaps already closed there | none |
| Milestones needed | M2, M3, M4, M5 |
| Findings in this document | L11.1 – L11.6 |

## 1. Story

**As a** data engineer, **I want** to re-run the enrichment over the last thirty days of the orders topic with the fixed code, joined against the customers topic, writing corrected records for the reporting pipeline, **so that** the fix lands as one auditable command and the live consumers of the original topic never notice.

**Context.** The enrichment had a currency bug for a month. The orders topic has 48 partitions and two terabytes in the window, in us-east-1; reporting runs in eu-west-1. The customers topic is compacted. Finance wants the corrected month held for seven years.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | the data engineer | a reader identity on `/queues/orders-us`, a writer identity on `/queues/reporting-eu` | reads must not commit production offsets (L11.6) |
| Approves | none; the write goes to a new topic version | | L11.2 |
| Pays | team data | compute in us-east-1, cross-region egress, MSK bytes out | three price shapes in one band |
| Audits | finance, for seven years | `replay --verify` against the held copy of the offset range | L11.4 |
| Operates the platform | platform team | MSK and Strimzi devices in the inventory, the EU artifact store | |

## 3. Preconditions

- **Capabilities**: `queue:kafka`, `indexed-job`, `artifact-store` in both regions.
- **Packages**: `enrich` and `fx-fix` as user tools with annotations.
- **Mounts**: `/queues/orders-us` on the MSK device (`ro`, `topics=orders.`); `/queues/reporting-eu` on the Strimzi device in prod-eu (`topics=reporting.`); `/artifacts` per region.
- **Annotations**: `enrich: pattern` with a side input; `fx-fix: stateless`.
- **Platform defaults** `explain` must print: how time cuts resolve to offsets, the order property of a topic read, coalescing of partitions, the hold copy and its price, and that no consumer group offsets are committed.

## 4. Main path

```bash
$ df -h /queues/orders-us
TOPIC            PARTS  BYTES    RETENTION  TIMESTAMPS
orders.placed    48     6.1 TiB  90d        CreateTime
customers        12     40 GiB   compacted  LogAppendTime

$ explain 'cat --from=@2026-10-01 --to=@2026-11-01 /queues/orders-us/orders.placed \
           | fx-fix | enrich --customers /queues/orders-us/customers \
           | tee /queues/reporting-eu/reporting.orders.v2 \
           | put /artifacts/' --retain=7y
stage 1  cat        bounded read   48 topic partitions, 2.0 TiB exact, coalesced to 96 plan partitions   indexed job @ prod-us
         time cut   CreateTime topic: offsets resolved per partition at plan time; late events before the cut are not included
         order      per-partition; no global order downstream
stage 2  fx-fix     stateless      96 partitions
stage 3  enrich     pattern        side input: customers snapshot 40 GiB, compacted → deterministic: false (salted id)
stage 4  tee        side effect    reporting.orders.v2 (new topic version), idempotent producer, txn id = artifact id
stage 5  put        copy           eu-west-1 store, 2.1 TiB egress us→eu $41
hold 7y: copies offset range 2.0 TiB + customers snapshot 40 GiB into cas-eu-west-1 ($47/mo)
consumer group: none (direct partition assignment, no offsets committed)
band: $96–$140   contract: floor 100 %  deadline none
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `cat --from --to` | bounded read, 48 topic partitions coalesced to 96 | Indexed Job in prod-us, direct partition assignment | artifacts in the US store |
| `enrich --customers` | pattern stage with a compacted side input | same Job, side input snapshot | salted ids |
| `tee reporting.orders.v2` | side-effect write | transactional producer per plan partition | the EU broker |
| `--retain=7y` | hold | copies into the EU store, held registry image | receipts |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ● | ids from offset ranges; compacted side input never cached; a rerun with the same fix is a full cache hit | second `explain` shows every stage but 3 as a hit |
| A2 | Command | ○ | pattern and stateless | |
| A3 | Session | ○ | two queue mounts in two regions | |
| A4 | Backend | ● | Indexed Job sized by bytes, not by topic partitions (L11.5) | 96 plan partitions from 48 topic partitions |
| A5 | Contract | ● | floor over plan partitions | one failed partition fails the run; the rest are hits on rerun |
| A6 | Device tree | ● | MSK and Strimzi devices, two regions | `mount` shows both with options |
| A7 | Signals | – | | |
| A8 | Secrets | ○ | reader and writer identities as references | |
| A9 | Cost | ● | compute, MSK bytes out, cross-region egress, hold storage | the band lists four lines |
| A10 | Identity | ○ | per-mount identities | |
| A11 | Defaults | ● | time-cut rule, order property, coalescing, hold copy | all in `explain` |
| A12 | Storage | ○ | the hold copy | priced monthly |
| A13 | Networking | ○ | us to eu egress | |
| A14 | Time | ○ | a time window on a CreateTime topic (L11.1) | |
| A15 | Audit | ● | receipt holds per-partition offsets; replay against the copy | `replay --verify` in year three |
| A16 | Durability | ○ | a rerun resumes from hits | |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | the new topic is created by `tee`; its `KafkaTopic` may be GitOps-managed | intent write (S4) | ok |
| P2 | POSIX cannot hold | a global `sort` or `uniq` after a 48-partition read | order is per partition | **L11.3** |
| P3 | Orphaned state | the old wrong records remain in `reporting.orders` | a log cannot be rewritten | **L11.2** |
| P4 | Identity leak | none | | ok |
| P5 | Invisible cost | the hold copy of two terabytes for seven years | priced in `explain` | L11.4 |
| P6 | Lying cache | `--from=@date` on a CreateTime topic | late events fall outside the cut | **L11.1** |
| P7 | Silent failure | reads that commit offsets of a production group | the read would move production consumers | **L11.6** |
| P8 | Partial fleet | one of the two regions unreachable | plan-time refusal | ok |
| P9 | Created elsewhere | none | | ok |
| P10 | Multi-party | none | | ok |
| P11 | Data at rest | the copy lands in the EU store | residency from the mount | ok |
| P12 | Inheritance | 48 partitions, or ten thousand | ADR 0012 says partitions are the topic's | **L11.5** |

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| a plan partition fails | Job index | 1 | partition, offsets | rerun; hits elsewhere |
| offsets before the cut already retained out | `--from` resolution | 1 | lost range | none; see US-10 L10.3 |
| writer identity lacks create on `reporting.` | plan-time ACL check | 3 | the rule | platform grants |
| customers topic changed between plan and run | compacted snapshot | 0 | snapshot offsets per partition | none; the side input is what it was |
| a production group's offsets moved | would be silent | | | L11.6 |

## 8. Acceptance criteria

- **AC-11.1** Given `--from=@t` on a CreateTime topic, then `explain` prints the per-partition offsets the cut resolved to and warns that late events are excluded, and the receipt stores those offsets.
- **AC-11.2** Given a bounded read, then no consumer group offsets change on the broker during or after the run.
- **AC-11.3** Given a 48-partition topic and two terabytes, then the plan has more partitions than the topic when the size warrants and each plan partition reads a contiguous offset range of one topic partition.
- **AC-11.4** Given `tee` to a topic that already holds output from an earlier run over the same window, then `explain` warns and names the earlier pid, and `--replace` is refused as impossible on a log.
- **AC-11.5** Given a downstream `uniq` without a `sort`, then `explain` warns that order is per partition.
- **AC-11.6** Given `--retain=7y`, then the receipt names the copied offset range and the compacted snapshot, and `replay --verify` two years later runs against the copies.

## 9. Persona lenses

- **DE**: wants a reprocess to be one command with a cache on rerun, and wants to be told that the old records stay.
- **SEC**: wants the held copy and the salted compacted snapshot to be the evidence.
- **FIN**: wants the four cost lines separate, because the hold is the one that recurs.
- **SRE**: wants certainty that a batch read cannot move production consumer offsets.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L11.1 | P6 | A time cut resolves to the first offset at or after the timestamp; on a CreateTime topic late events before the cut are excluded, and two runs may see different sets if the resolution is done at run time. | **rule**: `--from` and `--to` resolve to per-partition offsets at plan time; the receipt stores them; `explain` warns on CreateTime topics | ADR 0012 decision 2 |
| L11.2 | P3 | A rerun of a side-effect write appends; it cannot replace the records an earlier run produced. The idempotency key stops double-production of the same artifact, not the coexistence of old and new. | **amend** ADR 0012 decision 2: `explain` warns when the target topic already holds output for an overlapping input range and names the pid; versioned topic names are the recommended spelling; `--replace` is refused as impossible on a log | ADR 0012 |
| L11.3 | P2 | A topic read is ordered within a partition only; `uniq` or `join` downstream without a `sort` is wrong across partitions. | **rule**: queue reads carry `order: per-partition` as a stage property, like partitioned file reads; the compiler warns when an order-sensitive class follows without a global sort | ADR 0012, pipeline-compiler.md §4 |
| L11.4 | P5 | The hold copy of a large offset range is the recurring cost of the run and could exceed the run itself. | **rule**: `explain --retain` prices the copy per month in the band | ADR 0006 amendment 1, ADR 0012 decision 3 |
| L11.5 | P12 | ADR 0012 says plan partitions are the topic's partitions; a 12-partition topic with two terabytes would run twelve huge indexes, and a ten-thousand-partition topic ten thousand tiny ones. | **amend** ADR 0012 decision 2: topic partitions are the granularity, not the count; the planner splits large partitions by offset range and coalesces small ones by size, preserving per-partition order | ADR 0012 |
| L11.6 | P7 | A consumer that joins a named group commits offsets and moves that group; a batch read that borrowed a production group name would move production consumers. | **rule**: bounded reads use direct partition assignment and commit nothing; only units (follows) own a group, prefixed by the mount's `group=` option | ADR 0012 decisions 2 and 4 |
