# Roadmap

Build order follows the dependency chain the design uncovered: nothing above depends on anything below it.

| Milestone | Deliverable | Unblocks | Exit criterion |
|---|---|---|---|
| **M0 Design** (this repository) | README, pipeline compiler, debate review, command map, ADRs, stories, walkthroughs, user-story probes (`docs/user-stories/`) | everything | the three architecture-changing abstractions (session, identity, networking) have ADRs; the ten structural misroutes in `docs/user-stories/README.md` are each settled by an ADR decision or amendment |
| **M1 Toolbox** | image with a real init, coreutils pinned to `LC_ALL=C`, `convoy-get`/`convoy-put`/`convoy-get-all` with hashing and metering, merge programs (`sum-by-key`), the annotation database with ≥ 60 entries (PaSh coverage) | pipes, signals, cost, audit | a partitioned `grep \| sort \| uniq -c` over 64 local files produces byte-identical output to GNU coreutils |
| **M2 Session** | `run -it` terminals with job control, attach grants and end warnings in the terminal (ADR 0030), `write` and `wall` to the session journal of a user with no terminal attached (ADR 0027), the lend channel of the attach protocol with sparse lends of one root and snapshots current only when complete, and the client-side filter and scrub (ADR 0022), session pod restored from its journal, the attach protocol with token refresh and a shell per terminal, the controller as receipt writer, the `PS1` escapes (ADR 0020), the node health agent session with its verdict schema, action set and shadow mode (ADR 0021), the approval object with its `Approval` resource and the CEL two-person admission policy (ADR 0017), the `convoy.io/` label set and its three CEL admission policies (ADR 0018), WebSocket attach, parser (reuse mvdan/sh), planner with the local backend only, `explain` in the full form of ADR 0013 decision 3 and `type`, type-directed completion from the annotation database and the verb tables (ADR 0014, no model), audit stream, receipts, cost sources incl. package-metered spend under `/sys/cost` (gap G8) | every later verb | `explain` prints a plan with classes, partitions and a cost band for any POSIX pipeline; `history` works across two sessions |
| **M3 Distributed backend** | interactive placement by the smallest share near the data, and the start estimate with `--wait` (ADR 0030), reading in place or staging per input, the file-system capacity check and stage-in and stage-out steps in the plan (ADR 0029), placement by memory domain, domain capacity in `explain`, and memory and spill sizing for Spark and dataframe stages (ADR 0028), the image as a placement input with pulls from the nearest replica (ADR 0026), placement over every mounted computer by the ordered filter, partitions sized to `max_duration`, and `--on` (ADR 0025), `image-build` over rootless BuildKit with the pool's platforms and a `provenance: session` attestation (ADR 0022), Indexed Job map+merge, JobSet for coordinated groups (gap G5), completion contract incl. `--gang` (ADR 0004 amendment 1), `podFailurePolicy`, requests per class, do-not-disrupt, quota pre-check, `wait`, `time`, `timeout`, `ulimit` | batch wedge | the 412 GiB example in `pipeline-compiler.md` runs on a real cluster with a 95 % floor and a receipt; an 8-replica `--gang` job restarts all replicas from its checkpoint when one is preempted |
| **M4 Caching and Argo** | staged copies as cache entries by content id, verified before use, from pinned sources (ADR 0029), raise and delivery times of events in receipts (ADR 0027), pins on digests named by receipts, holds and running units (ADR 0026), settlement in each computer's price model, allocations with their balance (ADR 0025), stderr as one path across the pod's life, with the log device named in the receipt (ADR 0023), content-addressed artifact store with retention classes, holds and `replay --verify` (gap G9), a reverse lineage index for `secret scrub` (ADR 0007 amendment 5), no-cache for `deterministic: false` (gap G7), cache hits, limit pushdown, top-K rewrite, merge trees, Argo backend for `for`, `tee >(…)`, `wait` as the DAG join (gap G1), conditionals | ML pipelines | a rerun with one changed input re-executes only the affected partitions; the diamond in `walkthroughs.md` §1 compiles to one Workflow |
| **M5 Mounts** | the `fs` field in `/sys/compute/<id>/fs/`, linked-file states in `ls` and `du`, and `mv` across devices that deletes only after verifying (ADR 0029), the `/dev/shm` drivers with volatile semantics and declared backing (ADR 0028), the `/dev/notify` channel drivers and `/sys/notify/` with state, residency and price (ADR 0027), the `/dev/registry` driver with repositories, digests, tags as links and referrers, `cp` with referrers, `du` of shared layers (ADR 0026), the `/dev/iac` state device with drift (ADR 0024), the `/dev/metrics`, `/dev/logs` and `/dev/traces` drivers with bounded selector reads, the settle rule and query pricing (ADR 0023), the `worktree` source device with snapshots keyed by the git tree hash and `worktree_retention` (ADR 0022), mount namespace (`/pvc`, `/s3`, `/home`, `/scratch`, `/tmp`, `/artifacts`, `/queues`), session fileserver with per-type drivers reported by `caps`, `ls`/`cp`/`du`/`df` with cost, `umount` vs `rmvol`, snapshots, `fuser`, queue devices with bounded topic reads and schema-registry decode, encode and pinned reader schemas (ADR 0012 decision 10), `mkfifo` on a broker and `seek` (ADR 0012), table devices with snapshot-pinned reads, one-commit writes, `stat` on a table and `--at` (ADR 0015) | storage stories, edge buffering | `rmvol` refuses while in use and snapshots first; `df` lists every orphaned claim with its monthly cost; `cat` on a topic plans one partition per topic partition with an exact input size |
| **M6 Queueing and priority** | interactive priority within `interactive_quota`, and release on idle or detach (ADR 0030), domain claims, holds past `domain_hold_max` with approval, `domain_reserve`, and gang restarts into another healthy domain (ADR 0028), `batch` through Kueue, `nice`/`renice`, preemption as stop-with-checkpoint, grace from `checkpoint_seconds` | multi-tenant batch | an interactive pod preempting a training job never burns its floor and the job resumes from its checkpoint |
| **M7 Identity and secrets** | gateway login for interactive applications and launches from inside as the attached person (ADR 0030), per-group staging directories on computers without isolation and no staging of `class=restricted` data to them (ADR 0029), routes per group in `/etc/notify.d/` and answers only from channels with mapped, re-authenticated accounts (ADR 0027), repository ownership by group, pulls scoped to the pod's group, trusted signers per profile and vulnerability exceptions with expiry (ADR 0026), the per-kind credential exchange and `identity=per-user\|per-group\|shared` (ADR 0025), approvals pinned to plan hashes and the runner identity per module path (ADR 0024), telemetry tenancy, `label-enforce`, `telemetry_readers` and residency per device (ADR 0023), provenance levels `ci`, `session`, `any` per profile (ADR 0022), ADR 0016 (user and groups from the login token, group-owned objects, `svc:<group>` for unattended work, `wheel` assumes root with approval, `/etc/fstab.d/<group>`), `whoami`, `id`, owner labels + CEL admission, the group boundary as default-deny in the namespace template (ADR 0019), `sudo`, secrets as references, reference-only RBAC, workload identity, secret versions + reference index + `fuser` + `secret rotate`/`retire` + `kill -HUP` reload (gap G10) | multi-tenant safety | a user cannot `kill` a process owned by a group they are not in, or read a secret they can mount; a member can act on an absent colleague's job |
| **M8 Fleet** | drain and preemption notices to attached people, and allocation use outside Convoy (ADR 0030), removal of staged copies and per-process file systems after `stage_keep`, and idle file systems shown with their price (ADR 0029), domains that drain before reuse, device-memory scrub between groups, and the `fabric` class with the domain scope (ADR 0028), the notifier in the controller per residency with the closed event set, roll-up per process tree, the fallback route, daily test events and site spools (ADR 0027), `fuser` on a digest, mirror presence before a release and the inventory's check of lifecycle policies against pins (ADR 0026), the `/dev/compute/` class with `by-kind` views, `/sys/compute/<id>/` with `queue`, `/sys/proc/<pid>/backend` and `unknown` on an unreachable computer (ADR 0025), `mknod` with the runner's plan as `explain`, registration from module outputs, `/sys/<class>/<id>/iac/` and adoption by `import` blocks (ADR 0024), `top`, `ps` and `netstat` columns from the default telemetry devices, `?` with an age when unknown (ADR 0023), session hosts per residency, including edge sites (ADR 0020), device tree `/dev/k8s/by-id` + views, `/sys/k8s` metadata, hotplug events, the node tree under `/sys/k8s/<id>/nodes/` with `online` and `fenced` (ADR 0011 amendment 5), per-component `online` files, the `degraded` states and their exclusion from allocatable (ADR 0021), `mount` of clusters under `/clusters/…` with identity options, read-only `all` union, disconnected mode, cross-cluster artifact boundary with egress in the band (ADR 0011) | multi-cluster | `ps -A` across three clusters; a cross-cluster stage shows its egress cost at dry-run; a removed cluster fails `ls` fast instead of hanging |
| **M9 Packages** | the notebook hub driver and the Slurm and VM interactive shapes (ADR 0030), per-kind staging in the compute drivers and the transfer-service drivers (ADR 0029), the DRA driver mapping for domain claims and the modules for in-memory caches, stores and shuffle services (ADR 0028), channel driver packages and modules for chat, mail, incident tools and webhooks (ADR 0027), registry driver packages and modules for registries, caches and mirrors (ADR 0026), compute driver packages for `slurm`, `fn`, `batch` and `vm` under one contract (ADR 0025), device packages with a contract and Terraform modules per provider (ADR 0024), `pkg` (GitOps-reconciled user packages), `tool add` (toolbox layers with annotations), package manifest with `requires` against `caps` | platform adoption | installing a package with a missing requirement fails at pre-flight with the platform's instruction |
| **M10 Signals node agent** | the fabric burn-in probe on each domain (ADR 0028), preemption grace bounded by each pool's `preemption_notice` (ADR 0008, L27.13), cgroup freeze DaemonSet for STOP/CONT; group freeze with a Lease barrier (ADR 0008 amendment 1, gap G2); node health collectors (hung tasks, EDAC, GPU events) and the receipt spool (ADR 0011 amendment 5, ADR 0009 amendment 1); the probe package for GPU, shared-filesystem, memory and disk faults, peer attribution and the `convoy.io/untested` burn-in taint (ADR 0021); CRIU `pause`/`resume` where CRI-O is present | interactive control | `kill -STOP --group` on a 64-GPU job freezes all ranks within one tick and `kill -CONT --group` resumes them; STOP on one rank is refused |
| **M11 Networking and services** | gateway paths per interactive process (ADR 0030), `wait --slo` and `release --canary` on named unit conditions evaluated on the canary's series (ADR 0023), dev units named `<name>-<user>` with `--from`, `--like`, `--follow` and `--debug`, `attach --env-of` (ADR 0022), `expose` with reach, span and grants, `run --connect`, `netstat` and `/sys/unit/<name>/net/`, the session hosts view and `attach -L` (ADR 0019), default-deny policy, Gateway routes, service units with probes and rollouts, `release` that writes intent to the GitOps repository and waits for reconciliation (gap G3), `scale` delegating stateful scale-down to the operator or refusing (gap G6) | `run api-server` | a process exposed publicly has a certificate, a route, and a rollback in under a minute; a `release` to prod without a second approver exits 3 before any write |

