# Convoy — POSIX command map

**Status:** design reference. **Source list:** Wikipedia, *List of POSIX commands*, 155 utilities as of the revision fetched 2026-11-08 (IEEE Std 1003.1-2024 set). Mapping, approach, feasibility and pull are Convoy judgements, not Wikipedia content.

**Companion documents:** `pipeline-compiler.md` (how `|`, `xargs`, `for` and `tee` compile), `debate-review.md` (signals, exit codes, secrets, scheduling, cost, packages).

## Legend

| Column | Values | Meaning |
|---|---|---|
| Approach | **toolbox** | Runs unchanged inside the toolbox image, over partitions when its annotation class allows. The binary is never translated |
| | **translate** | Compiles to Kubernetes API objects or calls; the command keeps its POSIX name and flags where they make sense |
| | **mount** | Operates on the session mount namespace (`/pvc`, `/s3`, `/clusters`, `/artifacts`) through the session fileserver; cross-store operations become Jobs |
| | **session** | Runs only in the session pod; no cluster meaning beyond it |
| | **drop** | Not shipped. Legacy subsystem with no cloud meaning |
| Feasibility | **Native** | Works today with no semantic change |
| | **Mapped** | Clean translation; semantics preserved or extended, documented in `explain` |
| | **Partial** | Translation exists but semantics differ; the difference is surfaced, never hidden |
| | **Dropped** | Deliberately absent |
| Pull | High / Medium / Low / None | Demand for the *cluster-level* behaviour, judged from the batch, ML and platform use cases |

## Summary

| Approach | Count | | Feasibility | Count | | Pull | Count |
|---|---|---|---|---|---|---|---|
| toolbox | 54 | | Native | 79 | | High | 30 |
| translate | 39 | | Mapped | 38 | | Medium | 27 |
| mount | 18 | | Partial | 26 | | Low | 58 |
| session | 32 | | Dropped | 12 | | None | 40 |
| drop | 12 | |  |  | |  |  |

**30 commands carry high pull:** `awk`, `batch`, `cat`, `cd`, `cp`, `cron`, `cut`, `df`, `env`, `find`, `grep`, `head`, `id`, `jobs`, `kill`, `ls`, `ps`, `pwd`, `rm`, `sed`, `sh`, `sort`, `tail`, `time`, `timeout`, `ulimit`, `uniq`, `wait`, `wc`, `xargs`.

**Findings that change the design:**

- **`batch` is already the verb for the batch wedge.** POSIX defines it as "schedule commands to be executed in a batch queue". It maps one-to-one onto a Job admitted through a Kueue queue. No new `fork` spelling is needed for the queued case; `fork N` remains for explicit fan-out.
- **`xargs` is the fan-out primitive.** `-P` is parallelism, `-n` is partition size. It compiles to an Indexed Job or an Argo fan-out, and it is the reason a linear pipe becomes a DAG.
- **`make` is the pipeline compiler's ancestor.** Rules are stages, targets are artifacts, mtime caching becomes content-addressed caching. A Makefile can be accepted as a plan.
- **`time`, `timeout`, `wait`, `ulimit` are the completion contract.** `wait` returns the contract exit code, `timeout` sets the deadline, `ulimit` sets requests and limits, `time` prints the receipt.
- **`nohup` is the promotion verb.** Remote processes are already immune to hangup; `nohup` on a local-backend command is how a small command survives the session by becoming a Job.
- **`fuser` is a safety dependency.** `umount` and `rmvol` must refuse while a volume is in use; `fuser` is the index that makes that refusal possible.
- **`type` is the learning surface.** `type sort` prints the annotation class, so users discover what parallelizes without reading this document.
- **Text processing is untouched.** 50+ text and file utilities run as the real binaries over partitions. None is translated into an engine operator.

## Full table

### SCCS

