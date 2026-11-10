# US-21 · Keep my shell through a sleeping laptop, a drained node, an expired token and a dead link, and always know which cluster I am typing into

| Field | Value |
|---|---|
| Primary persona | DE |
| Secondary personas | SRE, SEC, PE, DS, FIN |
| Workload class | interactive: one engineer's working day across two residencies, a session host drain, a second terminal, an overnight detach, an edge site without its link |
| Walkthrough | written against [ADR 0020](../adr/0020-a-session-is-a-journal-not-a-pod.md) with [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md) (decisions 1, 9, 10 and 11), [ADR 0011](../adr/0011-device-tree-and-mount-table.md) (`self`, amendment 4), [ADR 0009](../adr/0009-stdout-artifact-stderr-log.md) (receipts, amendment 1), [ADR 0007](../adr/0007-secrets-are-references.md) (references), [ADR 0019](../adr/0019-a-unit-has-an-address-reach-is-a-grant.md) (the hosts view, the gateway) and [ADR 0008](../adr/0008-signal-routing.md) (drains); it reuses the edge sites of US-10 and the residency of US-08 |
| Gaps already closed there | none |
| Milestones needed | M2, M7, M8 |
| Findings in this document | L21.1 – L21.11 |

## 1. Story

**As a** data engineer who works from a laptop, a tablet and sometimes a plant floor, **I want** my shell to be where I left it whatever happened to the machine it runs on, and my prompt to tell me which cluster, namespace and identity the next command will use, **so that** I never rerun a load because a pod I never think about was drained, and never run a production command in the wrong place.

**Context.** `dana2` belongs to `payments-de` (residency `eu`, primary) and `ml-research` (residency `us`). The platform runs two session hosts in the EU and one in the US, and the Porto plant site of US-10 hosts its own sessions. Security has asked how a session pod can be inspected and where a value typed into `export` ends up. Today the team uses a jump host with tmux and a kubeconfig per cluster, and twice this year someone deleted a namespace in the wrong context.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | `dana2` | user `dana2`, groups `payments-de` (primary, eu), `ml-research` (us) | attaches from a laptop and a tablet |
| Operates the platform | `pe-sam` | groups `platform`, `wheel` | drains a session host node for a kernel patch |
| Audits | `sec-ola` | group `security` | journal location, break-glass on session pods |
| Pays | `payments-de` | primary group | session pod hours, journal bytes |
| Operates the platform | platform team | session hosts per residency, the gateway route for attach, the sandbox runtime class, `local_max = 10m`, `session_idle = 8h`, `session_expire = 30d`, the default `PS1` | |

## 3. Preconditions

- **Capabilities**: `session-host` on `eks-eu-a`, `eks-eu-b`, `eks-us-a` and `site-porto`; `sandbox: gvisor` on all four; `idp-revocation`.
- **Mounts**: the `payments-de` profile mounts `/clusters/eks-eu-prod` and `/s3/payments`; `dana2`'s own profile mounts `/clusters/eks-eu-staging`.
- **Platform defaults** `explain` must print: whether a stage runs locally and `local_max`, the session host and residency, the token expiry, the idle and expire windows, the journal location.

## 4. Main path