Not scheduled: debugging (ephemeral containers), devices (DRA claims), events and triggers, time (`at`), locks, named pipes, backups and environment forks. They are designed in `docs/stories.md` and will be slotted once M3 ships.

## Non-goals, permanently

Translating text tools into engine operators. Deciding from output size. Streams as the default pipe. Auto-upgrading a pipeline to a cluster engine because the input looks big. Emulating `fork`, `ptrace`, shared memory or real-time scheduling across pods.

## Rules bound to milestones · 2026-11-09

From the user-story probes. Each milestone's exit criteria include the rules listed under it.

**M1**

- **L4.2** the toolbox provides `convoy-metric key=value` writing to `/sys/receipt/$pid/metrics`; receipts carry a `metrics` map flagged self-reported (see US-07 L7.2).

**M2**

- **L21.3** the generated kubeconfig uses an exec credential to the session; no token in the file.
- **L21.6** bounded in-memory scrollback per terminal; a resume says what was dropped or lost in a restore.
- **L21.7** prompt marks carry the age of cached state; stale marks render `?`.
- **L21.8** dormancy stops local commands and resumes them at the next attach; ended `session-dormant` after `session_idle`.
- **L21.10** a local command past `local_max` warns once and is receipted `over local_max`.
- **L18.3** an approval object is `void` only for an enumerated reason (policy edited, hold lost, device removed, group cannot answer); re-created and re-paged; re-priced at resume.
- **L18.4** identical pending requests (same shape, same group) share one approval object with a waiter count; one answer resumes all; inputs held once.
- **L18.5** answers on an approval object are visible to the requesting actor regardless of L17.6.
- **L18.6** every approval policy has a bounded timeout (24 h default); expiry releases holds, pages and exits 3; `df` lists held bytes per pending object.
- **L19.1** `cost --by group` prints declared and settled per group with the allocating component, an idle share, a `?` row, `declared only` where no cost package maps the label, and `unverified` on shell-only clusters.
- **L19.2** ownership walks ownerReferences to the nearest labelled ancestor (`child of <ancestor>`); the controller labels children templates cannot reach; packages declare propagation in `caps`.
- **L7.2** metered entries are labelled `declared`; a later `settled` source (provider billing export) reconciles; FinOps sees both; a package whose declared spend is under settled by a platform threshold is flagged in `pkg list`.
- **L23.5** the health model is sent only the kernel facilities the policy names, never an audit record.
- **L23.10** the node health policy and probes run in each site's part of the controller; ambiguous verdicts queue for the link.
- **L23.11** infrastructure retries are counted apart from `--retries`, within `infra_retries`.
- **L25.2** `lend --dry-run` lists sent, excluded and withheld files; a build that needs one is refused.
- **L25.3** data files are refused by `lend_exclude`, naming the session's residency.
- **L31.3** `write` to a detached user is journaled; notify verbs exit on acceptance.

