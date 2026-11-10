# US-19 · Bill every team across two clouds from one label, back up one process, and adopt what we did not create

| Field | Value |
|---|---|
| Primary persona | PE |
| Secondary personas | FIN, SEC, SRE, DE, DBA |
| Workload class | platform plumbing: cost allocation by group, a selective backup, adoption of a foreign Deployment, a cluster without admission policies, a package with no label mapping |
| Walkthrough | written against [ADR 0018](../adr/0018-labels-are-the-signal-admission-is-the-decision.md) with [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md) (decisions 3 and 8), [ADR 0010](../adr/0010-platform-defaults-and-two-package-tiers.md) (amendment 1, packages, rule L7.2), [ADR 0011](../adr/0011-device-tree-and-mount-table.md) (`caps`, amendment 3) and [ADR 0013](../adr/0013-native-verbs-have-one-shape.md) (`chown`, `backup`, `caps`); it reuses the database unit of US-06, the cost lines of US-07 and the node of US-16 |
| Gaps already closed there | none |
| Milestones needed | M2, M8, M9 |
| Findings in this document | L19.1 – L19.7 |

## 1. Story

**As a** platform engineer, **I want** every object the shell writes to carry the same few labels, so that the cost tool, the backup tool, the queue and the residency policy all act on a team without my writing glue for each, and the shell to tell me plainly which objects are not its own and which clusters cannot enforce the labels, **so that** chargeback, backup and tenancy are one convention I can read with `kubectl` and not four scripts I maintain.

**Context.** The fleet runs an EKS cluster with OpenCost and a GKE cluster with the billing export; the same teams use both. Finance wants October by team. A data engineer wants one process's volumes backed up before a risky rerun. A legacy `ledger-admin` Deployment predates Convoy and nobody can `scale` it from the shell. A bare-metal k3s site runs a Kubernetes too old for CEL admission. Kueue was installed last week by a package whose manifest never said which label names the queue.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | `pe-sam`, platform | user `pe-sam`, groups `platform`, `wheel` | writes mappings, adopts objects under root |
| Reads the bill | `fin-rao`, FinOps | user `fin-rao`, group `finops`, read on `/sys/*/cost` | `cost --by group` across both clusters |
| Asks for the backup | `dana2`, data engineer | group `payments-de` | `backup --pid` of their own process |
| Audits | security | receipts; the admission policies' audit annotations | which clusters are `admission: shell-only` |
| Operates the platform | the same team | the inventory (slugs), OpenCost, the billing export, Velero, Kueue, the residency policy | |

## 3. Preconditions

- **Capabilities**: `admission:cel` on EKS and GKE, absent on the k3s site; `cost:opencost` (EKS) and `cost:billing-export` (GKE), each declaring a mapping from `convoy.io/group`; `backup:velero` declaring a mapping from `convoy.io/pid` and `convoy.io/unit`; `queue:kueue` installed without a mapping; `residency-policy` per cluster.
- **Packages**: `cnpg` (US-06) with `inheritedMetadata` set to carry `convoy.io/` keys to its children; `kueue` (no mapping yet).
- **Mounts**: `/clusters/eks-prod`, `/clusters/gke-prod`, `/clusters/site-k3s` under the platform profile; `/clusters/all` read-only.
- **Annotations**: `backup: verb, writes staged copy, class pattern`; `chown: verb, writes labels, admission`.
- **Platform defaults** `explain` must print: the label set written per object, the mappings each component declares, the idle-cost allocation rule, the admission mode per cluster, the slug registry source.

## 4. Main path

