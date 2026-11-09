# User stories and open abstractions

> This document lists short stories the research missed and the abstractions still to design. The nine corporate workloads, written as full persona-driven probes against a fixed template, are in [user-stories/](user-stories/README.md).

The research notes contained some twenty user stories. The critique of those is summarized in `debate-review.md` and in the research README. This document holds the stories the research missed, each in the same shape plus the part the research never wrote: what happens on failure. The second half lists the abstractions still to design.

## Stories the research missed

### 1. Change cluster like changing directory
- **Command:** `cd /clusters/prod-east/teams/data`, then `pwd` and `ps` scoped to it. `ls /clusters` lists the fleet.
- **Maps to:** kubeconfig contexts and namespaces; fleet inventory from Argo CD cluster secrets or Cluster API objects. No federation.
- **Failure:** an unreachable cluster shows as disconnected. Commands queue for reconciliation or are refused, by policy, never dropped.

### 2. `sudo` and permission boundaries
- **Command:** `whoami` shows the service account and scopes. `sudo --as=dba run migrate-db` requests elevated identity for one process.
- **Maps to:** RBAC, service-account impersonation, projected tokens bound to the process lifetime, an approval step for production.
- **Failure:** denial prints the exact rule that blocked it and who can approve.

### 3. Partial completion and `wait` with floors
- **Command:** `xargs -P 64 --min-success=95% analyze < parts.txt`, then `wait 5678`. `retry 5678 --failed` reruns only failed indexes.
- **Maps to:** Indexed Job, `backoffLimitPerIndex`, `successPolicy`, `podFailurePolicy`.
- **Failure:** the moment the floor is unreachable, remaining work is cancelled and `wait` returns 1 with the list of failed indexes.

### 4. Cost showback per process
- **Command:** `batch --budget=50 train.py` prints an estimate band at dry-run. `ps` shows COST and BURN/H. `time 5001` prints the receipt.
- **Maps to:** OpenCost or provider pod-level allocation by `convoy.io/pid`, artifact-store metering in the toolbox helpers, the audit log as run history.
- **Failure:** hitting the budget triggers the declared action, pause-with-checkpoint by default. Exit code 2 and a receipt either way.

### 5. Quotas, priority and noisy neighbours
- **Command:** `ulimit -a` shows the namespace ceiling and who is using it. `nice` and `renice` set priority inside the team's queue.
- **Maps to:** ResourceQuota, LimitRange, PriorityClass, Kueue queues with cohorts and borrowing.
- **Failure:** an unschedulable process says "waiting for 40 GPUs held by pid 3001, team X, expected free in 2h", not Pending. Preemption is a stop with checkpoint, never a kill.

### 6. Agent sandbox as a process
- **Command:** `sandbox --net=none --fs=snapshot run agent-task.py`. Credentials issued to the PID die with it. `exec` lets a human step inside.
- **Maps to:** gVisor or Kata runtime class, NetworkPolicy, ephemeral volumes, secrets with time to live.
- **Failure:** a policy violation kills the sandbox with exit code 3 and an audit entry naming the rule.

### 7. Package install from the platform catalog
- **Command:** `pkg install api-server-profile`. Output lists "will create" and "requires", checked against `caps`.
- **Maps to:** pre-approved Helm or OCI charts reconciled by Argo CD or Flux; Crossplane compositions; Terraform through a controller for out-of-cluster resources.
- **Failure:** a missing dependency blocks the install with the platform team's instruction. `pkg rollback` restores the previous release.

### 8. Follow the energy, and disconnected operation
- **Command:** `run inference --policy=follow-energy`. `explain` shows planned migrations from ephemeris and contact windows.
- **Maps to:** time-varying placement policy, checkpoint-driven migration, k3s or KubeEdge nodes, a session that queues commands while the control plane is out of contact.
- **Failure:** a missed migration window keeps the job in place and marks it degraded against the energy floor. Fault rates are high, so floors are mandatory.
- **Note:** in orbit this story is strong because migration is physically forced by solar incidence, thermal budget and contact windows, and orbits are deterministic so placement can be planned rather than guessed. Checkpoints move; datasets never do, since inter-satellite links are narrow.