**M3**

- **L5.1** `--gang` requires `--checkpoint` on a zone-independent device (object store or RWX); the planner refuses a per-replica PVC path for checkpoints; `explain` prints the rule.
- **L5.6** the session pre-creates PVCs named `<pid>-<index>` and mounts them by index through the downward API; they are retained on restart and on `kill` (ADR 0006); M3 exit criterion gains this case.
- **L24.5** a source-repository mount's `image=` (or its devcontainer) runs local stages under it in that image; `explain` names every stage's image.
- **L25.9** `image-build` steps run without registry credentials and with `build_egress`; the builder pushes.
- **L25.13** local stages under a worktree get a discarded writable overlay.
- **L27.6** cross-provider copies above `transfer_min` use the provider's transfer service, verified.
- **L29.8** a computer's log sink counts for residency.
- **L29.10** the remaining deadline compiles into an external scheduler's time limit.
- **L29.12** throttled and dropped function invocations count as failures.
- **L29.15** account-wide limits are checked net of what others hold.
- **L29.18** a stage over several topologies is split per topology.
- **L32.2** a gang with `--domain` is never split; the domain each member got is recorded.
- **L32.3** a domain's capacity is its healthy free size; `explain` refuses when none fits.
- **L32.16** `explain` sizes memory and spill for Spark and dataframe stages.
- **L32.17** no memory is shared across computers.
- **L33.4** staging is priced in the stage's band.
- **L33.5** a stage succeeds only when its outputs are verified at their destination.
- **L33.14** placement drops computers whose file systems cannot hold the stage.
- **L33.15** read in place unless staging is cheaper or faster; `explain` says why.
- **L33.16** `explain` warns on file counts above a metadata limit.

