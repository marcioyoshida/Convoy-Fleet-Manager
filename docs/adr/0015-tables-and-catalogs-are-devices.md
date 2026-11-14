# ADR 0015: Tables and catalogs are devices; a read pins a snapshot and a write is one commit

**Status:** Draft · 2026-11-09 · revised 2026-11-09 with user-story findings L15.1 and L15.5 (marked inline)
**Supersedes / relates to:** ADR 0001 (pipe is a staged artifact; amendment 2, store per topology); ADR 0002 (typed verbs are the only engine mapping; amendment 1, idempotency); ADR 0006 (amendments 1 and 2, holds); ADR 0010 (packages, intent writes); ADR 0011 (device classes, mount options, `caps` drivers); ADR 0012 (the three faces, partitions as granularity, misroute S12); ADR 0013 (verb shape, `explain`); docs/pipeline-compiler.md §8.3 and §8.4 (text tools on typed data, the structured path); docs/user-stories/README.md probes P6 (lying cache), P5 (invisible cost), P11 (data at rest)

## Context

The data engineer, data scientist and ML engineer personas spend most of their day on tables, not files or topics: lakehouse tables in Iceberg, Delta or Hudi format on an object store, behind a catalog, and warehouse tables in BigQuery, Snowflake or Redshift. Today the design reaches them only as Parquet files under an object-store mount (compiler §8.3), which loses the one thing a table format adds: a transaction log that says exactly which files make up the table at a given instant.

Facts that constrain the design:

- A catalog (Iceberg REST, Hive metastore, Glue, Unity, Nessie) is an API endpoint with its own identifier, opened under a credential. It lists namespaces and tables and points each table at its current metadata.
- A table's metadata names its snapshots; a snapshot names its manifests; manifests list data files with byte size, record count and column bounds. Planning a read therefore needs no listing of the object store at all: the input size, the file count and the partitioning are exact, and the cost of planning is a handful of small reads.
- Snapshots are immutable and addressable by id and by time. A commit is an atomic swap of the metadata pointer with optimistic concurrency. Tables can be read as of any retained snapshot (time travel). Snapshot expiration deletes the data files no retained snapshot references. Iceberg has tags and branches with their own retention; Delta has versions and a log retention period; Hudi has a timeline.
- Row-level deletes exist as separate delete files or deletion vectors (merge-on-read); a correct read applies them. Compaction rewrites data files without changing the table's content.
- Writers can be idempotent: Iceberg commits carry summary properties, Delta commits carry an application id and version, and a writer that finds its own key already committed does nothing.
- A warehouse table is read by a query the warehouse executes and bills, by bytes scanned (BigQuery, Athena, Trino on a metered cluster) or by compute time (Snowflake credits, Redshift). BigQuery estimates bytes scanned before running. There is no way to run a text tool inside a warehouse.
- Unity Catalog and Lake Formation carry grants (SELECT, MODIFY, CREATE on a schema or table) that POSIX modes cannot express, the same shape as Kafka ACLs in US-13.
- ADR 0002 forbids translating text tools into engine operators and permits exactly one mapping: typed verbs over typed data. ADR 0012 decided that a topic's partitions are the granularity of a read, not the count, and that a rerun of a write to a log appends.

## Decision

1. **Catalogs are devices under `/dev/table/`.** The `by-id` key is the catalog's own identifier: the REST catalog URI's registered id, the Glue catalog id, the Unity metastore id, the warehouse project or account. `by-provider`, `by-topology` and `by-label` are views; `self` is per session; `all` is the read-only union. A table is a path under the mount: `/tables/lake/sales/orders` is namespace `sales`, table `orders`. Mount options: `identity`, `ro`, `namespaces=<prefix>`, `residency` (ADR 0011 amendment 3), `at=<snapshot|timestamp>` to pin every read on the mount to one instant, `text=jsonl|csv` for the serialization text tools see (decision 2), `write=direct|branch` (decision 4). A catalog type is mountable only when a package provides its driver, reported by `caps` as `table:iceberg`, `table:delta`, `table:hudi`, `table:bigquery`, `table:snowflake` (ADR 0011 amendment 1).

   ```
   mount -o identity=role:lake-reader,namespaces=sales.,residency=eu,text=jsonl,ro \
         /dev/table/by-id/iceberg-rest-3f9a…-lake-eu /tables/lake
   ```

