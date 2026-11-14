# ADR 0011: Clusters are devices under `/dev`; the session mounts them

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** ADR 0005 (session owns the mount table); ADR 0010 (platform defaults); docs/stories.md story 1 (`cd /clusters/…`); docs/debate-review.md §multi-cluster; ROADMAP M5, M8

## Context

Every earlier document names `/clusters/<name>` as the tree a user walks with `cd`, and says the fleet inventory populates it. None of them says where a cluster *comes from* before it has a friendly name, who may use it, how two clusters with the same display name in two clouds are told apart, or what happens when a cluster appears or disappears while a session is attached.

The proposal under review lays the fleet out as a Linux-style device tree:

```
/dev/k8s/
├── aws/us-east-1/{prod-main,prod-data,staging}
├── aws/eu-west-1/prod-eu
├── azure/eastus/prod-aks
├── gcp/us-central1/prod-gke
├── current -> aws/us-east-1/prod-main
└── all                                   # federated view
```

with user-facing names mapped by symlink:

```
/cluster/prod-east-1/financial -> /dev/k8s/aws/us-east-1/financial-01234
```

Facts that constrain the design:

- Display names collide. `prod-main` can exist in EKS, AKS and GKE at once. Each cloud has a canonical identifier (an EKS ARN, an AKS resource id, a GKE `projects/…/locations/…/clusters/…` name) that does not.
- Provider and region are a cloud shape. On-prem, edge, Cluster API and the satellite constellation case (docs/stories.md, "follow the energy") have other topologies or none.
- A symlink carries no options and is global to the filesystem. A mount carries options, is per mount namespace, and appears in `mount` and `df`. The session already owns a per-user, Plan 9 style mount table (ADR 0005).
- Getting a usable credential for a cluster is a per-cloud exchange: `sts:AssumeRole` plus `eks get-token`, `az aks get-credentials`, `gcloud container clusters get-credentials`. Tokens expire.
- KubeFed is archived. A "federated view" that fans out to N API servers will see partial failures and has no placement answer for writes (docs/debate-review.md).
- Clusters appear and vanish: created by Convoy itself (ROADMAP M8), by autoscaling of the fleet, or by spot-backed node groups. A path that was valid a minute ago may point at nothing.
- Region in a path is enough for the planner to know a stage crosses an egress boundary, which is what the cost band at dry-run needs (ADR 0010, docs/pipeline-compiler.md §cost).

## Decision

1. **Clusters are character devices under `/dev/k8s/`.** A device node is the raw API endpoint plus the *right to obtain* a credential for it. It is not walkable. `ls /dev/k8s/by-id` lists the fleet; `cd` into a device node is an error that names the mount verb.

2. **`by-id` is the truth; every tree is a view.** The canonical directory is flat and keyed by the cloud's own identifier. Provider, region, label and topology trees are symlink farms generated from the inventory, exactly as udev generates `/dev/disk/by-*`. A topology with no region (on-prem, edge, constellation) is just another view with different path components.

   ```
   /dev/k8s/
   ├── by-id/
   │   ├── eks-123456789012-us-east-1-prod-main
   │   ├── aks-7f3c…-eastus-prod-aks
   │   ├── gke-signals-prod-us-central1-prod-gke
   │   └── capi-dc-sp-01-batch
   ├── by-provider/aws/us-east-1/prod-main -> ../../../by-id/eks-123456789012-us-east-1-prod-main
   ├── by-label/env=prod/prod-main          -> ../../by-id/eks-…
   ├── by-topology/leo/plane-3/sat-17       -> ../../../by-id/capi-leo-p3-s17
   └── self                                 -> by-id/eks-…        (per session)
   ```

