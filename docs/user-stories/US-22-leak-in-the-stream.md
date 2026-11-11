# US-22 · Find every copy of a password a producer wrote into an event stream, from the topic to the lake, while the stream's schema keeps changing

| Field | Value |
|---|---|
| Primary persona | SEC |
| Secondary personas | DE, DBA, PE, FIN |
| Workload class | incident response over typed data: an Avro topic read through its schema, a schema change mid-read, a copy into a lake table under a hold, a leaked secret scrubbed across artifacts, logs, a topic and a table |
| Walkthrough | written against [ADR 0012](../adr/0012-logs-and-queues-are-devices.md) decision 10 and [ADR 0007](../adr/0007-secrets-are-references.md) amendment 5, with [ADR 0015](../adr/0015-tables-and-catalogs-are-devices.md) (row-level writes, snapshot holds, rule L15.5), [ADR 0006](../adr/0006-detach-is-not-destroy.md) (holds), [ADR 0017](../adr/0017-approvals-are-objects-on-the-process.md) (approval for `--purge`) and [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md) (ownership by group); it reuses the topic of US-11 and the lake table of US-15 |
| Gaps already closed there | none |
| Milestones needed | M1, M4, M5, M7 |
| Findings in this document | L22.1 – L22.9 |

## 1. Story

**As a** security engineer on call, **I want** to find and contain every readable copy of a database password that a producer wrote into an event stream, wherever the stream's data went next, and to be told plainly which copies I could not reach, **so that** the incident report lists exposure by fact and not by hope, and the data engineers can keep reading the stream while I do it.

**Context.** The `ledger` service in `payments` produces Avro records to `payments.events`. On Tuesday it registered schema version 5, which added a `debug_ctx` string field; a bug put the `db-ledger` password (version 7, owned by `ledger-dba`) into it for six hours. A data engineer's nightly job pinned version 4 and never saw the field; a backfill pinned version 5 and copied 30 days into the lake table `payments.events` under a seven-year hold (US-15). A Flink job installed by a package and a consumer group nobody recognises also read the topic. Security learns of it on Wednesday morning from a dashboard screenshot.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | `sec-ola` | user `sec-ola`, group `security`, member of `ledger-dba` for this incident | `secret rotate --leaked`, `secret scrub` |
| Reads the stream | `dana2` | group `payments-de` | jobs pinned to v4 and v5 |
| Owns the secret | `ledger-dba` | group | rotation through the producer (ADR 0007 amendment 3) |
| Owns the topic and table | `payments` | group | approves a purge on its objects |
| Approves | `payments` lead | member of `payments` | the `--purge` gate |
| Pays | `ledger-dba` | the scrub's scan bytes and compute | |
| Operates the platform | platform team | registry driver `schema:confluent`, `caps: secret-scrub`, Loki with `log-delete: query`, CloudWatch with `log-delete: stream`, a SIEM forward, Iceberg with object lock | |

## 3. Preconditions

- **Capabilities**: `queue:kafka` with `schema:confluent (avro, protobuf, json)`; `table:iceberg` with `object-lock`; `secret-scrub`; log stores `loki` (`log-delete: query`) and `cloudwatch` (`log-delete: stream`); `log-forwards: siem-eu`.
- **Mounts**: `/queues/payments` with `schema=registry,text=csv`; `/tables/lake` (US-15).
- **Annotations**: `xxd`, `od`, `cksum` carry `input: bytes`.
- **Platform defaults** `explain` must print: the reader schema and its column list, the serialization of logical types, the scrub scope by source and its price, each store's `log-delete` and each device's scrub capability.

## 4. Main path