| Command | POSIX | Kubernetes backing | Approach | Feasibility | Pull | Notes |
|---|---|---|---|---|---|---|
| `admin` | Opt (XSI) | none (SCCS) | drop | Dropped | None | Version control is Git via GitOps; SCCS family is not shipped |
| `delta` | Opt (XSI) | none (SCCS) | drop | Dropped | None |  |
| `get` | Opt (XSI) | none (SCCS) | drop | Dropped | None |  |
| `prs` | Opt (XSI) | none (SCCS) | drop | Dropped | None |  |
| `rmdel` | Opt (XSI) | none (SCCS) | drop | Dropped | None |  |
| `sact` | Opt (XSI) | none (SCCS) | drop | Dropped | None |  |
| `sccs` | Opt (XSI) | none (SCCS) | drop | Dropped | None |  |
| `unget` | Opt (XSI) | none (SCCS) | drop | Dropped | None |  |
| `val` | Opt (XSI) | none (SCCS) | drop | Dropped | None |  |
| `what` | Opt (XSI) | none (SCCS) | drop | Dropped | None |  |

### Misc

| Command | POSIX | Kubernetes backing | Approach | Feasibility | Pull | Notes |
|---|---|---|---|---|---|---|
| `alias` | Mand | session profile on home volume | session | Native | Low | Aliases persist in the session pod; shared aliases via a team profile package |
| `ar` | Mand | toolbox | toolbox | Native | None | C toolchain only; non-parallel |
| `bc` | Mand | toolbox | toolbox | Native | Low | session or toolbox |
| `cal` | Opt (XSI) | session | session | Native | None |  |
| `cron` | Mand | CronJob with timeZone and concurrencyPolicy | translate | Mapped | High | crontab -e edits the user's CronJobs; missed-run and overlap policy exposed as flags |
| `date` | Mand | session clock | session | Native | Low | date -s is refused: there is no system clock to set |
| `env` | Mand | session env table → env: entries in the pod spec | translate | Mapped | High | Captured at launch; no ConfigMap per export. Secrets are references, never env values |
| `fc` | Opt (UP) | audit stream query | translate | Mapped | Medium | History is cross-session because the session writes the audit stream |
| `gencat` | Mand | toolbox | toolbox | Native | None |  |
| `getconf` | Mand | session limits in getconf form | session | Partial | Low | Reports quota, max partitions, artifact size limits alongside POSIX values |
| `gettext` | Mand | toolbox | toolbox | Native | None |  |
| `grep` | Mand | toolbox | toolbox | Native | High | stateless; limit pushdown; never translated to filter() |
| `hash` | Mand | session | session | Native | None |  |
| `id` | Mand | OIDC identity, groups, service account, namespace | translate | Mapped | High | Shows who you are to the cluster, not to the pod |
| `ipcrm` | Opt (XSI) | delete Lease / named queue | translate | Partial | Low |  |
| `ipcs` | Opt (XSI) | list Leases and named queues | translate | Partial | Low | Locks and FIFOs are the IPC objects that exist |
| `locale` | Mand | session; toolbox pins LC_ALL=C | session | Native | Low | Collation must be fixed or sort/merge results differ across nodes |
| `localedef` | Mand | session | session | Native | None |  |
| `logname` | Mand | OIDC login | translate | Mapped | Low |  |
| `m4` | Mand | toolbox | toolbox | Native | None |  |
| `mailx` | Mand | platform notifier (SES/SNS) | translate | Partial | Low | Outbound notification only |
| `man` | Mand | session, capability-aware pages | session | Native | Medium | A page for a verb the cluster lacks says so |
| `mesg` | Mand | allow/deny collaborative attach to my session | session | Partial | Low |  |
| `msgfmt` | Mand | toolbox | toolbox | Native | None |  |
| `newgrp` | Mand | switch active team → namespace context + service account | translate | Mapped | Low |  |
| `ngettext` | Mand | toolbox | toolbox | Native | None |  |
| `od` | Mand | toolbox | toolbox | Native | Low | offsets are global: non-parallel |
| `pax` | Mand | archive volume → artifact via Job; VolumeSnapshot preferred | mount | Mapped | Medium | pax -w /pvc/data > /s3/backups/x is a Job; snapshot is the better backup for consistency |
| `split` | Mand | toolbox; the planner's own partitioner | toolbox | Native | Medium | split -n N on an artifact is explicit partitioning |
| `stty` | Mand | pty on attach; resize channel | session | Native | Low |  |
| `tabs` | Mand | session terminal | session | Native | None |  |
| `talk` | Opt (UP) | none | drop | Dropped | None | Collaborative sessions cover the use case |
| `tput` | Mand | session terminal | session | Native | Low |  |
| `tty` | Mand | session terminal | session | Native | Low |  |
| `type` | Opt (XSI) | session: builtin, toolbox command with class, translate verb or package | session | Mapped | Medium | type grep prints "toolbox, stateless, limit pushdown"; this is how users learn what parallelizes |
| `ulimit` | Opt (XSI) | requests and limits for subsequent processes; reported from LimitRange and quota | translate | Mapped | High | ulimit -m 2G sets the memory limit of the next launches; ulimit -a shows quota headroom and VPA suggestions |
| `umask` | Mand | default mode on PVC files; default ownership labels on objects | mount | Partial | Low |  |
| `unalias` | Mand | session | session | Native | None |  |
| `uname` | Mand | cluster name, Kubernetes version, node OS/arch mix, Convoy version | translate | Mapped | Low |  |
| `uncompress` | Opt (XSI) | toolbox | toolbox | Native | Low | see compress |
| `write` | Mand | message to another session's terminal | session | Partial | Low | Collaborative sessions cover the richer case |