```bash
# October by team, across both clouds
$ cost --by group --month 2026-10 /clusters/all
GROUP          DECLARED    SETTLED     ALLOCATED BY                       IDLE SHARE   UNVERIFIED
payments-de    $18 402     $17 980     opencost (eks), billing-export (gke)  $1 210     $0
ml-platform    $61 118     $63 004     opencost (eks), billing-export (gke)  $4 870     $0
finance-de     $9 870      $9 410      opencost (eks), billing-export (gke)  $610       $0
?              $2 114      $2 290      none: 37 objects without convoy.io/group on eks-prod, 12 on gke-prod   (L19.1)
site-k3s       $3 300      $3 300      none: cost package absent; declared only, admission shell-only, unverified  (L19.4)
idle allocated by platform rule: proportional to requests (explain prints the rule)

# One process's volumes before a risky rerun
[dana2] $ backup --pid 7810 --to /s3/payments-archive/7810/
velero: selector convoy.io/pid=7810 (mapping from backup:velero) …
  pvc-7810-0 … pvc-7810-17   18 volumes, 212 GiB   receipt 8310
  skipped: pvc-db-2 (owned by unit db-2 through cnpg, label convoy.io/unit=db-2; not pid 7810)              (L19.2)
$ ls -l /clusters/eks-prod/payments/pvc/
pvc-7810-0     payments-de  dana2   pid 7810
pvc-db-2       payments-de  unit db-2 (child: cnpg Cluster db-2 via ownerReferences)
pvc-scratch-x  ?            (no convoy.io/group, no labelled owner)

# A Deployment that predates Convoy
$ scale ledger-admin 3
refused (exit 3): ledger-admin is not owned by Convoy (no convoy.io/group); chown adopts it:  chown payments-de ledger-admin
$ sudo chown payments-de /clusters/eks-prod/payments/deploy/ledger-admin
ledger-admin is managed by argocd (tracking label): labels written as intent, commit 5e2a… to gitops/payments, reconciled 40 s
labels: convoy.io/group=payments-de convoy.io/kind=unit convoy.io/unit=ledger-admin convoy.io/residency=us-east-1
managed-by left as is: app.kubernetes.io/managed-by=Helm (another manager)                                        (L19.7)
pods: 3 running pods carry no labels until the next rollout; the controller labels new pods as they appear      (L19.2)
$ scale ledger-admin 3
refused (exit 3): adopted unit ledger-admin has no unit record: release_path, image digest source, probes unknown
  alternative: unit add ledger-admin --from-live (writes the record from the Deployment as it is)                 (L19.3)

# The k3s site
$ caps /clusters/site-k3s
admission: shell-only (Kubernetes 1.27, no ValidatingAdmissionPolicy)   cost: none   backup: none   residency-policy: none
$ explain run --cluster /clusters/site-k3s -- sort big.csv
… writes Job sort-8322 with convoy.io/group=finance-de … admission: shell-only, labels unverified on this cluster
$ chown ml-platform /clusters/site-k3s/finance/job/sort-8322
refused (exit 3): on a cluster without admission, chown is root-only (ADR 0018 rule L19.4)

# The package that forgot to declare its mapping
$ batch --cluster /clusters/eks-prod -- spark-submit job.py
refused (exit 3): queue:kueue declares no mapping for convoy.io/group; the plan cannot name a LocalQueue
  caps shows: kueue  mapping: none   fix: packages/kueue/caps.yaml  labels: {convoy.io/group: kueue.x-k8s.io/queue-name via localqueue/<slug>}
$ pkg install kueue@0.9.1-platform.2        # the manifest now declares the mapping
$ caps /clusters/eks-prod | grep kueue
kueue  mapping: convoy.io/group to kueue.x-k8s.io/queue-name (localqueue per group slug)

# Actors
$ kubectl get job sort-8322 -o jsonpath='{.metadata.annotations.convoy\.io/actor}'
dana2                                                                                                            (L19.5)
$ id payments-de
group payments-de  slug g-7f3a  provider id 8c1e-…  since 2025-02-01   (a group recreated at the provider under the same name gets a new slug)  (L19.6)
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `cost --by group` | declared and settled lines per group | OpenCost allocation by the mapped key; billing export rows with the mapped resource label | `/sys/<id>/cost`, ledger |
| `backup --pid` | a staged copy with a receipt | a Velero Backup with the mapped selector | object store, receipt |
| `ls -l` on volumes | owner, actor, kind, child-of | labels; ownerReferences walked to a labelled ancestor | API server |
| `chown` on a managed object | labels as intent | a commit; the GitOps tool reconciles | repository, receipt |
| `caps` | admission mode, mappings per component | cluster version; package manifests | `/sys/k8s/<id>/caps` |
| `batch` through Kueue | a Job with the queue label | `kueue.x-k8s.io/queue-name` from the mapping | the Job |
| actor | an annotation | `convoy.io/actor` annotation; receipts | audit stream |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ○ | the hold label on snapshots; the staged backup copy | |
| A2 | Command | – | | |
| A3 | Session | – | | |
| A4 | Backend | ○ | the compiled Job carries the set | `explain` prints it |
| A5 | Contract | – | | |
| A6 | Device tree | ● | `caps` carries admission mode and mappings; `by-label` views stay the platform's | AC-19.4, AC-19.5 |
| A7 | Signals | – | | |
| A8 | Secrets | – | | |
| A9 | Cost | ● | declared, settled, allocated-by, idle share, unverified, `?` per group (L19.1) | AC-19.1 |
| A10 | Identity | ● | group label under admission; `chown` adoption; root-only on shell-only clusters; actor as annotation (L19.4, L19.5) | AC-19.3, AC-19.4 |
| A11 | Defaults | ● | mappings, idle rule, admission mode, slug source; a package without a mapping refuses | AC-19.5 |
| A12 | Storage | ● | selective backup by pid; children through ownerReferences (L19.2) | AC-19.2 |
| A13 | Networking | – | | |
| A14 | Time | ○ | settled lags declared (L7.2); reconciliation of adopted labels | |
| A15 | Audit | ● | the actor lives in the annotation and the receipt; the admission policies' audit | AC-19.6 |
| A16 | Durability | ○ | slugs never reused; a recreated group is a new group (L19.6) | |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | `chown` on an Argo-managed Deployment; `managed-by` on a Helm-managed one | labels as intent (ADR 0010 a1); `managed-by` never overwritten where another manager owns it | **L19.7** |
| P2 | POSIX cannot hold | `chown` on Linux changes one inode; here it changes labels on an object whose children keep the old ones | children are reached through ownerReferences and by the controller, not by `chown -R` alone | **L19.2** |
| P3 | Orphaned state | pods of an adopted Deployment have no labels until a rollout; a cnpg PVC has the unit's labels, not a pid | ownership is resolved through ownerReferences to a labelled ancestor; the controller labels children templates cannot reach | **L19.2** |
| P4 | Identity leak | `convoy.io/actor` as a label is readable by anyone who can list the namespace, and on cluster-scoped objects by everyone | the actor is an annotation and in the receipt; the label set drops it | **L19.5** |
| P5 | Invisible cost | idle node cost and `?` objects on a shared cluster are nobody's | `cost --by group` prints idle share by the platform's rule and a `?` row with counts | **L19.1** |
| P6 | Lying cache | settled cost lands days later; a slug lookup from a stale inventory | `declared` and `settled` side by side (L7.2); slugs are immutable so stale is still right | ok |
| P7 | Silent failure | a cluster without CEL admission: the shell checks, nothing else does | `caps` says `admission: shell-only`; `explain` prints it; cost and holds there are `unverified`; `chown` root-only | **L19.4** |
| P8 | Partial fleet | the k3s site has no cost package | the group's row says `declared only` with the reason, not a smaller number | ok with L19.1 |
| P9 | Created elsewhere | a Deployment with no labels; a PVC with no labelled owner | `?`, read-only, `chown` adopts; adoption writes labels only | **L19.3** |
| P10 | Multi-party | a package author who never declared a mapping | a verb that needs the mapping refuses and names the manifest line; `caps` shows `mapping: none` | ok (ADR 0018 d6) |
| P11 | Data at rest | the backup copies volumes selected by label into an object store under the group's residency | the staged copy carries `convoy.io/hold` and `residency`; the residency policy at the store refuses the wrong one | ok |
| P12 | Inheritance | a CronJob's jobTemplate carries the entry's pid; each run needs its own | the controller assigns and labels the run pid on Job creation | folded into L19.2 |

A thirteenth question for this story: the inventory keys slugs by the provider's group id, so a group deleted and re-created under the same name is a different group with a new slug and its old objects stay orphaned (L17.5). That is the correct reading of "never reused", and it needs saying. **L19.6**, accepted.

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| object without `convoy.io/group` | `ls -l`, `ps` show `?`; ownership verbs refuse | 3 | not owned by Convoy | `chown <group>` |
| adopted unit with no unit record | `scale`, `release` pre-flight | 3 | what is missing | `unit add --from-live` (L19.3) |
| `managed-by` already set by Helm or Argo | `chown` | ok | left as is | none |
| cluster without CEL admission | `caps`, `explain` | ok, marked | `admission: shell-only`, `unverified` | the platform upgrades or accepts |
| `chown` on a shell-only cluster by a non-root | pre-flight | 3 | root-only there | `sudo chown` |
| package controller with no mapping | the dependent verb | 3 | `mapping: none`, the manifest line | fix the package manifest, `pkg install` |
| children without labels (old pods, operator-created PVCs) | `ls -l` shows `child of <ancestor>` | ok | the ancestor | none; the controller labels what templates cannot |
| group recreated at the provider | `id <group>` shows a new slug and date | n/a | the old slug's objects orphaned | `sudo chown -R` (L17.5) |
| settled cost not yet available | `cost` shows `declared` with the lag | ok | `settled: pending` | none |

## 8. Acceptance criteria

- **AC-19.1** Given `cost --by group` across clusters with different cost sources, then each group's row shows declared, settled, which component allocated it and its idle share by the platform's rule, a `?` row counts unlabelled objects, and a cluster with no cost package or no admission is marked `declared only` or `unverified`, never silently smaller.
- **AC-19.2** Given `backup --pid <pid>`, then the volumes selected are those labelled with the pid or reached through ownerReferences from a labelled ancestor with that pid, the skipped ones are listed with their owner, and the receipt names the mapping used.
- **AC-19.3** Given `chown` on a foreign object, then the labels are written (as intent when the object is GitOps-managed), `managed-by` is left to any existing manager, children are labelled by the controller as they appear, and a verb that needs more than ownership refuses naming the unit record spelling.
- **AC-19.4** Given a cluster without CEL admission, then `caps` reports `admission: shell-only`, `explain` prints it on every plan that writes there, cost and holds on that cluster are marked `unverified`, and `chown` there is refused for non-root.
- **AC-19.5** Given a package whose controller selects on labels and declares no mapping, then `caps` shows `mapping: none` and the dependent verb refuses naming the manifest line; given the mapping, then the planner writes both keys and `explain` prints them.
- **AC-19.6** Given any object the shell writes, then `convoy.io/actor` is an annotation and the receipt carries the actor; no label carries a person's identity.
- **AC-19.7** Given a group deleted and re-created at the provider under the same name, then `id <group>` shows a new slug with its date, the old objects are orphaned by L17.5, and no slug is ever reused.

## 9. Persona lenses

- **PE**: wants one label set and one `caps` line per component that says whether the signal reaches it; refuses if the planner must learn each controller's keys.
- **FIN**: wants declared and settled side by side per group, idle cost allocated by a rule they can read, and `?` as a row with a count rather than a number that quietly shrinks.
- **SEC**: wants the actor out of labels, the admission mode of every cluster visible, and `chown` on an unenforced cluster restricted to root.
- **SRE**: wants `ls -l` to say "child of" instead of `?` for operator-created objects, and adoption to be one verb.
- **DE**: wants `backup --pid` to take their volumes and nobody else's.
- **DBA**: wants the operator's PVCs owned by the unit, not mislabelled with a pid, and never backed up by accident.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L19.1 | P5 | A group label on pods reaches OpenCost and the billing export only through a cost package's mapping; node idle cost and unlabelled objects belong to nobody, and a cluster with no cost package or no admission would show a smaller number without saying why. | **rule** (applied: ADR 0018 rules; command map; roadmap M2): `cost --by group` prints declared and settled per group with the allocating component, an idle share by the platform's rule (proportional to requests unless set), a `?` row with object counts per cluster, `declared only` where no cost package maps the group label, and `unverified` on clusters with `admission: shell-only`; `explain` prints the idle rule | ADR 0010 rule L7.2, ADR 0018 d6, ADR 0011 d7 |
| L19.2 | P2, P3, P12 | Labels reach only the objects the planner writes; pods of an adopted Deployment, PVCs an operator creates for a unit and Jobs a CronJob spawns carry them only if a template or the operator propagates, so `?` would appear inside Convoy's own units. | **rule** (applied: ADR 0018 rules; command map; roadmap M2): ownership is resolved by walking ownerReferences to the nearest labelled ancestor, and `ls -l` and `ps` print `child of <ancestor>` instead of `?`; the Convoy controller labels children templates cannot reach (run pids on Jobs a CronJob creates, pods of an adopted Deployment as they appear); a package declares in `caps` how it propagates `convoy.io/` keys to its children (cnpg `inheritedMetadata`, Strimzi templates) and `caps` lists `propagation: none` where it cannot | ADR 0018 d1, d5, d6; ADR 0010 |
| L19.3 | P9 | `chown` adopts a foreign Deployment by writing labels, but `scale` and `release` need a unit record (release path, digest source, probes) that adoption cannot invent. | **rule** (applied: ADR 0018 rules; ADR 0013 rules; command map; roadmap M11): adoption writes labels only; a verb that needs more refuses naming what is missing and the `unit add <name> --from-live` spelling, which writes the unit record from the object as it is and marks it `adopted` in the receipt | ADR 0018 d5, ADR 0013 (unit verbs) |
| L19.4 | P7 | A cluster whose API server cannot run CEL admission enforces nothing: anyone with write can set a group label and charge, back up or hold under another team's name; ADR 0018 says `admission: shell-only` and stops there. | **rule** (applied: ADR 0018 rules; command map; roadmap M8): on a shell-only cluster `explain` prints the mode on every plan that writes there, cost and holds are reported `unverified`, `backup` and `cost` selectors there require `--include-unverified`, and `chown` is root-only; the platform's choice to run such a cluster is listed by the inventory as an accepted risk | ADR 0018 d3, d5; ADR 0011 d7 |
| L19.5 | P4 | `convoy.io/actor` as a label is readable by anyone who can list the namespace and, on cluster-scoped objects, by everyone; L17.6 hides actors outside the group in the shell, so the label leaks what the shell conceals. | **amend** (applied: ADR 0018 d1, d2, d3, d5, d7 revised) ADR 0018 decision 1: the actor is an annotation on every object and a field of the receipt, never a label; the label set drops `convoy.io/actor`; `find --actor` reads receipts and the audit stream; this settles the open item | ADR 0018 d1, d2; ADR 0016 rule L17.6 |
| L19.6 | none | Slugs are keyed by the provider's group id, so a group deleted and re-created under the same name is a new group with a new slug and its old objects stay orphaned. | **accepted**: this is what "never reused" means; `id <group>` prints the slug, the provider id and the date, and `chown -R` under L17.5 is the move | ADR 0018 d7, ADR 0016 d9 |
| L19.7 | P1 | `app.kubernetes.io/managed-by` is already set to `Helm` or to the GitOps tool on most adopted objects and on every package's objects; writing `convoy` there breaks the other manager's tooling. | **amend** (applied: ADR 0018 d1 revised) ADR 0018 decision 1: `managed-by: convoy` is written only on objects the session creates directly and that no other manager owns; on adopted and package-managed objects the key is left as found and `convoy.io/kind` with `convoy.io/package` or `convoy.io/unit` carry the signal | ADR 0018 d1, ADR 0010 a1 |
