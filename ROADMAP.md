# Roadmap

Build order follows the dependency chain the design uncovered: nothing above depends on anything below it.

| Milestone | Deliverable | Unblocks | Exit criterion |
|---|---|---|---|
| **M0 Design** (this repository) | README, pipeline compiler, debate review, command map, ADRs, stories | everything | the three architecture-changing abstractions (session, identity, networking) have ADRs |
| **M1 Toolbox** | image with a real init, coreutils pinned to `LC_ALL=C`, `convoy-get`/`convoy-put`/`convoy-get-all` with hashing and metering, merge programs (`sum-by-key`), the annotation database with ≥ 60 entries (PaSh coverage) | pipes, signals, cost, audit | a partitioned `grep \| sort \| uniq -c` over 64 local files produces byte-identical output to GNU coreutils |
| **M2 Session** | session pod, WebSocket attach, parser (reuse mvdan/sh), planner with the local backend only, `explain`, `type`, audit stream, receipts | every later verb | `explain` prints a plan with classes, partitions and a cost band for any POSIX pipeline; `history` works across two sessions |
| **M3 Distributed backend** | Indexed Job map+merge, completion contract, `podFailurePolicy`, requests per class, do-not-disrupt, quota pre-check, `wait`, `time`, `timeout`, `ulimit` | batch wedge | the 412 GiB example in `pipeline-compiler.md` runs on a real cluster with a 95 % floor and a receipt |
| **M4 Caching and Argo** | content-addressed artifact store, cache hits, limit pushdown, top-K rewrite, merge trees, Argo backend for `for`, `tee`, conditionals | ML pipelines | a rerun with one changed input re-executes only the affected partitions |
| **M5 Mounts** | mount namespace (`/pvc`, `/s3`, `/home`, `/scratch`, `/tmp`, `/artifacts`), session fileserver, `ls`/`cp`/`du`/`df` with cost, `umount` vs `rmvol`, snapshots, `fuser` | storage stories | `rmvol` refuses while in use and snapshots first; `df` lists every orphaned claim with its monthly cost |
| **M6 Queueing and priority** | `batch` through Kueue, `nice`/`renice`, preemption as stop-with-checkpoint, grace from `checkpoint_seconds` | multi-tenant batch | an interactive pod preempting a training job never burns its floor and the job resumes from its checkpoint |
| **M7 Identity and secrets** | `whoami`, `id`, owner labels + CEL admission, `sudo --as`, secrets as references, reference-only RBAC, workload identity | multi-tenant safety | a user cannot `kill` another user's process or read a secret they can mount |
| **M8 Fleet** | `cd /clusters/…`, fleet inventory, disconnected mode, cross-cluster artifact boundary with egress in the band | multi-cluster | `ps -A` across three clusters; a cross-cluster stage shows its egress cost at dry-run |
| **M9 Packages** | `pkg` (GitOps-reconciled user packages), `tool add` (toolbox layers with annotations), package manifest with `requires` against `caps` | platform adoption | installing a package with a missing requirement fails at pre-flight with the platform's instruction |
| **M10 Signals node agent** | cgroup freeze DaemonSet for STOP/CONT; CRIU `pause`/`resume` where CRI-O is present | interactive control | `kill -STOP` on a 64-GPU job keeps memory intact and `kill -CONT` resumes it on the same node |
| **M11 Networking and services** | `expose`, `netstat`, default-deny policy, Gateway routes, service units with probes and rollouts | `run api-server` | a process exposed publicly has a certificate, a route, and a rollback in under a minute |

Not scheduled: debugging (ephemeral containers), devices (DRA claims), events and triggers, time (`at`), locks, named pipes, backups and environment forks. They are designed in `docs/stories.md` and will be slotted once M3 ships.

## Non-goals, permanently

Translating text tools into engine operators. Deciding from output size. Streams as the default pipe. Auto-upgrading a pipeline to a cluster engine because the input looks big. Emulating `fork`, `ptrace`, shared memory or real-time scheduling across pods.
