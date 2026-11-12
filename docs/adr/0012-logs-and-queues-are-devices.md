# ADR 0012: Logs and queues are devices; a topic has three faces

**Status:** Draft · 2026-11-09 · revised 2026-11-09 with user-story findings L11.2, L11.5, L12.1 and L13.3 (marked inline) · decision 10 added 2026-11-10 for the open finding L12.5
**Supersedes / relates to:** ADR 0001 (pipe is a staged artifact, streams opt-in); ADR 0003 (backends); ADR 0006 (detach is not destroy); ADR 0010 (packages); ADR 0011 decision 9 (`/dev/queue` reserved); docs/pipeline-compiler.md §8.5 (stream relay); docs/posix-command-map.md (`mkfifo`, `tail -f`); docs/stories.md (named pipes and queues, disconnected operation); docs/user-stories/README.md misroutes S4, S6, S9; a contributed review of Kafka as a Convoy primitive (Grok, 2026-11-09, kept under `research/sources/`, not in git)

## Context

The contributed review proposes Kafka as a first-class primitive: a `/kafka/` tree by provider and region with a `current` symlink and a federated `all`, a command table (`ls` topics, `mkdir topic`, `cat` and `tail -f` to consume, `echo | topic` to produce, `ps` for consumer groups, `kill group` to reset offsets), topics as pipeline sources and sinks, a streaming backend that emits Kafka Streams or Flink jobs for continuous pipelines, planner-inserted MirrorMaker2 for cross-cluster pipes, and lag as part of the completion contract. The owner's interest is coverage of edge computing, where Kafka and its relatives are the durable buffer between intermittently connected sites.

Facts that constrain the design:

- ADR 0011 already decided the shape of every device class: `by-id` is the truth, provider and region trees are views, devices are mounted with options rather than symlinked, `self` replaces `current`, and `all` is a read-only union that refuses writes. Decision 9 reserves `/dev/queue/…` for exactly this class.
- A topic is an append-only, partitioned, offset-addressed log. A broker reports partition count, log-end offsets and bytes per partition. The planner's one permitted size, the input size, is therefore known exactly, and the fan-out is given rather than derived.
- Retention deletes offset ranges on a schedule. Compaction rewrites what an offset range contains. Neither is under the session's control.
- A write to a topic is a side effect. Idempotent producers and transactions exist in the protocol, keyed by a transactional id.
- Brokers often run outside any Kubernetes cluster: managed services (MSK, Confluent Cloud), bare-metal edge boxes, Redpanda on k3s. Kafka has its own `cluster.id`. When brokers do run in a cluster, Strimzi's `Kafka`, `KafkaTopic` and `KafkaUser` resources are usually GitOps-managed.
- ADR 0001 makes the pipe a staged artifact and the stream opt-in. The README says Convoy is not a stream processor, and the roadmap lists translating text tools into engine operators and streams as the default pipe as non-goals.
- Pricing shapes differ: MSK bills broker-hours and bytes in and out, Confluent bills throughput units and partition-hours, a Strimzi cluster is pods that OpenCost already sees.
- A consumer group can use at most one member per partition.

## Decision

1. **Log and queue systems are devices under `/dev/queue/`.** The `by-id` key is the broker's own cluster id, the MSK ARN or the Confluent cluster id. `by-provider`, `by-topology` and `by-label` are views; `self` is per session; `all` is the read-only union. Mount options are the policy surface: `identity` (a KafkaUser secret reference, an IAM role, an mTLS certificate reference), `ro`, `topics=<prefix>` as the namespace scope, `group=<prefix>` as the consumer-group namespace, `residency` (ADR 0011 amendment 3), and `quota=partitions:<n>,bytes-in:<rate>,bytes-out:<rate>` because partitions are a scarce, billed resource with a per-broker limit: `mkdir` pre-flights the partition quota and the broker limit and `df` shows both (L13.3). Opening the device is the credential exchange. Hotplug comes from Strimzi resources where they exist and from a registry source in the inventory for brokers that live outside any cluster. A device type is mountable only when a package provides its fileserver driver, reported by `caps` as `queue:kafka`, `queue:nats`, `queue:sqs`.

   ```
   mount -o identity=secret:orders-reader,topics=orders.,group=etl-,ro \
         /dev/queue/by-id/kafka-7Hd3…-orders-prod \
         /queues/orders
   ```