### Text processing

| Command | POSIX | Kubernetes backing | Approach | Feasibility | Pull | Notes |
|---|---|---|---|---|---|---|
| `asa` | Opt (FR) | toolbox | toolbox | Native | None | stateless |
| `awk` | Mand | toolbox | toolbox | Native | High | pattern class: stateless if no BEGIN/END, variables, getline or NR; otherwise non-parallel. Never translated to DataFrame ops |
| `comm` | Mand | toolbox | toolbox | Native | Low | fan-in; requires sorted inputs; parallel only if range-partitioned |
| `csplit` | Mand | toolbox | toolbox | Native | Low | context dependent; non-parallel |
| `cut` | Mand | toolbox | toolbox | Native | High | stateless |
| `diff` | Mand | toolbox; kubectl diff for manifests | toolbox | Native | Medium | Files: non-parallel. Plans and manifests: dry-run vs live object diff |
| `ed` | Mand | toolbox on fileserver | toolbox | Native | None | non-parallel |
| `ex` | Opt (UP) | editor in the session pod over mounts | session | Native | Medium | Editing configs on /pvc and ConfigMap-backed /etc |
| `expand` | Mand | toolbox | toolbox | Native | Low | stateless |
| `fold` | Mand | toolbox | toolbox | Native | Low | stateless |
| `head` | Mand | toolbox | toolbox | Native | High | limit class: cancels upstream partitions (the SIGPIPE equivalent) |
| `iconv` | Mand | toolbox | toolbox | Native | Low | stateless on line boundaries; multibyte splits need record-aware partitioning |
| `join` | Mand | toolbox | toolbox | Native | Medium | fan-in; requires sorted; parallel only when both inputs are partitioned by the join key |
| `lp` | Mand | none | drop | Dropped | None | No printer; export to an artifact instead |
| `more` | Opt (UP) | session pager | session | Native | Low |  |
| `nl` | Opt (XSI) | toolbox | toolbox | Native | Low | needs global offsets: two-pass aggregate or non-parallel |
| `paste` | Mand | toolbox | toolbox | Native | Low | fan-in, line-aligned; non-parallel unless partitions align |
| `patch` | Mand | toolbox; strategic-merge patch on manifests | toolbox | Partial | Low |  |
| `pr` | Mand | toolbox | toolbox | Native | None | page numbers are global: non-parallel |
| `sed` | Mand | toolbox | toolbox | Native | High | pattern class: stateless without hold space, N/D/P or line-range addresses; otherwise non-parallel |
| `sort` | Mand | toolbox | toolbox | Native | High | aggregate: per-partition sort then sort -m; top-K rewrite with head |
| `tail` | Mand | toolbox; log/stream follow for -f | toolbox | Partial | High | tail -n: per-partition tail then merge. tail -f: opt-in stream on a log or queue device, never an artifact; on a topic it resumes from the committed offset (ADR 0012) |
| `tr` | Mand | toolbox | toolbox | Native | Medium | stateless |
| `tsort` | Mand | toolbox; used internally for plan ordering | toolbox | Native | Low |  |
| `unexpand` | Mand | toolbox | toolbox | Native | Low | stateless |
| `uniq` | Mand | toolbox | toolbox | Native | High | aggregate: requires sorted input; -c merges with sum-by-key |
| `vi` | Opt (UP) | editor in the session pod over mounts | session | Native | Medium | see ex |
| `wc` | Mand | toolbox | toolbox | Native | High | aggregate: partial counts then sum |
| `zcat` | Opt (XSI) | toolbox | toolbox | Native | Low | splittable only for bgzip-style block compression |