### 9. Killing a process that owns state
- **Command:** `kill 100` on a process with volumes scales compute to zero and retains state. `rmvol --with-data` is the only way to delete data, and it asks.
- **Maps to:** Deployment scale, reclaim policy Retain, deletion protection on compositions, the volume→pods index (`fuser`).
- **Failure:** a kill that would orphan data stops and lists what would be lost.

### 10. Capability discovery
- **Command:** `caps` lists the verbs this cluster supports and what each missing one needs (GPU plugin, Kueue, snapshots, node agent, CSI secrets).
- **Maps to:** discovery of installed CRDs, controllers and DaemonSets.
- **Failure:** an unsupported verb fails at pre-flight with the capability name, never at runtime with a Kubernetes error.

## Abstractions still to design

Three change the architecture:

| Abstraction | Why it changes the architecture |
|---|---|
| **Session placement and durability** | ADR 0005 decides the session is server-side; the design of the session pod, attach protocol, state persistence and promotion (`nohup`) is not written. Every other abstraction stores state here. |
| **Identity, ownership, tenancy** | RBAC is per kind and name, not per owner. "Only the owner may kill" needs owner labels plus a CEL admission policy. The tenancy tree (user, team, project; vCluster for hard isolation) underlies `cd`. |
| **Networking as sockets** | `expose pid 8080` → Service; `--public` → Gateway API route plus certificate; `netstat` → Services and endpoint slices; `/etc/hosts` → CoreDNS; `iptables` → NetworkPolicy, default-deny per namespace; `curl localhost:8080` → port-forward. Without this `run api-server` has no front door. |

The rest, in order of need:

| Abstraction | POSIX surface | Kubernetes backing | Trap |
|---|---|---|---|
| Executables | `which`, `$PATH`, `file`, `ldd` | image digest as inode, toolbox layers, registries as PATH entries, cosign as the signature bit | tags are mutable; pin digests in the receipt |
| Service units | `systemctl start/restart/reload/status`, `journalctl -u` | Deployment + Service + probes + rollout strategy; `rollout undo` | probes derived from nothing kill healthy processes; start with a generous startup probe |
| Events and triggers | `wait`, `inotify`, `dmesg`, `logger` | `kubectl wait` conditions, informers, Events, Argo Events or KEDA for `on … run …` | Events are kept one hour by default; export or `dmesg` is empty by lunchtime |
| Time | `at`, `cron`, `timeout`, windows | CronJob with `timeZone`, `activeDeadlineSeconds`, suspended Job released by a timer | no start-at in Kubernetes |
| Devices | `/dev`, `lsgpu`, `nvidia-smi` | device plugins, Dynamic Resource Allocation claims, MIG profiles, DCGM metrics | a count gives you a whole card; the claim is the unit |
| Debugging | `strace`, `gdb`, `tcpdump`, `lsof` | ephemeral debug containers, Inspektor Gadget or Tetragon | an ephemeral container cannot be removed; budget one |
| Configuration | `/etc/<app>/`, `sysctl`, edit and reload | ConfigMap as a directory, immutable ConfigMaps with a hash in the template, allowed sysctls | mounted ConfigMap updates take up to a minute and never reach env vars |
| Locks | `flock`, `mkdir` locks | Lease objects, `lock name -- cmd` | the honest fix for RWX races |
| Named pipes and queues | `mkfifo` | a topic on a `/dev/queue` device (Kafka, Redpanda, NATS JetStream, SQS); Job work-queue consumers or a service unit; drafted in ADR 0012 | object storage is not a FIFO; compaction and retention change what an offset range holds |
| Backup and environments | `pax`, `rsync`, `dump`, `env fork` | Velero by label, VolumeSnapshot, namespace or vCluster clones | backups copy Secrets across environments unless excluded |
| Mount namespace and fileserver | `ls`, `cp`, `du`, `df`, `mount`, `umount` on `/pvc`, `/s3`, `/home`, `/scratch`, `/tmp`, `/artifacts` | session fileserver pod, copy Jobs, kubelet volume stats, S3 Inventory | S3 has no append, random writes or atomic rename; RWO is per node; RWX is not atomic |
