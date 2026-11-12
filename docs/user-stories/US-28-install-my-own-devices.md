# US-28 · Install my team's own devices from the shell, adopt the ones we made by hand, and never let a plan destroy data nobody asked to touch

| Field | Value |
|---|---|
| Primary persona | DE |
| Secondary personas | PE, SEC, FIN, DO, SRE |
| Workload class | fleet operation: self-service infrastructure through device packages, a delegated module path, the platform's runner and approvals, adoption of hand-made resources, a contract change, a package upgrade and a move to another provider |
| Walkthrough | written against [ADR 0024](../adr/0024-a-device-is-installed-by-terraform.md) (all ten decisions), with [ADR 0017](../adr/0017-approvals-are-objects-on-the-process.md) and rule L18.3 (enumerated void reasons), [ADR 0010](../adr/0010-platform-defaults-and-two-package-tiers.md) and its amendment 1 as revised by L27.4, [ADR 0011](../adr/0011-device-tree-and-mount-table.md) decisions 7 and 8, [ADR 0006](../adr/0006-detach-is-not-destroy.md), [ADR 0007](../adr/0007-secrets-are-references.md) amendment 2, [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md) decision 5 and [ADR 0004](../adr/0004-completion-contract-and-exit-codes.md); it follows US-27 |
| Gaps already closed there | none |
| Milestones needed | M5, M7, M8, M9 |
| Findings in this document | L28.1 – L28.15 |

## 1. Story

**As a** data engineer, **I want** to create the queues, buckets and catalogs my team's pipelines need from the shell I already work in, change them as the pipelines grow, bring the ones we once made in the console under the same management, and move one to another provider when the business asks, **so that** I stop filing tickets and waiting days for the platform team, and the platform team stops discovering resources nobody declared, while nothing I do can delete data or grant rights beyond my team.

**Context.** After US-27 the platform runs ADR 0024. Infrastructure lives in the repository `acme/iac`, applied by Atlantis (`caps: iac:atlantis`), which plans each directory as one workspace and requires a review from the directory's code owners before it applies. The platform delegates `iac/groups/data/` to the `data` group: its runner identity is `svc:data-runner`, and its code owners are `data-leads`. Policy sets `iac=required` for the `queue` and `obj` classes, and a `prod-*` gate on `mknod` with `platform-iac` as approvers. `noor` (user `noor`, group `data`) needs a click-stream topic in us-east-1 and an Iceberg catalog in eu-west-1. The team has a bucket `acme-exports` that someone made in the console in 2024, with CORS rules and an S3 event notification to a Lambda function. The `kafka` device package is at 1.3; 2.0, released this week, renames `size_class` to `throughput_tier`.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | `noor` | user `noor`, group `data` | |
| Reviews the pull request | `data-leads` | GitHub accounts mapped to the IdP | code owners of `iac/groups/data/` |
| Approves production installs | `platform-iac` | members | the ADR 0017 gate on `mknod` to `prod-*` |
| Applies | Atlantis | `svc:data-runner` for the delegated path, `svc:platform` elsewhere | |
| Pays | `data` | devices, plan runs | |
| Operates the platform | platform team | the device packages, the delegation, the runner and its policies | |

## 3. Preconditions

- **Capabilities** (`caps`): `iac:atlantis`; `obj:s3` with `object-lock`; `queue:kafka-msk`; `table:glue`; on GCP `queue:pubsub`.
- **Packages**: `kafka` 1.3 and 2.0 with modules `aws-msk`, `gcp-managed-kafka`, `gcp-pubsub` (`unsupported: compaction, partitions`); `bucket` with `aws-s3`, `gcp-gcs`; `iceberg-catalog` with `aws-glue`, `rest`.
- **Platform defaults** `explain` must print: `iac_repo`, the delegated path and runner identity, `iac` per class, the approval policy on `mknod`, the class defaults (`retention_default=30d` for queues, `residency` per mount), the state backend.

