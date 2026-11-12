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
| `tty` | Mand | session terminal | session | Native | Low | Prints `/dev/pts/<n>`, one shell per attached terminal (ADR 0020) |
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
| `who` | Opt (XSI) | active session pods: user, cwd, since | translate | Mapped | Medium | Lists the user's sessions and terminals with host and state; root sees all (ADR 0020) |

## Reading the table

- A **toolbox** row with a class in its notes is already handled by `pipeline-compiler.md` §3; the class decides whether the command runs over partitions.
- A **translate** row is a verb the session implements against the API. Each one needs a dry-run form and an `explain` entry. The verbs POSIX does not define, and the POSIX verbs Convoy gives a new meaning (`batch`, `wait`, `time`, `timeout`, `mount`, `df`, `fuser`), are specified with one shape in [ADR 0013](adr/0013-native-verbs-have-one-shape.md). The identity rows (`id`, `whoami`, `newgrp`, `sudo`, `chown`, `chmod`) follow [ADR 0016](adr/0016-identity-is-a-user-rights-belong-to-groups.md): the user is the actor, rights and ownership belong to groups, and `sudo` assumes a group's service identity.
- A **mount** row depends on the session fileserver and the mount namespace design (`/pvc`, `/s3`, `/clusters`), which supersedes the disk-operations note.
- **Partial** rows are where leaks live. Each note names the difference the user will see; none of them is hidden behind emulation.
- The ten **Dropped** rows are SCCS, printing and terminal chat. Their use cases are covered by Git, artifacts and collaborative sessions.

## Rules from the user-story probes · 2026-11-09

Rules the sixteen stories in `docs/user-stories/` decided for specific commands. The row in the full table is unchanged; this table is where the command's behaviour on Convoy is refined.