2. **A topic is a path with three faces, and the planner keeps them apart.**
   - **A bounded read is a file.** `cat`, `head`, `tail -n`, `< path`, and any stage reading the path take a snapshot of the log-end offsets at plan time. The topic's partitions are the granularity of the read, not the count of plan partitions: the planner splits a large partition by offset range and coalesces small ones by size, preserving order within each topic partition (L11.5). The artifact id is `sha256(cluster id, topic, partition, offset range, read options)`. `--from` and `--to` accept offsets or timestamps. The completion contract applies as to any partitioned input: floor over partitions, deadline, budget.
   - **A follow is the opt-in stream.** `tail -f` and `|=` read from a committed offset and never produce an artifact; caching and replay are lost and `explain` says so (ADR 0001, compiler §8.5). A follow resumes from its committed offset after a link loss, which is what makes it usable at the edge.
   - **A write is a side effect, never an artifact.** `> path` and `tee path` produce with an idempotent, transactional producer whose transactional id is the artifact id of the input stage, so a retry or a replay never double-produces. A rerun still appends: the key prevents double production of the same artifact, not the coexistence of an earlier run's records beside the new ones, so `explain` warns when the target topic already holds output for an overlapping input range and names the earlier pid, versioned topic names are the recommended spelling, and `--replace` is refused as impossible on a log (L11.2). For a follow, which has no artifact id, a follow-to-write unit runs consume-transform-produce with the input offsets committed inside the producer transaction, keyed by group, partition and batch; `explain` prints `exactly-once` or, for a sink device without transactions, `at-least-once` with the reason (L12.1). `explain` marks every write as a side effect; it is never a cache hit.

3. **Retention and compaction are stated, not hidden.** Reads from a compacted topic are `deterministic: false` (ADR 0001 amendment 1): the id is salted and never served from cache. A hold (`--retain`) on a receipt whose input is an offset range copies that range into the artifact store at hold time, or refuses the hold with the reason, because the broker will delete it.

4. **There is no streaming backend.** A continuous pipeline is a service unit (ROADMAP M11): `run --unit NAME 'tail -f /queues/orders/orders | … | tee /queues/eu/enriched'` becomes a Deployment of the toolbox running the real binaries as a consumer group named after the unit. `scale` above the partition count is refused with the reason. Kafka Streams, Flink, ksqlDB, Kafka Connect and MirrorMaker2 are packages (ADR 0010) whose executables are annotated like `spark-submit`: `flink run` has `class: non-parallel`, `compiles_to: flinkdeployment`, `group: flink`. The planner never turns a shell pipeline into an engine job.

5. **Cross-device pipes are two stages and an egress line.** `cat /queues/us/orders | transform | tee /queues/eu/mirror` is one bounded read in one region and one write in another, with the egress priced from the two devices' topology (ADR 0011) and residency rules applied. A continuous mirror is a unit from the MirrorMaker2 or Cluster Linking package, chosen and named by the user. The planner never inserts one.

6. **`mkfifo` creates a topic.** The platform default `fifo_device` names the queue device that backs named pipes; `mkfifo /queue/human-review` creates a topic there with the platform's retention class and prints both. When the platform sets `stream_relay=queue`, the compiler's `|=` relay for stages that span nodes or sites is a topic on that device instead of a sidecar, so a stream survives a node loss and resumes from an offset.

7. **Verbs on the device.**

   | Verb | On a queue device | Rule |
   |---|---|---|
   | `ls -l` | topics with partitions, retention, bytes, consumer lag | from `/sys/queue/<id>/` |
   | `mkdir` | create a topic with the platform's partition and replication defaults | written as intent when the `KafkaTopic` is GitOps-managed (misroute S4) |
   | `rm` | delete a topic | lists bytes and retention, requires `--with-data` (ADR 0006); `umount` only detaches |
   | `df`, `du` | bytes, retention, price shape | `/sys/queue/<id>/` |
   | `ps`, `top` | consumer groups as units, with lag | groups created elsewhere show owner `?`; `chown` adopts (misroute S9) |
   | `seek <group> <topic> --to=<offset\|time>` | reset a group's position | a data verb with confirmation; `kill` never resets offsets |
   | `wait --lag-below=<n>` | a health condition on a unit | needs `caps: metrics`; it is not part of the completion contract |

8. **Cost has a second shape.** `/sys/queue/<id>/price` carries the device's billing shape (broker-hours and bytes, throughput units and partition-hours, or pod cost through OpenCost). The band for a read or write stage is computed from bytes and partitions against that shape; a shared broker's standing cost is shown as a rate and charged to a process only when the platform says so.

9. **The device is the edge buffer.** Brokers outside Kubernetes are devices like any other. A session cut off from a site queues commands with a visible count (debate-review, disconnected mode); producers at the site buffer locally; a follow resumes from its committed offset when the link returns. Nothing in the design is specific to a cloud-hosted broker.

