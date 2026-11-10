# US-17 · Offboard an engineer on Friday without stranding their jobs, and know on Monday who could still act as them

| Field | Value |
|---|---|
| Primary persona | SEC |
| Secondary personas | PE, DE, DO, SRE, FIN |
| Workload class | identity lifecycle: offboarding, approval under a shrinking group, root assumption, a group that vanishes |
| Walkthrough | none; written against [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md) and its consequences, with [ADR 0007](../adr/0007-secrets-are-references.md) (amendment 3, producers), [ADR 0013](../adr/0013-native-verbs-have-one-shape.md) (`approve`, `release`, `sudo`), [ADR 0008](../adr/0008-signal-routing.md) amendment 4 and [ADR 0011](../adr/0011-device-tree-and-mount-table.md) amendment 5 (fencing); it reuses the release of US-03, the cron DAG of US-01, the hold of US-08, the secret of US-09 and the node of US-16 |
| Gaps already closed there | none |
| Milestones needed | M2, M7, M8 |
| Findings in this document | L17.1 – L17.8 |

## 1. Story

**As a** security engineer, **I want** an engineer's departure to end their access the minute the identity provider says so, leave every job, hold, secret and release they made in the hands of their team, and leave me a record of who could act as whom at every step, **so that** offboarding is a checklist I can run in ten minutes on a Friday, not a weekend of finding what one person's credentials were holding up.

**Context.** Dana, a payments data engineer, leaves on Friday. They authored the nightly DAG (US-01), set the finance database password from a prompt (US-09), released `ledger-api` twice this month (US-03), hold three month-end artifacts for seven years (US-08) and have a 36-hour training run in flight (US-05). Their colleague Ben is the only other member of the release approvers group. On Monday an SRE must fence the node from US-16, which needs root. Two weeks later a reorg deletes the group at the provider. Today each of those is a ticket to a different team and a shared kubeconfig that nobody rotates.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | `sec-maria`, security | user `sec-maria`, groups `sec`, `sec-release`, `wheel` | reads other users' objects as `sec`; assumes `svc:root` through `wheel` |
| Leaves | `dana` | user `dana`, groups `payments-de`, `payments-release`; removed at the provider on Friday 17:31 | every object they made belongs to `payments-de` |
| Authors a release | `ben` | user `ben`, groups `payments-de`, `payments-release` | after Friday the only member of `payments-release` |
| Approves | the fallback group `sec-release`; a second `wheel` member (`sre-lee`) for root | | L17.3, ADR 0016 decisions 6 and 7 |
| Pays | cost centre `payments-de`, then `finance-de` after the reorg | chargeback follows group ownership | |
| Audits | the quarterly access review | receipts with actor, identity and approver; the provider's own log | |
| Operates the platform | the platform team | the inventory, the provider's event feed, the trust policies per device | |

## 3. Preconditions

- **Capabilities**: `idp-revocation` (the provider pushes user and group removals, back-channel logout or SCIM events), `idp-introspection` (membership can be checked at the provider at the moment of `sudo`), `node-pool:karpenter` (US-16), `object-lock` on the month-end store (US-08).
- **Packages**: `cnpg` (US-06) with the `cnpg-user` producer for `pg-finance` (ADR 0007 amendment 3).
- **Mounts**: `/etc/fstab.d/payments-de` gives every member `/clusters/prod-eu/payments`, `/s3/finance-eu` and `/tables/lake`; `sec-maria`'s profile adds `/clusters/all` read-only.
- **Annotations**: `release: verb, writes intent, policy prod-* approver payments-release, fallback sec-release`; `torchrun: gang, checkpoint_seconds 600`.
- **Platform defaults** `explain` must print: the login token lifetime (8 h) and the shorter cap for `wheel` (1 h), the scratch retention under `/home` (30 d), the orphan window for a group with no token seen (14 d), the fallback approver group per policy, the revocation source (`idp-revocation` or token expiry).

## 4. Main path