**M4**

- **L1.6** `crontab` derives `startingDeadlineSeconds` from `--deadline`, and a skipped run writes a receipt with exit 2 and the declared action.
- **L26.3** telemetry receipts record source and resolution; `replay --verify` fails across downsampling.
- **L26.6** artifacts from telemetry reads keep the source's retention and its scrub.
- **L27.9** decommission refused while holds; `hold move` keeps expiry and chain of custody.
- **L27.10** receipts and holds naming a broker's offsets are listed and copied before it goes.
- **L29.4** a scheduler's requeue is a new attempt with its stderr kept.
- **L29.6** receipts settle in each computer's price model.
- **L29.11** image conversions start from the digest and are cached by it.
- **L30.3** pinned digests cannot be deleted; `du` splits shared layers.
- **L30.15** receipts record every container that ran, injected ones included.
- **L31.6** messages are rolled up, rate limited and priced to the owning group.
- **L31.10** events at a disconnected site are spooled with receipts.
- **L33.2** staged copies are cache entries verified before use.
- **L33.9** transfers resume per file; a stage starts on a verified manifest.
- **L33.18** a stage-in reads a pinned snapshot of its source.

**M5**

- **L4.1** object-store mounts record the source artifact id as object metadata on `cp`; `ls -l` shows it; `run --model=` resolves and pins the id in the unit receipt; when the registry is the artifact store itself, `ln -s` is the recommended spelling.
- **L4.5** the fileserver resolves every path argument to a content id at plan time, by sha256 for small objects and by object version id plus size for large ones, and the receipt says which.
- **L6.4** `backup` excludes Secret objects by default; `--include-secrets` is audited; the receipt lists exclusions.
- **L22.3** the schema cache is keyed by registry and id with fingerprints; a mismatch is refused naming both registries.
- **L22.7** fixed text forms for timestamps, decimals, bytes, enums and unions, named in the column list.
- **L25.10** the worktree device id carries the user; snapshots are the lender's unless shared.
- **L26.7** an unestimable telemetry read is bounded by its upper bound.
- **L26.8** telemetry estimates use statistics endpoints only.
- **L26.10** a read across telemetry stores names those it missed and is degraded.
- **L27.3** account resources in no Terraform state are listed before translation.
- **L27.5** a table copy rewrites its metadata; byte copies of tables are refused.
- **L28.4** claimed `caps` are verified against the provider before registration.
- **L30.2** tags owned by GitOps, CI or replication are moved as intent.
- **L30.9** image copies carry their referrers.
- **L30.14** registries carry residency; copies never leave it.
- **L31.13** messages carry no output; channels carry residency.
- **L32.9** `/dev/shm` files are written once; no writable shared mapping across nodes.
- **L32.10** `/dev/shm` devices are volatile with declared backing; a lost object is `EIO`.
- **L32.14** spill to a bounded device; shuffle files owned per process and removed.
- **L33.6** `mv` across devices deletes only after verifying.
- **L33.10** a stage-out ships only files the process wrote.
- **L33.19** linked files show their load state in `ls -l` and `du`.