### Process management

| Command | POSIX | Kubernetes backing | Approach | Feasibility | Pull | Notes |
|---|---|---|---|---|---|---|
| `at` | Mand | Job with suspend:true released by a session timer | translate | Mapped | Medium | Kubernetes has no start-at; the session flips suspend, or emits a one-shot CronJob that deletes itself |
| `batch` | Mand | Job admitted through a Kueue LocalQueue | translate | Mapped | High | The POSIX verb for "run when load permits" is exactly a queued Job; the batch wedge needs no new verb |
| `bg` | Opt (UP) | detach TTY from the attached pod | translate | Mapped | Medium | The process is already an object and keeps running; bg only detaches. Never converts a pod into a Deployment |
| `fg` | Opt (UP) | attach TTY to the running pod (WebSocket attach) | translate | Mapped | Medium | Only pods created with stdin/tty can be reattached; the plan records that |
| `fuser` | Opt (XSI) | volume → pods index | translate | Mapped | Medium | Which processes mount a volume; required by umount and rmvol to refuse while in use |
| `jobs` | Opt (UP) | process table scoped to the session | translate | Mapped | High | ps filtered by session label |
| `kill` | Mand | signal routing table by backing object | translate | Mapped | High | TERM scales/suspends, KILL deletes, STOP freezes the cgroup, CONT unfreezes, HUP restarts; data is never deleted by kill |
| `nice` | Mand | PriorityClass tier / Kueue workload priority at launch | translate | Mapped | Medium | Interactive outranks pipelines outranks batch; batch never preempts batch |
| `nohup` | Mand | no-op for remote processes; promotes a local-backend command to a Job | translate | Mapped | Medium | Remote processes are already immune to hangup; nohup is how a small local command survives the session |
| `ps` | Mand | process table from labelled Pods, Jobs, Deployments, CronJobs, Workflows | translate | Mapped | High | Columns TYPE, REPLICAS, GPUS, COST; one row per process however many pods back it |
| `renice` | Mand | Kueue workload priority (mutable); rollout for Deployments | translate | Partial | Medium | Pod priority is immutable; queued workloads re-prioritize in place, running ones only via restart |
| `time` | Mand | the receipt | translate | Mapped | High | time cmd prints wall, CPU from metrics, partitions, cache hits and cost |
| `timeout` | Mand | activeDeadlineSeconds / contract deadline | translate | Mapped | High | Exit code 2 on expiry; the deadline is part of the completion contract |
| `uux` | Opt (UU) | run in another cluster context | translate | Mapped | Low | cd /clusters/b && run cmd; uux is the POSIX spelling of remote execution |
| `wait` | Mand | contract evaluation; kubectl wait conditions | translate | Mapped | High | Returns the contract exit code: 0, 0 degraded, 1 floor unreachable, 2 deadline/budget, 3 policy |

### Filesystem

