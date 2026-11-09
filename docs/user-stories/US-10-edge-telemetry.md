# US-10 · Aggregate telemetry at forty remote sites over links that drop daily

| Field | Value |
|---|---|
| Primary persona | DE (Data Engineer) |
| Secondary personas | SRE, PE, FIN, SEC |
| Workload class | edge batch and continuous unit, disconnected operation |
| Walkthrough | none; written against [ADR 0012](../adr/0012-logs-and-queues-are-devices.md) and the disconnected-mode story in [stories.md](../stories.md) |
| Gaps already closed there | none |
| Milestones needed | M2, M3, M5, M8, M11 |
| Findings in this document | L10.1 – L10.9 |

## 1. Story

**As a** data engineer at an operator of forty remote sites, **I want** each night's sensor data to be downsampled at the site where it was produced and only the aggregates shipped to the central region, with an anomaly detector running continuously at each site, **so that** a day without a link costs nothing but delay and I never pay to move raw bytes over LTE or satellite.

**Context.** Each site runs a single-node k3s cluster and a Redpanda broker that sensors publish to. Links drop for hours most days and for days a few times a year. Today the nightly job is a cron on each site box, the results arrive by rsync, and nobody knows which sites are behind until the dashboard shows holes.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | the data engineer, from a central session | `edge-batch` service account on each site mount; a reader identity on each queue mount | forty pairs of mounts, one session |
| Approves | none | | |
| Pays | operations | compute at the site is sunk cost; the link is metered per gigabyte | L10.5 |
| Audits | the reliability review, monthly | which sites ran, which were queued, what was lost to retention | needs receipts that survive an outage (L10.2) |
| Operates the platform | platform team | the inventory's registry source for off-cluster brokers, the per-site secret source, link metadata under `/sys` | |

## 3. Preconditions

- **Capabilities** per site device: `indexed-job`, `queue:kafka` driver, `node-agent`; `argo` is not required for a linear script and `explain` must say so when a script needs it (L10.9).
- **Packages**: `downsample`, `anomaly` as user tools with annotations; `mirrormaker2` as a platform package for the alert mirror.
- **Mounts**: `/sites/<n>` on `/dev/k8s/by-topology/edge/site-<n>` with `sa=edge-batch,ns=telemetry`; `/sites/<n>/queue` on `/dev/queue/by-topology/edge/site-<n>` with `identity=secret:site-reader,topics=telemetry.,group=edge-`; `/s3/eu/telemetry` for aggregates; `/queues/central` for alerts.
- **Annotations**: `downsample: stateless`, `anomaly: stateless, deterministic: false` (model inference with a changing model path is still deterministic per input id; the unit is a follow and never cached anyway).
- **Platform defaults** `explain` must print: placement by device locality (L10.7), queue policy for a disconnected site and its TTL, the secret source used at the site (L10.6), link price and bandwidth, broker retention versus oldest unread offset (L10.3).

## 4. Main path