3. **Devices are mounted, not symlinked.** A user-facing cluster path is a mount point in the session's mount table. The platform's fleet inventory supplies default mounts under `/clusters/` (the name every other document already uses); users add their own.

   ```
   mount -o role=arn:aws:iam::123456789012:role/fin-ro,ns=payments,ro \
         /dev/k8s/by-id/eks-123456789012-us-east-1-financial-01234 \
         /clusters/prod-east-1/financial
   ```

   Mount options are the policy surface: `role`/`sa`/`identity` (which identity opens the device), `ns` (namespace scope, repeatable), `ro`, `quota`, `toolbox`, `noexec` (list and read only, never `run`). `mount` and `df` show them. `umount` detaches and never destroys (ADR 0006).

4. **Opening a device is the credential exchange.** Permission bits on the node say who may open it; group ownership maps to a team. Open performs the cloud's token exchange under the mount's identity. Expiry triggers a silent reopen by the session. A failed open is a mount error with the cloud's reason, not a Kubernetes 401 at the first `ls`.

5. **`self`, not `current`.** The working directory already encodes cluster and namespace. `/dev/k8s/self` resolves per session (like `/proc/self`) for verbs that take no path, and `$CONVOY_CLUSTER` carries the same id into launched processes.

6. **`all` is a read-only union mount, never a federation.** The session builds it by binding every mounted cluster into one directory. Reads fan out, list partial results, print each unreachable member on stderr and exit degraded (exit code 0 with the degraded flag of ADR 0004). Writes are refused because placement is ambiguous. Explicit multi-target work stays on the command, as in `ps -A` or `--cluster=all`.

7. **`/sys/k8s/<id>/` holds the metadata.** Version, allocatable CPU, memory and accelerators, quota, node pools, price rate, topology labels and `caps` are plain readable files. `df -h /clusters/*` reports allocatable against requested from them. `/dev` is the handle; `/sys` is what the handle knows about itself.

8. **Hotplug is an event stream.** The inventory emits device-added and device-removed events; the session subscribes. A mount whose device is removed goes stale: `ls` fails fast with "device removed", running processes keep their own receipts, and `umount` cleans up. Nothing hangs on a dead endpoint.

9. **The same rules apply to every device class.** `/dev/k8s` is one subtree. Object stores (`/dev/obj/by-id/…`), cloud disks behind PVCs (`/dev/vol/by-id/…`), accelerators (`/dev/accel/…`, DRA claims) and queues behind `mkfifo` (`/dev/queue/…`) use by-id plus views, open-as-credential-exchange, mount-with-options and hotplug. The storage design that supersedes the disk-operations note (ROADMAP M5) is written against this ADR.

## Consequences

- Topology lives in the path, so the planner prices a cross-region or cross-cloud stage from the two device paths alone, and "follow the energy" becomes remounting a job's cluster under a new topology view.
- Two clusters named `prod-main` in two clouds no longer collide; a friendly name is a mount point the user or platform chose, not a property of the cluster.
- The platform team's fleet inventory becomes the udev of Convoy: it owns `by-id`, generates views, grants device permissions and emits hotplug events. Argo CD cluster secrets or Cluster API objects are sufficient sources.
- Identity (ROADMAP M7) gains a concrete anchor: the mount's identity is what every process launched under that path runs as, and `sudo --as` is a remount. ADR 0016 defines the identity: a user with groups from the login token, exchanged at open for the device's credential; options select among identities the group may assume, and `sudo` assumes a group's service identity. A launched process runs as the group's service identity, never under the user's token (ADR 0016 decision 5, L17.1), which narrows the first sentence of this bullet to the session's own calls.
- `all` can never silently do the wrong thing, at the price of being read-only.
- Cost: one more component (the inventory and its event stream) in M8, and `/sys` files must be refreshed on a cadence that does not hammer the API servers. Price rate files depend on the platform wiring OpenCost or the cloud price list in (ADR 0010).
- ADR 0012 adds `/dev/queue` as a class under decision 9, ADR 0014 drafts `/dev/model` for language models attached to the session, amendment 2 below adds `/dev/scm` and the per-topology artifact store, ADR 0015 drafts `/dev/table` for lakehouse catalogs and warehouses, and ADR 0023 drafts `/dev/metrics`, `/dev/logs` and `/dev/traces` for telemetry stores; all follow every rule above.
- `/dev/k8s/current` from the proposal is replaced by `self`; `/cluster/…` from the proposal is spelled `/clusters/…` to match every earlier document.