```bash
# Morning, laptop. Contexts are mounts; switching is cd.
laptop$ convoy attach
dana2@payments-de eks-eu-staging:etl $ mount -o identity=user /dev/k8s/by-id/eks-…-eu-west-1-reporting /clusters/reporting
dana2@payments-de eks-eu-staging:etl $ cd /clusters/eks-eu-prod/payments
dana2@payments-de eks-eu-prod:payments [prod] $ kubectl get pods | head -3       # generated kubeconfig for this mount and namespace
dana2@payments-de eks-eu-prod:payments [prod] $ cat ~/.kube/config | grep -c token
0        # exec credential to the session; no token in the file                                              (L21.3)

# A local sort, then the session host's node is drained for a kernel patch
$ sort -k3 /s3/payments/day.csv | uniq -c > /tmp/day.counts        # explain: local, estimate 4m (local_max 10m)
[pe-sam] $ echo 0 > /sys/k8s/eks-eu-a/nodes/ip-10-2-7-14/online     # cordon for a kernel CVE (US-16)
[pe-sam] $ sudo kill -TERM --node ip-10-2-7-14                       # drain
# dana2's terminal:
session host node draining in 5m (kernel CVE); 1 local command running: sort (pid 9301); finish, nohup it, or it will be lost   (L21.1)
...
session restored on eks-eu-b in 6 s; shells 1, 2; cwd kept; jobs: 9301 sort  lost (session-lost, exit 137)  replay: replay 9301
scrollback before 10:42:05 not kept across the restore                                                       (L21.6)

# A secret typed into env
$ export DB_URL=postgres://etl:S3cr3t@db-2:5432/ledger
warning: value looks like a credential; it is not journaled and will be unset after a restore; prefer: secret set DB_URL   (L21.2)

# A second terminal, and a colleague who wants to watch
tablet$ convoy attach --tty=1 --share          # same user: mirrors shell 1
[dana2, shell 2] $ umount /clusters/reporting
shell 1: mount table changed by pts/2: /clusters/reporting unmounted
[ds-kim] $ attach dana2 --tty=1 --watch
waiting for dana2 to accept a read-only watch by ds-kim (payments-de); input never shared                     (L21.4)

# Switching primary group to one in another residency
$ newgrp ml-research
refused (exit 3): ml-research is residency us; this session runs in eu (eks-eu-b). Use: convoy attach --residency=us   (L21.5)

# Evening: a follow left running, laptop closed, token expires at 02:10
$ tail -f /queues/eu/orders | grep FAILED &
...
[next morning] laptop$ convoy attach
session dormant since 02:10 (token expired); 1 command stopped at 02:10: tail -f (pid 9377), resumed now        (L21.8)
remote: batch 9350 finished 03:41 exit 0 (receipt written by the controller)

# A prompt that would have lied
dana2@payments-de eks-eu-prod:payments [prod ?] $        # device state older than prompt_stale: shown as ?    (L21.7)

# Security asks who can look inside a session pod
[sec-ola] $ who -a --host eks-eu-b
dana2   pts/1 pts/2   active   since 06:58   inspected: never
# pods/exec and pods/attach in the session namespace are granted to nobody; break-glass is sudo with a wheel approval and shows in who   (L21.9)

# A long local command that the estimate got wrong
$ sha256sum /s3/payments/archive/2025/*.parquet > /tmp/sums
local command running 10m (local_max 10m, estimate was 3m): it will be lost with this session; next time: nohup   (L21.10)

# Porto, link down
porto$ convoy attach --host site-porto
dana2@payments-de site-porto:telemetry [central: disconnected 2h ?] $ who
dana2  eks-eu-b    detached  (central session; unreachable from here)
dana2  site-porto  active    pts/1

# The US host is down for maintenance
laptop$ convoy attach --residency=us
restoring: residency us has one session host (eks-us-a), unavailable since 11:02; waiting                       (L21.11)
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `attach` | a terminal on a shell | WebSocket through the internal gateway route | the session; scrollback in memory |
| `mount`, `cd`, `export` | journal entries | none | journal in the residency's artifact store |
| `kubectl` | a generated kubeconfig | exec credential to the session | the session |
| drain | a notice, a checkpoint, a restore | eviction of the session pod; a new pod on another host | journal; receipts by the controller |
| lost local command | a receipt | none | the controller's receipt store |
| dormancy | stopped local commands | none | the session |
| `who` | sessions, terminals, state | Session objects the controller keeps | `/sys/session/<id>/` |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ○ | the journal is an object in the artifact store; a lost local stage has no artifact | |
| A2 | Command | ○ | `deterministic` decides the `replay` hint | |
| A3 | Session | ● | every decision of ADR 0020 | all |
| A4 | Backend | ○ | `local_max` decides local or Job (L21.10) | AC-21.7 |
| A5 | Contract | – | | |
| A6 | Device tree | ● | `self`, device marks in the prompt from cached hotplug state (L21.7) | AC-21.5 |
| A7 | Signals | ● | the drain notice, SIGSTOP at dormancy (L21.1, L21.8) | AC-21.1, AC-21.6 |
| A8 | Secrets | ● | a credential typed into `export`; no token in the kubeconfig (L21.2, L21.3) | AC-21.2, AC-21.3 |
| A9 | Cost | ○ | session pod hours by primary group | |
| A10 | Identity | ● | token refresh by the client, `newgrp` across residencies, a watch by another member (L21.4, L21.5) | AC-21.4 |
| A11 | Defaults | ● | `local_max`, `session_idle`, `session_expire`, `prompt_stale`, the default `PS1` | AC-21.5 |
| A12 | Storage | – | | |
| A13 | Networking | ○ | attach through the gateway; the hosts view after a restore | |
| A14 | Time | ● | idle, expiry, dormancy, the drain deadline | AC-21.6 |
| A15 | Audit | ● | commands recorded before they run; break-glass shown in `who` (L21.9) | AC-21.8 |
| A16 | Durability | ● | restore from the journal on another host; one-host residency (L21.11) | AC-21.1 |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | the session pod is evicted by the platform's own drain | the session is restored from intent (the journal), not kept alive; the drain must give local commands their chance | **L21.1** |
| P2 | POSIX cannot hold | a terminal's output buffer is not infinite and is in memory; a restore cannot replay what the pod held | scrollback is bounded and memory-only; resume says what was dropped and what was lost | **L21.6** |
| P3 | Orphaned state | a `tail -f` left running when the token expires keeps the pod alive and calls devices with no credential | at dormancy local commands are stopped, resumed on wake, ended after `session_idle` | **L21.8** |
| P4 | Identity leak | the generated kubeconfig, if it held the exchanged token, would be a credential file any local tool can copy | an exec credential to the session; no token on disk | **L21.3** |
| P5 | Invisible cost | session pods idle up to `session_idle`, twice for a user in two residencies | charged to the primary group at the time, `cost --session` per user and group | ok |
| P6 | Lying cache | the prompt reads cached device state; a missed hotplug event leaves `ready` on a device that went away | marks carry the cache age; stale state shows `?` | **L21.7** |
| P7 | Silent failure | a local command the estimate called short runs for an hour and dies with the next drain | the shell warns at `local_max`, the receipt marks it | **L21.10** |
| P8 | Partial fleet | a residency with one session host under maintenance | sessions wait in `restoring`; no session elsewhere, by residency | **L21.11** |
| P9 | Created elsewhere | a platform admin execs into a session pod and reads the user's token from memory | no exec or attach on session pods; break-glass by approval, visible in `who` | **L21.9** |
| P10 | Multi-party | a colleague wants to watch a shell; `--share` would hand them a shell holding `dana2`'s token | same user only for `--share`; a read-only watch for a group member with consent | **L21.4** |
| P11 | Data at rest | a password typed into `export` is journaled in the artifact store; `newgrp` to a group of another residency would move the session's context there | redacted values are not journaled; `newgrp` across residencies is refused | **L21.2**, **L21.5** |
| P12 | Inheritance | processes launched before and after a restore see the same env | launch captures env (ADR 0005); the restore replays the journal, minus values L21.2 drops, which the shell lists | ok with L21.2 |

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| session host node drained | the drain notice (L21.1) | n/a | per lost local command: `session-lost` | `nohup` before the deadline; `replay` after |
| session pod killed without notice | restore | 137 for local commands | `session-lost` | `replay` where deterministic |
| token expired while detached | the controller | n/a | session `dormant`; stopped commands listed | `attach` |
| credential typed into `export` | redaction patterns | ok, warned | not journaled | `secret set` |
| `newgrp` to another residency | pre-flight | 3 | residency of each | `attach --residency` |
| stale device state in the prompt | prompt age check | n/a | `?` | none; the next event refreshes |
| local command past `local_max` | the shell | n/a | `over local_max` | `nohup` next time |
| exec into a session pod | admission | refused | the attempt | `sudo` break-glass with approval |
| residency with no available host | `attach` | waits | `restoring` | wait, or the platform adds a host |

## 8. Acceptance criteria

- **AC-21.1** Given a drain of the session host node with local commands running, then every attached terminal is told the deadline and the commands at risk, the session is restored on another host of the same residency after the deadline, and each command still running is receipted `session-lost` with exit 137 and a `replay` hint when deterministic.
- **AC-21.2** Given `export` of a value the audit redactor flags, then the value is held in memory only, the shell says so with the `secret set` spelling, and after a restore the variable is unset and named.
- **AC-21.3** Given `kubectl` in the session, then the generated kubeconfig holds no token and obtains one per call from the session for the cwd's mount.
- **AC-21.4** Given `attach --share` by another user, then it is refused; given `attach --watch` by a member of the owner's group, then it waits for the owner's consent, is read-only and is receipted.
- **AC-21.5** Given a device whose cached state is older than `prompt_stale`, then the prompt shows `?` for its marks, and no prompt render makes an API call.
- **AC-21.6** Given a session that becomes dormant with local commands running, then they are stopped at the token's expiry, resumed at the next attach, and ended with `session-dormant` after `session_idle`.
- **AC-21.7** Given a local command that passes `local_max`, then the shell warns once with the `nohup` spelling and the receipt records it.
- **AC-21.8** Given any request to exec into or attach to a session pod, then admission refuses it unless it comes from `svc:root` under an approved `sudo`, and `who` shows the inspection.

## 9. Persona lenses

- **DE**: wants a lost sort to say so with a rerun line, a prompt that names the cluster and marks production, and contexts that are just paths.
- **SRE**: wants to drain any node, session hosts included, without paging the people whose shells run there.
- **SEC**: wants no credential on disk in the session, no secret in the journal, no inspection of a session pod without an approval that shows up, and nobody else's hands in a shell.
- **PE**: wants session hosts per residency, a sandbox for code users bring, and defaults for idle, expiry and `local_max` that `explain` prints.
- **DS**: wants the notebook kernel's session to come back with its mounts after a host loss.
- **FIN**: wants session hours by group and idle sessions to stop costing after the idle window.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L21.1 | P1, P7 | Decision 12 drains a session for a forced upgrade, but a platform drain of the host node, a consolidation or a spot reclaim evicts the pod with only the pod's grace, and local commands die with no warning. | **rule** (applied: ADR 0020 rules; roadmap M8): session pods carry a PodDisruptionBudget and `karpenter.sh/do-not-disrupt` while a local command runs, run only on capacity the session host marks non-spot, and any eviction is handled as the decision 12 drain: terminals told the deadline and the commands at risk, the platform's grace, then checkpoint and restore | ADR 0020 d3, d8, d12; ADR 0008 |
| L21.2 | P11 | Decision 2 says the journal never holds a secret value, but `export` takes any string; a password typed into env would be journaled into the artifact store, and the shell cannot tell a secret from a string. | **amend** (applied: ADR 0020 d2 revised) ADR 0020 decision 2: the journal is encrypted under the group's key in the residency's KMS; an exported value the audit redactor flags is kept in memory only, the shell says so and suggests `secret set`, and after a restore such variables are unset and listed by name | ADR 0020 d2; ADR 0007; S6 |
| L21.3 | P4 | The generated kubeconfig of decision 10 would hold the exchanged token in a file any local command can copy, and the file outlives the call. | **rule** (applied: ADR 0020 rules; command map; roadmap M2): the generated kubeconfig uses an exec credential that asks the session over a local socket for a token per call, scoped to the cwd's mount; no token is ever written to the file | ADR 0020 d10; ADR 0016 d4 |
| L21.4 | P10 | `--share` mirrors a shell that holds the user's token; decision 6 does not say who may share, and the ADR leaves it open. | **amend** (applied: ADR 0020 d6 revised) ADR 0020 decision 6: `--share` is for the same user only; a member of one of the owner's groups may `attach <user> --tty=<n> --watch`, read-only, after the owner accepts in that shell; every watch is receipted and shown by `who`; input is never shared across users | ADR 0020 d6; ADR 0016 d2 |
| L21.5 | P11 | `newgrp` (ADR 0016 decision 1) changes the primary group, whose residency places the session (decision 8); switching to a group of another residency would leave the session's context where that group's data may not be. | **rule** (applied: ADR 0020 rules; ADR 0016 rules; command map; roadmap M7): `newgrp` to a group of another residency is refused in the session, naming the residency of each and the `attach --residency` spelling | ADR 0020 d8; ADR 0016 d1, d11; S13 |
| L21.6 | P2 | Decision 5 resumes at the last acknowledged offset, but the output buffer is finite and lives in the pod, so a long detach or a restore cannot replay everything. | **rule** (applied: ADR 0020 rules; roadmap M2): each terminal keeps a bounded scrollback in memory (platform default); a resume beyond it says how many bytes were dropped; after a restore the shell says that scrollback before the restore time was not kept; output meant to last is redirected to a path | ADR 0020 d5; ADR 0009 |
| L21.7 | P6 | The prompt reads cached device state; a missed or delayed hotplug event leaves a stale mark, and a stale `ready` is worse than none. | **rule** (applied: ADR 0020 rules; command map; roadmap M2): each cached state carries its age; marks older than the platform's `prompt_stale` render as `?`, and `mount` shows the age | ADR 0020 d11; ADR 0011 a4 |
| L21.8 | P3 | Decision 4 lets local commands run until the token expires, then the session is dormant; a follow still running would keep calling devices without a credential and keep the pod from idling. | **rule** (applied: ADR 0020 rules; command map; roadmap M2): at dormancy local commands are stopped (SIGSTOP), listed by `who` and resumed at the next attach; after `session_idle` dormant they are ended with receipt `session-dormant` and the pod is checkpointed | ADR 0020 d4, d7; ADR 0008 |
| L21.9 | P9 | A session pod holds the user's access token in memory; anyone with `pods/exec` on the session namespace can read it and act as the user outside the audit stream. | **rule** (applied: ADR 0020 rules; ADR 0016 rules; command map; roadmap M7): `pods/exec`, `pods/attach`, `pods/portforward` and ephemeral containers in session namespaces are granted to no one; break-glass is `sudo` by `wheel` with a second member's approval (ADR 0016 decision 6), the session is told, and `who` shows `inspected by` with the approval | ADR 0020 d8; ADR 0016 d6; ADR 0017 |
| L21.10 | P7 | `local_max` is applied to an estimate; a local command that runs longer stays local and is lost with the next restore, with no warning. | **rule** (applied: ADR 0020 rules; command map; roadmap M2): when a local command passes `local_max` the shell warns once in its terminal with the `nohup` spelling for next time, the receipt records `over local_max`, and the planner's estimate for that command class learns from it | ADR 0020 d3; ADR 0013 (`explain`) |
| L21.11 | P8 | A residency with a single session host leaves its users without a shell while it is down; residency forbids a session elsewhere. | **accepted**: that is what residency means; the inventory flags a residency with one session host, `caps` prints `session-host: 1`, and `attach` says why it waits | ADR 0020 d8; ADR 0016 d11 |