```bash
# Friday 16:40, before HR acts
$ id
uid=sec-maria gid=sec groups=sec,sec-release,wheel   token expires 17:02   assumed: none
$ id dana
uid=dana gid=payments-de groups=payments-de,payments-release   sessions 2 (laptop 09:12 attached, cron-debug 08:40 detached)   tokens expire 17:12, 16:40

$ find --actor dana
artifact  payments-de  1 204 under /artifacts/payments-de, 3 held (month-end 2026-10, 7 y, object lock)
cron      payments-de  nightly-dag (runs as svc:payments-de, since 2026-03-02), reconcile-weekly (since 2026-07-19)
secret    payments-de  pg-finance v3, set 2026-09-14 from prompt
unit      payments-de  ledger-api, released v1.19 on 2026-11-06
process   payments-de  7790 training run, runs as svc:payments-de, 14 h of 36 h                    (L17.1)
scratch   dana         /home/dana, 2 artifacts, retention 30 d, no holds
owner is payments-de for every line but the last

$ secret list --actor dana
pg-finance   v3   payments-de   set 2026-09-14 from prompt   known to actor: yes   producer: cnpg-user       (L17.8)
$ secret rotate pg-finance
v4 produced by cnpg-user (rotation: dual) … 3 holders reloaded with kill -HUP … v3 retires after 24 h cooling   receipt 7841

# Friday 17:31, HR removes dana at the provider. Convoy is not asked.
$ id dana
uid=dana   provider: removed 17:31 (idp-revocation)   sessions 2 ended 17:31   tokens revoked
# without idp-revocation the same line reads: provider: unknown; sessions end at token expiry 17:12, 16:40   (L17.2)

$ ps --group payments-de
PID   TYPE     OWNER        ACTOR            RUNS AS           STATE
7790  jobset   payments-de  dana (removed)   svc:payments-de   running 15 h of 36 h
$ cron list payments-de
nightly-dag        svc:payments-de   next 02:00   actor dana (removed 2026-11-13)                                    (L17.7)
reconcile-weekly   svc:payments-de   next Sun     actor dana (removed 2026-11-13)

# Monday 10:02, ben releases ledger-api v1.20
$ release ledger-api v1.20 --to /clusters/prod-eu/payments
refused (exit 3): policy prod-* needs approval by payments-release; the group has 1 member (ben) and ben is the author
  alternatives: --approver=sec-release (the policy's fallback), or a second member at the provider                  (L17.3)
$ release ledger-api v1.20 --to /clusters/prod-eu/payments --approver=sec-release
approval 7850 queued for sec-release (3 members) … approved by sec-maria 10:14 … rollout done 10:21
receipt 7850: author ben, approver sec-maria (fallback; payments-release had 1 member), intent commit 8c1e…

# Monday 10:30, sre-ana fences the node from US-16 (instance off since Friday)
$ sudo echo 1 > /sys/k8s/self/nodes/gpu-b-14/fenced
refused (exit 3): the redirect is applied by your shell as sre-ana; fenced is a root file
  alternative: echo 1 | sudo tee /sys/k8s/self/nodes/gpu-b-14/fenced                                                 (L17.4)
$ echo 1 | sudo tee /sys/k8s/self/nodes/gpu-b-14/fenced
sudo: assuming svc:root for one command; wheel membership re-checked at the provider; approval 7852 queued for wheel (4 members)
[sre-lee, own session] $ approve 7852
node gpu-b-14 fenced: out-of-service taint set, pvc-7810-17 detached
receipt 7852: actor sre-ana, identity svc:root, approver sre-lee, audit stream pinned, assumed for 4 s

# Two weeks later, a reorg deletes payments-de at the provider
$ ls -l /artifacts/
drwxrw----  payments-de (orphaned: no token seen for 14 d)   1 206 artifacts, 3 holds, 2 cron entries, 1 unit, 1 secret   (L17.5)
$ sudo chown -R finance-de /artifacts/payments-de
approval 7901 queued for wheel … approved by sre-lee … 1 213 objects moved from payments-de to finance-de
cron entries run as svc:finance-de from their next run; the hold keeps its object lock; receipt 7901
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `id`, `id <user>` | the login token's claims, open sessions, token expiry | the provider's token; session pods | session, provider |
| `find --actor` | objects by actor across classes | labels on Jobs, Secrets, units; artifact store metadata | artifact store, reference index, cron table |
| `secret rotate` | a new version through the producer | the operator's user reconciliation; `secretKeyRef` holders reloaded | vault, reference index |
| removal at the provider | sessions ended, tokens revoked | provider event; session pods deleted | provider, session controller |
| `release --approver` | an approval record answered by a fallback group member | the GitOps commit as intent | approvals queue, receipt |
| `sudo tee … fenced` | `svc:root` assumed for one command with a second `wheel` member | out-of-service taint | node receipt, audit stream |
| `chown -R` between groups | ownership transfer under root | owner labels rewritten; cron entries rebound to `svc:finance-de` | artifact store, cron table, receipts |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ○ | held artifacts and scratch survive their actor; a hold cannot live under `/home` | AC-17.5 |
| A2 | Command | – | | |
| A3 | Session | ● | sessions end at revocation or at token expiry, never before; `id` says which | AC-17.2 |
| A4 | Backend | – | | |
| A5 | Contract | ○ | a 36-hour deadline exceeds an 8-hour token; the process runs as the group's service identity (L17.1) | `explain` prints `runs as` |
| A6 | Device tree | ○ | the node file is a root file; mounts come from the group profile | `mount` shows the origin |
| A7 | Signals | ○ | `kill -HUP` on rotation; sessions ended by revocation | |
| A8 | Secrets | ● | a secret set from a prompt is known to its actor and is rotated on offboarding (L17.8) | AC-17.3 |
| A9 | Cost | ○ | chargeback follows the group through the `chown` | the receipt names both groups |
| A10 | Identity | ● | user, groups, service identity, root through `wheel`, fallback approver (L17.2, L17.3, L17.5) | AC-17.1 to AC-17.7 |
| A11 | Defaults | ● | token lifetime and the `wheel` cap, scratch retention, orphan window, fallback group, revocation source | all in `explain` |
| A12 | Storage | – | | |
| A13 | Networking | – | | |
| A14 | Time | ○ | token lifetime is the upper bound on stale access (L17.2) | `id` prints expiry |
| A15 | Audit | ● | every receipt names actor, identity and approver; the root audit stream cannot be detached | AC-17.6 |
| A16 | Durability | ● | jobs, cron entries, holds, secrets and units survive their author and their group | AC-17.5, AC-17.7 |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | a group removed at the provider is a fact Convoy cannot refuse; decision 9 says `rm` on a group with objects is refused | Convoy observes; the refusal only holds inside the shell | **L17.5** |
| P2 | POSIX cannot hold | `sudo echo 1 > file` applies the redirect as the user, on Linux and here | the refusal must name the `tee` spelling | **L17.4** |
| P3 | Orphaned state | objects whose group has no living member | an orphan window, `ls -l` marks, root `chown -R` | **L17.5** |
| P4 | Identity leak | `ps -A` shows actor names to every user on the fleet | the owner column is the group; the actor is for members and root | **L17.6** |
| P5 | Invisible cost | a service identity per group at every device is trust policies times groups times devices | inventory cost, stated in ADR 0016; accepted | accepted |
| P6 | Lying cache | the groups claim in a token is a cache of membership for its lifetime; a user removed from `wheel` can still assume root until expiry | revocation events where available; re-check at the provider for `wheel`; a shorter cap | **L17.2** |
| P7 | Silent failure | without `idp-revocation` nothing happens at removal until the token expires | `id` must say `provider: unknown` and print the expiry, never `removed` | folded into L17.2 |
| P8 | Partial fleet | a device whose trust policy was not updated for the new group after the reorg | `mount` fails with the device's reason; the `chown` receipt lists devices not yet rebound | ok, with L17.5 |
| P9 | Created elsewhere | a Secret created by `kubectl` with no owner label | owner `?` (L6.6); `find --actor` cannot see it; `fuser --unknown` can (L9.4) | ok |
| P10 | Multi-party | the approving group is one person, or that person is the author | a fallback group in the policy; never root | **L17.3** |
| P11 | Data at rest | the leaver's scratch, and artifacts they can no longer reach | scratch expires on its retention; group objects are the group's; the provider holds the only copy of membership history | ok |
| P12 | Inheritance | who is accountable for a cron entry whose submitter has left; what credential does the running job hold once the token is revoked | the receipt names the removed actor and the service identity; a launched process never held the user's token | **L17.1**, **L17.7** |

A thirteenth question for this story: ADR 0016 says the login token is exchanged at open and that the mount's identity is what a launched process runs as. A pod cannot refresh a person's token, and a 36-hour run outlives an 8-hour one. **L17.1**.

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| user removed at the provider, no revocation capability | `id <user>` prints `provider: unknown` and the expiry | n/a | sessions ending at expiry | `sudo pkill -u <user>` ends the sessions now (L17.2) |
| approving group has one member who is the author | `explain` or the verb, pre-flight (ADR 0016 d7) | 3 | the group and its size | `--approver=<fallback>` (L17.3) |
| no fallback group in the policy | pre-flight | 3 | the policy line | a member at the provider; policy change by PE |
| `sudo` with a shell redirect | pre-flight | 3 | the file and the user the redirect ran as | `sudo tee` (L17.4) |
| `wheel` membership gone at the provider but still in the token | re-check at `sudo` (`idp-introspection`) | 3 | provider says no | none; the token was stale |
| group deleted at the provider while owning objects | orphan window in `ls -l`, `df` (L17.5) | n/a | last token seen | `sudo chown -R <group>` |
| a device not yet bound to the new group after `chown` | `mount` or the next stage | 1 | the device's reason | the inventory rebinds; the `chown` receipt lists the devices |
| secret set from a prompt by the leaver | `secret list --actor` (L17.8) | n/a | known to actor | `secret rotate` through the producer |
| a hold whose only member left | nothing fails; the hold is the group's | n/a | | none needed |

## 8. Acceptance criteria

- **AC-17.1** Given `id <user>`, then it prints user, primary group, supplementary groups, open sessions, token expiry per session, and after removal either `provider: removed <time>` with sessions ended or `provider: unknown` with the expiry, never a bare success.
- **AC-17.2** Given a user removed at a provider with `idp-revocation`, then every session of that user ends within the event's delivery time and no later call succeeds; given no such capability, then `sudo pkill -u <user>` ends them and the receipt names the actor.
- **AC-17.3** Given `secret list --actor <user>`, then every version set from a prompt or stdin is marked `known to actor`, and `secret rotate` through its producer invalidates it with the dual-version rotation of ADR 0007.
- **AC-17.4** Given a release whose approving group cannot supply a member other than the author, then the verb is refused at plan time naming the group, its size and the policy's fallback group; given `--approver=<fallback>`, then the approval is answered by that group and the receipt records that the fallback was used and why.
- **AC-17.5** Given a removed user, then every artifact, hold, cron entry, secret, unit and running process they made remains owned by their group, runs or holds unchanged, and `cron list` marks the entries whose actor is removed.
- **AC-17.6** Given `sudo` on a root file, then a redirect form is refused naming the `tee` form; the assumption re-checks `wheel` membership at the provider, queues an approval for `wheel`, and the receipt names actor, identity `svc:root`, approver, duration and the pinned audit stream.
- **AC-17.7** Given a group deleted at the provider, then after the orphan window `ls -l` and `df` mark its objects orphaned with the last token seen, `sudo chown -R` moves them with one receipt, cron entries rebind to the new group's service identity, and holds keep their locks.

## 9. Persona lenses

- **SEC**: wants removal at the provider to be the only switch, with Convoy ending sessions on the event and saying honestly when it cannot; refuses if a leaver's prompt-set secret is not flagged.
- **PE**: owns the fallback groups, the orphan window and the trust policies; wants a `chown` across groups to tell them which devices still need rebinding.
- **DE**: wants the nightly DAG to run on Saturday as if nothing happened, and the 36-hour run not to die at token expiry.
- **DO**: wants the Monday release to be refused with a way out, not stuck in a queue nobody can answer.
- **SRE**: wants `sudo` to work like `sudo`, including the redirect trap, and the fence to need a second person without a ticket.
- **FIN**: wants the bill to follow the group through the reorg with one receipt.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L17.1 | P12 | ADR 0016 says the mount's identity is what a launched process runs as and that the login token is exchanged at open; a pod cannot refresh a person's token and the token has a lifetime, so a 36-hour run launched under a user, or any run that outlives its author, dies at expiry. In Kubernetes a launched pod only ever runs as a ServiceAccount, which is a service identity. | **amend** (applied: ADR 0016 d4, d5 revised) ADR 0016 decision 5: every launched process runs as `svc:<primary group>`; the user's token acts only from the session (API calls, reads the session performs itself); `explain` prints `runs as svc:<group>` for every stage and the receipt records actor and identity, as `replay` does; the group's rights are reached through two credentials with the same bindings | ADR 0016 d4, d5; ADR 0011 consequence |
| L17.2 | P6, P7 | Removal at the provider reaches Convoy only when the token is refreshed, so a removed user keeps every right for up to the token lifetime, and a user dropped from `wheel` can still assume root until then. | **amend** (applied: ADR 0016 d6, d9 revised) ADR 0016 decision 9: where `caps: idp-revocation` exists the session controller ends the user's sessions on the event; otherwise `id` prints `provider: unknown` with the expiry and root may `pkill -u <user>`; `sudo` re-checks `wheel` at the provider (`caps: idp-introspection`) and, without it, the `wheel` token lifetime is capped by policy and printed by `explain` | ADR 0016 d6, d9; ADR 0010 |
| L17.3 | P10 | Decision 7 refuses an approval the group cannot supply, which is right, but gives the author no way out other than the provider; a Monday release waits on an IT ticket. | **rule** (applied: ADR 0016 rules; roadmap M7): an approval policy names a group and may name a fallback group; the refusal prints both; `--approver=<fallback>` asks the fallback and the receipt records that it was used and the size of the primary group; root never approves | ADR 0016 d7, ADR 0013 (`approve`, `release`) |
| L17.4 | P2 | `sudo echo 1 > /sys/…/fenced` fails on Linux and here because the shell applies the redirect as the user; the refusal must say so. | **rule** (applied: ADR 0013 rules; command map; roadmap M7): a refused write to a root file names the user the redirect ran as and the `echo … \| sudo tee <file>` spelling; the command map's `sudo` row carries the note | ADR 0013 d10, command map |
| L17.5 | P1, P3 | Decision 9 refuses `rm` on a group that owns objects, but groups are removed at the provider, where Convoy has no veto; a vanished group is the hostage case at group level. | **amend** (applied: ADR 0016 d9 revised) ADR 0016 decision 9: Convoy cannot prevent a removal; the inventory records the last token seen per group; a group with objects and no token for the orphan window is marked `orphaned` in `ls -l` and `df` with the last-seen time; root `chown -R` moves the objects, rebinds cron entries to the new group's service identity, keeps holds and their locks, and lists devices whose trust policies are not yet rebound; `rm` on a group inside Convoy stays refused while it owns objects | ADR 0016 d9, ADR 0011 |
| L17.6 | P4 | `ps -A` and `top` show actor names fleet-wide (US-16 printed `dana`), which tells every user who runs what in every team. | **rule** (applied: ADR 0013 rules; command map; roadmap M7): the owner column of `ps`, `top` and `ls -l` is the group; the actor column is shown to members of that group and to root, and reads `?` otherwise; receipts are readable by the group | ADR 0013 d5, ADR 0009 |
| L17.7 | P12 | A cron entry whose submitter has left keeps running as the service identity, which is what the design wants, but nobody is told that its author is gone. | **rule** (applied: ADR 0013 rules; command map; roadmap M7): `cron list` marks entries whose actor is removed with the date; `/sys/k8s/<id>/orphans` counts them per group; they keep running | ADR 0013 (`cron`), ADR 0011 d7 |
| L17.8 | P11 | A secret set from a prompt or stdin was seen by its actor; nothing records that, so offboarding cannot tell which secrets the leaver knows. | **rule** (applied: ADR 0007 rules; roadmap M7): the receipt of `secret set` records the source (`prompt`, `stdin`, `producer`); `secret list --actor <user>` marks prompt and stdin versions `known to actor`; a producer-set version is never known to anyone | ADR 0007 a3, ADR 0013 (`secret`) |
