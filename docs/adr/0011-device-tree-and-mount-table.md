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
- Identity (ROADMAP M7) gains a concrete anchor: the mount's identity is what every process launched under that path runs as, and `sudo --as` is a remount.
- `all` can never silently do the wrong thing, at the price of being read-only.
- Cost: one more component (the inventory and its event stream) in M8, and `/sys` files must be refreshed on a cadence that does not hammer the API servers. Price rate files depend on the platform wiring OpenCost or the cloud price list in (ADR 0010).
- ADR 0012 adds `/dev/queue` as a class under decision 9, and ADR 0014 drafts `/dev/model` for language models attached to the session, mounted with `identity`, `residency` and `scope` options; both follow every rule above.
- `/dev/k8s/current` from the proposal is replaced by `self`; `/cluster/…` from the proposal is spelled `/clusters/…` to match every earlier document.
