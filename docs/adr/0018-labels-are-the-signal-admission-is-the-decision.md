# ADR 0018: Labels are the signal to the platform; admission is the decision

**Status:** Draft · 2026-11-10 · revised 2026-11-10 with user-story findings L19.5 and L19.7 (marked inline)
**Supersedes / relates to:** docs/pipeline-compiler.md §6.3 and §9 (the `convoy.io/pid`, `session`, `plan`, `stage` labels on compiled objects); ADR 0010 amendment 1 (management detected from the GitOps tracking label); ADR 0011 decision 2 (`by-label` views) and amendment 3 (`nsmap`, the reverse recorded on the namespace; `residency`); ADR 0012 rule L12.6 (members not labelled by Convoy); ADR 0013 decision 3 (`explain` writes); ADR 0016 decisions 3 and 8 (group-owned objects, owner labels compared by CEL admission); ADR 0017 (the `Approval` resource); rule L6.6 (`?` owner for unlabelled namespaces)

## Context

The design signals the platform through labels in four places and never says that this is their job. The compiler stamps every Job and Workflow with pid, session and plan so `ps`, `cost` and the audit log need no session-side store. ADR 0016 makes the owner label the thing a CEL admission policy compares with the requester's groups. ADR 0010 reads the tracking label Argo CD or Flux sets to decide whether a mutation must be written as intent. ADR 0011 builds the `by-label` view from cluster labels and records the path-to-namespace map on the namespace. Rule L6.6 shows `?` as owner for anything unlabelled, and L12.6 refuses to join a consumer group whose members Convoy did not label.

Facts that constrain the design:

- A label is a selector: at most 63 characters of a restricted alphabet per value, and every controller on the platform, Kueue, Karpenter, OpenCost, Velero, Argo, Kyverno, the cloud's cost export, chooses what to act on by selecting labels. An annotation is free text and selects nothing.
- A label is written by anyone with write on the object. It is not a permission. What enforces is admission: a ValidatingAdmissionPolicy with CEL can refuse a create or update by comparing labels with the requester's identity and groups, and can make a label immutable.
- Kubernetes already reserves `app.kubernetes.io/` keys that tooling understands, among them `managed-by`.
- The groups claim of a login token is not label-safe: it may carry slashes, spaces and more than 63 characters.
- The shell is one writer among many. Packages, operators and people with `kubectl` create objects in the same namespaces, and the shell has to tell its own from theirs without a database.
- Receipts, plan hashes, pinned `explain` artifacts and approval states are long, change often and are never selected on.

## Decision

1. **One prefix, one fixed set, and this record owns it** (revised 2026-11-10, L19.5 and L19.7). Every object the session or its controller writes carries labels under `convoy.io/` from the set below and nothing else under that prefix. A new key is an amendment here. Packages and people do not write `convoy.io/` keys; admission refuses the write unless the requester is a session identity (ADR 0016) or the controller.

   | Key | Value | On |
   |---|---|---|
   | `convoy.io/group` | the owner group's label-safe slug (decision 7) | every object |
   | `convoy.io/kind` | `process`, `unit`, `session`, `approval`, `hold`, `package` | every object |
   | `convoy.io/pid` | the process id | processes, their pods, services, PVCs and snapshots |
   | `convoy.io/session` | the session id | processes created from an interactive session |
   | `convoy.io/plan` | the plan's shape hash (ADR 0017 decision 6) | processes |
   | `convoy.io/stage` | the stage index | compiled stage objects |
   | `convoy.io/unit` | the unit name | a unit's Deployment, Service, route, PDB, HPA |
   | `convoy.io/package` | the package name and version | objects a `pkg install` creates |
   | `convoy.io/residency` | the topology the object must stay in (ADR 0011 amendment 3) | every object that holds data or runs code |
   | `convoy.io/hold` | the hold id | PVCs, VolumeSnapshots and buckets' objects under a hold (ADR 0006) |
   | `convoy.io/path` | the mount path segment the namespace maps to (ADR 0011 amendment 3, `nsmap`) | namespaces |
   | `app.kubernetes.io/managed-by` | `convoy` | only objects the session creates directly and no other manager owns; left as found on adopted and package-managed objects, where `convoy.io/kind` with `convoy.io/package` or `convoy.io/unit` carry the signal, and absent on objects written as intent, where the GitOps tool's own tracking label is the truth (ADR 0010 amendment 1) |

   The actor is never a label (L19.5). A label is readable by anyone who can list the namespace, and on cluster-scoped objects by everyone, so a person's identity in a label would leak what rule L17.6 hides in the shell. The actor is the `convoy.io/actor` annotation on every object and a field of the receipt; `find --actor` reads receipts and the audit stream, never a selector. The `managed-by` key belongs to whichever manager set it first (L19.7): Helm, Argo CD and Flux read it, and overwriting it with `convoy` would break their tooling.

