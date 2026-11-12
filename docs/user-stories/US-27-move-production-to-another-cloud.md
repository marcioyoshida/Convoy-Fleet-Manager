# US-27 · Move production from AWS to Google Cloud: translate the infrastructure, replicate the data, cut over and back, and know the bill before we start

| Field | Value |
|---|---|
| Primary persona | PE |
| Secondary personas | FIN, DBA, SRE, SEC, DE, DO |
| Workload class | fleet operation: a provider migration of one production environment, with infrastructure as code translated between providers, data replicated one-shot and continuously, a period of dual running, a cutover with a way back, and a cost forecast before any of it starts |
| Walkthrough | written against [ADR 0011](../adr/0011-device-tree-and-mount-table.md) (clusters and stores as devices, topology in the path, `caps`), [ADR 0001](../adr/0001-pipe-is-a-staged-artifact.md) amendment 2 (the artifact store per topology), [ADR 0006](../adr/0006-detach-is-not-destroy.md) amendments 1 and 2 (retention and holds), [ADR 0007](../adr/0007-secrets-are-references.md) amendments 2 to 4 (sinks, producers, a mount's secret source), [ADR 0008](../adr/0008-signal-routing.md) (preemption as stop with checkpoint), [ADR 0010](../adr/0010-platform-defaults-and-two-package-tiers.md) amendment 1 (intent), [ADR 0012](../adr/0012-logs-and-queues-are-devices.md), [ADR 0015](../adr/0015-tables-and-catalogs-are-devices.md), [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md), [ADR 0019](../adr/0019-a-unit-has-an-address-reach-is-a-grant.md), [ADR 0023](../adr/0023-telemetry-is-a-device-the-shell-reads.md) and [ADR 0014](../adr/0014-assisted-completion-the-model-proposes.md) (the model proposes); it reuses the release of US-03, the holds of US-08, the topic of US-11, the training gang of US-05 and the labels of US-19 |
| Gaps already closed there | none |
| Milestones needed | M4, M5, M7, M8, M9, M10, M11 |
| Findings in this document | L27.1 – L27.17 |

## 1. Story

**As a** platform engineer leading the move, **I want** to take the production environment from one cloud to another without a dark weekend: translate the infrastructure it stands on, copy and then continuously replicate its data, run both sides until the new one is proven, switch traffic, and keep a way back, **so that** the company gets the commercial terms that motivated the move, nobody loses data or a legal hold, and finance signs off on a forecast that turns out to be right.

**Context.** Acme's `orders` platform runs in `prod-us`, an EKS cluster in us-east-1, built from the Terraform repository `infra/aws` (412 resources, state in S3). It holds an Aurora PostgreSQL cluster `orders-db` (2 TiB), a lake bucket with Iceberg tables in the Glue catalog (180 TiB, 41 million objects), an MSK topic `orders` with 30-day retention, an EFS file system, secrets in Secrets Manager, IRSA roles, CloudWatch logs, Karpenter with Spot GPU pools for training, and the artifact store `cas-aws-us-east-1` holding seven-year receipts and held inputs under S3 Object Lock (US-08). Google has offered committed-use credits worth the move if production runs on GKE in us-east4 within 90 days. The target is a GKE Standard cluster `prod-us-gcp`, with AlloyDB, GCS with an Iceberg REST catalog, Google Managed Service for Apache Kafka, Filestore, Secret Manager, Workload Identity, Cloud Logging and Managed Service for Prometheus, and Spot VMs for training. Users sign in with Google Workspace accounts after the move. Both regions are residency `us`. `lee` (user `lee`, group `platform`) leads; `morgan` (FIN) must approve the budget; `sam` (DBA) owns the database.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | `lee` | user `lee`, group `platform` | mounts both clusters and both accounts' stores |
| Approves the budget | `morgan` | member of `finance` | reads the forecast before anything is copied |
| Owns the database cut | `sam` | member of `orders-dba` | the write freeze and the replication lag |
| Approves cutover | `orders-leads` | members | the `release` gate (ADR 0017) |
| Applies infrastructure | CI for `infra/gcp` | `svc:platform` with the Terraform runner's credentials | outside the shell today |
| Pays | `platform` for the move; each group for its dual-running units | | |
| Audits | `audit` | | the holds, their chain of custody, and the decommission of the old account |

## 3. Preconditions

- **Capabilities** (`caps`): on `prod-us`: `node-agent`, `dcgm`, `fqdn-policy`, `admission:cel`, `cost:opencost`, `object-lock` on `cas-aws-us-east-1`; on `prod-us-gcp`: `admission:cel`, `cost:billing-export`, `metrics:promql`, `bucket-lock` on the GCS stores; `logs:cloud-logging`, `node-agent` and `dns-weights` are the questions the story asks.
- **Packages**: `transfer:storage-transfer` (Google Storage Transfer Service), `replicate:dms` (Google Database Migration Service), `replicate:mirrormaker2`.
- **Mounts**: `/clusters/prod-us`, `/clusters/prod-us-gcp`; `/s3/lake`, `/gcs/lake`; `/tables/glue`, `/tables/rest-gcp`; `/queues/msk-us`, `/queues/gmk-us`; `/src/infra` (the Terraform repository as a source device).
- **Platform defaults** `explain` must print: the price files of both providers, egress between them, the residency of every store, the retention and lock of every held artifact, each cluster's preemption notice, each cluster's `caps`.

## 4. Main path

```bash
# Week 0: what are we moving, and what will it cost?
[lee] $ cost --by group --since=-30d /clusters/prod-us
orders $41 200  ml $18 900  data $12 700  platform $6 100   (settled from the CUR)
[lee] $ ls /dev/
k8s  obj  table  queue  scm  metrics  logs  traces  model
#  the 412 resources the environment stands on (VPC, Aurora, MSK, IAM, Route 53, KMS) are in
#  Terraform's state; none of them is a device, and nothing in the design reads or writes state  (L27.1)
[lee] $ cd /src/infra/aws && run terraform show -json | head
"master_password": "Zq…"       # the state holds Aurora's master password in clear              (L27.2)
#  the assisted shell (ADR 0014) is asked to translate main.tf to google_* resources; it proposes
#  328 resources and says nothing about 37 resources in the AWS account that are not in the state
#  (a console-made bucket policy, two Route 53 records, a KMS grant)                                  (L27.3)
[lee] $ diff <(explain --at=/clusters/prod-us 'release orders-api …') <(explain --at=/clusters/prod-us-gcp 'release orders-api …')
#  the diff covers profiles, limits and egress (L24.4); it does not list that prod-us-gcp has no
#  node agent for kill -STOP, no fs probes for Filestore, no log driver for Cloud Logging, and
#  that a Spot VM gets about 30 s of notice where EC2 Spot gives two minutes                       (L27.14)
[lee] $ explain migrate.sh
band $14 900 – $19 600: cp /s3/lake → /gcs/lake (egress $9 200 – $16 600, requests $220), backup of orders topic …
#  the replication units and both environments running side by side have no end date, so the band
#  leaves them out; morgan asks what October would have cost on GCP, and no verb answers           (L27.15)
#  the AWS Savings Plan runs until March and Google's credits are not in any price file            (L27.16)

# Week 2: infrastructure
#  infra/gcp is applied by CI. lee creates two topics with mkdir under /queues/gmk-us and a bucket
#  policy with chmod; the next terraform apply deletes the policy as drift                           (L27.4)

# Week 3: data, one-shot
[lee] $ cp -r /s3/lake/ /gcs/lake/
plan: copy Job in /clusters/prod-us, 41 M objects, 180 TiB, est. 9 days at the Job's throughput   (L27.6)
[lee] $ cp -r /tables/glue/sales /tables/rest-gcp/sales
#  the data files arrive in GCS, but every manifest still names s3://acme-lake/…; the new table
#  reads S3 across clouds on every query, and the egress shows up in the next month's bill       (L27.5)

# Week 3: data, continuous
[sam] $ run --unit orders-db-repl --package=replicate:dms --from=/db/aurora/orders --to=/db/alloydb/orders
[lee] $ run --unit orders-mm2 --package=replicate:mirrormaker2 --from=/queues/msk-us/orders --to=/queues/gmk-us/orders
#  wait --lag-below has a source for a topic (L12.4), none for a database replication               (L27.7)
#  consumers moved to gmk-us resume from translated offsets; receipts and holds that name MSK
#  offsets of orders (US-11) will point at a broker that is about to be deleted                     (L27.10)
[lee] $ secret ls /secrets/orders/
db-uri  stripe-key  sqs-dlq-token        # references; values never reach the session (ADR 0007)
#  nothing in the design moves a secret's value from Secrets Manager to Secret Manager             (L27.11)

# Week 4: dual running
[lee] $ release orders-api sha256:e1f0… --to=/clusters/prod-us-gcp
[ana] $ newgrp orders-leads
newgrp: you are not a member of orders-leads
#  on the Google side ana's token carries no groups; Google ID tokens never do                     (L27.12)
[ml] $ batch --gang --checkpoint=/gcs/ckpt train.py --gpu=64 --spot     # on prod-us-gcp
#  the grace for a final checkpoint (up to 300 s) cannot hold: GKE gives Spot pods about 15 s     (L27.13)

# Week 6: cutover
[sam] $ scale orders-api 0 --to=/clusters/prod-us && wait --lag-below=0 orders-db-repl
[lee] $ release orders-api sha256:e1f0… --to=/clusters/prod-us-gcp --canary=10%
#  the canary is a Gateway weight inside one cluster; the traffic between clouds is split by
#  Route 53 weighted records, which Terraform owns and the shell cannot see                          (L27.17)
#  at 40% the GCP region has an incident; rolling back sends writes to Aurora, which stopped
#  receiving them at the freeze: nothing replicates from AlloyDB back                              (L27.8)

# Week 10: decommission
[lee] $ umount /dev/obj/by-id/cas-aws-us-east-1
#  the store holds 7-year receipts and held inputs under Object Lock; closing the AWS account
#  would destroy them, and a copy to GCS starts a new lock with no chain of custody                 (L27.9)
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| inventory and forecast | `cost`, `explain` of a script | the CUR, both price files | receipts |
| infrastructure | none today | Terraform state, plans, applies by CI | the IaC repository, the state bucket |
| one-shot copy | `cp`, `backup` | a copy Job, or the provider's transfer service | both stores; the receipt |
| continuous replication | a unit from a package | DMS, MirrorMaker 2 | the package's own state; `/sys` |
| dual running | units released to both mounts | Deployments in both clusters | the GitOps repository |
| cutover and rollback | `scale`, `wait`, `release --canary` | Gateway weights, DNS weighted records | the GitOps and IaC repositories |
| decommission | `umount`, `rm --with-data` | the old account's resources | receipts; the audit record |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe: staged artifact, content id, cache | ● | content ids survive a copy; topic-range ids do not survive a broker change (L27.10) | AC-27.10 |
| A2 | Command: class, annotation, executable fields | ○ | transfer and replication executables from packages | |
| A3 | Session: mount table, env, cwd, attach and detach | ○ | two providers' mounts in one session | |
| A4 | Backend selection | ● | a 180 TiB copy placed in a Job in the source cluster (L27.6) | AC-27.6 |
| A5 | Completion contract | ● | cutover conditions on replication lag; a gang on Spot with 15 s of grace (L27.7, L27.13) | AC-27.7, AC-27.13 |
| A6 | Device tree | ● | clusters and stores of two providers; capability parity (L27.14) | AC-27.14 |
| A7 | Signals and lifecycle | ● | preemption notice per provider (L27.13) | AC-27.13 |
| A8 | Secrets as references | ● | values held in Terraform state; values that must cross vaults (L27.2, L27.11) | AC-27.2, AC-27.11 |
| A9 | Cost | ● | the forecast of a migration with no end date; repricing a month on the target; commitments (L27.15, L27.16) | AC-27.15 |
| A10 | Identity and authorization | ● | tokens without groups (L27.12) | AC-27.12 |
| A11 | Platform defaults and `caps` | ● | two providers' defaults, notices and drivers | |
| A12 | Storage | ● | object lock that cannot be transferred (L27.9); a table whose metadata names the old store (L27.5) | AC-27.5, AC-27.9 |
| A13 | Networking | ● | traffic between clouds is DNS, owned by Terraform (L27.17) | AC-27.17 |
| A14 | Time and queueing | ○ | the 90-day deadline; replication lag over time | |
| A15 | Audit, receipts, retention and `replay` | ● | holds and receipts that outlive the account they name (L27.9, L27.10) | AC-27.9 |
| A16 | Durability | ● | a rollback with nothing replicating back (L27.8) | AC-27.8 |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative verb, declarative owner | (a) a topic and a bucket policy created from the shell on resources Terraform manages; (b) DNS weights for the cutover, which Terraform owns | (a) ADR 0010 amendment 1 writes intent to the GitOps repository, not to Terraform; (b) `release --canary` writes Gateway weights only | **L27.4**, **L27.17** |
| P2 | POSIX semantics that cannot hold | preemption as stop with a final checkpoint when the provider gives 15 s | the grace is sized from `checkpoint_seconds`, up to 300 s, on any provider | **L27.13** |
| P3 | Orphaned or lost state | replication units and the old environment after cutover | units end with `kill`; decommission is `umount` and `rm --with-data` | folded into **L27.7** and **L27.9** |
| P4 | Identity leak or escalation | (a) Terraform state holds secret values; (b) moving secret values between vaults | (a) state is a file nobody classified; (b) values never reach the session, so the design has no path | **L27.2**, **L27.11** |
| P5 | Invisible cost | (a) a table copied by bytes reads the old cloud on every query; (b) dual running and replication have no end date; (c) commitments and credits | (a) egress is priced for the copy, not for the reads its metadata causes; (b) bands cover finite stages; (c) price files are list prices | **L27.5**, **L27.15**, **L27.16** |
| P6 | Lying cache | topic-range artifacts, receipts and holds that name MSK offsets after the consumers move to translated offsets | artifact ids of topic reads include the broker's id (ADR 0012 decision 2) | **L27.10** |
| P7 | Silent failure | a copied Iceberg table that works and reads S3 | `cp` of a table path copies data files | folded into **L27.5** |
| P8 | Partial fleet | (a) the target cluster lacks capabilities the source had; (b) the target region fails during cutover | (a) each mount's `caps` are read separately; (b) `release --rollback` reverts the route, not the data | **L27.14**, **L27.8** |
| P9 | Created elsewhere | (a) the infrastructure under the environment exists only in Terraform; (b) resources in the account that are in no state | (a) no device; (b) no inventory beyond Kubernetes and mounted stores | **L27.1**, **L27.3** |
| P10 | Multi-party action | the budget approved by finance, the database cut by the DBA, the cutover by the leads | ADR 0017 approvals; the forecast and the cut have no gate of their own | folded into **L27.15** and **L27.7** |
| P11 | Data at rest | held artifacts under Object Lock in an account about to close | holds forbid deletion (ADR 0006); nothing says how a hold moves | **L27.9** |
| P12 | Process-tree inheritance | groups on the target provider, where tokens carry none | ADR 0016 reads groups from the login token | **L27.12** |

A thirteenth question: what is "the same environment" on another provider? Aurora and AlloyDB, MSK and Google's Kafka, IAM roles and service accounts are not equivalent, and no table of resource types makes them so. The translation of L27.1 can only be a proposal that a person reviews and a target plan that proves it, never an equivalence; that is the shape the open finding asks for.

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| secret values in Terraform state | the scrub on read (L27.2) | none | attributes withheld, the rule | none needed |
| account resources missing from the state | the inventory diff (L27.3) | 0 degraded | each resource, its type, its creator where known | import, or accept as out of scope |
| shell write to a Terraform-managed resource | plan (L27.4) | 3 under `iac=refuse`, else intent | the resource address in state | the IaC pull request |
| table copied by bytes | plan (L27.5) | 3 | the manifests that name the source store | the typed table copy |
| replication lag with no source | plan (L27.7) | 3 | the unit and the missing `/sys` file | the package's lag source |
| rollback with no reverse replication | plan (L27.8) | 3 | the units that would lose writes | start the reverse unit |
| decommission with holds | plan (L27.9) | 3 | each hold, its expiry, its store | move the holds first |
| receipts naming a deleted broker | `replay` (L27.10) | 3 | the broker id and the hold copy, if any | `--hold` before decommission |
| capabilities lost at the target | `release --to` (L27.14) | 3 under `--strict`, else a warning | each capability and the verbs it disables | install, or accept |
| a gang on a pool whose notice is below its checkpoint time | plan (L27.13) | 3 | the notice, the checkpoint interval | a shorter interval or another pool |

## 8. Acceptance criteria

- **AC-27.1** Given a Terraform state and its repository, then an infrastructure device lists the resources with addresses, providers and dependencies; a translation to another provider is a proposal (ADR 0014) reviewed as a diff; and the target's `terraform plan` is the `explain` the apply is gated on.
- **AC-27.2** Given a read of Terraform state or plan output, then attributes the provider marks sensitive and values matching the scrub rules are withheld by name, and the state is classed as a secret sink.
- **AC-27.3** Given an infrastructure device and the provider account's own inventory, then resources in the account that no state holds are listed with their type before a translation is proposed.
- **AC-27.4** Given a shell write to a resource whose address is in a Terraform state, then it is written as intent to the IaC repository (a pull request), or refused where the platform sets `iac=refuse`.
- **AC-27.5** Given `cp` of a table path to another store, then the copy rewrites the table's metadata to the target's paths or registers a new table from the copied files, and a byte copy of a table is refused, naming the manifests that would still point at the source.
- **AC-27.6** Given a cross-provider copy above the platform's `transfer_min`, then the planner uses a transfer executable from a package where one exists, prices egress and requests on both sides, and the receipt verifies object counts and checksums.
- **AC-27.7** Given a replication unit, then its lag is read from `/sys/unit/<name>/lag` provided by its package, and `wait --lag-below=<n> --for=<duration>` is a valid cutover condition.
- **AC-27.8** Given `release --to=<mount>` that moves the writes of a unit whose data is replicated, then a rollback is refused unless a reverse replication unit is running with its lag below the bound.
- **AC-27.9** Given `umount` or decommission of a store with holds, then it is refused naming each hold; a hold moved to another store keeps its original expiry under the target's lock, and the receipt records the chain of custody (source, target, checksums, times, actors).
- **AC-27.10** Given a broker device about to be removed, then `ls -l` lists the receipts and holds that name its offsets, and `--hold` copies the held ranges to artifacts before removal.
- **AC-27.11** Given a mount whose secret source changes provider, then `secret move` rotates each secret on the target by default, and copies values only with `--copy-values`, controller-side between the two vaults under an approved grant.
- **AC-27.12** Given a login token with no groups or a group overage, then the controller resolves groups from the provider's directory API, caches them with a TTL, and refuses group-gated actions with the reason when the directory is unreachable.
- **AC-27.13** Given a pool with a preemption notice shorter than a process's final checkpoint, then the plan says that a preemption loses up to one checkpoint interval, and `--gang` refuses a checkpoint interval longer than the platform's bound for that pool.
- **AC-27.14** Given two cluster mounts, then `diff` of `explain --at` lists the `caps` that differ and the verbs and conditions each difference disables, and `release --to` to the target prints them.
- **AC-27.15** Given `explain` of a script that starts units, then the band includes each unit's daily cost over `--horizon=<duration>`; and `cost --reprice-at=<mount> --since=<range>` reprices settled receipts with the target's price files, marking resources with no counterpart.
- **AC-27.17** Given a `release --canary` across clusters on different providers, then the weights are written as DNS records through `caps: dns-weights` (as intent to the IaC repository where the zone is in a Terraform state), or the plan is refused naming the zone.

## 9. Persona lenses

- **PE**: wants the infrastructure under the environment to be visible from the shell, the translation proposed and proven by a plan rather than trusted, and every capability the target lacks listed before the release.
- **FIN**: wants one forecast that includes the months of dual running and the copies, a repricing of a real month on the target, and the commitments it cannot see named as such.
- **DBA**: wants the cut to wait on replication lag, and no cutover without replication back.
- **SRE**: wants rollback to be real, and preemption on the new provider described honestly.
- **SEC**: wants Terraform state treated as the secret sink it is, secrets rotated rather than copied on the move, and groups resolved on the new identity provider.
- **DE**: wants a moved table that reads its own store and topic receipts that still mean something after the old broker is gone.
- **DO**: wants traffic between clouds shifted where the DNS is owned, as intent, never behind the IaC repository's back.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L27.1 | P9 | The infrastructure the environment stands on (VPC, managed databases, brokers, IAM, DNS, keys) exists only in Terraform state; the design places Terraform outside Convoy (debate-review §holds), so the shell can neither see it nor help translate it, and a provider move is mostly that translation. | **open** (drafted 2026-11-12 in [ADR 0024](../adr/0024-a-device-is-installed-by-terraform.md): Terraform is the install phase of every device; a device package carries a contract and a module per provider, `mknod` installs through the platform's runner with the plan as `explain`, state is a device, and a device moves to another provider as a new instance of the same contract): the design needs Terraform as part of the shell: state and plans as an infrastructure device (`/dev/iac/`) with resources as paths and dependencies as `ls -l`; translation between providers as a proposal under ADR 0014 (the model proposes, a person reviews the diff), with mappings per resource type from packages and the resources with no counterpart named; the target's `terraform plan` as the `explain` of the apply, which runs through the platform's runner and its approvals | debate-review; ADR 0010; ADR 0011; ADR 0014; S9 |
| L27.2 | P4 | Terraform state holds secret values in clear (a database master password, generated keys); reading it from the shell would put them on the terminal and in the journal. | **rule** (applied: ADR 0007 rules): Terraform state and plan output are a secret sink: reads pass the scrub and withhold attributes the provider marks sensitive by name; the state device is readable by the platform group only | ADR 0007 a2; ADR 0020 L21.2 |
| L27.3 | P9 | Resources made in the account outside Terraform are in no state, so a translation from state silently leaves them behind. | **rule** (applied: ADR 0011 rules): an infrastructure device compares the state with the provider account's own inventory (AWS Config or Resource Explorer, Cloud Asset Inventory, Azure Resource Graph) and lists resources that no state holds before any translation is proposed; the result is degraded when the inventory is unavailable | ADR 0011 d7; ADR 0018 |
| L27.4 | P1 | ADR 0010 amendment 1 writes intent to the GitOps repository for objects a reconciler owns; resources Terraform owns get a direct write from the shell, which the next apply reverts or deletes as drift. | **amend** (applied: ADR 0010 a1 revised) ADR 0010 amendment 1: a write to a resource whose address is in a Terraform state the platform names is written as intent to the IaC repository (a pull request against the module that declares it) and waits for the apply, or is refused where the platform sets `iac=refuse`; `explain` prints the resource address | ADR 0010 a1; S4 |
| L27.5 | P5, P7 | An Iceberg table's metadata names its files by absolute URI; a byte copy to another store produces a table that works and reads the old cloud on every query, at egress rates, from then on. | **rule** (applied: ADR 0015 rules): `cp` of a table path to another store is a typed copy that rewrites the metadata to the target's paths (or registers a new table from the copied files), and a byte copy of a table is refused, naming the manifests that would still point at the source; `explain` of any table read prices the files outside the stage's store | ADR 0015 d2, d4; ADR 0011 |
| L27.6 | P5 | A 180 TiB, 41-million-object copy is planned as a copy Job in the source cluster, at that Job's throughput, while each provider has a transfer service that is faster and priced differently. | **rule** (applied: ADR 0003 rules; command map): a cross-provider copy above the platform's `transfer_min` uses a transfer executable from a package (Storage Transfer Service, DataSync, AzCopy) where `caps` lists one, prices egress and requests on both sides, and verifies object counts and checksums in the receipt | ADR 0003 a1; ADR 0011; debate-review |
| L27.7 | P3, P10 | Continuous replication is a unit from a package, but only a topic has a lag source (L12.4); a database cutover has no condition to wait on, and the write freeze is improvised. | **rule** (applied: ADR 0012 rules; ADR 0013 rules): a replication package provides `/sys/unit/<name>/lag` (seconds and bytes behind); `wait --lag-below=<n> --for=<duration>` is the cutover condition; the cutover is spelled `scale <writers> 0 && wait --lag-below=0 --for=30s <repl> && release …`, which `explain` checks end to end | ADR 0012; ADR 0013 d5; L12.4 |
| L27.8 | P8 | `release --rollback` reverts the route, not the data; after a cutover, writes go to the target, and rolling back sends traffic to a database that stopped receiving them. | **rule** (applied: ADR 0013 rules): a `release --to` that moves the writes of a unit whose data is replicated requires a reverse replication unit running before the switch, and `--rollback` is refused unless its lag is below the bound | ADR 0013 d7; ADR 0006 |
| L27.9 | P11 | Held artifacts are under the source store's object lock, which cannot be transferred; closing the account destroys them, and a copy starts a new lock with no proof it is the same evidence. | **rule** (applied: ADR 0006 rules): `umount` or decommission of a store with holds is refused naming each hold; `hold move <store>` copies held artifacts to a store with `caps: object-lock` or `bucket-lock`, sets the target lock to the original expiry, verifies content ids, and records the chain of custody (source, target, checksums, times, actors) in the hold's receipt | ADR 0006 a1, a2; ADR 0001 a2; L8.6 |
| L27.10 | P6 | Topic-range artifacts, receipts and holds name the broker's cluster id and its offsets; after consumers move to the target broker with translated offsets, those receipts point at a broker about to be deleted, and `replay` will fail. | **rule** (applied: ADR 0012 rules): `ls -l` of a broker device lists the receipts and holds that name its offsets; removing the device is refused while holds name it, and `--hold` copies the held ranges to artifacts first; offset translation from the replication package is recorded so a receipt can say which target offsets correspond | ADR 0012 d2; ADR 0006 a2; L11.4 |
| L27.11 | P4 | Values never reach the session (ADR 0007), so nothing in the design moves a secret from one provider's vault to another's. | **rule** (applied: ADR 0007 rules): `secret move <path> --to=<source>` rotates on the target by default (a new value from the producer of amendment 3, the old version retired after cutover); `--copy-values` copies controller-side between the two vaults under an approved grant (ADR 0017), never through the session | ADR 0007 a3, a4; ADR 0017 |
| L27.12 | P12, P10 | ADR 0016 takes groups from the login token, but Google ID tokens carry none, and Entra replaces a long group list with an overage link; on the target, users have no groups. | **amend** (applied: ADR 0016 d1 revised) ADR 0016 decision 1: when the token carries no groups or an overage claim, the controller resolves groups from the provider's directory API (Cloud Identity Groups, Microsoft Graph), caches them with the token's lifetime, and refuses group-gated actions with the reason when the directory is unreachable; `id` prints the source | ADR 0016 d1, d2; S15 |
| L27.13 | P2 | Preemption is a stop with a final checkpoint, with grace up to 300 s; Spot VMs on GCP and Azure give about 30 s and GKE gives pods about 15 s, so a final checkpoint of a large gang cannot complete. | **amend** (applied: ADR 0008 revised) ADR 0008 (preemption): the grace is bounded by the pool's preemption notice, which `/sys/k8s/<id>/preemption_notice` carries per node pool; where the final checkpoint does not fit, a preemption loses up to one checkpoint interval and the plan says so; `--gang` on such a pool refuses a checkpoint interval longer than the platform's `spot_loss_max` | ADR 0008; ADR 0004 a1; S10 |
| L27.14 | P8 | The release diff of L24.4 compares profiles, not clusters' capabilities; a target without a node agent, health probes for its filesystem, a log driver or a span capability is found one refusal at a time. | **rule** (applied: ADR 0013 rules): `diff` of `explain --at` covers the two cluster devices' `caps`, and `release --to` prints each capability the target lacks with the verbs and conditions it disables (`kill -STOP`, `health`, `wait --slo` on logs, `expose --global`); refused under `--strict` | ADR 0013 d3; ADR 0011; L24.4 |
| L27.15 | P5, P10 | A migration's cost is mostly units with no end date (replication, both environments running); a script's band covers finite stages, and nothing reprices real usage on the target. | **rule** (applied: ADR 0013 rules; ADR 0010 rules): `explain` of a script includes each unit it starts at its daily cost over `--horizon=<duration>`; `cost --reprice-at=<mount> --since=<range>` reprices settled receipts with the target's price files (compute, storage, requests, egress), marking resources with no counterpart; the forecast is an artifact the budget approval pins (ADR 0017) | ADR 0013 d7; ADR 0010; ADR 0017; L7.2 |
| L27.16 | P5 | Commitments, credits, negotiated discounts and reservations that outlive the move are billing objects the price files do not hold. | **accepted**: forecasts use list prices, or the effective rates a billing export gives where the cost source provides them (L7.2), and say which; commitments and credits are finance's to apply; `explain` names them as not included | ADR 0010; L7.2 |
| L27.17 | P1 | Traffic between providers is split by DNS weighted records, not a Gateway; `release --canary` writes weights inside one cluster, and the zone is owned by Terraform. | **rule** (applied: ADR 0019 rules): a `release --canary` whose targets are on different providers writes weighted records through `caps: dns-weights`, as intent to the IaC repository where the zone is in a Terraform state (L27.4), and waits for propagation within the records' TTL; without the capability it is refused naming the zone | ADR 0019 d2; ADR 0013 d7; L27.4 |
