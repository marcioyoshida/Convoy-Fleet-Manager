# US-26 · The dashboard said green: find out at 2 a.m. why the service is slow, with metrics, logs and traces from the shell, without paying for the investigation twice or leaking what I read

| Field | Value |
|---|---|
| Primary persona | SRE |
| Secondary personas | DO, SEC, FIN, PE, DEV |
| Workload class | incident investigation across telemetry stores: a canary gated on a metric condition, log and trace reads across residencies, an assisted shell proposing queries, a postmortem that replays the night's reads |
| Walkthrough | written against [ADR 0023](../adr/0023-telemetry-is-a-device-the-shell-reads.md) (all twelve decisions), with [ADR 0013](../adr/0013-native-verbs-have-one-shape.md) decisions 5 and 7 (`wait`, `top`, `release --canary`), [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md) decisions 5, 6 and 11 (service identities, `sudo`, residency), [ADR 0017](../adr/0017-approvals-are-objects-on-the-process.md) (break-glass approval), [ADR 0020](../adr/0020-a-session-is-a-journal-not-a-pod.md) rule L21.2 (the redactor), [ADR 0007](../adr/0007-secrets-are-references.md) amendments 2 and 5 (the scrub, `log-delete`), [ADR 0014](../adr/0014-assisted-completion-the-model-proposes.md) (proposals) and [ADR 0004](../adr/0004-completion-contract-and-exit-codes.md) (degraded); it reuses the canary of US-03 and the edge site of US-10 |
| Gaps already closed there | none |
| Milestones needed | M4, M5, M7, M8, M11 |
| Findings in this document | L26.1 – L26.16 |

## 1. Story

**As an** on-call SRE, **I want** to go from a page to the cause with the metrics, logs and traces the platform already keeps, from the same shell I run everything else in, and to trust that a condition that says "healthy" measured the thing it names, **so that** an incident costs minutes of my night rather than hours, the investigation does not cost more than the outage, and nothing I read ends up somewhere it should not be.

**Context.** `ana` (user `ana`, group `sre`) is on call from Toronto; their session runs in residency `us`. At 01:30 UTC `kai` started `release orders-api sha256:e7d0… --to=/clusters/prod-eu/orders --canary=10% && wait --healthy orders-api --slo=error_rate<0.5% --timeout=6h && release orders-api --promote …` from their own session, using the `slo:` the team declared in the image's annotation. At 02:10 the page fires: orders-api p99 latency in prod-eu is 2.4 s against a 400 ms objective. The platform runs, for prod-eu: `mimir-eu` for metrics (tenants by group header; the edge tenant `factory` has a six-hour out-of-order window), `thanos-lt` for long-term metrics (downsampled to 5 minutes after 40 days), `loki-eu` for logs behind a label-enforcing proxy (14-day retention), CloudWatch Logs for the managed databases (`cwlogs-eu`, priced per gigabyte scanned), a legacy `opensearch-eu` for the payments team's application logs, and `tempo-eu` for traces; prod-us has `mimir-us`, `tempo-us` and Datadog for the payments team. The platform's observability repository defines a Pyrra SLO `orders-api-availability`, counted at the gateway.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Investigates | `ana` | user `ana`, group `sre` | on call; owns no namespace in prod-eu |
| Started the canary | `kai` | user `kai`, group `orders`; the waiting process runs in the controller | their token expires at 02:30 |
| Approves break-glass | platform lead | member of `platform` | ADR 0017 |
| Second responder | `raj` | user `raj`, group `sre`, residency `us` | looks at payments in prod-us |
| Pays | `sre` for the investigation's queries; `orders` for the canary's conditions | | |
| Operates the platform | platform team | the stores, the label proxy, the default devices per topology, the Pyrra SLOs, `telemetry_query_max` | |

## 3. Preconditions

