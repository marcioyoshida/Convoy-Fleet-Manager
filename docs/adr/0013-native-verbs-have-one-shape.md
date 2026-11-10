# ADR 0013: The native verbs are a closed set with one shape, and `explain` is the gate they all pass through

**Status:** Draft · 2026-11-09
**Supersedes / relates to:** ADR 0003 (backends, `--engine`); ADR 0004 (contract flags and exit codes); ADR 0005 (session verbs); ADR 0006 (`rmvol`, `--retain`, `replay`); ADR 0007 (`secret`); ADR 0010 (`pkg`, `tool`, `caps`, `release_path`); ADR 0011 (`mount` options, `df`, `caps` per device); ADR 0012 (`seek`, `wait --lag-below`); docs/pipeline-compiler.md §5.2 (`explain`) and §7 (contract); docs/posix-command-map.md (the verbs POSIX already defines); docs/user-stories/README.md (every story's "platform defaults `explain` must print")

## Context

The POSIX verbs are specified to the letter because POSIX wrote the letter: `docs/posix-command-map.md` maps all 155 of them against a text that already exists. The verbs Convoy adds have no such text. They were introduced one at a time, in a research note, a walkthrough or a user story, each time by example, and each example spelled the verb the way that story needed.

What the thirteen stories and twelve ADRs show when read together:

- `explain` is named about a hundred times and defined nowhere. Every story lists what `explain` "must print" for that story; no document says what `explain` prints in general, in what order, or what its exit code means.
- `run` appears as a bare pipeline, as `run --unit NAME 'pipeline'`, as `run --unit db --package=cnpg/cluster --replicas=3 --disk=per-replica:200Gi --zone-spread`, as `run --unit churn-api --gpu=1 --image=… --model=…`, as `run api-server --expose=global` and as `run inference --policy=follow-energy`. Nothing says which of these forms exist.
- `wait` takes a pid, `--healthy`, `--error-rate-below`, `--lag-below`, `--timeout`, and `$!`; two of those conditions need `caps: metrics` and one story says so.
- `expose` is spelled `expose pid 8080`, `expose notebook --internal`, `expose --public` and `--expose=global` across four documents, on top of a networking design that is still open.
- `pkg` has `install`, `remove`, `list` and `rollback`; `secret` has `set`, `rotate` and `retire`; `tool` has `add`. None has a synopsis.
- `replay <pid> --verify`, `retry <pid> --failed`, `status <pid>`, `seek <group> <topic> --to=`, `approve <pid>`, `caps`, `rmvol`, `pause` and `resume` each exist in exactly one place.

A probe cannot test a verb for consistency when there is nothing to be inconsistent with. Several findings in the register are, on inspection, the same defect: a flag or a behaviour that one story needed and another story could not know about (`--keep-going` in L4.4, `--deadline-from` in L5.5, `explain --retain` in L11.4, `explain` refusing on a follow in L12.2, `caps` per device in L10.9, the degraded channel in L9.5).

The remark that prompted this record: the non-Linux commands could be more comprehensive. They can, and the way to make them so is one shape, written once.

## Decision

1. **The native verb set is closed and listed here.** A verb is native when POSIX does not define it or when Convoy gives a POSIX verb a meaning POSIX does not have. Adding a verb amends this record. Anything else is a flag on an existing verb, a path under a mount, or a package executable.

   | Family | Verbs | Defined by |
   |---|---|---|
   | plan | `explain`, `type` | this record, decision 3 |
   | create | `run`, `batch`, `nohup` | decision 4 |
   | contract | `wait`, `status`, `retry`, `time`, `timeout` | decision 5 |
   | lineage | `replay` | decision 6 |
   | unit | `scale`, `release`, `expose`, `kill -HUP` | decision 7 |
   | device | `mount`, `umount`, `caps`, `df`, `du`, `fuser`, `rmvol`, `seek` | decision 8 |
   | platform | `pkg`, `tool`, `secret`, `approve`, `pause`, `resume` | decision 9 |

   `cd`, `pwd`, `ps`, `jobs`, `kill`, `nice`, `renice`, `cron`, `at`, `xargs`, `tee`, `mkfifo`, `mkdir`, `rm`, `chmod`, `chown`, `id`, `whoami`, `sudo` keep their POSIX synopsis and are specified by the command map and the ADR that gives them their backing object. They are not repeated here.

2. **Every native verb has the same seven fields, and a verb with an empty field is not finished.**

   | Field | What it states |
   |---|---|
   | synopsis | the exact form, with every flag |
   | reads | what the verb reads: metadata under `/sys`, receipts, the mount table, the annotation database, data |
   | writes | what the verb creates or mutates: Kubernetes objects, intent in a GitOps repository, receipts, the ledger, the mount table; or "nothing" |
   | flags | each flag and the contract field, mount option or platform default it sets |
   | exit | which of the five contract codes it can return and what 3 means for it |
   | explain | what `explain` prints for it and what it refuses |
   | devices | which device classes it applies to |

   The entries in decisions 3 to 9 follow this shape. Where a field is open the entry says `open` and names the design that must settle it; `open` is a value, not an omission.

3. **`explain` is the gate.** Nothing creates or mutates a thing on a cluster without a plan, and `explain` is the plan printed without being run.

   - **synopsis** `explain [--json] [--retain=<d>] [contract flags] '<command line>'` and `explain <pid>`. Every creating verb accepts `--explain` as the same thing.
   - **reads** the annotation database, the mount table, `/sys/<class>/<id>/` metadata, `caps` per target device, the artifact index (for cache hits), the platform profile, the receipt of `<pid>` for the second form. It never reads data. Object listings come from the session's metadata cache and carry their age.
   - **writes** nothing. Not an object, not a receipt, not a ledger line. A plan id is printed so the same line run later can be matched to it.
   - **flags** `--json` for the machine form; `--retain` so the hold copy is priced before anything is held (L11.4); any contract flag, so the contract shown is the one that would apply.
   - **exit** 0 when the plan would run; 3 when it would be refused, with the refusal text; 1 only when `explain` itself cannot reach the metadata it needs, and then it says which device.
   - **explain** prints, in this order and always all of them:
     1. plan id, backend per stage with the reason (ADR 0003), or "unit" and "package target" where no backend applies (US-02, US-12);
     2. input size, partitions and how they were derived, including coalescing and splitting (L11.5);
     3. stages with class, cache hits marked `cached`, side effects marked `side effect`, non-deterministic stages marked `never cached`, streams marked `stream, no cache, no replay` (ADR 0001, 0012);
     4. the contract: floor, retries, deadline and what it counts from (L5.5), budget, the action on each, `--gang`, degraded policy (ADR 0004);
     5. the cost band by line: compute, requests, egress, metered declared, hold per month (ADR 0012 decision 8, L11.4);
     6. placement and egress lines from the device paths (ADR 0011, L10.7);
     7. platform defaults used, each with the profile it came from (ADR 0010);
     8. `requires:` every capability checked against the target device, with a mark per device (L10.9);
     9. warnings: fail-fast can cancel independent siblings (L4.4), order is per partition (L11.3), late events excluded by a time cut (L11.1), a target topic already holds output for the window (L11.2), retention margin under threshold (L10.3), the deadline counts while a device is disconnected (L10.4);
     10. refusals, each with the rule, the capability or policy it comes from, and at least one alternative (L12.2, L13.5).
   - **devices** all. `explain` on a path under any mount prints the device, its state (`ready`, `stale`, `disconnected`) and the identity the mount would open it under.

   `type <name>` is the small form: one line per command from the annotation database (class, pushdowns, group, checkpoint, signals, `compiles_to`), or "builtin", "verb" or "package" (command map). `type` is how a user learns what parallelizes; `explain` is how a user learns what a line will do.

4. **`run` is the one creating verb; `batch` is `run` through a queue; `nohup` promotes.**

   - **synopsis**
     ```
     run  [contract flags] [--engine=<backend>] [--strict] [--stream] '<pipeline>'
     run  [contract flags] --unit <name> [--replicas=<n>] [--gpu=<n>] [--disk=<spec>] [--zone-spread]
          [--image=<ref>] [--model=<path>] [--policy=<name>] ( '<pipeline>' | --package=<pkg>[/<kind>] )
     batch [the same flags] …
     nohup <local command>
     ```
   - **reads** everything `explain` reads, plus the data the stages read.
   - **writes** the objects the plan names (Indexed Job, JobSet, Argo Workflow, Deployment and Service for a unit, a package CRD for `--package`), labelled with the `convoy.io/` set of ADR 0018 (group, kind, pid, session, plan, stage, residency; the actor is an annotation, L19.5), printed per object; one receipt per process; ledger lines as cost settles; for a GitOps-managed target, intent in the repository instead of the object (L3.2, L9.3).
   - **flags** the contract flags are one table, shared by `run`, `batch`, `explain` and `cron`:

     | Flag | Sets | Record |
     |---|---|---|
     | `--min-success=<pct\|n>` | floor | ADR 0004 |
     | `--retries=<n>` | retries per partition, or group restarts under `--gang` | ADR 0004 |
     | `--deadline=<duration\|clock>` | deadline; a clock is read in the session's `TZ` and `explain` prints it (L8.4) | ADR 0004 |
     | `--deadline-from=submission\|start` | what the deadline counts from; default `submission` | L5.5 |
     | `--budget=<amount>` | budget over compute plus declared metered spend | ADR 0004, US-07 |
     | `--on-deadline=`, `--on-budget=` `pause\|cancel\|page:<route>` | the action; `pause` is refused for `checkpoint: none` (L2.6); `page:` needs `caps: alerting` (L8.7) | ADR 0004 |
     | `--accept-degraded` | `&&` proceeds on a degraded 0 | ADR 0004 |
     | `--keep-going` | independent siblings are not cancelled by fail-fast | L4.4 |
     | `--gang` | floor 100 %, group restart, all-or-nothing admission; requires `--checkpoint` on a non-zonal device (L5.1) | ADR 0004 amendment 1 |
     | `--checkpoint=<path>` | the checkpoint device | ADR 0008, US-05 |
     | `--retain=<duration>` | a hold on the receipt, outputs, inputs and toolbox digest | ADR 0006 amendment 1 |
     | `--engine=<backend>`, `@remote` | backend override | ADR 0003 |
     | `--strict` | unknown commands are a refusal, not a one-node fallback | compiler §7 |
     | `--stream` | the opt-in live channel for the whole line | ADR 0001 |

     Unit flags: `--replicas`, `--gpu`, `--disk=per-replica:<size>\|shared:<size>`, `--zone-spread`, `--image`, `--model`, `--policy` set the unit's spec and are recorded in the unit receipt with the image digest, not the tag (US-04). `--package` makes the unit an instance of a package CRD, and the package's annotations (`scale: delegate`, `reload`, `member_signals`) govern the unit verbs.
   - **exit** `run` on a pipeline returns the contract code when run in the foreground and prints the pid when run with `&`. `run --unit` returns 0 once the unit is created and its first readiness probe passes, 1 if it never does within the platform's rollout deadline, 3 on refusal. `batch` returns 3 if the queue refuses admission outright; waiting for admission is not an error and `ps` shows `QUEUED` with position and holder.
   - **explain** every section of decision 3. For `--unit` with a follow, the planner refuses aggregate and fan-in classes with the two alternatives (L12.2). For `--unit` whose consumer group already exists with members not labelled by Convoy, refusal (L12.6). For `scale` above the partition count, refusal with the count.
   - **devices** all. The target device is the cwd's, or the device of the stage's input when the platform's placement rule says locality (L10.7).

   Contract fields inherit: a process started by a process gets the parent's remaining budget and deadline as its ceiling unless the parent tightens them (ADR 0004 amendment 3).

5. **`wait` has three subjects, and the contract code is what it returns.**

   - **synopsis** `wait [<pid>…]`, `wait <unit> --healthy [--error-rate-below=<pct>] [--lag-below=<n>] [--timeout=<duration>]`, `wait --lag-below=<n> <unit>`.
   - **reads** receipts for pids; the rollout status and, for the metric conditions, the metrics source named by `caps: metrics` (L3.5); lag from the queue device, not from the unit (L12.4).
   - **writes** nothing.
   - **flags** `--healthy` is rollout complete and probes passing; `--error-rate-below` and `--lag-below` are conditions that need `caps: metrics` or a queue device, and their absence is a plan-time refusal, never a silent fallback to rollout status; `--timeout` makes `wait` itself exit 2.
   - **exit** the contract code of the waited process: 0, 0 with the degraded flag, 1, 2, 3. With several pids, the worst code. The degraded flag is readable after `wait` as `CONVOY_DEGRADED` and at `/sys/proc/<pid>/degraded`; `set -o nodegraded` turns it into exit 1 (ADR 0004 amendment 2). `wait` with no argument joins every process of the session, which is the barrier spelling (walkthrough gap G1).
   - **explain** `explain 'wait …'` prints which source each condition reads from and refuses conditions with no source.
   - **devices** processes on any device; metric conditions per target device's `caps`.

   `status <pid>` lists partitions with state, attempts and failure reason; `retry <pid> --failed` reruns failed indexes only with the rest as cache hits; `time <line>` runs and prints the receipt; `timeout <duration> <line>` sets the deadline, exit 2 (command map).

6. **`replay` re-executes from the receipt, under the caller.**

   - **synopsis** `replay <pid> [--verify] [--engine=<backend>]`.
   - **reads** the receipt of `<pid>`: input content ids, env subset hash, toolbox digest, partition plan; the held closure when a hold exists.
   - **writes** a new process with a new pid and receipt that links to `<pid>` as its prior; with `--verify`, a report of whether output ids match, in the new receipt.
   - **flags** `--verify` compares ids; without it the replay is a rerun with lineage.
   - **exit** the contract code of the new process; with `--verify`, 0 when every id matches, 1 when any differs, with the differing stage named. 1 with `missing input id` or `missing digest` when the closure was not held (US-08).
   - **explain** names every input by id and where it is, every stage marked `never cached` or `side effect`, and refuses with the identity rule when the caller lacks read on the held closure (L8.5).
   - **devices** the artifact store and the registry; topics only when the offset range was copied at hold time (ADR 0012 decision 3).

7. **Unit verbs act on a unit as a whole and write intent where the platform says.**

   | Verb | Synopsis | Reads | Writes | Exit 3 when | `explain` prints |
   |---|---|---|---|---|---|
   | `scale` | `scale <unit> <n>` | the unit's annotations, partition count for a consumer unit | replicas, or a request to the operator under `scale: delegate`; intent under GitOps; volumes always retained (ADR 0006) | above the partition count; scale-down of a stateful unit with no operator (gap G6) | what is retained and its monthly cost |
   | `release` | `release <unit> <digest\|version> [--to=<mount>…] [--canary=<pct>] [--wait]` | the image digest, `release_path` | the version and the route weight as intent in the GitOps repository by default, directly only under `release_path=direct`; waits for reconciliation; receipt with commit and ReplicaSet hashes (gap G3, L3.2) | policy requires an approver other than the author (L3.1) | the repository and branch written, the probe source, the weight written as intent |
   | `expose` | `expose <unit> [<port>] [--internal\|--public\|--global]` | the networking capabilities | a Service; a Gateway route and certificate for `--public`; `open`: the networking design (stories.md, L4.6) | `--public` without approval; no gateway capability | the address, the route, who can reach it |
   | `kill -HUP` | `kill -HUP <unit>` | `reload` on the unit | SIGHUP to the process under `reload: hup`; a rolling restart under `reload: restart`, written as intent under GitOps (L9.3); maxSurge 0 and a cooperative assignor for consumer units (L12.3) | never; HUP is refused only where the unit declares `reload: none` | the reload mechanism and the probe that confirms it |

   A unit's receipt is one record per lifecycle event: rollouts, rebalances, scale changes, idle stops (L4.3), with cost per interval.

8. **Device verbs are POSIX verbs with device semantics, and `caps` is one of them.**

   - `mount -o <options> /dev/<class>/by-id/<id> <path>` and `umount <path>`: edit the session mount table; `umount` detaches and never deletes (ADR 0006). Options are the policy surface and are listed per class in ADR 0011 and 0012: `identity`, `ro`, `nsmap` (L6.1), `residency` (ADR 0011 amendment 3), `secrets=central\|site-replica` (ADR 0007 amendment 4), `topics`, `group`, `quota` (L13.3), `schema` (L12.5, open), `completion` (ADR 0014). An option the driver does not know is a refusal, never ignored.
   - `caps [<path>…]`: prints the capabilities of the device each path resolves to, with the package that provides each (ADR 0010) and the driver for the device type (L2.1). With no path, the cwd's device. `caps` is evaluated per device, never per fleet (L10.9). Exit 0; exit 1 when a device cannot be reached, with that device marked `disconnected`, never omitted.
   - `df [<path>]`, `du <path>`: capacity, usage, cost and retention per mount; for queue devices bytes, partitions against quota, oldest unread offset against retention (US-10, US-13); `du` warns before a priced listing (command map).
   - `fuser <path> [--version=vN] [--cluster=<mount>\|all]`: holders of a volume or secret with a TYPE column (`process`, `unit`, `policy`, `snapshot`), partial and degraded on an unreachable member (ADR 0007 amendment 1, L6.3).
   - `rmvol <path> [--with-data] [--no-snapshot]`: the destructive verb `rm` points to; checks `fuser`, snapshots first, confirms; refuses to break a hold (ADR 0006).
   - `seek <group> <topic> --to=<offset\|timestamp>`: resets a consumer group's position, with confirmation; never done by `kill` (ADR 0012).
   - `backup <path> --to=<path>`: copies the retained range of a topic, or a snapshot of a table or volume, to an object-store path as a staged copy with a receipt; the verb `rm --with-data` names as the alternative on a device that holds data (US-13, AC-13.6). Two stages and an egress line, priced by `explain`; never implied by `rm`.

9. **Platform verbs have subcommands, and each subcommand has the seven fields.**

   | Verb | Subcommands | Writes | Refuses when |
   |---|---|---|---|
   | `pkg` | `install <name>[@<version>]`, `remove <name>`, `list`, `rollback <name>` | a namespaced release reconciled by Argo CD or Flux; `remove` prints what is retained and the destroy verb for it (ADR 0010) | `requires` cannot be satisfied by the target device's `caps`; the manifest is unsigned |
   | `tool` | `add <name>[@<version>]`, `remove <name>`, `list` | a toolbox image layer, pinned in the session; an annotation entry is mandatory or the tool is `non-parallel` (ADR 0002) | no annotation entry and `--strict` |
   | `secret` | `set <name>`, `rotate <name>`, `retire <name> <vN> [--when=<condition>] [--force]`, `list` | versions at the vault and the reference index; never prints a value (ADR 0007) | `retire` while a reachable reference exists; `--force` without a second identity (L9.8) |
   | `approve`, `deny` | `approve <pid>[.<n>] [-m <note>]`, `deny <pid>[.<n>] -m <reason>` | an answer on the approval object `/sys/proc/<pid>/approval/<n>` (ADR 0017), by a user who is a member of the approving group or its fallback, other than the requesting actor, from their own session; the pinned plan hash is what is approved; the process resumes or exits 3 | the answerer is the actor, a service identity, outside the group, or in the requesting session; the plan hash changed (the object is `void`); `explain` refuses at plan time when the group has no second member (ADR 0016 decision 7) |
   | `pause`, `resume` | `pause <pid>`, `resume <pid>` | a CRIU checkpoint artifact and the pod released; a new pod from the checkpoint (ADR 0008) | no CRI-O on the node; a group without `freeze_max_seconds` (L5.2) |

10. **Refusals are uniform.** A refusal is exit 3, one line naming the rule, the capability or the policy it comes from, and at least one alternative. A verb never silently does less than asked: no fallback from a metric condition to rollout status, from a follow to a bounded read, from a group freeze to a per-member signal, from a mount option to no option. Where the design prefers a weaker action it says so in `explain` before and in the receipt after.

11. **Flags not in this record do not exist.** A story or walkthrough that needs a new flag amends the table in decision 4, 5 or 7 and says which contract field, mount option or default it binds to. The findings register's `rule` disposition for a verb-shaped finding is implemented by an amendment here, not by a sentence in the story.

## Declined

| Proposal | Why not | What replaces it |
|---|---|---|
| a `topic`, `cluster` or `namespace` noun verb | the POSIX surface is the filesystem (ADR 0011, 0012) | paths under mounts and the POSIX file verbs |
| `kill group` to reset offsets | destroys position, not a process | `seek` |
| a `fork N` verb for fan-out | `xargs -P` and `batch` already are the fan-out and the queue (command map) | no new verb |
| a `stream` or `follow` verb | the follow face is `tail -f` and `\|=` (ADR 0012) | no new verb |
| per-story flag spellings (`--expose=global`, `expose pid 8080`) | one verb, one synopsis | the `expose` entry in decision 7, marked `open` where networking is |
| `explain` as an optional dry run | nothing may create or mutate without a plan | `explain` as the gate, decision 3 |

## Consequences

- The thirteen stories can now be checked against a text. A first pass finds eight findings that were verb defects in disguise and are closed by entries here rather than by an ADR amendment of their own: L3.5, L4.4, L5.5, L8.7, L10.9, L11.4, L12.2, L13.5. L9.5 and L7.3 are decided in ADR 0004 amendments 2 and 3 and only referenced here.
- `explain --json` needs a schema; it is the contract between the planner and any tool that reads plans, including the assisted completion in ADR 0014, which ranks candidates by what `explain` would print for them.
- ROADMAP M2 gains `explain` in the full form of decision 3 and `type`; M3 the contract flags table; M11 the unit verbs; M5 and M8 the device verbs with `caps` per device; M9 the platform verbs with their refusals.
- The approvals object behind `approve` (L3.1) is ADR 0017, which adds `deny` and a fourth `wait` subject, `wait --approval <pid>`. Open: the networking design behind `expose` (L4.6), the schema registry option (L12.5), the `--json` schema. Each is marked `open` in its entry and nothing here pre-empts it.
- [US-14](../user-stories/US-14-assisted-shell.md) probes decisions 3 and 10 from the completion side and adds L14.3 (a gloss per candidate is a plan per candidate; `explain` from cache never populates a priced listing). A probe story against the whole record (every verb, every refusal, the uniform exit 3) is worth writing once the register's amendments have been decided, so the story checks the decided text and not this draft.

## Rules · 2026-11-09 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L14.3** (US-14) completion glosses at most the platform's `gloss_top_k` candidates after grammar ranking, never populates a priced listing (a cold path shows `size unknown` instead), and the plan it shows is the one `explain` would print from cache.
- **L17.4** (US-17) a refused write to a root file under `/sys` names the user the shell applied the redirect as and the `echo … \| sudo tee <file>` spelling, since `sudo echo … > <file>` redirects as the caller on Linux and here.
- **L17.6** (US-17) the owner column of `ps`, `top` and `ls -l` is the group; the actor column is shown to members of that group and to root and reads `?` otherwise; receipts are readable by the owning group. The answers on an approval object are the one exception, visible to the requesting actor (ADR 0017 rule L18.5).
- **L17.7** (US-17) `cron list` marks entries whose actor was removed at the provider with the removal date; `/sys/k8s/<id>/orphans` counts them per group; they keep running as the group's service identity.
- **L19.3** (US-19) a foreign object adopted by `chown` carries labels only; `scale` and `release` on it refuse naming the missing unit fields (release path, digest source, probes) and the `unit add <name> --from-live` spelling, which writes the unit record from the object as it is and marks it `adopted` in the receipt (ADR 0018 rule L19.3).