```bash
$ ls /dev/queue/by-topology/edge/ | wc -l
40
$ df -h /sites/17/queue
TOPIC           PARTS  BYTES   RETENTION  OLDEST-UNREAD(edge-nightly)  LINK
telemetry.raw   12     38 GiB  7d         22h                          up  LTE 9 Mb/s  $0.08/GB

$ cat nightly.sh
cat --from=@$DAY --to=@$NEXT telemetry.raw | downsample --1m | tee telemetry.1m | put /s3/eu/telemetry/$SITE/$DAY/

$ (cd /sites/17/queue && explain nightly.sh)
stage 1  cat         bounded read   12 partitions, 5.4 GiB exact      parallel-local at site-17 (1 node)
stage 2  downsample  stateless      12 partitions                     same pod
stage 3  tee         side effect    telemetry.1m, idempotent producer, txn id = artifact id
stage 4  put         copy           42 MiB to eu-west-1 over LTE      egress $0.003, eta 40 s
placement: stages 1–3 at site-17 (device locality: 5.4 GiB never leaves the site)
contract: floor 100 %  deadline none  budget none        band $0.02–$0.04
secret source at site-17: site-local replica (synced 3h ago)

$ for s in $(ls /sites); do (cd /sites/$s/queue && batch nightly.sh --deadline=6h --on-deadline=keep-partial); done
…
convoy: site-23 is disconnected (link down since 02:14). 1 command queued for site-23 (policy: queue, ttl 24h).

$ ps -A --cluster=all | grep -e queued -e disconnected
site-23   7733  queued      link down 1h12m    ttl 22h48m

$ (cd /sites/17 && run --unit anomaly 'tail -f queue/telemetry.raw | anomaly --model=/models/vib/2.1 | tee queue/alerts')
unit anomaly  Deployment 1/1 at site-17  consumer group edge-anomaly  12 partitions  follow resumes from committed offset
$ scale anomaly 16
convoy: telemetry.raw has 12 partitions; members above 12 would idle. Refusing; `scale anomaly 12` is the maximum.

$ run --unit alerts-mirror --package=mirrormaker2 --from '/sites/*/queue/alerts' --to /queues/central/alerts
unit alerts-mirror  Deployment 1/1 in prod-eu  40 sources, resumes per site on reconnect
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `cat --from --to` | bounded read, 12 partitions | one toolbox pod at the site (parallel-local) reading the broker | artifacts in the site-local store, then receipts |
| `tee telemetry.1m` | side-effect write | transactional producer | the broker |
| `put /s3/eu/…` | copy with egress | copy from the site over the link | the central object store |
| queued command | session-side object | nothing yet | the session; runs on reconnect or expires |
| `run --unit anomaly` | service unit, consumer group | Deployment at the site | offsets in the broker |
| `run --unit alerts-mirror` | user-named mirror | MirrorMaker2 from the package | central broker |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ● | bounded reads keyed by offset range; a rerun of a finished night is a cache hit | the id contains cluster id, topic, partitions and offsets; the rerun executes nothing |
| A2 | Command | ○ | two stateless tools and a package executable | `type anomaly` shows the class |
| A3 | Session | ● | eighty mounts, commands queued for a disconnected site, a loop over `ls /sites` | `ps` shows queued commands with their TTL (L10.1) |
| A4 | Backend | ● | parallel-local at a one-node site; placement by device locality | `explain` prints the placement and the reason (L10.7) |
| A5 | Contract | ● | floor over partitions, deadline during an outage, `keep-partial` | a deadline that expires while queued is explicit, not silent (L10.4) |
| A6 | Device tree | ● | off-cluster brokers from a registry source; `by-topology/edge`; a device that is disconnected rather than removed | `mount` shows `disconnected since` (L10.1) |
| A7 | Signals | ○ | TERM on the unit commits offsets and stops | `kill anomaly` leaves no uncommitted work |
| A8 | Secrets | ● | identity references resolved at a site that cannot reach the vault | `explain` names the site-local source (L10.6) |
| A9 | Cost | ● | link egress per gigabyte; compute at the site | the band separates link egress from compute (L10.5) |
| A10 | Identity | ○ | one service account per site mount | the receipt names the site identity |
| A11 | Defaults | ● | queue policy and TTL, secret source, placement, retention warning | all in `explain` |
| A12 | Storage | ○ | the broker's disk is the only buffer; retention versus outage | `df` shows oldest unread against retention (L10.3) |
| A13 | Networking | ● | the link is a metered, failing resource | `/sys/k8s/<site>/link` exists (L10.5) |
| A14 | Time | ● | nightly, deadlines, TTL on queued commands | AC-10.3 |
| A15 | Audit | ● | receipts written at the site and synced | no receipt is lost to an outage (L10.2) |
| A16 | Durability | ● | the site finishes its night with no link; the unit runs for months | AC-10.2, AC-10.5 |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | the mirror is a unit the user named | no silent insertion (ADR 0012 d5) | ok |
| P2 | POSIX cannot hold | `cat --from` older than the earliest offset | the bytes are gone | **L10.3** |
| P3 | Orphaned state | aggregates written at a site whose link never returns | they sit in the site store with a retention class | ok |
| P4 | Identity leak | none | | ok |
| P5 | Invisible cost | the link is billed per gigabyte by a carrier | no price shape for links in `/sys` | **L10.5** |
| P6 | Lying cache | a rerun after compaction | `telemetry.raw` is not compacted; a compacted topic would be `deterministic: false` (ADR 0012 d3) | ok |
| P7 | Silent failure | a Job that finishes during an outage and is garbage-collected before the session sees it | ADR 0009 has the session watch the object | **L10.2** |
| P8 | Partial fleet | a site is down | ADR 0011 d8 knows only "removed" | **L10.1** |
| P9 | Created elsewhere | brokers that are not custom resources; a replaced broker with a new cluster id | inventory registry source; by-id changes | L10.8 |
| P10 | Multi-party | none | | ok |
| P11 | Data at rest | raw bytes stay on the site disk for 7 days; aggregates go central | the placement decision is the residency decision | ok, via L10.7 |
| P12 | Inheritance | the deadline while a command is queued | finding L5.5 left the choice to a flag | L10.4 |

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| link down at submit | device state | queued | queued since, TTL | automatic on reconnect |
| link down for longer than the TTL | queue expiry | 2 | expired unrun, site, outage length | rerun with `--from` the same day |
| outage longer than retention | `--from` before earliest offset (L10.3) | 1 | offset range lost, hours of data | none; the platform raises retention |
| Job finished during an outage, pod TTL passed | nothing today (L10.2) | would be unknown | | site-side receipt spool |
| vault unreachable at the site | pod cannot mount the secret (L10.6) | 1 | secret source central, link down | site-local replica |
| 16 members on 12 partitions | plan time | 3 | partition count | `scale anomaly 12` |
| script uses `tee >(…)` at a site without Argo | `caps` at plan time (L10.9) | 3 | `caps: argo` missing at site-17 | linear script or install Argo |

## 8. Acceptance criteria

- **AC-10.1** Given `cat --from --to` on a topic, then `explain` shows one partition per topic partition and the exact byte count, and a rerun of a finished range is a cache hit with no bytes read from the broker.
- **AC-10.2** Given a site whose link drops after `batch` was accepted, then the stages at the site complete, the receipt is persisted at the site and reaches the session on reconnect with the original exit code and timestamps.
- **AC-10.3** Given a site that is disconnected at submit time, then the command is queued with a visible TTL in `ps`, runs on reconnect, and expires with exit 2 and a receipt if the TTL passes.
- **AC-10.4** Given a mount whose device stops answering, then `mount` shows it as disconnected with a timestamp, `ls` returns the cached listing marked stale or fails fast according to the mount option, and the device is not treated as removed.
- **AC-10.5** Given a consumer unit on a 12-partition topic, then `scale` above 12 is refused with the partition count and `kill` on the unit commits offsets before stopping.
- **AC-10.6** Given `df` on a queue mount, then it shows the oldest unread offset per consumer group against retention, and `explain` warns when the margin is under the platform's threshold.
- **AC-10.7** Given a stage whose input device is local to one cluster, then `explain` places the stage there, prints the reason, and prices only the output's egress.

## 9. Persona lenses

- **DE**: wants one script for forty sites and a cache hit on every night that already ran; refuses if a night can vanish without a receipt.
- **SRE**: wants disconnected to be a state with a timestamp and a TTL, not an error at the first `ls`.
- **PE**: owns the registry source for off-cluster brokers, the site-local secret replica and the link metadata; wants `caps` evaluated per device.
- **FIN**: wants link egress as its own line, since it is the only cost that varies.
- **SEC**: wants raw telemetry to stay on site by placement, and the receipt to prove it did.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L10.1 | P8 | ADR 0011 decision 8 knows removed devices only. A site whose link is down is not removed; treating it so fails every `ls` and drops queued work. | **amend** (applied: ADR 0011 a4) ADR 0011 decision 8: a device has a `disconnected` state distinct from `removed`; `mount` shows `disconnected since`; commands against it follow the mount's queue policy (`queue=<ttl>` or `fail`); `ls` uses `stale=cache` or `stale=fail` per the mount; hotplug emits connected and disconnected events | ADR 0011, debate-review.md (disconnected mode) |
| L10.2 | P7 | ADR 0009 has the session watch the object and persist the exit code before TTL deletion. With the link down longer than the TTL, the exit code and timestamps are lost. | **amend** (applied: ADR 0009 a1) ADR 0009: the node agent (or a per-device receipt spool) persists the receipt at the site when the object completes; TTL deletion is deferred until the session acknowledges the receipt (a finalizer or `ttlSecondsAfterFinished` set on acknowledgement) | ADR 0009, ROADMAP M10 |
| L10.3 | P2 | A bounded read whose `--from` lies before the earliest retained offset silently reads less than asked; an outage longer than retention loses data with no signal before it happens. | **rule** (applied: ADR 0012 rules): `df` on a queue device shows oldest unread offset per consumer group against retention; `explain` warns under a platform margin; `cat --from` before the earliest offset is exit 1 naming the lost range, never a partial success | ADR 0012 decisions 3 and 7 |
| L10.4 | P12 | A deadline counted from submission expires while a command is queued for a disconnected site. | **accepted**: finding L5.5 already gives `--deadline-from=start`; `explain` warns when the target device is disconnected at submit time | ADR 0004 (via L5.5) |
| L10.5 | P5 | Carrier links are billed per gigabyte and have a bandwidth; the device tree has no price shape or state for a link. | **rule** (applied: ADR 0011 rules): `/sys/k8s/<id>/link` holds state, bandwidth and price per gigabyte; the band separates link egress from compute; placement prefers the side of the link with the data | ADR 0011 decision 7 |
| L10.6 | P4, P8 | Secret references resolve against a central vault; a pod at a disconnected site cannot start. | **amend** (applied: ADR 0007 a4) ADR 0007: a mount declares its secret source (`secrets=central\|site-replica`); platforms provide a site-local replica with a sync age; `explain` prints the source and the age; a reference with no reachable source is exit 3 at plan time, not a pod stuck in creation | ADR 0007, ADR 0011 |
| L10.7 | P11, P5 | The cwd chooses the cluster, so a stage reading a site-local device from a central cwd would pull raw bytes over the link. Nothing in ADR 0003 places a stage where its input is. | **amend** (applied: ADR 0003 a1) ADR 0003: among the clusters the session has mounted, a stage runs where its largest input device is local when that cluster has the capabilities; `explain` prints `placement` and the reason; only outputs cross the link | ADR 0003, ADR 0011 |
| L10.8 | P9 | Off-cluster brokers come from a registry source, and a replaced broker has a new cluster id, so a `by-id` mount goes stale while the `by-topology` path still names the site. | **accepted**: mounting by a view path is allowed and re-resolves on hotplug; receipts always record the `by-id`; the inventory marks the replacement as a removed and an added device | ADR 0011 decision 2 |
| L10.9 | P7 | A script with `tee >(…)` or `&&` needs the Argo backend, which a one-node site may not have; capabilities are per device, not per fleet. | **rule** (applied: ADR 0003 rules; roadmap M8): `caps` is evaluated per target device at plan time; a backend the site lacks is a plan-time refusal naming the capability and the site | ADR 0003, ADR 0010, ROADMAP M8 |
