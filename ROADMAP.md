# Roadmap

Build order follows the dependency chain the design uncovered: nothing above depends on anything below it.

| Milestone | Deliverable | Unblocks | Exit criterion |
|---|---|---|---|
| **M0 Design** (this repository) | README, pipeline compiler, debate review, command map, ADRs, stories, walkthroughs, user-story probes (`docs/user-stories/`) | everything | the three architecture-changing abstractions (session, identity, networking) have ADRs; the ten structural misroutes in `docs/user-stories/README.md` are each settled by an ADR decision or amendment |
| **M1 Toolbox** | image with a real init, coreutils pinned to `LC_ALL=C`, `convoy-get`/`convoy-put`/`convoy-get-all` with hashing and metering, merge programs (`sum-by-key`), the annotation database with ≥ 60 entries (PaSh coverage) | pipes, signals, cost, audit | a partitioned `grep \| sort \| uniq -c` over 64 local files produces byte-identical output to GNU coreutils |
| **M2 Session** | session pod, the approval object with its `Approval` resource and the CEL two-person admission policy (ADR 0017), the `convoy.io/` label set and its three CEL admission policies (ADR 0018), WebSocket attach, parser (reuse mvdan/sh), planner with the local backend only, `explain` in the full form of ADR 0013 decision 3 and `type`, type-directed completion from the annotation database and the verb tables (ADR 0014, no model), audit stream, receipts, cost sources incl. package-metered spend under `/sys/cost` (gap G8) | every later verb | `explain` prints a plan with classes, partitions and a cost band for any POSIX pipeline; `history` works across two sessions |
| **M3 Distributed backend** | Indexed Job map+merge, JobSet for coordinated groups (gap G5), completion contract incl. `--gang` (ADR 0004 amendment 1), `podFailurePolicy`, requests per class, do-not-disrupt, quota pre-check, `wait`, `time`, `timeout`, `ulimit` | batch wedge | the 412 GiB example in `pipeline-compiler.md` runs on a real cluster with a 95 % floor and a receipt; an 8-replica `--gang` job restarts all replicas from its checkpoint when one is preempted |
| **M4 Caching and Argo** | content-addressed artifact store with retention classes, holds and `replay --verify` (gap G9), no-cache for `deterministic: false` (gap G7), cache hits, limit pushdown, top-K rewrite, merge trees, Argo backend for `for`, `tee >(…)`, `wait` as the DAG join (gap G1), conditionals | ML pipelines | a rerun with one changed input re-executes only the affected partitions; the diamond in `walkthroughs.md` §1 compiles to one Workflow |
| **M5 Mounts** | mount namespace (`/pvc`, `/s3`, `/home`, `/scratch`, `/tmp`, `/artifacts`, `/queues`), session fileserver with per-type drivers reported by `caps`, `ls`/`cp`/`du`/`df` with cost, `umount` vs `rmvol`, snapshots, `fuser`, queue devices with bounded topic reads, `mkfifo` on a broker and `seek` (ADR 0012), table devices with snapshot-pinned reads, one-commit writes, `stat` on a table and `--at` (ADR 0015) | storage stories, edge buffering | `rmvol` refuses while in use and snapshots first; `df` lists every orphaned claim with its monthly cost; `cat` on a topic plans one partition per topic partition with an exact input size |
| **M6 Queueing and priority** | `batch` through Kueue, `nice`/`renice`, preemption as stop-with-checkpoint, grace from `checkpoint_seconds` | multi-tenant batch | an interactive pod preempting a training job never burns its floor and the job resumes from its checkpoint |
| **M7 Identity and secrets** | ADR 0016 (user and groups from the login token, group-owned objects, `svc:<group>` for unattended work, `wheel` assumes root with approval, `/etc/fstab.d/<group>`), `whoami`, `id`, owner labels + CEL admission, `sudo`, secrets as references, reference-only RBAC, workload identity, secret versions + reference index + `fuser` + `secret rotate`/`retire` + `kill -HUP` reload (gap G10) | multi-tenant safety | a user cannot `kill` a process owned by a group they are not in, or read a secret they can mount; a member can act on an absent colleague's job |
| **M8 Fleet** | device tree `/dev/k8s/by-id` + views, `/sys/k8s` metadata, hotplug events, the node tree under `/sys/k8s/<id>/nodes/` with `online` and `fenced` (ADR 0011 amendment 5), `mount` of clusters under `/clusters/…` with identity options, read-only `all` union, disconnected mode, cross-cluster artifact boundary with egress in the band (ADR 0011) | multi-cluster | `ps -A` across three clusters; a cross-cluster stage shows its egress cost at dry-run; a removed cluster fails `ls` fast instead of hanging |
| **M9 Packages** | `pkg` (GitOps-reconciled user packages), `tool add` (toolbox layers with annotations), package manifest with `requires` against `caps` | platform adoption | installing a package with a missing requirement fails at pre-flight with the platform's instruction |
| **M10 Signals node agent** | cgroup freeze DaemonSet for STOP/CONT; group freeze with a Lease barrier (ADR 0008 amendment 1, gap G2); node health collectors (hung tasks, EDAC, GPU events) and the receipt spool (ADR 0011 amendment 5, ADR 0009 amendment 1); CRIU `pause`/`resume` where CRI-O is present | interactive control | `kill -STOP --group` on a 64-GPU job freezes all ranks within one tick and `kill -CONT --group` resumes them; STOP on one rank is refused |
| **M11 Networking and services** | `expose`, `netstat`, default-deny policy, Gateway routes, service units with probes and rollouts, `release` that writes intent to the GitOps repository and waits for reconciliation (gap G3), `scale` delegating stateful scale-down to the operator or refusing (gap G6) | `run api-server` | a process exposed publicly has a certificate, a route, and a rollback in under a minute; a `release` to prod without a second approver exits 3 before any write |