2. **A bounded read pins a snapshot.** Any stage that reads a table path resolves the table's current snapshot at plan time, or the one `at=` or `--at=` names, and the receipt records it. Partitions of the plan are groups of the snapshot's data files, split and coalesced by size as for topics (ADR 0012 decision 2): the table's files are the granularity, not the count. The artifact id is `sha256(catalog id, table id, snapshot id, projection, predicate, file group, serialization)`, so a rerun is a cache hit for as long as the snapshot exists and never after the table changed underneath. Delete files and deletion vectors are applied by the driver inside the read and `explain` prints `merge-on-read: <n> delete files`. Text tools (`grep`, `awk`, `sort`, `wc`) read the table as rows serialized per the mount's `text=` option, one record per line, and run as the real binaries over the file groups (ADR 0002); the raw Parquet bytes are still available under the object-store mount, where compiler §8.3 applies. Typed verbs (`select`, `filter`, `group`, `tjoin`) read the table's own schema as their type and push projection and predicate into the scan, so the file groups and the byte count `explain` prints shrink with the predicate.

3. **A follow is an incremental read, and only on an append-only table.** `tail -f` on a table follows new snapshots and emits the records each appended, resuming from the last snapshot id it saw, under the opt-in stream rules of ADR 0001. On a table whose history has overwrites, deletes or compaction the follow is refused with the reason, because the snapshots do not form a log; the alternative `explain` names is a changelog view where the format provides one, read as a bounded range.