- **Capabilities** (`caps`): `metrics:promql` (mimir-eu, mimir-us, thanos-lt), `logs:logql` with `label-enforce` (loki-eu), `logs:cloudwatch` (cwlogs-eu), `logs:opensearch` (opensearch-eu), `traces:traceql` (tempo-eu, tempo-us), `metrics:datadog` (prod-us payments), `log-delete: query` (loki-eu).
- **Mounts**: the platform's default telemetry devices per topology, under `/metrics/<region>`, `/logs/<region>` and `/traces/<region>`; `/logs/eu-rds` and `/logs/eu-legacy` mounted by the platform for `sre`.
- **Annotations**: orders-api's image declares `slo: error_rate = sum(rate(http_requests_total{unit="orders-api",code=~"5.."}[5m])) / sum(rate(http_requests_total{unit="orders-api"}[5m]))`.
- **Platform defaults** `explain` must print: the default device per topology and class, each device's residency, tenant model and settle window, `telemetry_query_max`, the price shape per device, the retention of each store.

## 4. Main path

```bash
# 02:12, Toronto, session in residency us
[ana] $ cat /logs/eu/namespace=orders/unit=orders-api --since=-30m | grep -F timeout | head
refused (exit 3): log device loki-eu has residency eu; this session is us
[ana] $ attach --residency=eu                         # a second session, hosted in eu
[ana@eu] $ cat /logs/eu/namespace=orders/unit=orders-api --since=-30m | grep -F timeout | head
refused (exit 3): loki-eu is label-enforced; group sre owns no namespace under /clusters/prod-eu   (L26.5)
#  the only route is sudo to svc:platform, gated; the platform lead approves at 02:31
#  terminal output of the eu session is on a laptop in Toronto; nothing in the design says
#  whether that is allowed                                                                         (L26.4)

[ana@eu] $ ps | grep orders-api
orders-api  1.4.2 90% / 1.5.0 10%  canary  wait --slo=error_rate<0.5%: 0.31%  passing
#  the condition sums both versions; the canary's own 5xx rate is 4.2%; and 503s the gateway
#  returns for a crash-looping canary pod are labelled upstream=none, not unit=orders-api;
#  the platform's Pyrra SLO orders-api-availability, counted at the gateway, is breaching   (L26.1, L26.13)
#  at 02:30 kai's token expires; the condition reads unknown and the canary stays at 10%
#  until the 6 h timeout                                                                           (L26.16)

[ana@eu] $ cat /metrics/eu/http_request_duration_seconds_bucket --since=-24h --step=15s | head
#  the estimate ran count() over 2.1 M series for 40 s and tripped mimir-eu's per-tenant
#  query limit, so every team's dashboards failed for a minute                                  (L26.8)

[ana@eu] $ cat /logs/eu-legacy/service=payments --since=-1h | grep -F Deadline | wc -l
0
#  opensearch analyses text: the pushed-down match for "Deadline" never matches the token
#  "deadlineexceedederror", so the store returned nothing and grep had nothing to print      (L26.9)

[ana@eu] $ cat /logs/eu-rds/orders-db --since=-7d | grep -F 'lock timeout' | wc -l
estimate: unknown (cwlogs-eu reports stored bytes per log group, not per range)
2214                                      # 1.3 TB scanned, $6.50
#  the assisted shell proposes the same line for the other five databases; ana accepts all: $39  (L26.7)

[ana@eu] $ cat /traces/eu/id=4bf9e1… | head -50
41 spans; 1.8 s between orders-api's call and its response from billing-api
#  billing-api runs in prod-us; its spans are in tempo-us, which an eu session may not read;
#  the read returned what tempo-eu had, exit 0, and the gap looks like network latency      (L26.10)
[ana@eu] $ cat /traces/eu/id=4bf9e1… | grep -F authorization
"http.request.header.authorization": "Bearer eyJhbGciOi…"
#  printed to the terminal and written to the session journal                                    (L26.14)

[ana@eu] $ cat /logs/eu/namespace=orders --since=-2h | grep -F refund_id > /artifacts/inc-2231/refunds.txt
#  42 MiB of lines with customer e-mail addresses, artifact retention 90 days; loki-eu keeps 14,
#  and a scrub of loki-eu does not reach the copy                                               (L26.6)

[ana@eu] $ cat /proc/7701/fd/2 | tail -20
#  7701 is the nightly refund reconciliation, 1 648 partitions, failed at 02:05; the lines of
#  1 648 pods arrive interleaved with no partition or attempt                                     (L26.12)

[raj@us] $ top -g payments
UNIT          CPU   MEM   RPS
payments-api  ?     ?     ?            # Datadog drops convoy.io/* labels by its allowlist     (L26.11)
#  fourteen people open top during the incident; each refresh of each column of each session is
#  one query to mimir-eu: 7 000 queries a minute                                                   (L26.15)

# 03:00: compare the edge site
[ana@eu] $ cat /metrics/eu/factory/line_throughput/site=factory-3 --since=-3h --until=-1h > /artifacts/inc-2231/f3.tsv
#  mimir-eu's settle window is 30 min, so the read is settled and cached; factory-3's collector
#  ships two buffered hours at 05:00 inside the tenant's six-hour out-of-order window            (L26.2)

# Thursday: the postmortem
[ana] $ replay 5120 --verify          # the same pattern, 60 days ago
replay ok
#  the original read raw 15 s samples from mimir-eu; the replay read thanos-lt's 5 min
#  downsampled series; p99 differs by 30%, and "ok" says nothing about it                        (L26.3)
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| a bounded read | a read of a telemetry device | a PromQL, LogQL, TraceQL, CloudWatch Insights or OpenSearch query | the store; the artifact when cached or redirected |
| a condition | a named condition evaluated by `wait` | repeated queries to the default metrics device | the process receipt |
| break-glass | an approval on `sudo` | the platform group's store credential | `/sys/proc/<pid>/approval/<n>` |
| live columns | `top`, `ps` | queries per refresh | none |
| `cat /proc/<pid>/fd/2` | stderr across the pods' lives | pod logs, then the log device | the store, within its retention |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe: staged artifact, content id, cache | ● | a settled read that was not settled (L26.2); an artifact copied from a log store (L26.6) | AC-26.2, AC-26.6 |
| A2 | Command: class, annotation, executable fields | ○ | the image's `slo:` | |
| A3 | Session: mount table, env, cwd, attach and detach | ○ | a second session in another residency; live columns per session | |
| A4 | Backend selection | ○ | text tools over rows a store returns | |
| A5 | Completion contract | ● | a canary condition that passes on a diluted rate, and one that reads unknown after a token expires (L26.1, L26.16) | AC-26.1, AC-26.16 |
| A6 | Device tree | ● | seven telemetry devices with different tenant models and dialects (L26.5, L26.9) | AC-26.5, AC-26.9 |
| A7 | Signals and lifecycle | – | | |
| A8 | Secrets as references | ● | a bearer token in a span (L26.14) | AC-26.14 |
| A9 | Cost | ● | an unestimable scan, an estimate that is itself a heavy query, live columns (L26.7, L26.8, L26.15) | AC-26.7, AC-26.8, AC-26.15 |
| A10 | Identity and authorization | ● | an on-call group that owns nothing; a waiting process whose author's token expired (L26.5, L26.16) | AC-26.5, AC-26.16 |
| A11 | Platform defaults and `caps` | ● | settle windows, pushdown kinds, label keys a store keeps (L26.2, L26.9, L26.11) | AC-26.11 |
| A12 | Storage | ○ | store retention against artifact retention | |
| A13 | Networking | ○ | gateway-generated responses; a cross-region call | |
| A14 | Time and queueing | ● | relative ranges, settle windows, out-of-order ingestion, downsampling (L26.2, L26.3) | AC-26.3 |
| A15 | Audit, receipts, retention and `replay` | ● | a replay that read different data (L26.3); the journal holding a token (L26.14) | AC-26.3 |
| A16 | Durability | ○ | a store in another residency treated as missing | |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative verb, declarative owner | the image's `slo: error_rate` and the platform's Pyrra SLO of the same name disagree | decision 5 lets a unit declare its own conditions; decision 6 leaves SLO objects to the observability repository | **L26.13** |
| P2 | POSIX semantics that cannot hold | `cat /proc/<pid>/fd/2` of a process with 1 648 pods | decision 7 makes stderr one path across "the pod's" life; a process has many pods and attempts | **L26.12** |
| P3 | Orphaned or lost state | incident log lines redirected into an artifact | the artifact's retention class, not the store's | folded into **L26.6** |
| P4 | Identity leak or escalation | a span attribute holding a bearer token, read to the terminal and the journal | the redactor (rule L21.2) covers what is typed, not what a read returns | **L26.14** |
| P5 | Invisible cost | (a) a store that cannot estimate a range; (b) an estimate that is a heavy query; (c) live columns from many sessions | decision 9 estimates "from the store's own statistics" and assumes they exist and are cheap; live columns are not mentioned | **L26.7**, **L26.8**, **L26.15** |
| P6 | Lying cache | (a) a tenant whose out-of-order window exceeds the store's settle window; (b) a replay across a downsampling boundary; (c) a pushed-down filter that drops lines `grep` would keep | (a) decision 3 reads the settle window per store; (b) decision 3 promises the same data "while the store retains it"; (c) decision 10 assumes the store's filter is a substring match | **L26.2**, **L26.3**, **L26.9** |
| P7 | Silent failure | a canary condition summed over both versions and blind to gateway-generated 5xx passes at 0.31% while the canary fails at 4.2% | decision 5 evaluates the declared selector as written | **L26.1** |
| P8 | Partial fleet | a trace whose spans are split across two stores, one of them unreadable from this residency | decision 8 refuses the cross-residency read; nothing marks the result partial | **L26.10** |
| P9 | Created elsewhere | a collector that drops `convoy.io/*` labels by its allowlist | decision 6 assumes the platform's collectors keep them | **L26.11** |
| P10 | Multi-party action | the on-call group must read logs of namespaces it does not own, at night | decision 8 gives a label-enforced store to owners and the platform group; the rest is `sudo` with an approval | **L26.5** |
| P11 | Data at rest | (a) log lines copied to an artifact outlive the store and its scrub; (b) an eu session's output is read on a laptop in Toronto | (a) the artifact store's own retention; (b) residency binds where a session runs | **L26.6**, **L26.4** |
| P12 | Process-tree inheritance | a waiting canary condition whose author's login has expired | the process reads under its author's identity (ADR 0016) | **L26.16** |

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| canary condition over both versions | plan (L26.1) | 3 | the selector and the version label it lacks | `--slo` on the canary's series |
| a settled read that receives late data | the tenant's window (L26.2) | none; the read is not cached | the settle state per tenant | rerun later |
| replay across a downsampling boundary | `replay --verify` (L26.3) | 1 | resolution then and now | `--hold` the original read |
| on-call has no read grant | plan (L26.5) | 3 | the device, the readers group, how to join it | the platform's `telemetry_readers` |
| artifact outliving its source | put (L26.6) | 0 | retention capped to the source's | `--hold` with a reason |
| unestimable scan | plan (L26.7) | 3 above `telemetry_query_max` | the upper bound used | a narrower range or selector |
| pushdown on an analysed store | plan (L26.9) | none | `narrowed: no (token store)` | none needed |
| partial trace | the read (L26.10) | 0 degraded | the stores missing | `attach --residency` |
| collector without Convoy labels | `caps` (L26.11) | none | `joined by pod name` | the platform's label allowlist |
| a secret in a read | the redactor (L26.14) | none | count of redactions, store notified | `secret scrub` |
| condition reader without a token | the controller (L26.16) | none | read as `svc:<group>` | none |

## 8. Acceptance criteria

- **AC-26.1** Given `release --canary` with a condition, then the condition is evaluated on the canary version's series only (by the version or ReplicaSet label the store keeps), the gateway's own responses for the route are included, and a store that lacks the version label refuses the plan, naming the label.
- **AC-26.2** Given a read from a store whose tenants have their own out-of-order windows, then the settle state uses the tenant's window, and a range inside it is never cached.
- **AC-26.3** Given `replay --verify` of a telemetry read, then the receipt's recorded resolution and source are compared with what the store would return now, and a difference fails the verify, naming both.
- **AC-26.4** Given a session in residency R, then `explain` of a telemetry read states that output reaches the terminal wherever the user is, and the platform's access policy, not residency, governs who may read it.
- **AC-26.5** Given a device with `telemetry_readers=<group>`, then members read every tenant or namespace the device covers, each read is audited with the selector, and `mount` by anyone else is refused naming the group.
- **AC-26.6** Given a redirect of a telemetry read into the artifact store, then the artifact's retention is at most the source store's, unless held, and the artifact records its source store and selector so a scrub of the store reaches it.
- **AC-26.7** Given a store that cannot estimate a range, then the plan uses the upper bound (the stored bytes of the selected streams), binds `telemetry_query_max` and `--budget` to it, and the receipt records the actual.
- **AC-26.8** Given an estimate, then it is computed from the store's statistics or index endpoints only, never from a query over the data, and a selector the store cannot estimate cheaply needs a narrower selector.
- **AC-26.9** Given a log driver, then it reports `pushdown: substring|token|none`, only `substring` stores narrow by `grep -F`, and `explain` prints whether the read was narrowed.
- **AC-26.10** Given a read whose result spans stores (a trace across regions), then stores that could not be read are named and the result is degraded (exit 0 with the flag).
- **AC-26.11** Given a store whose collector drops `convoy.io/*` labels, then `caps: label-keys` says so, joins fall back to pod names from the inventory, and columns from that store are marked `joined by pod`.
- **AC-26.12** Given a process with several pods or attempts, then `/proc/<pid>/fd/2` is ordered by time with an `[index.attempt]` prefix, and `/proc/<pid>/task/<index>/fd/2` reads one partition.
- **AC-26.13** Given a platform SLO object for a unit, then `slo:` may refer to it by name, and a unit condition that shadows its name is refused.
- **AC-26.14** Given a telemetry read, then its output passes the session's redactor before the terminal, the journal and any artifact, and matches are counted and reported to the store's owning group.
- **AC-26.15** Given live columns, then the controller queries each store once per interval per selector for all sessions, and the cost is charged to `/sys/cost/session/telemetry` of the sessions that watched.
- **AC-26.16** Given a process waiting on a condition, then its reads run as `svc:<group>` with the group's telemetry grant once the author's token expires, and the receipt records the switch.

## 9. Persona lenses

- **SRE**: wants to read every store the incident touches without a break-glass approval at 2 a.m., and every partial or unknown result said out loud.
- **DO**: wants a canary condition to measure the canary, and the same definition of availability as the platform's SLO.
- **SEC**: wants tokens in spans redacted before any screen or journal, incident copies of logs to die with the logs, and the residency claim stated precisely.
- **FIN**: wants unestimable scans bound by their worst case, and fourteen people watching `top` to cost one query, not fourteen.
- **PE**: wants an estimate never to be the query that takes the store down, and the stores' differences (settle windows, analysers, label allowlists) in `caps` rather than in incidents.
- **DEV**: wants the `slo:` they wrote in the image to mean what the platform's SLO means.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L26.1 | P7 | A canary condition evaluates the unit's selector as declared, so it sums both versions and dilutes the canary's error rate tenfold at a 10% weight; and 503s the gateway itself returns for an unhealthy canary pod carry no unit label. The canary passed at 0.31% while failing at 4.2%. | **amend** (applied: ADR 0023 d5 revised) ADR 0023 decision 5: under `release --canary`, a condition is evaluated on the canary version's series only, selected by the version or ReplicaSet label the store keeps; the gateway default includes responses the gateway generated for the route; a store that lacks the version label refuses the plan, naming the label | ADR 0023 d5; ADR 0013 d7; ADR 0019 |
| L26.2 | P6 | The settle window is read per store, but a tenant may have its own out-of-order window: the edge tenant's six hours against the store's thirty minutes. A read was cached as settled and the site's buffered samples arrived after it. | **amend** (applied: ADR 0023 d3 revised) ADR 0023 decision 3: the settle state is computed from the tenant's or stream's window where the store has one, from the longest window the device reports where the tenant is not known, and from the edge collector's buffered range (decision 12) where one is reported | ADR 0023 d3, d12; ADR 0001 a1 |
| L26.3 | P6 | Decision 3 promises the same data "while the store retains it", but long-term stores downsample. A replay 60 days later read 5-minute aggregates instead of 15-second samples, and `--verify` passed. | **rule** (applied: ADR 0023 rules): a read's receipt records the source store and the resolution; `replay --verify` fails when the store would now answer from another source or resolution, naming both; `--hold` on a telemetry read stores the result as an artifact, which is what a later replay reads | ADR 0023 d3; ADR 0006 a2; ADR 0011 |
| L26.4 | P11 | Residency binds where a session runs and stores, but its terminal output is read wherever the user is: an eu session's log lines are displayed in Toronto. | **accepted**: residency governs processing and storage, not the location of the person reading; who may read is the platform's access policy (groups, `telemetry_readers`); `explain` of a read from a residency-bound device says so | ADR 0016 d11; ADR 0020; ADR 0005 |
| L26.5 | P10 | On a label-enforced store the readers are the owning groups and the platform group, so the on-call group, which owns nothing, reaches logs only through a gated `sudo` in the middle of the night. | **amend** (applied: ADR 0023 d8 revised) ADR 0023 decision 8: a device may name `telemetry_readers=<group>…`, groups that read every tenant or namespace the device covers without owning them; every read by a reader is audited with its selector and visible to the owning groups; the grant is a platform default, listed by `ls -l /dev/<class>/` | ADR 0023 d8; ADR 0016 d3, d6; ADR 0017 |
| L26.6 | P11, P3 | A telemetry read redirected into the artifact store takes the artifact's retention, so incident copies of log lines with personal data outlive the store's retention, and a scrub of the store does not reach them. | **rule** (applied: ADR 0023 rules; ADR 0007 rules): an artifact written from a telemetry read records its source store and selector, takes a retention no longer than the source's unless a hold with a reason covers it, and is reached by `secret scrub` of the source through the reverse lineage index | ADR 0023 d3; ADR 0007 a2, a5; ADR 0006 |
| L26.7 | P5 | A store that cannot estimate a range (CloudWatch Logs reports bytes per log group) runs the scan unestimated, and an accepted assisted proposal multiplies it six times. | **rule** (applied: ADR 0023 rules): with no range estimate the band is the upper bound, the stored bytes of the selected streams; `telemetry_query_max` and `--budget` bind to it (README principle 9); the receipt records actual; an assisted proposal that repeats a priced read shows the total before it is accepted (ADR 0014 decision 6) | ADR 0023 d9; ADR 0014 d6 |
| L26.8 | P5 | The estimate of decision 9 may be "a `count` query", which on a high-cardinality metric is itself the heavy query: it hit the tenant's query limit and failed every team's dashboards. | **rule** (applied: ADR 0023 rules): estimates use only the store's statistics and index endpoints (cardinality, label values, stream statistics), never a query over the data; a selector those endpoints cannot bound needs a label the store reports as indexed, and the plan says which | ADR 0023 d9 |
| L26.9 | P6 | `grep -F` pushed down to a full-text store becomes a token match, which drops lines a substring match keeps; the real `grep` then has nothing to print, and the result differs from Linux. | **amend** (applied: ADR 0023 d10 revised) ADR 0023 decision 10: a log driver reports `pushdown: substring\|token\|none`; only `substring` stores narrow by `grep -F`; a `token` store narrows only on a keyword field with a wildcard, or not at all; `explain` prints whether the read was narrowed | ADR 0023 d10; ADR 0001; ADR 0002 |
| L26.10 | P8 | A trace whose spans are in two stores returns what the readable store holds with exit 0; the missing spans look like latency. | **rule** (applied: ADR 0023 rules; ADR 0004 rules): a read whose result spans stores names the stores it could not read (residency, unreachable, no grant) and is degraded (exit 0 with the flag); a trace read lists the services whose spans are missing | ADR 0023 d8, d12; ADR 0004 a2 |
| L26.11 | P9 | Decision 6 assumes collectors keep `convoy.io/*` labels; a vendor agent with a label allowlist drops them, so `top` reads `?` for a team's units. | **rule** (applied: ADR 0023 rules; ADR 0018 rules): `caps: label-keys` lists the Convoy labels a store keeps; where they are missing, joins fall back to pod names from the inventory, columns from that store are marked `joined by pod`, and the platform is told which allowlist lacks them | ADR 0023 d6; ADR 0018 |
| L26.12 | P2 | A process has many pods and attempts; "stderr across the pod's life" interleaves 1 648 partitions with no way to tell them apart. | **amend** (applied: ADR 0023 d7 revised) ADR 0023 decision 7: `/proc/<pid>/fd/2` is the time-ordered merge of every pod and attempt of the process, each line prefixed `[index.attempt]`; `/proc/<pid>/task/<index>/fd/2` reads one partition, as `ps -T` lists them | ADR 0023 d7; ADR 0009; ADR 0013 d5 |
| L26.13 | P1 | The image's `slo: error_rate` and the platform's SLO object of the same purpose disagree; the canary passes while the platform's SLO is breaching. | **rule** (applied: ADR 0023 rules): where the platform has SLO objects (OpenSLO, Pyrra, Sloth) for a unit, `slo:` may name one (`slo: availability = platform:orders-api-availability`); a unit condition that shadows a platform SLO's name is refused; `explain` prints both definitions where a unit declares its own | ADR 0023 d5, d6; ADR 0010 a1 |
| L26.14 | P4 | A read returns whatever the store holds, and spans and logs hold bearer tokens; the redactor of rule L21.2 covers what is typed, not what a read prints, so the token reached the terminal and the journal. | **rule** (applied: ADR 0023 rules; ADR 0020 rules): telemetry reads pass the session's redactor before the terminal, the journal and any artifact; matches are counted in the receipt and reported to the store's owning group with the selector, so the store can be scrubbed (ADR 0007 amendment 5) | ADR 0023 d2; ADR 0020 L21.2; ADR 0007 a5 |
| L26.15 | P5 | Live columns (`top`, `ps`, `netstat`) query the store per refresh, per column, per session; fourteen watchers became 7 000 queries a minute, priced nowhere. | **rule** (applied: ADR 0023 rules; ADR 0013 rules): live columns come from a per-store cache in the controller, one query per interval per selector for all sessions; the cost is shared among the watching sessions under `/sys/cost/session/telemetry` and `top` shows the interval and age | ADR 0023 d4, d9; ADR 0013 d5; ADR 0020 |
| L26.16 | P12 | A waiting process reads its condition under its author's identity; when the author's token expires, every read is `unknown`, and the canary holds 10% of traffic until the timeout. | **rule** (applied: ADR 0023 rules; ADR 0016 rules): conditions of a process that outlives its author's login are read as `svc:<group>` with the group's telemetry grant, the switch recorded in the receipt; `unknown` for longer than the condition's `unknown_max` (default 15 minutes) rolls a canary back rather than holding it | ADR 0023 d4, d5; ADR 0016 d5; ADR 0013 d7 |