## Amendment 1 · 2026-11-09 · a device type needs a driver from a package (user story US-02, L2.1)

**Context.** Decision 9 names device classes; a filesystem device (`/dev/fs/by-id/hdfs-…`) is usable only if the session fileserver has a driver for its type, and nothing said where the driver comes from.

**Decision.** A device type is mountable only when a package provides its fileserver driver, discovered by `caps` as `fs:<type>`, `queue:<type>`, `obj:<type>`, `model:<provider>` and so on. `mount` of a device whose type has no driver on the session's platform is refused with the capability name. `caps` lists every driver with the package that provides it.

**Consequences.** Drivers are platform packages (ADR 0010) and go through GitOps like operators. ROADMAP M5 is a set of drivers, not one fileserver.

Note · 2026-11-12: the draft [ADR 0024](0024-a-device-is-installed-by-terraform.md) gives a device package a device contract and a Terraform module per provider beside its driver; `mknod` installs a device through the platform's runner, the module's outputs register it here, and `/sys/<class>/<id>/iac/` records the module instance (`iac: managed` or `iac: none`).

Note · 2026-11-12: the draft [ADR 0025](0025-a-computer-is-a-device-kubernetes-is-one-driver.md) makes compute one device class, `/dev/compute/`, with Kubernetes as one kind beside `slurm`, `fn`, `batch` and `vm`; decision 1 is the `k8s` kind of it, and `/dev/k8s` and `/sys/k8s/<id>/` remain as views of `/dev/compute/by-kind/k8s` and `/sys/compute/<id>/`.

Note · 2026-11-13: the draft [ADR 0026](0026-a-registry-is-a-device-a-tag-is-a-name-for-a-digest.md) adds registries to decision 9 as the class `/dev/registry/` (kinds `registry`, `cache`, `mirror`, `held`): repositories are directories, digests files and tags symbolic links under a registry mount, and `/sys/registry/<id>/` holds residency, lifecycle, replication and pins.

## Amendment 2 · 2026-11-09 · source repositories and artifact stores are devices (user stories US-03 and US-08, L3.3 and L8.1)

**Context.** `release` writes to a GitOps repository that is not in the device tree: no identity, no mount options, no `df`. The artifact store, where every pipe and receipt lives, was never placed at all; a single global store breaks residency for every intermediate artifact (ADR 0001 amendment 2).

**Decision.** Two more classes under decision 9. Source repositories are `/dev/scm/by-id/<id>`, mounted with a write identity and a `branch=` option, so `release` and every other intent write (ADR 0010 amendment 1) has a place to write and the credential is a reference. Artifact stores are `/dev/obj/by-id/cas-<topology>`, one per topology the platform serves, selected per stage by residency; the platform's inventory supplies them and the planner never writes to a store outside the stage's residency. Both appear in `df` with usage, cost and retention, and both have `/sys` metadata including topology.

**Consequences.** The GitOps repository is now something `fuser` can report holders of and `ls -l` can show the identity of. Receipts name the store of every artifact. A fleet with one region has one store and sees no difference.

Note · 2026-11-11: the draft [ADR 0022](0022-the-laptop-lends-a-tree-the-cluster-builds-it.md) adds a source-repository device kind `worktree`, `/dev/scm/by-id/wt-<user>-<host>-<name>`: a working tree lent from a laptop as read-only snapshots, `disconnected` while the laptop is away.

## Amendment 3 · 2026-11-09 · `nsmap` and `residency` are mount options (user stories US-06 and US-08, L6.1 and L8.2)

**Context.** Namespaces are flat and paths are hierarchical; `tenants/acme` and `teams/acme` both want to be `acme`, and the hierarchy under a cluster mount was a convention nobody declared. Residency as a per-command flag is bypassable by forgetting it; the walkthrough's own `cp` moved data across regions unchecked.