4. **A write is one commit, keyed by the input, and it can replace.** `> table` and `tee table` stage data files per partition and make exactly one atomic commit when every partition has finished, carrying the input stage's artifact id as the commit's idempotency key (Iceberg summary property, Delta application id and version). A rerun with the same key is a no-op commit, which differs from a topic (ADR 0012, misroute S12): a table write does not append beside its earlier self. `--replace=<predicate>` is permitted and means an atomic overwrite of the rows the predicate selects, or of the whole table with `--replace=all`; the predicate is required, because partitions derived from the input would remove rows the input never contained (late records written to the table after the input's cut), so the implicit form is refused with the predicate spelling. `explain` prints, per partition, the rows removed and the rows written, and warns when removed rows fall outside the input's declared time range (L15.1). A write under `--on-deadline=cancel` or a budget stop leaves no partial commit. When the mount says `write=branch`, the commit lands on a branch named `convoy/<pid>` (Iceberg branches; a staged version on formats without branches) and `release <table> <snapshot>` fast-forwards the main branch, so write, audit and publish are three verbs rather than one; the receipt carries both snapshot ids. A catalog whose tables are managed by GitOps or Terraform receives `mkdir`, `chmod` and `rm` as intent (ADR 0010 amendment 1).

5. **Snapshots are where holds and caches meet.** A hold (`--retain`) on a read of a table is a tag on the snapshot with a retention at least as long as the hold, where the format has tags, so expiration cannot remove its files. A tag is a table property that any identity with MODIFY can drop, so where the object store has `caps: object-lock` the hold also locks the snapshot's data files for the hold's duration; without the capability `explain` prints `hold: tag only, removable by MODIFY` and a regulated profile refuses the hold. A tag that is removed is recreated by the driver and the remover is named in the receipt (L15.5). On formats without tags, or when the platform says so, the hold copies the file groups into the held store (ADR 0006 amendment 2). `explain --retain` prints which mechanism and its monthly price. Snapshot expiration, compaction and other maintenance are package units chosen and named by the user or the platform, never inserted by the planner; a maintenance unit that would expire a tagged snapshot is refused by the driver.

6. **Warehouses are devices whose reads are priced queries.** A warehouse table is read by a query the warehouse runs: for typed verbs the projection, predicate and aggregation push down as SQL (`compiles_to: sql` on the typed verbs when the input device is a warehouse, the one engine mapping ADR 0002 permits); for text tools the read is a priced export of the serialization, since no text tool can run inside the warehouse. `/sys/table/<id>/price` carries the shape (bytes scanned, credits per second, slot-hours) and `explain` prints the estimated bytes scanned from the warehouse's own estimate where one exists, as its own cost line, before anything runs. A warehouse mount without `ro` on a regulated profile requires `--allow-write` on the mount, because a warehouse write is billed and audited by the warehouse too.

7. **Verbs on the device.**

   | Verb | On a table device | Rule |
   |---|---|---|
   | `ls -l` | tables with rows, bytes, files, current snapshot, partition spec, format, last commit | from `/sys/table/<id>/`, refreshed on the inventory cadence; `ls --fresh` reads the catalog |
   | `stat` | snapshot history with ids, times, operations and parents; tags and branches | the table's metadata |
   | `mkdir` | create a namespace, or a table with `--schema=<file>` and `--partition=<spec>` | written as intent when the catalog is managed |
   | `rm` | drop a table | refused without `--with-data` (purge), lists bytes and the last snapshot, snapshots the metadata pointer first (ADR 0006) |
   | `chmod`, `getfacl`, `setfacl` | catalog grants | `r` is SELECT, `w` is MODIFY, `x` is CREATE under the namespace; the rest is `setfacl`; `ls -l` marks `+` when grants exceed the mode (US-13, L13.2) |
   | `df`, `du` | bytes by table, by partition, by snapshot retained; price shape | `/sys/table/<id>/` |
   | `cat`, `head`, `tail -n`, `< path` | bounded read at the pinned snapshot | decision 2 |
   | `tail -f` | incremental follow | decision 3, append-only tables only |
   | `>`, `tee` | one commit | decision 4 |
   | `release <table> <snapshot>` | publish a branch commit to main | decision 4, `write=branch` |
   | `cp` between a table and an object store or a topic | two stages and an egress line | ADR 0012 decision 5; a topic with a schema registry is read typed into a table |

8. **Cost has a third shape.** Lakehouse reads cost object-store requests per data file plus compute, and `explain` prices them from the file count the manifests give, with no listing. Warehouse reads cost what decision 6 says. Egress between the table's store and the stage's cluster is priced from topology (ADR 0011). The hold cost of a tag is zero storage beyond what the snapshot already retains; the hold cost of a copy is the file groups' size per month.

9. **The schema is the type.** A table's schema is read by the `typed` class directly, which settles the first half of ADR 0012's open item: a topic with a schema registry is read through the same driver interface as a table and can feed typed verbs or be written to a table with `tee`. The second half, which registries and which encodings, is a driver matter under `caps`, and ADR 0012 decision 10 (2026-11-10) states how a topic read through its schema is decoded, pinned, serialized for text tools and written.

## Declined

| Proposal | Why not | What replaces it |
|---|---|---|
| a `sql` verb as the way to read tables | the POSIX surface is paths and pipes; SQL engines are tools | typed verbs push down; `duckdb`, `trino` and `spark-sql` are toolbox executables with their own annotations |
| translating `awk` or `grep` into SQL or dataframe operators | ADR 0002's one rule | text tools read a serialization and run as binaries; typed verbs are the engine path |
| the warehouse as an execution backend | it runs queries, not stages; it cannot run the toolbox | a device whose reads are priced queries (decision 6) |
| a `/lake/<provider>/<region>/` tree as the truth | the layout ADR 0011 reviewed and changed | `/dev/table/by-id` with views |
| planner-inserted compaction or snapshot expiration | a hidden write to the user's table | package units, named by the user or the platform |
| reads at "latest" that may see a concurrent commit mid-read | the lying cache probe | every read pins a snapshot at plan time |
| treating a table's data files as the input (compiler §8.3 alone) | loses the transaction log, applies no deletes, lists the bucket at a price | the table device; the object-store mount remains for raw bytes |
| refusing `--replace` as on topics | a table commit is atomic and can overwrite | permitted, printed, keyed |

## Consequences

- Tables join topics as inputs whose size and partitioning are exact at plan time. For the lakehouse case, the plan also costs nothing to compute, because manifests replace listing.
- The lying-cache probe (P6) is answered structurally: the snapshot id is in the artifact id. The invisible-cost probe (P5) is answered for warehouses by printing the estimated bytes scanned as a line before the run.
- Misroute S12 gets its mirror image: on a log a rerun appends and a replace is impossible; on a table a rerun is a no-op and a replace is atomic. `explain` prints which kind of sink a path is, so a pipeline that ends in `tee` to either is unambiguous.
- Cost: four drivers in M5 (`table:iceberg`, `table:delta`, `table:hudi`, one warehouse to start), a third price shape, `stat` on a table, `--at`, `--replace`, `write=branch` with `release` on a table, and a serialization option per mount. The chart gains one infrastructure box.
- Open, to be settled when M5 is designed against this ADR: which serialization defaults are safe for text tools (nested types, timestamps); whether `tail -f` should follow a changelog view by default where one exists; how `release` on a table interacts with a GitOps-managed catalog; grants on warehouses that have no catalog of their own.
- [US-15](../user-stories/US-15-lakehouse-backfill.md) probes this record with a backfill from a topic into a partitioned Iceberg table under a seven-year hold, a branch write published by `release`, and a BigQuery mirror read under a bytes-scanned budget. It raised seven findings; L15.1 (`--replace` requires an explicit predicate and prints rows removed per partition) and L15.5 (a hold locks the snapshot's files where the store can, and a regulated profile refuses a tag-only hold) are applied inline in decisions 4 and 5 as of 2026-11-09; L15.2, L15.3, L15.4 and L15.7 are rules on `release` against a moving main, capacity billing, staged files and per-dataset residency; L15.6 accepts the lossy text serialization.

## Rules · 2026-11-09 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L15.2** (US-15) `release` on a table replays the branch commit onto the current main when the partitions it touches were not written on main since the branch point, and is refused with the conflicting partitions otherwise; the receipt names the snapshot it replayed onto.
- **L15.3** (US-15) `/sys/table/<id>/price` states the billing model; under capacity pricing the band is slot-time from the device's job history and `explain` prints `estimate: capacity, from history`; `--budget` then binds at run time from job statistics, and the receipt shows estimated and settled.
- **L15.4** (US-15) staged files are written under a per-pid staging prefix the platform names; the session deletes them on abort, cancel or deadline and `du` shows staged bytes per pid; a staging prefix with no live pid is reported by `df`, never silently kept.
- **L15.7** (US-15) for warehouses `/sys/table/<id>/` carries topology per dataset, `ls -l` shows the location, and residency is enforced per table path, not per mount; a dataset outside the mount's residency is listed as unreadable.
- **L27.5** (US-27) `cp` of a table path to another store is a typed copy that rewrites the table's metadata to the target's paths, or registers a new table from the copied files; a byte copy of a table is refused, naming the manifests that would still point at the source; `explain` of any table read prices the data files that lie outside the stage's store.
- **L33.6, L33.18** (US-33) `mv` across devices deletes only after the destination is verified; a stage-in reads a pinned table snapshot (full rules in [ADR 0029](0029-a-computers-file-systems-are-in-its-contract-staging-is-a-planned-copy.md)).