| Command | POSIX | Kubernetes backing | Approach | Feasibility | Pull | Notes |
|---|---|---|---|---|---|---|
| `basename` | Mand | session / toolbox | toolbox | Native | Low | stateless |
| `cat` | Mand | toolbox; fileserver for mounts | toolbox | Native | High | fan-in concat; reading /pvc or /s3 goes through the session fileserver; on a topic it is a bounded read of the log-end offsets at plan time, one partition per topic partition (ADR 0012) |
| `cd` | Mand | session cwd = (cluster, namespace, path) | translate | Mapped | High | A kubeconfig context switch plus namespace; /clusters tree from the fleet inventory |
| `chgrp` | Mand | owner labels + admission policy; fsGroup on volumes | translate | Partial | Medium | Objects: convoy.io/group label checked by a CEL admission policy. PVC files: fsGroup. S3: no groups |
| `chmod` | Mand | POSIX bits on PVC via fileserver; none on S3 | mount | Partial | Low | Object storage has no mode bits; sharing an artifact is a bucket policy or presigned URL |
| `chown` | Mand | owner label transfer with admission check; chown Job on volumes | translate | Partial | Medium | Transferring a process requires the new owner to accept; recursive chown on large volumes is slow, use OnRootMismatch |
| `cksum` | Mand | artifact manifest hash; toolbox for streams | toolbox | Native | Low | Artifacts already carry sha256; a CRC over a stream is non-parallel |
| `cmp` | Mand | toolbox | toolbox | Native | Low | two inputs; parallel only on aligned partitions |
| `compress` | Opt (XSI) | toolbox; artifacts compressed transparently | toolbox | Native | Low | whole-stream compression is non-splittable; the artifact store compresses at rest anyway |
| `cp` | Mand | fileserver within a volume; copy Job across volumes/buckets | mount | Mapped | High | Cross-store copies price egress and requests in the dry-run band; never silently replicate or repartition |
| `dd` | Mand | artifact/volume block copy; /dev targets refused | mount | Partial | Low | bs and count honoured on artifacts and PVC files; no raw devices |
| `df` | Mand | kubelet volume stats, bucket inventory, namespace storage quota | translate | Mapped | High | Adds a COST column; the only inventory of orphaned volumes most teams will ever have |
| `dirname` | Mand | session | session | Native | Low |  |
| `du` | Mand | fileserver walk on PVC; inventory/list on S3 | mount | Partial | Medium | Listing a large bucket costs requests; du warns and prefers S3 Inventory |
| `file` | Mand | toolbox; artifact media type; image SBOM | toolbox | Partial | Low | file on image:// shows layers and SBOM |
| `find` | Mand | fileserver walk; S3 list/inventory; -exec → fan-out Job | mount | Mapped | High | -exec and -print0 \| xargs compile to a partitioned stage; S3 listing is priced |
| `link` | Opt (XSI) | hard link on PVC; none on S3 | mount | Partial | None |  |
| `ln` | Mand | PVC links via fileserver; catalog alias for artifacts | mount | Partial | Low | ln -s on /s3 or /artifacts creates a catalog alias, not a filesystem link |
| `ls` | Mand | fileserver list; object listings for /clusters /pvc /s3 /models | mount | Mapped | High | ls -l shows owner, size and cost for volumes and artifacts |
| `mkdir` | Mand | fileserver on PVC; S3 prefix no-op; namespace under /clusters | mount | Mapped | Medium | mkdir /clusters/x/teams/y creates a namespace from the platform quota template; volumes need mkvol |
| `mkfifo` | Mand | a topic on the platform's `fifo_device` (Kafka, Redpanda, NATS JetStream, SQS) under `/dev/queue` | translate | Mapped | Medium | Object storage is not a FIFO; a named pipe needs a broker (ADR 0012 decision 6). Consumers are Job work-queue mode or a service unit |
| `mv` | Mand | rename on PVC (atomic); copy+delete on S3 (not atomic); Job across stores | mount | Partial | Medium | The S3 case is warned; objects: rename is a label edit |
| `pathchk` | Mand | session | session | Native | None |  |
| `pwd` | Mand | (cluster, namespace, path) | translate | Mapped | High |  |
| `readlink` | Mand | resolve catalog aliases and PVC symlinks | mount | Partial | Low |  |
| `realpath` | Mand | resolve to artifact id or volume path | mount | Partial | Low | realpath of an artifact is its content hash |
| `rm` | Mand | fileserver / object delete; refused on volumes and namespaces | mount | Mapped | High | rm /pvc/x is refused and points to rmvol; rm on a namespace points to the destructive verb with confirmation; rm on a topic lists bytes and retention and needs --with-data |
| `rmdir` | Mand | fileserver; S3 prefix no-op; empty namespace via destructive verb | mount | Partial | Low |  |
| `touch` | Mand | fileserver on PVC; zero-byte put on S3 | mount | Partial | Low |  |
| `unlink` | Opt (XSI) | fileserver | mount | Native | None |  |

### C programming