2. **Everything else is an annotation or lives in the resource.** The receipt id, the full plan hash, the pinned `explain` artifact id, the approval object path, the contract flags, the cordon reason and evidence, the intent commit, the full group name, the actor (decision 1): `convoy.io/` annotations, never labels, so the label set stays small and selectable. The `Approval` resource (ADR 0017) and the receipt carry their own fields.

3. **Labels signal; admission decides.** Three CEL policies ship with the session (M2) and are the enforcement of ADR 0016 on the API server: on create, `convoy.io/group` must be one of the requester's groups or the requester must be the controller acting for a session whose primary group it is; `convoy.io/group` and the `convoy.io/actor` annotation are immutable except through `chown`, which runs as root or as a member of both groups (ADR 0016 decision 3), and the controller writes the annotation from the requester, never from the manifest; `convoy.io/` keys are written only by session identities and the controller. A label is never a permission and no verb grants anything by writing one. ADR 0016 decision 8 names these as its mechanism.

4. **The planner writes the full set, and `explain` prints it.** Every object a plan names (ADR 0013 decision 3, writes) carries the set of decision 1 in the manifest, including manifests written as intent to a GitOps repository; the "writes" section of `explain` prints the labels of each object, so the signal is visible before it is sent. A compiled example in docs/pipeline-compiler.md §9 shows the labels of its objects.

5. **An object without the group label is foreign, and the shell only looks at it.** `ls -l` shows `?` as owner (L6.6), `ps` lists it with owner `?` (L12.6), `kill`, `rm`, `scale`, `release` and `chown` refuse with "not owned by Convoy" and name `chown` as the adoption verb; `chown <group> <path>` writes the group label and the actor annotation through admission and is receipted. A foreign object is never deleted, patched or restarted by a verb that needs ownership; the eviction path of a drain (ADR 0008 rule L16.3) is the one exception and goes through the eviction API, which asks the object's own PDB.

6. **Packages declare how the signal reaches their controller.** A platform component that selects on its own keys declares in `caps` a label mapping, `convoy.io/group` to `kueue.x-k8s.io/queue-name` through the group's local queue, `convoy.io/group` to the cost allocation key OpenCost or the cloud export reads, `convoy.io/pid` to a Velero backup selector, `convoy.io/residency` to the topology key a scheduling policy checks. The planner writes both keys and `explain` prints the mapping; a component with no mapping sees no signal, and `caps` says so. Convoy never learns a controller's keys by itself.

7. **Values are slugs the inventory assigns.** A group's label value is a slug the fleet inventory assigns once and never reuses; the full group name is an annotation and `id`, `ls -l` and receipts print the name, not the slug. Actors are not labels (decision 1) and need no slug. A slug collision is impossible by construction; a renamed group at the provider keeps its slug.

8. **Residency is labelled and checked twice.** Beyond the plan-time check of ADR 0011 amendment 3, every cluster carries a policy that refuses a pod or a PVC whose `convoy.io/residency` does not match the cluster's own topology, so a plan that slipped past the session, or an object written as intent and reconciled later, is still refused where it would land.

9. **Views read the platform's labels, not Convoy's.** The `by-label` view of the device tree (ADR 0011 decision 2) and the fleet inventory select on the cluster labels the platform already has; Convoy's keys are written on objects inside clusters, never on the devices. The two directions do not mix: the platform labels devices, the shell labels workloads.

## Declined

| Proposal | Why not | What replaces it |
|---|---|---|
| labels as permissions ("anyone with `convoy.io/group=x` may act on it") | a label is written by anyone with write; it signals, it cannot enforce | the CEL policies of decision 3 |
| a user flag `--label k=v` on creating verbs | a second channel beside names, units and env, with no reader | `--unit` names, env, annotations the platform maps |
| the receipt, plan hash and approval state as labels | long, changing, never selected on; label values are capped | annotations and the resource (decision 2) |
| a Convoy custom resource wrapping every object so the shell can find its own | plain objects stay plain for every other tool; a wrapper is a second source of truth | labels on the plain object |
| the session id as the ownership key | sessions end (ADR 0016 decision 9); ownership is the group's | `convoy.io/group`; the session id is a trace, not an owner |
| Convoy learning each controller's keys (Kueue, OpenCost, Velero) | every new component is a code change in the planner | the mapping declared in `caps` (decision 6) |
| the full group name as the label value | not label-safe; renames break selectors | a slug, with the name in an annotation (decision 7) |
| a `convoy.io/managed=true` key | `app.kubernetes.io/managed-by` already exists and tooling reads it | the standard key |