**M6**

- **L5.5** the deadline counts from submission by default, queue and pause time included; `--deadline-from=start` opts out; `explain` prints the choice and the queue estimate.
- **L32.5** `domain_reserve` keeps domain pools for claims; stranded members priced to the platform.
- **L32.13** whole-domain holds past `domain_hold_max` need approval.
- **L32.15** `--domain` and `--gang` are not inherited.

**M7**

- **L21.5** `newgrp` to a group of another residency is refused, naming `attach --residency`.
- **L21.9** no exec, attach, port-forward or ephemeral container on session pods; break-glass by approved `sudo`, shown by `who`.
- **L6.6** `ls -l` under a cluster mount shows `?` as owner for unlabelled namespaces; `chown` adopts them; the inventory counts them as unmanaged.
- **L7.6** `egress:` with hostnames requires `caps: fqdn-policy`; without it the planner refuses or routes through the platform's egress proxy package; `explain` names the mechanism.
- **L17.3** an approval policy may name a fallback group; the refusal prints both; `--approver=<fallback>` is receipted with the primary group's size; root never approves.
- **L17.4** a refused write to a root file names the user the redirect ran as and the `sudo tee` spelling.
- **L17.6** the owner column of `ps`, `top` and `ls -l` is the group; the actor is visible to members and root only.
- **L17.7** `cron list` marks entries whose actor was removed at the provider; they keep running as the group's service identity.
- **L17.8** `secret set` records its source (`prompt`, `stdin`, `producer`); `secret list --actor` marks what a user knows.
- **L9.4** `secret retire` is two-phase: disable (version kept, reads denied and logged) for a cooling period, then delete; `fuser --unknown` lists Secret objects referenced by any pod path (env, envFrom, volumes, CSI) that the index did not create; denied reads reopen the retirement.
- **L22.2** each device class reports its `scrub:` capability; matches it cannot remove are reported with their removal date.
- **L22.4** consumer groups past matched offsets are listed with owners and paged.
- **L22.5** `--purge` needs an approver from each purged object's owning group.
- **L22.6** scrub match locations are readable by security and the owning groups only.
- **L22.8** schema-aware scans of string fields; `--budget` binds in order of exposure, unscanned ranges listed.
- **L24.9** provenance is a mount requirement checked at plan time; secret-shaped files in image layers are refused everywhere.
- **L26.14** telemetry reads pass the redactor before terminal, journal and artifacts.
- **L27.2** Terraform state and plans are a secret sink.
- **L27.11** `secret move` rotates on the target; values copied only controller-side.
- **L28.3** a re-plan voids the approval and re-asks with the diff.
- **L28.7** delegated paths take package instances only; principals limited to the group.
- **L28.11** where the runner publishes plans is declared; sensitive inputs are references.
- **L29.5** computers declare per-user, per-group or shared identity.
- **L30.5** repositories belong to groups; pulls are as the pod's group.
- **L30.8** stale scans are rescanned or refused.
- **L30.10** each computer reports whether it enforces signatures.
- **L30.13** vulnerability exceptions are approvals with an expiry.
- **L31.4** events go to groups, never only to a person.
- **L31.12** approval requests go to the approving group's on-call.
- **L33.8** staging into per-group directories only on computers without isolation.
- **L33.12** staging into a shared quota past `stage_share_max` needs approval.

