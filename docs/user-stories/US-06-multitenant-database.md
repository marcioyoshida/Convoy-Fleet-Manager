# US-06 · Give a tenant a database as a directory and never lose a replica's data

| Field | Value |
|---|---|
| Primary persona | DBA (Stateful service owner) |
| Secondary personas | PE, SEC, SRE |
| Workload class | stateful service, multi-tenant |
| Walkthrough | [walkthroughs.md §6](../walkthroughs.md) |
| Gaps already closed there | G2, G6 |
| Milestones needed | M2, M5, M7, M9, M11 |
| Findings in this document | L6.1 – L6.7 |

## 1. Story

**As a** database administrator on a platform team, **I want** to create a tenant as a directory, run a three-replica database in it from the platform's operator, resize it, back it up and see every tenant's database at once, **so that** tenant onboarding is minutes and no scale-down or cleanup can destroy data.

**Context.** Forty tenants, one operator, two incidents last year caused by a namespace deletion that cascaded to volumes. The operator's CRD is fine; the surrounding namespace, quota, network policy and backup wiring is where mistakes happen.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | the DBA | personal identity on `/clusters/prod-east-1/platform`, then `sudo --as platform` for cross-tenant views | identity design is open |
| Approves | none for create; a second person for `rm -r` of a tenant | | L6.2 |
| Pays | tenant acme | cost centre from the namespace template | `df` shows per-volume cost |
| Audits | security, quarterly | who created, resized, backed up, deleted | receipts per verb |
| Operates the platform | the same team | operator package, quota template, default-deny policy, backup target | |

## 3. Preconditions

- **Capabilities**: `operator:cnpg`, `snapshots`, `networkpolicy`, `resourcequota`, `objstore` for backups, `cel-admission` for owner rules.
- **Packages**: `cnpg` as a platform package with `scale: delegate, decommission-first`.
- **Mounts**: `/clusters/prod-east-1` with `nsmap=` declaring how paths become namespace names (L6.1); `/s3/backups` write for the tenant identity.
- **Annotations**: `cnpg/cluster: unit, stateful, group cnpg, scale delegate, reload hup, member_signals none`.
- **Platform defaults** `explain` must print: quota template, network policy, PDB, do-not-disrupt, reclaim policy Retain, namespace name derived from the path, backup secret exclusion.

## 4. Main path

