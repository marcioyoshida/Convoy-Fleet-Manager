# US-15 · Backfill a month into the lake table from the topic, publish it when audited, and query the warehouse mirror within a budget

| Field | Value |
|---|---|
| Primary persona | DE (Data Engineer) |
| Secondary personas | SEC, FIN, DS, SRE |
| Workload class | regulated batch into a lakehouse table; branch write and publish; priced warehouse read |
| Walkthrough | none; written against [ADR 0015](../adr/0015-tables-and-catalogs-are-devices.md) decisions 1 to 8, with the topic read of US-11 and the hold of US-08 |
| Gaps already closed there | none |
| Milestones needed | M3, M4, M5, M8 |
| Findings in this document | L15.1 – L15.7 |

## 1. Story

**As a** data engineer, **I want** to recompute October in the enriched orders table from the raw topic, land it where an auditor can check it before it becomes the table everyone reads, keep the evidence for seven years, and **I want** the analysts' warehouse copy to answer a question about it without a surprise bill, **so that** a bug fix becomes a corrected month rather than a second copy of the data or a deleted one.

**Context.** The enriched orders table is written continuously by the unit of US-12 and read by forty dashboards through a BigQuery mirror. A geo-enrichment bug affected October. Last time this happened the team overwrote the month with a Spark job, lost two days of late-arriving records nobody had counted, and the analysts' first query over the repaired month cost more than the repair.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | the data engineer | `role:lake-writer` on `/tables/lake`, `secret:orders-reader` on `/queues/orders` | the branch write needs MODIFY, the publish needs it too |
| Approves | the data steward, for `release` to main | a second identity runs `release` (US-03 L3.1) | |
| Pays | the orders domain | compute, topic bytes out, the hold copy, warehouse bytes scanned | four shapes (L15.3) |
| Audits | the regulator's reviewer, within seven years | `cat --at=<tag>` under their identity, `replay --verify` | the hold must survive table writers (L15.5) |
| Operates the platform | the platform team | the Iceberg REST catalog and the BigQuery project as devices in the inventory, object lock on the lake bucket | |

## 3. Preconditions

- **Capabilities**: `table:iceberg`, `table:bigquery`, `queue:kafka`, `object-lock` on the lake's object store, `artifact-store` in `eu-west-1`, `indexed-job`.
- **Packages**: `enrich` (US-12).
- **Mounts**: `/tables/lake` on the Iceberg catalog with `identity=role:lake-writer,namespaces=sales.,residency=eu,text=jsonl,write=branch`; `/tables/bq` on the BigQuery project with `identity=sa:bq-reader,ro,residency=eu`; `/queues/orders` as in US-11.
- **Annotations**: `enrich: stateless`; `select`, `filter`, `group: typed`.
- **Platform defaults** `explain` must print: the serialization for text tools, the branch naming and who may `release`, the hold mechanism per device (tag, lock, copy) and its price, the warehouse billing model the estimate assumes, the staging prefix for an in-flight write.

## 4. Main path