Not scheduled: debugging (ephemeral containers), devices (DRA claims), events and triggers, time (`at`), locks, named pipes, backups and environment forks. They are designed in `docs/stories.md` and will be slotted once M3 ships.

## Non-goals, permanently

Translating text tools into engine operators. Deciding from output size. Streams as the default pipe. Auto-upgrading a pipeline to a cluster engine because the input looks big. Emulating `fork`, `ptrace`, shared memory or real-time scheduling across pods.

## Rules bound to milestones · 2026-11-09

From the user-story probes. Each milestone's exit criteria include the rules listed under it.

**M1**

- **L4.2** the toolbox provides `convoy-metric key=value` writing to `/sys/receipt/$pid/metrics`; receipts carry a `metrics` map flagged self-reported (see US-07 L7.2).

**M2**

- **L18.3** an approval object is `void` only for an enumerated reason (policy edited, hold lost, device removed, group cannot answer); re-created and re-paged; re-priced at resume.
- **L18.4** identical pending requests (same shape, same group) share one approval object with a waiter count; one answer resumes all; inputs held once.
- **L18.5** answers on an approval object are visible to the requesting actor regardless of L17.6.
- **L18.6** every approval policy has a bounded timeout (24 h default); expiry releases holds, pages and exits 3; `df` lists held bytes per pending object.
- **L7.2** metered entries are labelled `declared`; a later `settled` source (provider billing export) reconciles; FinOps sees both; a package whose declared spend is under settled by a platform threshold is flagged in `pkg list`.

**M3**

- **L5.1** `--gang` requires `--checkpoint` on a zone-independent device (object store or RWX); the planner refuses a per-replica PVC path for checkpoints; `explain` prints the rule.
- **L5.6** the session pre-creates PVCs named `<pid>-<index>` and mounts them by index through the downward API; they are retained on restart and on `kill` (ADR 0006); M3 exit criterion gains this case.

**M4**

- **L1.6** `crontab` derives `startingDeadlineSeconds` from `--deadline`, and a skipped run writes a receipt with exit 2 and the declared action.

**M5**

- **L4.1** object-store mounts record the source artifact id as object metadata on `cp`; `ls -l` shows it; `run --model=` resolves and pins the id in the unit receipt; when the registry is the artifact store itself, `ln -s` is the recommended spelling.
- **L4.5** the fileserver resolves every path argument to a content id at plan time, by sha256 for small objects and by object version id plus size for large ones, and the receipt says which.
- **L6.4** `backup` excludes Secret objects by default; `--include-secrets` is audited; the receipt lists exclusions.

**M6**

- **L5.5** the deadline counts from submission by default, queue and pause time included; `--deadline-from=start` opts out; `explain` prints the choice and the queue estimate.

**M7**

- **L6.6** `ls -l` under a cluster mount shows `?` as owner for unlabelled namespaces; `chown` adopts them; the inventory counts them as unmanaged.
- **L7.6** `egress:` with hostnames requires `caps: fqdn-policy`; without it the planner refuses or routes through the platform's egress proxy package; `explain` names the mechanism.
- **L17.3** an approval policy may name a fallback group; the refusal prints both; `--approver=<fallback>` is receipted with the primary group's size; root never approves.
- **L17.4** a refused write to a root file names the user the redirect ran as and the `sudo tee` spelling.
- **L17.6** the owner column of `ps`, `top` and `ls -l` is the group; the actor is visible to members and root only.
- **L17.7** `cron list` marks entries whose actor was removed at the provider; they keep running as the group's service identity.
- **L17.8** `secret set` records its source (`prompt`, `stdin`, `producer`); `secret list --actor` marks what a user knows.
- **L9.4** `secret retire` is two-phase: disable (version kept, reads denied and logged) for a cooling period, then delete; `fuser --unknown` lists Secret objects referenced by any pod path (env, envFrom, volumes, CSI) that the index did not create; denied reads reopen the retirement.

**M8**

- **L10.9** `caps` is evaluated per target device at plan time; a backend the site lacks is a plan-time refusal naming the capability and the site.

**M9**

- **L8.6** holds copy the image by digest into a platform-held registry device (`/dev/registry/by-id/held`), or refuse when none exists.

**M11**

- **L3.5** `caps: metrics` is required for any metric condition; absence is a plan-time refusal, never a fallback.
- **L4.3** platform default `idle_timeout` per unit class with a GPU; idle is scale-to-zero with volumes retained (ADR 0006); `ps` shows IDLE and the receipt the idle cost.
- **L12.3** consumer units default to `maxSurge: 0`, the cooperative-sticky assignor and a drain on TERM that commits before leaving; printed by `explain`.