| Commands | Rule | Finding |
|---|---|---|
| `crontab` | `crontab` derives `startingDeadlineSeconds` from `--deadline`, and a skipped run writes a receipt with exit 2 and the declared action | L1.6 |
| `cp` | copy Jobs pre-flight the destination's free capacity × replication factor from `/sys`; `explain cp` prints it | L2.5 |
| `wait` | `caps: metrics` is required for any metric condition; absence is a plan-time refusal, never a fallback | L3.5 |
| `cp`, `ln` | object-store mounts record the source artifact id as object metadata on `cp`; `ls -l` shows it; `run --model=` resolves and pins the id in the unit receipt; when the registry is the artifact store itself, `ln -s` is the recommended spelling | L4.1 |
| `xargs` | `explain` warns when fail-fast can cancel independent siblings; `--keep-going` runs every index and exits 1 with the failed list; platforms may set it as the `xargs -P` default | L4.4 |
| `rmdir`, `rm` | `rmdir` refuses a non-empty namespace; `rm -r` lists everything that would be lost and requires `--with-data` and, for namespaces with volumes, a second approval (US-03 L3.1); reclaim policy Retain is the platform default | L6.2 |
| `ls`, `chown` | `ls -l` under a cluster mount shows `?` as owner for unlabelled namespaces; `chown` adopts them; the inventory counts them as unmanaged | L6.6 |
| several | prompts in non-interactive sessions are exit 3 with the required flag (`--yes`), consistent with `rm -i` | L6.7 |
| `crontab`, `at` | wall-clock deadlines are read in the session's `TZ`; receipts store UTC and the zone; `crontab` sets `timeZone` | L8.4 |
| `chmod`, `getfacl` | `chmod` maps `r` to Read and Describe, `w` to Write, `x` to Create under the prefix; everything else is `setfacl`/`getfacl`; `ls -l` marks a `+` when ACLs exceed the mode, as POSIX ACLs do | L13.2 |
| `xargs` | reading a queue by redirection is a bounded drain to the depth observed at plan time and compiles to a work-queue Job; a follow into `xargs` is continuous and requires `run --unit`, scaled by queue depth when `caps: keda` exists | L13.5 |
| `sudo` | a shell redirect in a `sudo` line is applied by the caller's shell as the caller; a refused write to a root file names that and the `echo … \| sudo tee <file>` spelling; `sudo` assumes a group's service identity or `svc:root` through `wheel` with a second member's approval (ADR 0016) | L17.4 |
| `ps`, `top`, `ls -l` | the owner column is the group; the actor column is shown to members of that group and to root, `?` otherwise | L17.6 |
| `cron` | `cron list` marks entries whose actor was removed at the provider with the date; they keep running as the group's service identity | L17.7 |
| `df`, `ls` | `df` lists bytes held per pending approval object; `ls /sys/approvals/<group>/` is the group's inbox and shows the waiter count when identical requests share one object (ADR 0017) | L18.4, L18.6 |
| `cost` | `cost --by group` prints declared and settled per group with the allocating component, an idle share by the platform's rule, a `?` row with object counts per cluster, `declared only` where no cost package maps the group label, and `unverified` on shell-only clusters; `explain` prints the idle rule (ADR 0018) | L19.1 |
| `ls -l`, `ps` | ownership walks ownerReferences to the nearest labelled ancestor and prints `child of <ancestor>` instead of `?` (ADR 0018) | L19.2 |
| `chown`, `scale`, `release` | adoption by `chown` writes labels only; a verb that needs a unit record refuses naming what is missing and `unit add <name> --from-live` | L19.3 |
| `explain`, `backup`, `cost`, `chown` | on an `admission: shell-only` cluster `explain` prints the mode, cost and holds are `unverified`, `backup` and `cost` selectors need `--include-unverified`, and `chown` is root-only | L19.4 |
| `expose` | on a GitOps-managed unit the Service, routes, NetworkPolicy and grants are written as intent; hostname and path are pre-flighted against every route on the Gateway; a withdrawn hostname is held for its group; `--internal` certificates come from a private issuer; revoking a grant with running callers needs `--yes` and notifies them (ADR 0019) | L20.1, L20.3, L20.4, L20.6, L20.7 |
| `explain`, `run`, `batch` | `--connect` prints the path class (same zone, cross-zone, cross-region) and its price per gigabyte; Services prefer the same zone | L20.5 |
| `nohup` | promotion to a Job re-plans reach: units reached through the session's hosts view must be granted `--connect`, else refused | L20.9 |
| `release` | weights move new connections only; open long-lived connections on the old version are reported and drained within `--drain` | L20.10 |
| `netstat` | foreign routes to a unit show owner `?`; `--held` lists held hostnames | L20.3, L20.7 |
| `newgrp` | to a group of another residency is refused in the session, naming `attach --residency` (ADR 0020) | L21.5 |
| `who`, `tty` | `who` lists stopped commands of a dormant session, watches by other members and any root inspection of the session pod (ADR 0020) | L21.4, L21.8, L21.9 |
| `kubectl` (toolbox) | the generated kubeconfig asks the session for a token per call; no token in the file | L21.3 |
| `mount`, `PS1` | `mount` shows the age of each device's cached state; prompt marks older than `prompt_stale` render `?` | L21.7 |
| `nohup` | the shell warns once when a local command passes `local_max`; the receipt records it | L21.10 |
| `secret` | `secret scrub` follows the device writes of the processes that held the version and reads forward from them (ADR 0007 amendment 5 decision 2), applies each device class's `scrub:` capability and reports the removal date of what it cannot remove, lists consumer groups past matched offsets, gates `--purge` on each object's owning group, shows match locations to security and owners only, and scans string fields first within `--budget` | L22.1, L22.2, L22.4, L22.5, L22.6, L22.8 |
| `cat`, `cut`, `head` on a typed topic | fixed text forms: ISO 8601 UTC timestamps, decimals as strings with scale, base64 bytes, enum symbols, bare unions; the schema cache is keyed by registry and id, with fingerprints | L22.3, L22.7 |
| `kill` | a drain caused by a component the checkpoint needs (a member's GPU, the checkpoint device's client) skips the checkpoint and names the last good one | L23.4 |
| `status`, `wait` | print the estimated start including the burn-in of joining nodes; receipts count infrastructure retries apart from `--retries` | L23.8, L23.11 |
| `env`, `export` | a unit gets only the variables it declares with `--env`; a session variable reaches a unit only when named (ADR 0013 decision 4) | L24.3 |
| `diff` | `diff` of two `explain --at=<path>` plans lists what differs between two environments; `release` prints the same differences from the mount where the digest last passed (ADR 0013 decisions 3 and 7) | L24.4 |
| `make` and any local stage | under a source-repository mount with `image=`, runs in that image; `explain` names the image of every stage | L24.5 |
| `ps`, `cost` | units from a non-default branch show their daily cost and their TTL | L24.8 |
| `netstat` | dropped connections per unit with the host, where `caps: flows` exists; `egress: undeclared` for an image with no annotation | L24.10 |
| `make` and any local stage under `/src/<name>` | runs with a per-process writable overlay over the read-only worktree, discarded at exit (ADR 0022) | L25.13 |
| `ps` | a followed dev unit shows its digest, its tree and the updates since the build; a debugged one shows `paused (debugger)` with its duration; `--follow` shows its builds and their cost | L25.5, L25.6, L25.8 |
| `run --unit --from`, `attach --env-of` | refused in a mount with `release_path=git`; a secret-shaped plain `--env` value is refused and `--env-of` writes none; `explain` of `--like` lists the target's policies it does not reproduce | L25.12, L25.14, L25.15 |
| `attach --lend`, `lend` | `--dry-run` prints sent, excluded and withheld files; data files refused by `lend_exclude`; the device id carries the user and the snapshot is the lender's unless `--share=<group>` | L25.2, L25.3, L25.10 |