| Command | POSIX | Kubernetes backing | Approach | Feasibility | Pull | Notes |
|---|---|---|---|---|---|---|
| `cc` | Opt (CD) | build Job (buildkit) producing a toolbox layer | translate | Partial | Low | Compiles inside a build Job; the output is an image layer, not a file in PATH |
| `cflow` | Opt (XSI) | toolbox | toolbox | Native | None |  |
| `ctags` | Opt (SD) | toolbox | toolbox | Native | None |  |
| `cxref` | Opt (XSI) | toolbox | toolbox | Native | None |  |
| `lex` | Opt (CD) | toolbox | toolbox | Native | None |  |
| `nm` | Optional(SD, XSI) | toolbox | toolbox | Native | None |  |
| `strings` | Mand | toolbox | toolbox | Native | Low | stateless on byte boundaries with overlap |
| `strip` | Opt (SD) | toolbox | toolbox | Native | None |  |
| `xgettext` | Opt (CD) | toolbox | toolbox | Native | None |  |
| `yacc` | Opt (CD) | toolbox | toolbox | Native | None |  |

### Shell programming

| Command | POSIX | Kubernetes backing | Approach | Feasibility | Pull | Notes |
|---|---|---|---|---|---|---|
| `command` | Mand | session builtin | session | Native | Low |  |
| `echo` | Mand | session / toolbox | session | Native | Low |  |
| `expr` | Mand | session | session | Native | Low |  |
| `false` | Mand | session | session | Native | Low |  |
| `getopts` | Mand | session | session | Native | Low |  |
| `logger` | Mand | Kubernetes Event on the process object + audit line | translate | Mapped | Medium | Events are retained one hour; the audit stream keeps the durable copy |
| `printf` | Mand | session | session | Native | Low |  |
| `read` | Mand | session | session | Native | Low |  |
| `sh` | Mand | the session interpreter; scripts go through the planner | session | Native | High | A script is a plan; set -o pipefail is on by default |
| `sleep` | Mand | session; timer gate in a plan | session | Mapped | Low | Inside a && chain or pipeline, sleep compiles to a delay on the edge, never to a pod that sleeps |
| `tee` | Mand | toolbox fan-out to artifacts and mounts | toolbox | Native | Medium | fan-out class; writing to a topic is a side effect through an idempotent producer keyed by the input artifact id, never an artifact (ADR 0012) |
| `test` | Mand | session | session | Native | Low |  |
| `true` | Mand | session | session | Native | Low |  |
| `xargs` | Mand | fan-out stage: -P → parallelism, -n → partition size | translate | Mapped | High | xargs is where DAG width comes from; compiles to an Indexed Job or Argo fan-out |

### Programming

| Command | POSIX | Kubernetes backing | Approach | Feasibility | Pull | Notes |
|---|---|---|---|---|---|---|
| `make` | Opt (SD) | plan stages with content-addressed caching | translate | Partial | Medium | A Makefile is a DAG with mtime caching; rules compile to stages, targets to artifacts, -j to fan-out. Closest POSIX ancestor of the pipeline compiler |

### Network

| Command | POSIX | Kubernetes backing | Approach | Feasibility | Pull | Notes |
|---|---|---|---|---|---|---|
| `uucp` | Opt (UU) | alias of cp across /clusters | translate | Mapped | Low | System-to-system copy is a cross-cluster artifact copy with egress priced |
| `uudecode` | Mand | toolbox | toolbox | Native | None |  |
| `uuencode` | Mand | toolbox | toolbox | Native | None |  |
| `uustat` | Opt (UU) | ps filtered to cross-cluster copies | translate | Partial | None |  |

### System administration

| Command | POSIX | Kubernetes backing | Approach | Feasibility | Pull | Notes |
|---|---|---|---|---|---|---|
| `who` | Opt (XSI) | active session pods: user, cwd, since | translate | Mapped | Medium |  |

## Reading the table

- A **toolbox** row with a class in its notes is already handled by `pipeline-compiler.md` §3; the class decides whether the command runs over partitions.
- A **translate** row is a verb the session implements against the API. Each one needs a dry-run form and an `explain` entry. The verbs POSIX does not define, and the POSIX verbs Convoy gives a new meaning (`batch`, `wait`, `time`, `timeout`, `mount`, `df`, `fuser`), are specified with one shape in [ADR 0013](adr/0013-native-verbs-have-one-shape.md).
- A **mount** row depends on the session fileserver and the mount namespace design (`/pvc`, `/s3`, `/clusters`), which supersedes the disk-operations note.
- **Partial** rows are where leaks live. Each note names the difference the user will see; none of them is hidden behind emulation.
- The ten **Dropped** rows are SCCS, printing and terminal chat. Their use cases are covered by Git, artifacts and collaborative sessions.