```bash
# Tuesday: two reads of the same topic, two pinned reader schemas
[dana2] $ explain cut -d, -f6,8 < /queues/payments/payments.events --from=-24h
reader schema: payments.events-value v4 (id 211), pinned
columns: 1 _partition, 2 _offset, 3 _timestamp, 4 _key, 5 _schema, 6 account_id, 7 amount (decimal as string), 8 currency   (L22.7)
[dana2] $ cut -d, -f6,8 < /queues/payments/payments.events --from=-24h > /artifacts/daily-ccy
receipt 8801: 4 112 330 records; 912 004 with writer schema v5 (id 233), fields dropped: debug_ctx
[dana2] $ cat /queues/payments/payments.events --from=-30d --schema-version=5 | tee /tables/lake/payments/events
text=csv refused for v5 (nested field debug_ctx.headers); using the typed path: columns map by name into the table
commit 7710 (snapshot 88213), hold: tag + object lock until 2033-11-10

# Wednesday: the leak is known
[sec-ola] $ secret rotate db-ledger --leaked
producer: cnpg user reconciliation ok; version 8 written; v7 retire phase 1 (disabled) started; incident inc-0412
scrub of db-ledger v7 planned as pid 9500:
  scope from the reference index: ledger (pid 8122) mounted v7: 2 artifacts, Loki 6 h, CloudWatch 6 h
  scope from writes of those processes: payments.events partitions 0-11, offsets 18.2M-18.9M (6 h)   (L22.1)
  scope forward from reads of those ranges: pid 8803 (backfill) -> table payments.events snapshot 88213; pid 8801 (v4 read)
  scan: 41 GB topic range (string fields only, schema known), 3.1 TB table files (string columns only), 220 MB logs   (L22.8)
  price: $96-$130 (scan) within --budget=$200
$ wait 9500
exit 0 (degraded)
  artifacts: 1 match in pid 8122 stdout -> quarantined; pid 8801 artifact: no match (v4 dropped debug_ctx)
  Loki: 3 lines deleted by query;  CloudWatch: stream ledger/2026-11-09 readers narrowed (log-delete: stream)
  siem-eu: outside reach (forward)
  topic payments.events: 1 412 matches in 12 partitions; scrub: none on queue devices; removed by retention on 2026-11-16   (L22.2)
  consumers past matched offsets: flink-enrich (package flink, owner payments), etl-x (owner ?)  outside reach, groups paged   (L22.4)
  table payments.events: 1 412 rows; held snapshot 88213 (tag + object lock until 2033-11-10): readable by time travel   (L22.9)
  match locations visible to security and to each object's owning group only                                     (L22.6)

# Purging the table rows needs the table's owner
[sec-ola] $ secret scrub db-ledger v7 --purge --only=/tables/lake/payments/events
gate: purge on objects owned by payments needs an approver from payments (policy secret-purge)               (L22.5)
[payments lead] $ approve 9512
row-level delete commit 7742 (snapshot 88540); current and future reads clean; snapshot 88213 held, not expired

# A DR registry with different schema ids
[dana2] $ cat /queues/payments-dr/payments.events --from=-1h | head -1
refused (exit 3): schema id 233 on registry reg-dr has fingerprint 4f1c…, cached id 233 from reg-eu is 9a02…; ids are per registry   (L22.3)
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| typed read | an artifact keyed with the reader schema id | broker fetch, registry lookups by id | artifact store, receipt |
| `tee` to a table | one commit | Iceberg snapshot with tag and object lock | catalog, object store |
| `rotate --leaked` | an incident id, a new version, a retire phase | producer hook, vault version | vault, receipts |
| `scrub` | a process with a receipt | scans in the controller; quarantine class changes; Loki delete requests; CloudWatch policy | receipts, artifact store |
| `--purge` on a table | an approval, a commit | Iceberg row-level delete | catalog, `/sys/proc/<pid>/approval/<n>` |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ● | artifacts keyed by reader schema; lineage forward from a topic range (L22.1) | AC-22.1 |
| A2 | Command | ○ | `input: bytes` for raw readers | |
| A3 | Session | – | | |
| A4 | Backend | ○ | the scrub runs in the controller, not the session | |
| A5 | Contract | ● | unresolvable records against the floor; a degraded scrub | AC-22.4 |
| A6 | Device tree | ● | `caps` for registries, `log-delete`, scrub per device (L22.2) | AC-22.2 |
| A7 | Signals | – | | |
| A8 | Secrets | ● | rotation with `--leaked`, scrub, keyed-hash matching | AC-22.1 |
| A9 | Cost | ○ | scan price and budget (L22.8) | AC-22.6 |
| A10 | Identity | ● | purge approved by the object's owning group; match locations by group (L22.5, L22.6) | AC-22.5 |
| A11 | Defaults | ● | serialization of logical types, scrub capabilities, the purge policy | AC-22.3 |
| A12 | Storage | ● | table row deletes and held snapshots; topic retention (L22.2, L22.9) | AC-22.2 |
| A13 | Networking | – | | |
| A14 | Time | ○ | retention date of matched topic ranges; hold expiry | |
| A15 | Audit | ● | the incident across receipts; outside-reach lists | AC-22.4 |
| A16 | Durability | ● | the hold that outlives the incident (L22.9) | |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | schema v5 was registered through the GitOps repository | the scrub never removes a schema; a v6 without the field is a normal registry write | ok |
| P2 | POSIX cannot hold | Avro decimals, timestamps, bytes and unions have no text form; Avro's JSON encoding wraps unions as `{"string": …}` | decision 10 fixes the columns, not the value forms | **L22.7** |
| P3 | Orphaned state | quarantined copies kept with no end | class `sensitive-quarantine` has a platform retention; after retire and hold expiry they go | ok |
| P4 | Identity leak | the scrub receipt lists partitions and offsets where the value sits in a topic nobody can quarantine | locations are readable only by security and each object's owning group | **L22.6** |
| P5 | Invisible cost | scanning 30 days of a topic and a 3 TB table byte by byte | with a schema, only string and bytes fields are scanned; the budget binds and unscanned ranges are listed | **L22.8** |
| P6 | Lying cache | schema ids are per registry; a DR registry assigns 233 to a different schema | the cache is keyed by registry and id, and fingerprints are checked | **L22.3** |
| P7 | Silent failure | the v4 read silently drops `debug_ctx` | the receipt counts the records and the dropped fields (decision 10) | ok |
| P8 | Partial fleet | a site broker is unreachable during the scrub | listed, never assumed clean (amendment 5 decision 2) | ok |
| P9 | Created elsewhere | a Flink job and a foreign consumer group read the matched offsets into sinks the scrub cannot see | consumer groups whose committed offsets passed a match are listed with their owner and paged | **L22.4** |
| P10 | Multi-party | the secret belongs to `ledger-dba`, the table to `payments`; who approves deleting rows in `payments`' table | the purge gate names an approver from the owning group of each object | **L22.5** |
| P11 | Data at rest | the value is in a topic and a table, which are not artifacts; amendment 5 seeds its scope from the reference index and lineage of artifacts only, and quarantine has no meaning on a log | scope follows the writes of processes that held the version into devices, and each device class says what scrub it can do | **L22.1**, **L22.2** |
| P12 | Inheritance | the backfill read the topic, not the secret; nothing ties its table commit to v7 | lineage forward from the matched device ranges (L22.1) | folded into L22.1 |

A thirteenth question: the table's snapshot 88213 is under a seven-year hold with object lock (US-15, L15.5); the leaked rows stay readable by time travel there until 2033, whatever the scrub does. **L22.9**, accepted.

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| record not resolvable into the reader schema | decode | counts against the floor | partition, offset, writer id | `--schema-version`, or fix the subject |
| CSV over a nested reader schema | plan | 3 | the nested field | `text=jsonl` or the typed path |
| schema id fingerprint mismatch across registries | decode | 3 | both registries and fingerprints | mount the right registry |
| matches in a topic | scrub | 0, degraded | count, retention date | wait for retention, or a versioned topic (S12) |
| matches in a held table snapshot | scrub | 0, degraded | hold and expiry | none (accepted, L22.9) |
| a consumer read the matches into its own sink | scrub | 0, degraded | groups and owners | the owners act |
| purge on another group's object | gate | 3 until approved | the approving group | `approve` by that group |
| scan over budget | plan or runtime | 0, degraded | unscanned ranges | raise `--budget`, rerun on the ranges |

## 8. Acceptance criteria

- **AC-22.1** Given `secret scrub <name> <vN>`, then the scope includes the device ranges written by every process that held the version, and every artifact, table commit or unit read computed from those ranges, transitively, each listed with its source.
- **AC-22.2** Given matches on a device, then the scrub applies the action the device class reports (table: row-level delete under `--purge`, snapshot expiry where unheld; topic: none, truncate or key tombstone) and states the date retention or expiry removes what it could not.
- **AC-22.3** Given a reader schema with decimals, timestamps, bytes, enums or unions, then `text=jsonl` and `text=csv` print them in the fixed forms the column list names, and CSV is refused for nested schemas.
- **AC-22.4** Given matched offsets in a topic, then the receipt lists every consumer group whose committed offset passed them, with its owner or `?`, and pages each owning group.
- **AC-22.5** Given `--purge` on an object owned by another group, then the gate requires an approver from that group, and the commit records it.
- **AC-22.6** Given a scrub receipt, then match locations are readable by the security group and by the owning group of each matched object, and nobody sees content.
- **AC-22.7** Given a schema id read through two registries, then the cache is keyed by registry and id, and a fingerprint mismatch is refused.
- **AC-22.8** Given a scrub whose estimate exceeds `--budget`, then it scans in the order of exposure (logs and artifacts, then device ranges) and lists the unscanned ranges in a degraded receipt.

## 9. Persona lenses

- **SEC**: wants one incident id across rotation, retire and scrub, an exposure list that names what was not reached and why, and match locations kept from people who could act on them.
- **DE**: wants reads to keep working during the incident, a pinned schema that keeps their columns, and value forms a `cut` and a `sort` can use.
- **DBA**: wants the rotation through the producer and the old version disabled before anyone argues about scrubbing.
- **PE**: wants each device class to say what scrub it can do, and registries keyed by their own id.
- **FIN**: wants the scan priced before it runs and charged to the secret's owning group.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L22.1 | P11, P12 | Amendment 5 seeds the scope from the reference index and the lineage of artifacts, but a process holding the secret writes to devices (a topic, a table), and what read those ranges next is not in any artifact's inputs; the backfill and its table commit would never be scanned. | **amend** ADR 0007 amendment 5 decision 2: the scope also follows every write the holding processes made to a device (offset ranges, table commits, object keys, from their receipts) and continues forward from every read of those ranges recorded in a receipt, transitively; each candidate is listed with the path that reached it | ADR 0007 a5; ADR 0012 d2; ADR 0015 d4; S6 |
| L22.2 | P11 | Quarantine is a class on an artifact; a topic is an append-only log with no per-record delete or reader set, and a table keeps old snapshots readable by time travel. | **rule**: each device class reports `scrub:` in `caps`: tables `row-delete` (a delete commit under `--purge`, then expiry of unheld snapshots that contain the rows), topics `none`, `truncate` or `key-tombstone` on compacted keyed topics; the scrub never truncates by default and `--purge` on a `truncate` topic lists the records it would destroy; every match it cannot remove is reported with the date retention or expiry removes it | ADR 0007 a5; ADR 0012 d3, d10; ADR 0015 d4, d5; S12 |
| L22.3 | P6 | Decision 10 caches schemas by id because ids are immutable, but ids are per registry; a DR or edge registry can give the same id to a different schema. | **rule**: the schema cache is keyed by registry id and schema id, each entry carries the schema's fingerprint, and a read whose id resolves to a different fingerprint in the cache is refused naming both registries | ADR 0012 d10 |
| L22.4 | P9 | Consumers outside the shell (a package's Flink job, a foreign group) read the matched offsets into sinks the scrub cannot see. | **rule**: for every topic range with matches the scrub lists the consumer groups whose committed offsets passed it, with their owner or `?` (L12.6), pages each owning group, and lists them as `outside reach` | ADR 0007 a5; ADR 0012 rule L12.6; S9 |
| L22.5 | P10 | Amendment 5 gates `--purge` by an approval but does not say whose; the secret's owner is not the owner of the table whose rows would be deleted. | **rule**: the purge gate requires, for each object to be purged, an approver from that object's owning group (ADR 0016 decision 7), and the commit or deletion records the approver | ADR 0007 a5; ADR 0017 d2 |
| L22.6 | P4 | A scrub receipt lists the partitions and offsets where a live value sits in a topic that cannot be quarantined; anyone who reads the receipt can read the value. | **rule**: match locations in a scrub receipt are readable by the security group and by the owning group of each matched object only; the rest of the receipt follows rule L17.6 | ADR 0007 a5; ADR 0016 rule L17.6 |
| L22.7 | P2 | Decision 10 fixes the column list but not the value forms: decimals, timestamps, bytes, enums and unions have no single text form, and Avro's JSON encoding wraps unions in an object. | **rule**: text serializations print timestamps as ISO 8601 UTC, decimals as strings with their scale, bytes as base64, enums by symbol and unions as the bare value; the column list `explain` prints names the form of each column | ADR 0012 d10; ADR 0015 d2 |
| L22.8 | P5 | A full scan of a topic range and a 3 TB table is the scrub's dominant cost, and a byte scan reads columns no string can be in. | **rule**: where a schema is known the scrub scans only string and bytes fields; the band is in `explain`; `--budget` binds, the scan runs in order of exposure (logs, artifacts, device ranges) and a degraded receipt lists the ranges left unscanned | ADR 0007 a5; ADR 0013 d3 |
| L22.9 | none | A held snapshot under object lock keeps the leaked rows readable by time travel until the hold expires; neither quarantine nor purge can touch it. | **accepted**: the hold is a legal obligation (ADR 0006); rotation makes the value dead; the receipt names the snapshot, the hold and its expiry, and the table's grants are the only narrowing available | ADR 0007 a5 d5; ADR 0015 d5, rule L15.5 |