10. **An encoded topic is read through its schema, and text tools see a serialization, never the bytes** (added 2026-11-10, settles L12.5). Avro, Protobuf and JSON Schema records carry the id of the schema they were written with (the registry wire format: a magic byte and a schema id, message indexes for Protobuf, a version id for Glue). A queue device whose package provides a registry driver lists it in `caps` (`schema:confluent`, `schema:apicurio`, `schema:glue`, with the encodings each decodes), and the registry is opened with the mount's identity like the broker.
    - **The mount says how records are seen.** `schema=registry|raw` and `text=jsonl|csv` are mount options, as `text=` is on a table mount (ADR 0015 decision 1); `schema=registry` is the default where `caps` lists a registry. Each record is decoded with the writer schema its own id names; schemas are immutable by id, so the driver caches them and a read whose ids are all cached runs while the registry is unreachable (decision 9).
    - **A bounded read pins a reader schema.** At plan time the planner resolves the subject's latest version, or the one `--schema-version=` names, records its id in the plan and the receipt, and projects every record into it by the format's resolution rules. The artifact id adds the reader schema id and the serialization, so a cache hit is never decoded differently. A record that cannot be resolved into the reader schema is written to stderr with its partition and offset and counts against the stage's floor (ADR 0004); the receipt counts records written with a newer writer schema and the fields they lost.
    - **Text tools get stable columns.** Under `text=jsonl` each record is one JSON object per line; under `text=csv`, allowed only when the reader schema is flat, one row per record with a header. The leading fields are `_partition`, `_offset`, `_timestamp`, `_key` (decoded with the key subject when it has one) and `_schema`, then the reader schema's fields in order. `explain` prints the column list, so `cut -f3` names the field it reads and a schema change can never move it during the read.
    - **Encoded bytes are never handed to a text tool silently.** The driver samples each partition at plan time; when records carry a registry wire-format prefix and the mount says `schema=raw` or no registry is reachable for an uncached id, a stage that would feed those bytes to a command is refused with the reason and the `--raw` spelling, unless the command's annotation declares `input: bytes` (a new annotation field; `xxd`, `od`, `cksum` carry it); `--raw` hands the bytes over on request.
    - **Typed verbs read the reader schema as their type** (ADR 0015 decision 9): `select`, `filter`, `group` and `tjoin` push projection and predicate into the decode, and `tee` into a table maps fields by name.
    - **A write encodes against a pinned version.** `> path` and `tee path` encode each line against the subject's version at plan time, pinned in the plan's shape (ADR 0017 decision 6); a line that does not validate aborts the stage's producer transaction (decision 2), so nothing is produced, and the stage exits 1 naming the first failing line and the count. The shell never auto-registers a schema.
    - **A schema change is a write to the registry, checked by its compatibility mode.** A new version is `cp <file> /sys/queue/<id>/schemas/<subject>`, written as intent where the registry is GitOps-managed (misroute S4); `explain` prints the subject's compatibility mode and the registry's verdict before anything is written; `mkdir --schema=<file>` creates a topic with its first version, as for tables (ADR 0015).
    - **A unit keeps its pinned reader schema across a change.** A follow (a unit) decodes with the reader schema pinned at its last `release`; `ps` marks the unit `schema drift` when the subject's latest version is newer, the unit's receipt records each new writer schema id it sees, and `release` re-pins.

## Declined from the review

| Proposal | Why not | What replaces it |
|---|---|---|
| `/kafka/` tree by provider and region as the truth, `current`, federated `all` | the layout ADR 0011 reviewed and changed | `/dev/queue/by-id` with views, `self`, read-only union |
| a session's "current Kafka cluster" | the mount table holds several devices at once; a topic is a path | mounts under `/queues/…` |
| `topic` as a verb (`echo … \| topic orders`) | the POSIX surface is the filesystem | redirections and `tee` on paths |
| a streaming backend emitting Kafka Streams or Flink | translation of text tools into engine operators, a stated non-goal | service units for continuous pipelines; engines as packages |
| planner-inserted MirrorMaker2 or Cluster Linking | a hidden default and a federation by another name | two stages with an egress line, or a user-named mirror unit |
| `kill group` to reset offsets | destroys position, not a process | `seek` with confirmation |
| lag in the completion contract | a service-level condition, not convergence | `wait --lag-below` on a unit |
| "a typed, content-addressed artifact boundary, or a streaming boundary" | the "or" hides three different things | the three faces in decision 2 |

## Consequences