**Decision.** A cluster mount carries `nsmap=<rule>`, a declared bijection from path segments to namespace names, with the reverse recorded as labels on the namespace. `explain mkdir` prints the resulting name; a collision is refused; a path with no mapping is a read-only view. Every mount may carry `residency=<topology>`, enforced for every command that touches the mount: a stage whose inputs and outputs span residencies is refused unless `--allow-cross-region` is given, which is audited per command and printed in the receipt. Residency of a process is the intersection of its mounts' options, inherits to children (ADR 0004 amendment 3) and selects the artifact store (ADR 0001 amendment 2). The platform sets both options per namespace through its profile (ADR 0010), so users never have to remember them.

**Consequences.** A mount is the unit of policy, which is where misroute S3 said it should be. `cp` across residencies is a refusal with the override spelling, not a silent transfer.

## Amendment 4 · 2026-11-09 · disconnected is a state, not removal (user story US-10, L10.1)

**Context.** Decision 8 knows removed devices only. A site whose link is down is not removed; treating it as removed fails every `ls` and drops queued work, and treating it as present hangs every command.

**Decision.** A device has a `disconnected` state distinct from `removed` and `ready`, with `disconnected since` shown by `mount` and in `/sys/<class>/<id>/state`. Commands against a disconnected device follow the mount's `queue=<ttl>|fail` option: queued commands are listed with their count and expire at the TTL with a receipt that says so. Listings follow `stale=cache|fail`: the metadata cache is served with its age, or the command fails fast. Hotplug emits `connected` and `disconnected` events, and a device disconnected longer than the platform's threshold raises an alert through the alerting capability. Nothing is ever deleted because a device is disconnected.

**Consequences.** A session cut off from a site keeps working against the fleet it can reach, and the part it cannot is visible rather than absent. Completion and `ls` read the cache under this rule (ADR 0014 decision 7). The per-device link metadata (`/sys/k8s/<id>/link`) the stories asked for lives here.

## Amendment 5 · 2026-11-09 · nodes are a tree under `/sys`, with `online` and `fenced` writable (user story US-16, L16.1)

**Context.** A cluster is a character device and a pod is a process. Nothing in the design names a node, so the layer where disks hang, GPUs fall off the bus and memory throws errors has no path, no health file, no verb and no column in `ps`. Every hardware incident starts outside the shell, and the one verb that exists for a hung process (`kill -KILL`) cannot work there (ADR 0008 amendment 4).