**M8**

- **L21.1** session pods: PDB, do-not-disrupt while local work runs, no spot; every eviction is a drain with notice and restore.
- **L20.3** a withdrawn hostname stays reserved to its group for a hold period (30 days default); `netstat --held` lists held names.
- **L19.4** on an `admission: shell-only` cluster `explain` prints the mode, cost and holds are `unverified`, `backup` and `cost` selectors need `--include-unverified`, `chown` is root-only; the inventory lists the cluster as an accepted risk.
- **L10.9** `caps` is evaluated per target device at plan time; a backend the site lacks is a plan-time refusal naming the capability and the site.
- **L23.7** `caps: node-repair` says whether the repairer honours a drain; where not, the controller drains within its wait.
- **L23.8** burn-in capacity is listed as joining, with the estimated start in `status` and `wait`.
- **L24.2** the plan checks the digest's platforms against the target pool's.
- **L24.7** `explain` prints the namespace's defaulted limits, quota headroom and Pod Security level with their sources.
- **L26.11** `caps: label-keys`; joins fall back to pod names.
- **L26.15** live columns from one controller cache per store, cost shared.
- **L27.14** the release diff covers the clusters' `caps` and the verbs they disable.
- **L27.15** script bands include units over a horizon; `cost --reprice-at`.
- **L28.2** `mknod` shows its stage; `kill` closes the PR before apply, defers during it.
- **L28.5** destroys of data resources a change did not name are refused; a workspace per instance.
- **L28.6** a partial apply registers `iac: partial`, priced and completable.
- **L28.9** an unreachable runner is reported; out-of-band devices marked for adoption.
- **L28.10** adoption needs a no-change plan.
- **L29.9** `release` and `scale` on a module-installed VM pool write intent.
- **L29.13** processes on an unreachable computer are `unknown`, with the backend job id.
- **L29.14** node verbs on an externally operated computer become requests.
- **L30.4** lifecycle policies never expire a pinned digest.
- **L30.11** a release checks the target's mirror first.
- **L30.12** `fuser` on a digest lists every pod running it.
- **L31.2** routes never edit schedules, escalations or alert rules.
- **L31.7** messages are updated or superseded as the object changes.
- **L31.8** a failed delivery is an event to the fallback route.
- **L31.11** events carry a deduplication key from the object path.
- **L31.14** `--notify` covers the process tree; children roll up.
- **L31.15** every event class has an urgency.
- **L31.17** expiring objects warn their owning group.
- **L32.7** a domain drains before reuse.
- **L32.8** device memory is cleared between groups; `mem_scrub=` declared.
- **L33.3** staged copies keep their residency and are removed at the end.
- **L33.7** staged copies and per-process file systems removed after `stage_keep`.