- Topics are the best-behaved input the planner has: the size is exact and the partitions are given. The 412 GiB example in the README would plan the same way over a topic with no listing cost.
- Named pipes and the stream relay become durable and resumable, which closes the edge and disconnected-operation gap in `docs/stories.md` with no new abstraction.
- `flink run` and `spark-submit` are the same kind of thing, so walkthrough 2 and user story US-02 already describe the engine path.
- Cost: a Kafka protocol driver in the session fileserver (`caps: queue:kafka`), a second price shape under `/sys`, a consumer-group index for `ps`, and a wider M5. The `seek` verb and the `--from`/`--to` read options are new surface.
- Schema registries are decision 10 (2026-11-10, L12.5), built on ADR 0015 decision 9. [US-22](../user-stories/US-22-leak-in-the-stream.md) probes it with two pinned reader schemas over one topic and a DR registry; it raised rules L22.3 (the cache is keyed by registry and id, with fingerprints) and L22.7 (fixed text forms for logical types), written below on 2026-11-10.
- Decision 10 adds a registry driver per package (`caps: schema:<registry>`), the `schema=` and `text=` options on queue mounts, an annotation field `input: bytes` for the few commands that may read encoded records raw (ADR 0002), `/sys/queue/<id>/schemas/` and a `schema drift` state for units in `ps`; all are M5.
- Open, to be settled when M5 is designed against this ADR: keyed writes (`tee --key=`) and whether order is per partition or global for a downstream `sort`; exactly-once across a pipeline that ends in a side-effect sink; what `ls` shows for a topic with ten thousand partitions.
- User stories [US-10](../user-stories/US-10-edge-telemetry.md) (edge telemetry), [US-11](../user-stories/US-11-topic-reprocess.md) (reprocessing by offset range), [US-12](../user-stories/US-12-continuous-consumer-unit.md) (a continuous consumer unit and the engine package) and [US-13](../user-stories/US-13-tenant-topics-and-work-queues.md) (tenant topics, ACLs, quotas and work queues) exercise every decision above. They raised 27 findings; the four that amend this ADR, L11.2 (a rerun appends), L11.5 (partitions are the granularity, not the count), L12.1 (exactly-once for follows) and L13.3 (partition quotas), are applied inline in decisions 1 and 2 as of 2026-11-09 and collected in `docs/user-stories/README.md` under misroute S12.

## Rules · 2026-11-09 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L10.3** (US-10) `df` on a queue device shows oldest unread offset per consumer group against retention; `explain` warns under a platform margin; `cat --from` before the earliest offset is exit 1 naming the lost range, never a partial success.
- **L11.1** (US-11) `--from` and `--to` resolve to per-partition offsets at plan time; the receipt stores them; `explain` warns on CreateTime topics.
- **L11.3** (US-11) queue reads carry `order: per-partition` as a stage property, like partitioned file reads; the compiler warns when an order-sensitive class follows without a global sort.
- **L11.6** (US-11) bounded reads use direct partition assignment and commit nothing; only units (follows) own a group, prefixed by the mount's `group=` option.
- **L12.4** (US-12) lag is computed from the broker's group offsets through the queue device, under `/sys/queue/<id>/groups/`; no metrics capability required.
- **L12.6** (US-12) unit groups are prefixed by the mount's `group=` option; a group that exists with members not labelled by Convoy is a plan-time refusal; `ps` shows foreign groups with owner `?`.
- **L13.1** (US-13) `rm` on a topic with retained bytes requires `--with-data`, a second approval above a platform threshold (US-03 L3.1), and prints the `backup` spelling that copies the retained range into the artifact store first.
- **L13.2** (US-13) `chmod` maps `r` to Read and Describe, `w` to Write, `x` to Create under the prefix; everything else is `setfacl`/`getfacl`; `ls -l` marks a `+` when ACLs exceed the mode, as POSIX ACLs do.
- **L13.4** (US-13) `mkfifo` creates a dead-letter topic by default with `max-deliveries` from the platform template; consumers move an item after the limit and the process exits degraded with the count; `mkfifo --no-dead-letter` is explicit.
- **L13.5** (US-13) reading a queue by redirection is a bounded drain to the depth observed at plan time and compiles to a work-queue Job; a follow into `xargs` is continuous and requires `run --unit`, scaled by queue depth when `caps: keda` exists.
- **L22.3** (US-22) the schema cache of decision 10 is keyed by registry id and schema id, each entry carries the schema's fingerprint, and a read whose id resolves to a different fingerprint in the cache is refused naming both registries.
- **L22.7** (US-22) text serializations print timestamps as ISO 8601 UTC, decimals as strings with their scale, bytes as base64, enums by symbol and unions as the bare value; the column list `explain` prints names the form of each column.
- **L27.7** (US-27) a replication package (database migration services, MirrorMaker 2) provides `/sys/unit/<name>/lag` in seconds and bytes; `wait --lag-below=<n> --for=<duration>` is the cutover condition, spelled `scale <writers> 0 && wait --lag-below=0 --for=30s <repl> && release …`, and `explain` checks the chain end to end.
- **L27.10** (US-27) `ls -l` of a broker device lists the receipts and holds that name its offsets; removing the device is refused while a hold names it, and `--hold` copies held ranges to artifacts first; the replication package's offset translation is recorded so a receipt can say which target offsets correspond.