**Decision.** Decision 7 gains a node tree: `/sys/k8s/<id>/nodes/<name>/` holds per-node files fed by the API server and the node agent. Read-only: `health` (a verdict line and one line per source: disk hung tasks, GPU health and XID events, EDAC correctable and uncorrectable counts, kubelet conditions, each with its age and its source capability, and `unknown` for any source whose capability is absent, never `ok`), `dmesg`, `conditions`, `gpu/<i>/health`, `edac`, `price` (the node's rate and its cordoned hours), `managed-by` (the pool manager and its object, or `none`), `receipt` (cordon, drain, fence and replacement events with identities, times, reasons and the evidence artifact). Writable under the platform identity: `online` (writing `0` cordons the node after prompting for a reason and pinning `health` and a `dmesg` excerpt as an evidence artifact with class `sensitive`; writing `1` uncordons; this is the sysfs convention for taking a CPU offline), `fenced` (the out-of-service taint, under the rule of ADR 0008 amendment 4) and `gpu/<i>/reset` (a device reset through the node agent, the least destructive repair). `rm` on the node directory is replacement: intent written to the object named by `managed-by` (ADR 0010 amendment 1), refused with the `page:` route when there is none. `ls` lists nodes with state (`ready`, `cordoned`, `fenced`, `draining`, `replacing`). Cordoned and fenced nodes are excluded from the allocatable figures in `/sys/k8s/<id>/` and from the plan-time capacity check (ADR 0003).

**Consequences.** Receipts record the node of every partition attempt and group member, and `status`, `top` and `ps --node` show it (ADR 0013 decision 5). The node agent (ROADMAP M10) gains collectors for hung tasks, EDAC and GPU events, and `caps` reports `dcgm` and `node-agent` as the sources `health` depends on. A replacement arrives as a hotplug event inside the device, so the inventory's event stream (decision 8) carries node-added and node-removed as well as device events. See docs/user-stories/US-16-node-fault.md.

Note · 2026-11-10: the draft [ADR 0021](0021-a-node-is-tested-before-it-is-trusted.md) gives the thresholds of this amendment a reader. Probes test each component and feed `health`; a policy and a health agent session in the controller exclude a GPU or a shared-filesystem client before the node, and cordon by writing `online 0` within a disruption budget; drain, replacement and fencing stay requests to a person. It adds per-component `online` files, a `probe` file, the node state `degraded: <component>` and, beside `disconnected` of amendment 4, a device state `degraded` for a shared filesystem that fails on many nodes at once.


Note · 2026-11-14: the draft [ADR 0028](0028-memory-has-a-distance-and-a-domain-is-claimed-whole.md) says what decision 9's `/dev/accel` DRA claims are for domains (a claim compiled from `--domain`), adds `/sys/compute/<id>/mem/` under decision 7, and adds `/dev/shm/` as a class of volatile in-memory caches, stores and shuffle services.

Note · 2026-11-14: the draft [ADR 0029](0029-a-computers-file-systems-are-in-its-contract-staging-is-a-planned-copy.md) makes decision 9's `/dev/fs` devices the file systems that outlive processes, installed by modules, and lists each computer's own file systems under `/sys/compute/<id>/fs/`.
## Rules · 2026-11-09 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L2.5** (US-02) copy Jobs pre-flight the destination's free capacity × replication factor from `/sys`; `explain cp` prints it.
- **L4.1** (US-04) object-store mounts record the source artifact id as object metadata on `cp`; `ls -l` shows it; `run --model=` resolves and pins the id in the unit receipt; when the registry is the artifact store itself, `ln -s` is the recommended spelling.
- **L5.3** (US-05) `/sys/k8s/<id>/preemption_rate` per node group (ADR 0011 decision 7); the band widens by expected restart loss on spot; the receipt separates productive from lost time.
- **L7.7** (US-07) named queues carry a retention class and sit on a device with a topology; `mkfifo` prints both.
- **L10.5** (US-10) `/sys/k8s/<id>/link` holds state, bandwidth and price per gigabyte; the band separates link egress from compute; placement prefers the side of the link with the data.
- **L25.10** (US-25) the `worktree` source-repository device (ADR 0022) has the id `wt-<user>-<host>-<name>` and is readable by its lender unless shared with `lend --share=<group>` (full rule in ADR 0022).
- **L27.3** (US-27) an infrastructure device compares its states with the provider account's own inventory (AWS Config or Resource Explorer, Cloud Asset Inventory, Azure Resource Graph) and lists the resources no state holds, with type and creator where known, before any translation or `mknod --like` is proposed; the listing is degraded when the inventory is unreachable (ADR 0024 decision 7).
- **L28.4** (US-28) `caps` claimed by a module's outputs are verified against the provider before a device registers (full rule in ADR 0024).
- **L29.8, L29.13, L29.15** (US-29) a computer's log and metric sinks carry a topology that residency binds; `/sys/proc/<pid>/backend` holds the backend job id and an unreachable computer's processes are `unknown`; account-wide limits are shown with what others hold (full rules in [ADR 0025](0025-a-computer-is-a-device-kubernetes-is-one-driver.md)).
- **L30.11, L30.12, L30.14** (US-30) `/sys/compute/<id>/` names the registry or mirror each computer pulls from; `fuser` on a digest lists every pod running it; registries and repositories carry a residency (full rules in [ADR 0026](0026-a-registry-is-a-device-a-tag-is-a-name-for-a-digest.md)).
- **L31.8, L31.10, L31.13** (US-31) channels are devices with a state, a residency and a retention; a failing channel is shown and its events go to the fallback route (full rules in [ADR 0027](0027-convoy-tells-a-group-about-its-own-objects.md)).