**M9**

- **L20.4** `--internal` certificates only from an issuer `caps` lists as private; otherwise refused unless `internal-ct: allowed`.
- **L8.6** holds copy the image by digest into a platform-held registry device (`/dev/registry/by-id/held`), or refuse when none exists.
- **L24.8** branch units carry the profile's `branch_unit_ttl` and show their daily cost.
- **L28.8** usage-priced bands from contract inputs; plans reused per input hash.
- **L28.13** package upgrades never re-plan instances.
- **L30.17** production takes public images only from a platform mirror.
- **L32.11** stores inside a process count in its bound; installed stores are devices.
- **L32.12** domain layout is read from the driver, never written.
- **L33.11** lasting file systems from modules, per-process ones from drivers.

**M10**

- **L23.4** a drain caused by a component the checkpoint needs skips the checkpoint.
- **L23.6** readings carry the collector version; a step that coincides with a rollout is the rollout's.
- **L23.9** at most one abandoned probe per component per node.
- **L29.2** signals a computer does not deliver are refused.

**M11**

- **L20.1** `expose` and grants on a GitOps-managed unit are written as intent; the receipt carries the commit.
- **L20.5** `explain --connect` prints the path class and price per gigabyte; Services prefer the same zone.
- **L20.6** revoking a grant lists running callers, needs `--yes`, and notifies their group.
- **L20.7** `expose` pre-flights hostname and path against every route on the Gateway; foreign routes show owner `?`.
- **L20.9** `nohup` promotion re-plans reach as granted `--connect`, or refuses.
- **L20.10** `release` reports open long-lived connections on the old version and applies a drain window.
- **L19.3** adoption writes labels only; `scale` and `release` on an adopted object refuse naming `unit add <name> --from-live`, which writes the unit record and marks it `adopted`.
- **L3.5** `caps: metrics` is required for any metric condition; absence is a plan-time refusal, never a fallback (ADR 0023 decision 5 makes it a named condition on a metrics device).
- **L4.3** platform default `idle_timeout` per unit class with a GPU; idle is scale-to-zero with volumes retained (ADR 0006); `ps` shows IDLE and the receipt the idle cost.
- **L12.3** consumer units default to `maxSurge: 0`, the cooperative-sticky assignor and a drain on TERM that commits before leaving; printed by `explain`.
- **L24.6** `--connect=<path>@<range>` checks the target's version from `/sys`.
- **L24.10** undeclared egress is printed; units declare `--egress`; dropped connections counted per unit.
- **L25.5** a followed unit is digest and tree; a digest passes only for the tree it was built from.
- **L25.6** debugger pauses are shown and bounded by `debug_pause_max`.
- **L25.8** `--follow` coalesces builds, caps them per hour and shows their cost.
- **L25.12** dev units are refused in mounts with `release_path=git`.
- **L25.14** secret-shaped plain `--env` values are refused; `--env-of` writes none.
- **L25.15** `explain` of `--like` lists the target's policies it does not reproduce.
- **L26.13** `slo:` may name a platform SLO; shadowing is refused.
- **L26.16** unattended conditions read as `svc:<group>`; a long `unknown` rolls a canary back.
- **L27.7** replication packages provide lag; the cutover waits on it.
- **L27.8** rollback after moving writes needs reverse replication.
- **L27.17** a canary across providers writes DNS weights as intent.
- **L29.16** `expose` on a VM pool compiles to its firewall as module intent.