## 4. Main path

```bash
# Monday 09:00
[noor] $ explain mknod /dev/queue/clicks kafka:aws-msk --var partitions=12 --owner=data
plan (draft PR acme/iac#1204, iac/groups/data): + 7 resources: aws_msk_cluster, 2 topics, IAM …
retention 7d (module default)                                                                     (L28.12)
band $0.38/h – ?        # the broker's storage and throughput depend on traffic                   (L28.8)
#  each explain opened a new draft PR and a plan run; noor ran explain four times               (L28.8)
[noor] $ mknod /dev/queue/clicks kafka:aws-msk --var partitions=12 --var retention=30d --owner=data
gate: mknod to prod-us needs platform-iac (policy prod-iac) and the PR needs a data-leads review  (L28.1)
#  two people approve the same change in two places, and nothing ties the two answers together
#  10:40 both approved; Atlantis holds the PR: the workspace is locked by acme/iac#1199
[noor] $ ps
PID   TYPE   STATE
8810  mknod  running          # waiting on what, for how long, and can kill stop it?             (L28.2)
#  13:10 #1199 merges; Atlantis re-plans #1204 against the new main: a different plan, and the
#  approval names the old hash                                                                     (L28.3)

# 14:05 apply
#  the MSK cluster is created; the IAM policy fails (eventual consistency on the new role);
#  Atlantis reports the apply failed; the cluster runs, costs $9 a day, and no device exists        (L28.6)
#  a clean plan did not predict it                                                                  (L28.15)

# Tuesday: the catalog, and a lock that is not the lock it says
[noor] $ mknod /dev/table/lake-eu iceberg-catalog:aws-glue --residency=eu --owner=data
[noor] $ cat /sys/obj/by-id/s3-acme-lake-eu/caps
object-lock
#  the module output says object-lock; the bucket's lock is GOVERNANCE mode, which a privileged
#  identity can lift; holds (L27.9) rely on COMPLIANCE                                              (L28.4)

# Wednesday: an unrelated change replaces a topic
[ravi] $ echo 18 > /sys/queue/by-id/msk-clicks/iac/vars/partitions        # a colleague, same workspace
#  partitions had been raised to 24 in the console during an incident; Atlantis plans the whole
#  directory: the topic resource must be replaced to go from 24 to 18, and the plan shows
#  "-/+ replace" on aws_msk_topic.clicks, which holds 30 days of events                           (L28.5)

# Thursday: adopt the hand-made bucket
[noor] $ mknod --adopt /dev/obj/by-id/s3-acme-exports bucket:aws-s3
plan: import 1 bucket; ~ update: cors_rule removed, notification removed
#  the contract has no CORS or notification inputs; adoption would delete both                     (L28.10)
#  the plan is posted as a comment on acme/iac#1231, readable by every engineer in the org,
#  with the Lambda's ARN and the bucket's tags                                                     (L28.11)

# Thursday: the delegated path is wider than it looks
#  the kafka module has an input extra_iam_members for consumers; a contractor in data sets it to
#  a role in the finance account, and svc:data-runner may create IAM bindings                       (L28.7)

# Friday: the package upgrade and the move
[noor] $ pkg upgrade kafka 2.0
#  every instance of kafka in the repository now plans against 2.0; size_class is gone; seven
#  teams' PRs show changes they did not make                                                       (L28.13)
[noor] $ mknod /dev/queue/clicks-gcp kafka:gcp-pubsub --like=/dev/queue/by-id/msk-clicks
plan: + topic, + subscription        unsupported: compaction, partitions
#  the plan applies; the dedup consumer relies on the compacted topic and silently re-reads
#  every key                                                                                         (L28.14)

# the runner is down for maintenance on Saturday
[noor] $ mknod /dev/obj/scratch bucket:aws-s3
#  hangs; a colleague creates the bucket in the console "for now"                                   (L28.9)
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `explain mknod` | a draft pull request and a plan | a plan run in the runner | the PR, the runner |
| `mknod` | a process with an approval | a merged PR and an apply | the IaC repository, the state backend, the inventory |
| a contract change | a write to `/sys/<class>/<id>/iac/vars/` | a PR, a plan, an apply | the same |
| adoption | `mknod --adopt` | `import` blocks | the same |
| a move | `mknod --like` | a module instance for another provider | the same |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe: staged artifact, content id, cache | ○ | plans reused per input hash (L28.8) | |
| A2 | Command: class, annotation, executable fields | ○ | | |
| A3 | Session: mount table, env, cwd, attach and detach | ○ | `mknod` outlives a detach | |
| A4 | Backend selection | – | | |
| A5 | Completion contract | ● | a long-running `mknod`, its waits and its kill (L28.2); a partial apply (L28.6) | AC-28.2, AC-28.6 |
| A6 | Device tree | ● | registration, `caps` claimed by a module (L28.4), `iac: partial`, adoption (L28.10) | AC-28.4, AC-28.10 |
| A7 | Signals and lifecycle | ● | `kill` before and during an apply (L28.2) | AC-28.2 |
| A8 | Secrets as references | ○ | sensitive contract inputs (L28.11) | AC-28.11 |
| A9 | Cost | ● | usage-priced devices, plan runs, a partial apply that bills (L28.6, L28.8) | AC-28.8 |
| A10 | Identity and authorization | ● | two approval gates (L28.1); a delegated runner identity (L28.7) | AC-28.1, AC-28.7 |
| A11 | Platform defaults and `caps` | ● | module defaults against platform defaults (L28.12) | AC-28.12 |
| A12 | Storage | ● | a topic replaced by an unrelated plan (L28.5); lock modes (L28.4) | AC-28.5 |
| A13 | Networking | – | | |
| A14 | Time and queueing | ● | runner locks and outages (L28.2, L28.9) | AC-28.9 |
| A15 | Audit, receipts, retention and `replay` | ● | the approval and the plan hash (L28.1, L28.3) | AC-28.3 |
| A16 | Durability | ○ | an apply that fails halfway | |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative verb, declarative owner | (a) the runner plans a whole directory, so one change's plan carries another device's drift; (b) a package upgrade re-plans every instance | (a) decision 4 prints "the plan" as if it were the change's; (b) packages are upgraded by `pkg` (ADR 0010) | **L28.5**, **L28.13** |
| P2 | POSIX semantics that cannot hold | `mknod` takes hours, waits on locks and reviews, and cannot be interrupted mid-apply | decision 3 says it "waits for the runner's plan and apply" | **L28.2** |
| P3 | Orphaned or lost state | an apply that created some resources and failed | decision 5 registers the device on apply | **L28.6** |
| P4 | Identity leak or escalation | a module input that names principals; a delegated path that accepts any resource | decision 10 scopes the runner identity to the path, not to the package | **L28.7** |
| P5 | Invisible cost | usage-driven prices; a plan run per `explain` | decision 4 prices the plan's resources | **L28.8** |
| P6 | Lying cache | (a) an approval pinned to a plan the runner replaced; (b) `caps` taken from a module's outputs | (a) the apply runs only for the pinned hash, and then nothing; (b) decision 5 registers outputs | **L28.3**, **L28.4** |
| P7 | Silent failure | `--like` onto a module that does not support an input the source uses | decision 9: `explain` prints the unsupported inputs | **L28.14** |
| P8 | Partial fleet | the runner is down | decision 10 refuses without `caps: iac`, not when the runner is unreachable | **L28.9** |
| P9 | Created elsewhere | adopting a bucket with settings the contract does not model | decision 8 promises a no-change plan | **L28.10** |
| P10 | Multi-party action | a Convoy approval and a repository review for the same change | decision 4 creates an approval object; the runner requires its own review | **L28.1** |
| P11 | Data at rest | plans posted as pull-request comments | decision 7 scrubs state reads in the shell, not what the runner publishes | **L28.11** |
| P12 | Process-tree inheritance | contract inputs the user omits: the module's default or the platform's? | decision 2 does not say | **L28.12** |

A thirteenth question: a clean plan does not guarantee a clean apply (eventual consistency, quotas, API-side validation). **L28.15**, accepted.

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| plan changed after approval | the runner's re-plan (L28.3) | none; the approval is void | both hashes and the diff | answer the new approval |
| plan replaces a data-holding resource the change did not name | plan (L28.5) | 3 | the resource, the device, the cause (drift) | reconcile the drift first |
| partial apply | the runner (L28.6) | 1 | the resources that exist and their cost | retry, or `rm --with-data` |
| `caps` claimed and not found | registration (L28.4) | 0 degraded | the claim and the provider's setting | fix the module |
| inputs naming principals outside the group | runner policy (L28.7) | 3 | the input and the rule | none |
| adoption plan with changes | plan (L28.10) | 3 | each attribute the contract would remove | `extra` passthrough or accept the loss |
| `--like` dropping inputs | plan (L28.14) | 3 without `--drop` | the inputs and the consumers that use them | `--drop=<input>` |
| runner unreachable | `mknod` (L28.9) | 2 at `--deadline` | the runner and its last contact | wait; never the console |

## 8. Acceptance criteria

- **AC-28.1** Given a `mknod` whose path has code owners and whose target has an approval policy, then one approval object records both: a review from a code owner mapped to a member of the approving group satisfies it, and an `approve` in a session posts the review; the approver can never be the author.
- **AC-28.2** Given a running `mknod`, then `ps` shows its stage (drafting, planning, awaiting approval, queued behind a lock with the holder, applying, registering) and its age; `kill` before the apply closes the pull request; `kill` during an apply is deferred until the apply ends and says so; `--deadline` applies to the wait, not to an apply in progress.
- **AC-28.3** Given an approval pinned to a plan hash and a re-plan with another hash, then the approval is void with reason `plan changed`, and the new request shows the diff between the two plans.
- **AC-28.4** Given an apply, then each `caps` entry the module's outputs claim is verified against the provider's API before the device registers, and a claim that does not hold is dropped and reported.
- **AC-28.5** Given a plan for a change, then every destroy or replace of a data-holding resource that the change did not name is refused, naming the resource, its device and the drift that causes it; the platform default `iac_workspace=per-device` gives each instance its own workspace.
- **AC-28.6** Given an apply that fails after creating resources, then the device registers as `iac: partial` with the resources that exist, priced in `df` and `cost`, and a retry or `rm --with-data` completes or removes it.
- **AC-28.7** Given a delegated path, then the runner's policy accepts only instances of the platform's device packages, and contract inputs that name principals are limited to the owning group's service identities.
- **AC-28.8** Given `explain mknod`, then usage-priced resources are banded from the contract inputs with stated assumptions or an upper bound, and a plan is reused for the same commit and inputs instead of a new draft and plan run.
- **AC-28.9** Given an unreachable runner, then `explain mknod` and `mknod` report it with its last contact, `mknod` waits within `--deadline`, and use of existing devices is unaffected.
- **AC-28.10** Given `mknod --adopt`, then adoption completes only on a plan with no changes; attributes the contract does not model are listed, and the instance may carry them through the package's `extra` passthrough.
- **AC-28.11** Given a runner that publishes plans, then `caps: iac` declares where and to whom, `explain` warns when that audience is wider than the owning group, and sensitive contract inputs are passed as references.
- **AC-28.12** Given a contract input the user omits, then the platform's class default fills it before any module default, and `explain` prints each input's source.
- **AC-28.13** Given `pkg upgrade` of a device package, then no instance re-plans; each instance shows `upgrade available` and is upgraded by a change to its own module version, with `explain` showing the plan.
- **AC-28.14** Given `mknod --like` onto a module that lacks an input the source uses, then the plan is refused unless `--drop=<input>` names it, and the receipt records what was dropped.

## 9. Persona lenses

- **DE**: wants to create and change their own devices in minutes, see what a `mknod` is waiting on, and never meet a destroy they did not ask for.
- **PE**: wants self-service that stays inside approved packages, `caps` that are verified rather than claimed, no resources created around a down runner, and package upgrades that do not re-plan the world.
- **SEC**: wants one approval record per change, no principals beyond the group in contract inputs, lock modes that mean what holds need, and plans published only where the owning group can read them.
- **FIN**: wants usage-priced devices banded honestly, partial applies visible in the bill, and plan runs that are not repeated for every `explain`.
- **DO**: wants the runner's locks, re-plans and review flow to be the source of truth the shell reports, not a second process beside it.
- **SRE**: wants an apply never to replace a topic during an unrelated change, and `kill` never to leave a half-applied state.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L28.1 | P10 | A `mknod` to a gated target needs a Convoy approval (ADR 0017) and the runner needs a code-owner review of the pull request; two people approve the same change in two places, and nothing ties the answers together. | **amend** ADR 0024 decision 4: the approval object is the record for both; a pull-request review by a code owner whose account maps to a member of the approving group satisfies it, `approve` in a session posts the review, and the author can satisfy neither (ADR 0017 decision 5) | ADR 0024 d4; ADR 0017 d1, d5 |
| L28.2 | P2 | `mknod` waits for reviews, locks and an apply that may take an hour, and an apply cannot be interrupted safely; the shell shows a process with no stage and an undefined `kill`. | **rule** (ADR 0024 rules; ADR 0013 rules): `ps` and `status` show a `mknod`'s stage (drafting, planning, awaiting approval, queued behind a lock with its holder, applying, registering) and age; `kill` before the apply closes the pull request; `kill` during an apply is deferred to the end of the apply and says so; `--deadline` binds the waits, never an apply in progress | ADR 0024 d3, d5; ADR 0004; ADR 0013 d5 |
| L28.3 | P6 | The runner re-plans when the base branch moves, so the plan that applies is not the one approved; decision 4 runs the apply only for the pinned hash and says nothing about what happens next. | **rule** (ADR 0024 rules; ADR 0017 rules): `plan changed` is an enumerated void reason (rule L18.3); the new approval request shows the diff between the two plans, and an unchanged diff for the device's own resources may be answered by the same approvers within the policy's reuse window | ADR 0024 d4; ADR 0017; L18.3 |
| L28.4 | P6 | `caps` are taken from the module's outputs, which are claims: a module reported `object-lock` for a bucket whose lock is governance mode, which holds (L27.9) cannot rely on. | **rule** (ADR 0024 rules; ADR 0011 rules): before a device registers, the driver verifies each claimed `caps` entry against the provider's API (lock mode, encryption key, versioning, replication); a claim that does not hold is dropped and reported, and the device registers degraded | ADR 0024 d5; ADR 0011 d7; ADR 0006 |
| L28.5 | P1 | The runner plans a whole directory as one workspace, so a change's plan carries every other instance's drift, including a replace of a topic holding 30 days of events. | **rule** (ADR 0024 rules): every destroy or replace of a data-holding resource in a plan that the change did not name is refused, naming the resource, its device and the drift; the platform default `iac_workspace=per-device` gives each module instance its own workspace, and `explain` prints the plan for the change's own instance apart from the rest | ADR 0024 d4, d6; ADR 0006 |
| L28.6 | P3 | An apply that fails halfway leaves resources that run and bill with no device, since decision 5 registers only on success. | **rule** (ADR 0024 rules): a partial apply registers the device as `iac: partial` with the resources that exist, priced in `df` and `cost` and listed in the receipt; `mknod --retry` completes it and `rm --with-data` removes it | ADR 0024 d5; ADR 0006; ADR 0009 |
| L28.7 | P4 | A delegated module path accepts any resource the runner identity may create, and a module input that names principals lets a group grant roles outside itself. | **rule** (ADR 0024 rules; ADR 0016 rules): the runner's policy on a delegated path accepts only instances of the platform's device packages; contract inputs that name principals are limited to the owning group's service identities (ADR 0016 decision 5); the runner identity is bounded by the platform to the resource types the package's modules declare | ADR 0024 d2, d10; ADR 0016 d5 |
| L28.8 | P5 | A plan prices resources, not usage: a broker's storage and throughput cost depend on traffic; and every `explain` opens a draft and a plan run. | **rule** (ADR 0024 rules): usage-priced resources are banded from contract inputs (retention, throughput tier) with the assumptions printed, or from the class's upper bound; a plan is reused for the same commit and inputs, and draft pull requests opened by `explain` are closed when the session ends | ADR 0024 d4; ADR 0010 |
| L28.9 | P8 | `mknod` is refused without `caps: iac` but hangs when the runner is down, and a colleague works around it in the console. | **rule** (ADR 0024 rules): an unreachable runner is reported by `explain` and `mknod` with its last contact; `mknod` waits within `--deadline` and exits 2; use of existing devices is unaffected; under `iac=required` a device that appears in the account outside the runner is listed by L27.3's comparison and marked for adoption | ADR 0024 d8, d10; L27.3 |
| L28.10 | P9 | Adoption promises a plan with no changes, but a hand-made resource has settings the contract does not model, and the adoption plan deletes them. | **rule** (ADR 0024 rules): `mknod --adopt` completes only on a plan with no changes; attributes the contract does not model are listed, and the instance may carry them through the package's `extra` passthrough input, which `explain` marks as outside the contract and `--like` does not carry | ADR 0024 d8 |
| L28.11 | P11 | The runner publishes plans as pull-request comments readable across the organisation; the scrub of decision 7 covers what the shell reads, not what the runner posts. | **rule** (ADR 0024 rules; ADR 0007 rules): `caps: iac` declares where plans are published and to whom; `explain` warns when that audience is wider than the owning group; sensitive contract inputs are passed as secret references (ADR 0007), never as values in the instance | ADR 0024 d7, d10; ADR 0007 a2 |
| L28.12 | P12 | When a contract input is omitted, the module's own default applies (retention 7 days) instead of the platform's class default (30 days), and `explain` does not say which. | **amend** ADR 0024 decision 2: contract inputs are filled from the platform's class defaults (ADR 0010) before any module default, and a module default is used only where the platform has none; `explain` prints each input with its source (user, platform default, module default) | ADR 0024 d2; ADR 0010 |
| L28.13 | P1 | Upgrading a device package moves every instance in the repository to the new version, so seven teams' pull requests show changes they did not make. | **rule** (ADR 0024 rules; ADR 0010 rules): an instance pins its module version; `pkg upgrade` of a device package never re-plans an instance; `/sys/<class>/<id>/iac/` shows `upgrade available` and each instance is upgraded by a change to its own version, with `explain` showing the plan and the contract's renamed inputs mapped by the package | ADR 0024 d2, d6; ADR 0010 |
| L28.14 | P7 | `--like` onto a module that lacks an input the source uses prints the input as unsupported and applies anyway; consumers that rely on it (compaction) fail silently. | **amend** ADR 0024 decision 9: `mknod --like` is refused when the target module does not support an input the source device uses, unless `--drop=<input>` names it; `explain` lists the consumers of the source that depend on each dropped input, where the device knows them; the receipt records what was dropped | ADR 0024 d9; ADR 0012 |
| L28.15 | none | A clean plan does not guarantee a clean apply: eventual consistency, quotas and API-side validation surface only at apply. | **accepted**: the runner's plan is the best preview there is; failures are handled as partial applies (L28.6) and `explain` says that a plan is a preview, not a promise | ADR 0024 d4 |