```bash
$ mkdir tenants/acme && cd tenants/acme
$ run --unit db --package=cnpg/cluster --replicas=3 --disk=per-replica:200Gi --zone-spread
$ df -h .
$ scale db 2            # delegated to the operator; db-2 decommissioned, volume retained
$ rmvol /pvc/db-2       # refused: held by the operator's retention policy
$ backup db --to /s3/backups/acme/
$ sudo --as platform ps -A --cluster=prod-east-1 | grep ' db '
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `mkdir tenants/acme` | directory | Namespace `tenant-acme` from the template, ResourceQuota, NetworkPolicy, owner labels | API server |
| `run --unit db` | stateful unit | cnpg `Cluster`, 3 PVCs, PDB | operator |
| `scale db 2` | delegated scale | `Cluster.spec.instances: 2` | operator decommissions |
| `rmvol` | refused | nothing | |
| `backup db` | snapshot + copy | VolumeSnapshot × 2, copy Job | object store |
| `rm -r tenants/acme` (not in the walkthrough) | cascading delete | Namespace deletion | **refused without `--with-data` and a second approval** (L6.2) |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | – | no pipelines | |
| A2 | Command | ○ | a stateful unit from a package | `type cnpg/cluster` prints `scale: delegate` |
| A3 | Session | ● | `cd` into a tenant; cwd namespace drives every verb | `pwd` shows cluster and namespace; `mkdir` prints the namespace name it will create (L6.1) |
| A4 | Backend | – | no stages | |
| A5 | Contract | – | units converge rather than exit | |
| A6 | Device tree | ● | hierarchical path over a flat namespace list (L6.1); volumes as devices | `ls /clusters/prod-east-1/platform/tenants` lists tenants only |
| A7 | Signals | ● | STOP on a quorum member refused (G2); HUP = reload | `kill -STOP db` is refused |
| A8 | Secrets | ● | the database password is a reference the operator creates | `ls -l /secrets` in the tenant shows it; `backup` excludes it (L6.4) |
| A9 | Cost | ○ | `df` with cost per volume | three lines with zone and monthly cost |
| A10 | Identity | ● | tenant isolation by owner labels and CEL; `sudo --as platform` | depends on the open identity design (L6.5) |
| A11 | Defaults | ● | seven defaults printed by `run` | all seven in `explain` |
| A12 | Storage | ● | per-replica disks, snapshots, retained volumes, `rmvol` refused | AC-6.3 |
| A13 | Networking | ○ | default-deny per tenant | a pod in acme cannot reach globex |
| A14 | Time | – | | |
| A15 | Audit | ○ | every verb a receipt | `scale` receipt names the decommissioned replica |
| A16 | Durability | ● | scale-down and tenant deletion never lose data | AC-6.3, AC-6.4 |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | `scale db 2` on an operator-owned object | delegated to the operator by writing its spec | ok; if the `Cluster` is GitOps-managed, US-03 L3.2 applies |
| P2 | POSIX cannot hold | `mkdir a/b` where namespaces are flat; `rmdir` on a non-empty namespace | path to name is a convention; Kubernetes deletion cascades | **L6.1**, **L6.2** |
| P3 | Orphaned state | `scale` down leaves `/pvc/db-2` | retained by design; `df` lists it with cost | ok |
| P4 | Identity leak | backups copy the tenant's Secrets to the backup bucket | Velero-style trap named in stories.md | **L6.4** |
| P5 | Invisible cost | retained volumes after scale-down | `df` shows them; GC never deletes them | ok |
| P6 | Lying cache | none | | ok |
| P7 | Silent failure | `scale db 2` with an operator that does not support scale-down | ADR 0006 note: delegate or refuse | ok |
| P8 | Partial fleet | one cluster | n/a | ok |
| P9 | Created elsewhere | a tenant created by `kubectl create ns` without the template | invisible to `ls tenants/` or listed without owner | L6.6 |
| P10 | Multi-party | the `[y/N]` prompt in a script | a prompt in a non-interactive batch | L6.7 |
| P11 | Data at rest | backups in `/s3/backups/acme` under the tenant's identity | retention class from the template | ok |
| P12 | Inheritance | quota from the template; the DB pods are not Kueue jobs | ResourceQuota only | ok, stated |

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| namespace name collision | `nsmap` at plan time | 3 | existing namespace and its owner | pick another path |
| scale-down on an operator without decommission | annotation | 3 | reason | `backup` then operator-specific steps |
| `rmvol` on a held volume | `fuser` (holder is a policy) | 1 | holder TYPE policy | `snapshot` first |
| `rm -r tenants/acme` with volumes | pre-flight listing | 3 | what would be lost; needs `--with-data` and a second approval | `backup`, then approved delete |
| backup includes a Secret | default exclusion | n/a | excluded objects listed | `--include-secrets` with audit |
| `[y/N]` prompt in batch | non-interactive session | 3 | needs `--yes` | rerun with `--yes` |

## 8. Acceptance criteria

- **AC-6.1** Given `mkdir tenants/acme` under a mount with `nsmap`, when `explain` runs, then it prints the namespace name, the quota template, the network policy and the owner labels; when the name already exists, then it is refused with exit 3.
- **AC-6.2** Given `scale db 2` on a `cnpg` unit, then the operator decommissions one replica, `/pvc/db-2` remains, and `df` lists it with its monthly cost.
- **AC-6.3** Given `rmvol /pvc/db-2` while a policy holds it, then it is refused naming the holder with type `policy`.
- **AC-6.4** Given `rm -r tenants/acme` with retained volumes, then it lists what would be lost and refuses without `--with-data` and a second approval; `rmdir tenants/acme` on a non-empty namespace is refused as POSIX requires.
- **AC-6.5** Given `backup db`, then no Secret object is in the backup unless `--include-secrets` was given, and the receipt lists exclusions.
- **AC-6.6** Given `kill -STOP db`, then it is refused as a coordinated group member with the `--group` alternative.

## 9. Persona lenses

- **DBA**: wants scale, backup and restore as verbs that cannot lose data; refuses if a namespace delete can cascade to volumes.
- **PE**: owns the path-to-namespace mapping and the templates; wants collisions impossible, not just unlikely.
- **SEC**: wants backups without secrets by default and tenant deletion two-person.
- **SRE**: wants `fuser` to tell policy holders from process holders at a glance.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L6.1 | P2 | Namespaces are flat; `tenants/acme` and `teams/acme` both want to be `acme`. The hierarchy under a cluster mount is a convention nobody has declared. | **amend** (applied: ADR 0011 a3) ADR 0011: a cluster mount carries `nsmap=`, a declared bijection from path segments to namespace names (and back, via labels); `explain mkdir` prints the result; collisions are refused; paths with no mapping are read-only views | ADR 0011, ROADMAP M7 |
| L6.2 | P2, P3 | Namespace deletion cascades to every object and, by reclaim policy, to volumes; POSIX `rmdir` refuses non-empty directories. | **rule** (applied: ADR 0006 rules; command map): `rmdir` refuses a non-empty namespace; `rm -r` lists everything that would be lost and requires `--with-data` and, for namespaces with volumes, a second approval (US-03 L3.1); reclaim policy Retain is the platform default | posix-command-map.md (`rmdir`, `rm`), ADR 0006, ADR 0010 |
| L6.3 | P7 | `fuser` reporting a retention policy as the holder of a volume stretches the POSIX meaning. | **accepted**: holders have a TYPE column (`process`, `unit`, `policy`, `snapshot`); the refusal names the type | posix-command-map.md (`fuser`) |
| L6.4 | P4 | `backup` of a tenant copies its Secret objects into the backup target unless excluded. | **rule** (applied: ADR 0007 rules; roadmap M5): `backup` excludes Secret objects by default; `--include-secrets` is audited; the receipt lists exclusions | ADR 0007, ROADMAP M5 |
| L6.5 | P9 | Cross-tenant visibility (`sudo --as platform`) and tenant isolation by owner labels depend on the identity design, which does not exist. | **open**: identity and ownership is one of the three architecture-changing gaps; this story is its acceptance test for tenancy | stories.md, ROADMAP M7 |
| L6.6 | P9 | A namespace created outside Convoy has no owner labels and no template; it appears in `ls` without an owner or not at all. | **rule** (applied: command map; roadmap M7): `ls -l` under a cluster mount shows `?` as owner for unlabelled namespaces; `chown` adopts them; the inventory counts them as unmanaged | posix-command-map.md (`ls`, `chown`), ROADMAP M7 |
| L6.7 | P10 | `scale db 2` asks `[y/N]`; in a batch there is nobody to answer. | **rule** (applied: ADR 0005 rules; command map): prompts in non-interactive sessions are exit 3 with the required flag (`--yes`), consistent with `rm -i` | posix-command-map.md, ADR 0005 |