```bash
$ cd /clusters/prod-eu/orders
$ explain --retain=7y 'cat --from=@2026-10-01T00:00Z --to=@2026-11-01T00:00Z /queues/orders/orders.placed \
    | enrich --geo /s3/ref/geo.parquet \
    | tee --replace="day >= 2026-10-01 and day < 2026-11-01" /tables/lake/sales/orders_enriched'
plan 9c2e…  input: topic 48 partitions, 2.0 TiB → 16 file groups   backend: Indexed Job in prod-eu (placement: input local)
  stage 1  read     48 partitions, offsets pinned per partition (late events after the cut excluded)
  stage 2  enrich   stateless, 16 × 128 GiB
  stage 3  write    one commit on branch convoy/<pid>, overwrite 31 partitions (412 M rows removed, 412 M written, delta per partition below)
             key: input artifact id   serialization: n/a (typed sink)   staging: s3://lake-eu/_convoy/staging/<pid>/
hold 7y: topic range copied 2.0 TiB into cas-eu-west-1 ($47/mo); table snapshot tagged hold-<pid>, files object-locked 7y ($0 beyond retention)
cost band: $61–88  (compute $40–55, MSK bytes out $14, egress 0, hold $47/mo)
requires: queue:kafka ✓  table:iceberg ✓  object-lock ✓  indexed-job ✓

$ batch --retain=7y '…the same line…'
pid 7702

$ wait 7702 && stat /tables/lake/sales/orders_enriched
exit 0
snapshots:  8812  main            2026-11-09 06:10  append     (unit enrich-orders)
            8813  convoy/7702     2026-11-09 06:42  overwrite  31 partitions   tag hold-7702 (7y, locked)
            8814  main            2026-11-09 06:43  append     (unit enrich-orders)

$ cat --at=convoy/7702 /tables/lake/sales/orders_enriched | filter day = 2026-10-14 | group region | head
…                                                     # the auditor reads the branch, nothing on main changed

(steward)  $ release /tables/lake/sales/orders_enriched 8813
main: 8814 → 8815 (overwrite replayed onto main; partitions day=2026-10-* only; no conflict with 8814)
receipt 7702: branch 8813, published 8815, approver steward

$ cat --at=2026-11-01T00:00Z /tables/lake/sales/orders_enriched | wc -l       # time travel pins snapshot 8790
412 180 331

$ explain --budget=5 'filter day >= 2026-10-01 < /tables/bq/sales/orders_enriched | group region'
refused (exit 3): estimated 1.8 TiB scanned, $11.25 on-demand, exceeds --budget=5
  alternatives: narrow the predicate (day >= 2026-10-25 scans 290 GiB, $1.81); read the lake table instead (compute only)

$ batch --budget=5 'filter day >= 2026-10-25 < /tables/bq/sales/orders_enriched | group region'
pid 7704   bytes scanned 290 GiB   $1.81 estimated, $1.79 settled
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `cat --from --to` on the topic | bounded read, 48 partitions pinned | Indexed Job over 16 file groups | receipt, artifact store |
| `tee --replace=<predicate>` | one commit on a branch | staged Parquet files, one Iceberg commit on `convoy/7702` | catalog metadata, lake bucket |
| `--retain=7y` | hold | topic range copied; snapshot tag; object lock on data files | cas-eu-west-1, catalog, bucket |
| `stat`, `cat --at=` | metadata and time-travel reads | catalog metadata, manifests | |
| `release <table> <snapshot>` | publish | main branch advanced by the overwrite | catalog, receipt with both ids |
| `filter … < /tables/bq/…` | priced warehouse read | a BigQuery job with the pushed-down query | BigQuery job history, ledger |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ● | snapshot ids in the artifact id; a rerun of the read is a cache hit; a rerun of the write is a no-op commit | second `explain` shows stages 1 and 2 cached and stage 3 `no-op: key present` |
| A2 | Command | ○ | `enrich` stateless; typed verbs on both table devices | `type filter` shows pushdown to SQL on a warehouse |
| A3 | Session | ○ | two table mounts with different write rights | |
| A4 | Backend | ○ | Indexed Job placed where the topic is | `explain` prints placement |
| A5 | Contract | ● | a budget refused at plan time from the warehouse estimate; a deadline stop that leaves no partial commit | AC-15.5, AC-15.4 |
| A6 | Device tree | ● | catalog and warehouse as devices; residency per dataset (L15.7) | `ls -l /tables/bq` shows location per dataset |
| A7 | Signals | – | | |
| A8 | Secrets | ○ | roles and service accounts as references | |
| A9 | Cost | ● | compute, topic bytes out, hold, bytes scanned; on-demand versus capacity billing (L15.3) | the band has four lines; the settled figure appears |
| A10 | Identity | ○ | writer, steward, auditor | `release` by a second identity |
| A11 | Defaults | ● | serialization, branch policy, hold mechanism, billing model, staging prefix | all in `explain` |
| A12 | Storage | ● | staged files of an aborted write (L15.4); held files locked | `du` shows staging; a lock survives a table writer |
| A13 | Networking | – | | |
| A14 | Time | ○ | time cuts on the topic (L11.1); time travel on the table | `--at=<time>` names the snapshot it pinned |
| A15 | Audit | ● | branch read before publish; receipt with both snapshot ids; `replay --verify` from the hold | AC-15.3, AC-15.6 |
| A16 | Durability | ○ | the branch survives the session; the hold survives the table's writers (L15.5) | |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | `release` fast-forwards main while a unit keeps appending to it | ADR 0015 d4 says fast-forward; main has moved | **L15.2** |
| P2 | POSIX cannot hold | `cut`, `sort -n` on JSON lines of decimals and timestamps | the serialization is lossy for types | **L15.6** |
| P3 | Orphaned state | an aborted write's staged files | "no partial commit" says nothing about the files | **L15.4** |
| P4 | Identity leak | the auditor reads a branch with `role:lake-writer`'s hold on it | reads under their own identity; grants are catalog grants | ok |
| P5 | Invisible cost | the warehouse estimate under capacity pricing is zero dollars | the estimate assumes on-demand | **L15.3** |
| P6 | Lying cache | the table changed between plan and run | the snapshot id is in the artifact id (d2) | ok |
| P7 | Silent failure | a replace that deletes rows the input never contained | `--replace` overwrites whatever the predicate selects | **L15.1** |
| P8 | Partial fleet | the catalog unreachable | `disconnected` state, `stale=cache` for `ls`, writes queued or failed per mount (ADR 0011 a4) | ok |
| P9 | Created elsewhere | the unit's appends, snapshots made by Spark jobs | `stat` shows every snapshot with its writer where the summary names one, else `?` | ok |
| P10 | Multi-party | the steward publishes what the engineer wrote | `release` by a second identity, both in the receipt | ok once L3.1 exists |
| P11 | Data at rest | a hold tag is a table property a writer with MODIFY can drop | ADR 0015 d5 says tag or copy | **L15.5** |
| P12 | Inheritance | residency of a BigQuery dataset in the US under a mount that says `eu` | topology is per device in ADR 0011; a warehouse's is per dataset | **L15.7** |

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| replace would remove rows outside the input's time range | plan time (L15.1) | 3 | partitions and rows that would be removed, the implicit form refused | narrow or widen the predicate |
| main advanced between the branch commit and `release` | `release` | 0 or 3 | replayed onto main, or the conflicting partitions | re-run the month on a new branch |
| deadline reached during the write | contract | 2 | no commit; staged bytes deleted | `batch` again, stages 1 and 2 cached |
| warehouse estimate exceeds the budget | plan time | 3 | estimate, billing model, alternatives | narrow the predicate or read the lake |
| warehouse under capacity pricing | plan time (L15.3) | plan runs | `estimate: capacity, from history`; budget binds from job statistics | none |
| hold tag removed by a table writer | lock (L15.5) | n/a | files remain locked; the tag is recreated and the writer named | none |
| catalog unreachable | device state | queued or 3 per mount | `disconnected since` | wait |

## 8. Acceptance criteria

- **AC-15.1** Given a read of a table, then the receipt and the artifact id contain the snapshot id pinned at plan time, and a rerun after the table has changed is still a cache hit for that snapshot.
- **AC-15.2** Given `tee --replace=<predicate>` to a table, then exactly one commit exists after the run, it carries the input artifact id as its key, a rerun with the same key is a no-op commit, and `explain` printed rows removed and written per partition before the run.
- **AC-15.3** Given `write=branch` on the mount, then the commit is on a branch the auditor can read with `--at`, main is unchanged until a second identity runs `release`, and the receipt carries the branch snapshot, the published snapshot and both identities.
- **AC-15.4** Given a deadline reached during the write, then no commit exists, the staged files are gone and `du` of the staging prefix is zero.
- **AC-15.5** Given a warehouse read whose estimate exceeds `--budget`, then it is refused at plan time with the estimate, the billing model and two alternatives, and nothing is billed.
- **AC-15.6** Given `--retain=7y`, then the snapshot is tagged for seven years, its data files are object-locked where the capability exists, `explain` printed which mechanism and its price, and `replay --verify` two years later runs from the held closure.
- **AC-15.7** Given a BigQuery dataset in a location outside the mount's `residency`, then the mount lists it as unreadable with the location, and a read of it is refused at plan time.
- **AC-15.8** Given `tail -f` on a table whose history has an overwrite, then it is refused with the changelog alternative.

## 9. Persona lenses

- **DE**: wants the month repaired in place with the rows it removed counted, and the rerun to cost nothing; refuses if a replace can silently delete late records.
- **SEC**: wants the branch readable before it is published and the hold to survive anyone with write access to the table.
- **FIN**: wants the warehouse estimate as a line before the run and the settled figure after, and the hold's recurring cost separate from the run's.
- **DS**: wants `filter` and `group` to behave identically on the lake table and the warehouse mirror, with the price the only difference.
- **SRE**: wants an aborted write to leave nothing to clean up and a publish that cannot race the streaming unit.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L15.1 | P7 | `--replace` overwrites the partitions the predicate selects, which removes rows the input never contained: records for October that arrived after the topic read's offset cut (L11.1) were in the table and are gone after the backfill. ADR 0015 decision 4 prints the partitions, not the rows that leave. | **amend** ADR 0015 decision 4: `--replace` requires an explicit predicate; `explain` prints rows removed and rows written per partition and warns when the removed rows fall outside the input's declared time range; the implicit form (partitions derived from the input) is refused with the predicate spelling | ADR 0015 |
| L15.2 | P1 | `release` is specified as a fast-forward of main, but a continuous unit appends to main between the branch commit and the publish, so main is never where the branch left it. | **rule**: `release` on a table replays the branch commit onto the current main when the partitions it touches were not written on main since the branch point, and is refused with the conflicting partitions otherwise; the receipt names the snapshot it replayed onto | ADR 0015 decision 4 |
| L15.3 | P5 | The bytes-scanned estimate assumes on-demand billing. Under capacity pricing (slots, editions) the estimate prices to zero and the real cost is compute time the warehouse reports only after the job. | **rule**: `/sys/table/<id>/price` states the billing model; under capacity pricing the band is slot-time from the device's job history and `explain` prints `estimate: capacity, from history`; `--budget` then binds at run time from job statistics, and the receipt shows estimated and settled | ADR 0015 decisions 6 and 8, ADR 0010 |
| L15.4 | P3 | "A write leaves no partial commit" is true of the catalog and false of the object store: the staged data files of an aborted write are orphans until a maintenance procedure removes them. | **rule**: staged files are written under a per-pid staging prefix the platform names; the session deletes them on abort, cancel or deadline and `du` shows staged bytes per pid; a staging prefix with no live pid is reported by `df`, never silently kept | ADR 0015 decisions 4 and 5 |
| L15.5 | P11 | A hold tag is a table property; anyone with MODIFY on the table can drop it or shorten its retention, and the snapshot's files then expire. | **amend** ADR 0015 decision 5: where the object store has `caps: object-lock`, a hold tags the snapshot and locks its data files for the hold's duration; without the capability `explain` prints `hold: tag only, removable by MODIFY` and a regulated profile refuses the hold; a removed tag is recreated by the driver and the remover named in the receipt | ADR 0015, ADR 0006 amendment 2 |
| L15.6 | P2 | The text serialization is lossy for types: decimals, timestamps and nested fields become strings, so `sort -n`, `cut -f` and `uniq` on them do not mean what they mean on a typed read. | **accepted**: `explain` prints `serialization: jsonl, types flattened`; typed verbs are the alternative it names; the open item in ADR 0015 on safe defaults stands | ADR 0015 decision 2 |
| L15.7 | P12 | A warehouse project is one device but its datasets have locations; a mount with `residency=eu` on a project with a US dataset would read it unless residency is checked below the device. | **rule**: for warehouses `/sys/table/<id>/` carries topology per dataset, `ls -l` shows the location, and residency is enforced per table path, not per mount; a dataset outside the mount's residency is listed as unreadable | ADR 0015 decision 1, ADR 0011 amendment 3 |