## Consequences

- The four scattered uses become one convention, and ADR 0016 decision 8 gets its mechanism: the owner label, the admission policies and `chown` are named in one place.
- Chargeback, backup, queueing and residency enforcement by third-party controllers follow from one group label and the mappings packages declare, without the planner knowing any of them.
- The reverse signal is a rule, not a habit: foreign means `?`, `?` means read-only, `chown` adopts. Every verb that needs ownership has the same refusal.
- Cost: the three CEL policies in M2, the mapping field in `caps` and the package manifest (M9), the slug registry in the inventory (M8), the residency policy per cluster (M8), the labels in every compiled example, and `explain` output that grows by one line per object.
- A cluster whose API server cannot run CEL admission (older than 1.30) gets the shell-side checks only, and `caps` reports `admission: shell-only`; `explain` prints it on every plan that creates an object there.
- Settled 2026-11-10: the actor is an annotation, not a label (L19.5, decision 1).
- Open, to be settled against this ADR: the exact slug alphabet and length; the `caps` syntax for a mapping; whether objects written as intent should carry `app.kubernetes.io/managed-by: convoy` in the manifest for the GitOps tool to keep.
- [US-19](../user-stories/US-19-label-the-fleet.md) probes this record under the Platform Engineer and FinOps personas with a cost report by group across two clouds, a Velero backup of one process, a foreign Deployment adopted by `chown`, a cluster without CEL admission and a package without a mapping. It raised seven findings; two were applied inline the same day: L19.5 (the actor is an annotation, never a label, which settled an open item; decisions 1, 2, 3, 5 and 7) and L19.7 (`managed-by` is left to any existing manager; decision 1). Four are rules, written below, in ROADMAP M2, M8 and M11 and in the command map: cost by group with declared, settled, idle, `?` and unverified rows (L19.1), ownership walked through ownerReferences with the controller labelling children templates cannot reach (L19.2), adoption writing labels only with `unit add --from-live` for the record (L19.3), and the marks and restrictions on a cluster without admission (L19.4). L19.6 accepts that a recreated group is a new group with a new slug.

## Rules · 2026-11-10 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L19.1** (US-19) `cost --by group` prints declared and settled per group with the component that allocated each (ADR 0010 rule L7.2), an idle share by the platform's rule (proportional to requests unless set), a `?` row with object counts per cluster, `declared only` where no cost package maps the group label (decision 6), and `unverified` on clusters with `admission: shell-only`; `explain` prints the idle rule.
- **L19.2** (US-19) ownership is resolved by walking ownerReferences to the nearest labelled ancestor; `ls -l` and `ps` print `child of <ancestor>` instead of `?`; the Convoy controller labels the children templates cannot reach (run pids on Jobs a CronJob creates, pods of an adopted Deployment as they appear); a package declares in `caps` how it propagates `convoy.io/` keys to its children (cnpg `inheritedMetadata`, Strimzi templates) and `caps` lists `propagation: none` where it cannot.
- **L19.3** (US-19) adoption by `chown` writes labels only; a verb that needs more (`scale`, `release`) refuses naming what is missing and the `unit add <name> --from-live` spelling, which writes the unit record from the object as it is and marks it `adopted` in the receipt (ADR 0013).
- **L19.4** (US-19) on a cluster with `admission: shell-only`, `explain` prints the mode on every plan that writes there, cost and holds are reported `unverified`, `backup` and `cost` selectors there require `--include-unverified`, and `chown` is root-only; the inventory lists the platform's choice to run such a cluster as an accepted risk.
- **L26.11** (US-26) `caps: label-keys` lists the Convoy labels a telemetry store keeps; without them joins fall back to pod names and are marked (full rule in ADR 0023).
- **L30.5, L30.10, L30.15** (US-30) admission refuses images from repositories the pod's group may not read, reports its signature mode as `verify=`, and injected containers are recorded in receipts (full rules in [ADR 0026](0026-a-registry-is-a-device-a-tag-is-a-name-for-a-digest.md)).
- **L31.11** (US-31) events carry a deduplication key from the object's path, matching the object labels alerts carry (full rule in [ADR 0027](0027-convoy-tells-a-group-about-its-own-objects.md)).
