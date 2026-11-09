# US-09 · Rotate a leaked database password across the fleet and prove the old one is dead

| Field | Value |
|---|---|
| Primary persona | SEC (Security & Compliance) with PE (Platform Engineer) |
| Secondary personas | SRE, DO, DBA |
| Workload class | fleet operation |
| Walkthrough | [walkthroughs.md §9](../walkthroughs.md) |
| Gaps already closed there | G10 |
| Milestones needed | M2, M7, M8, M10 |
| Findings in this document | L9.1 – L9.8 |

## 1. Story

**As a** security engineer, **I want** to rotate a database password that appeared in a log, roll every workload in every cluster that uses it, and retire the old version only when nothing can still be using it, **so that** the incident closes with a transcript and not with a hope.

**Context.** Four clusters, one unreachable at the time. The password has a producer (the database) and dozens of consumers. Today the rotation is a runbook of twelve manual steps across three tools, and the step that fails is always "find everyone who uses it".

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | the security engineer | `sudo --as platform`, fleet-wide read, per-cluster write | |
| Approves | a second platform engineer for `secret retire --force` | | L9.8 |
| Pays | platform | negligible | |
| Audits | the incident review | every step with timestamps, who deferred retirement and why | |
| Operates the platform | the same team | vault behind the Secrets Store CSI, the producer hook for the database (L9.1) | |

## 3. Preconditions

- **Capabilities**: `secrets-csi` with versions, `secret-index` (reference index per cluster), `node-agent` for HUP delivery, `gitops` awareness (L9.3).
- **Packages**: `payments-api` with `reload: hup` and a declared HUP-capable entrypoint (L9.6); `payments-worker` with `reload: restart`; `cnpg` as the secret's producer with a rotation hook.
- **Mounts**: all four clusters; the `all` union read-only.
- **Annotations**: `reload` per unit; `producer` per secret (L9.1); `rotation: dual | cutover` (L9.2).
- **Platform defaults** `explain` must print: the producer hook, the rotation strategy, the retire cooling period, how restart is delivered to GitOps-managed units.

## 4. Main path

```bash
$ sudo --as platform
$ fuser -v /secrets/payments-db-password --cluster=all        # partial: dev-sandbox unreachable
$ secret rotate payments-db-password                           # producer hook, then v8 in the vault
$ for c in prod-east-1 prod-eu staging; do
    kill -HUP --cluster=$c $(fuser /secrets/payments-db-password --cluster=$c); done
$ fuser -v /secrets/payments-db-password --version=v7 --cluster=all
$ secret retire payments-db-password v7 --when="fuser --version=v7 is empty on all clusters"
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `fuser --cluster=all` | reference index query over the union | pod specs, CSI SecretProviderClass bindings, mounts the session made | per-cluster index |
| `secret rotate` | producer hook, then version write | the database user's password changed by the operator; vault version v8 | vault, operator |
| `kill -HUP` | reload per unit declaration | signal to pid 1 via the node agent, or an intent-written rolling restart (L9.3) | pods |
| `secret retire --when` | deferred two-phase retirement (L9.4) | version disabled, then deleted | vault, audit stream |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ○ | `fuser` output piped into `kill` | the degraded flag of the producer is visible to the consumer (L9.5) |
| A2 | Command | ● | `reload`, `producer`, `rotation` fields | `type payments-api` shows `reload: hup` and the entrypoint claim |
| A3 | Session | ● | one session, four identities, a fleet-wide loop | `mount` lists four entries; the loop is explicit |
| A4 | Backend | – | | |
| A5 | Contract | ● | degraded exit on a partial union; the `--when` condition | AC-9.5 |
| A6 | Device tree | ● | `all` read-only union, degraded on partial failure | `fuser --cluster=all` says which cluster is missing |
| A7 | Signals | ● | HUP as reload (G10) | AC-9.3 |
| A8 | Secrets | ● | versions, producer, index, rotate, retire | the whole story |
| A9 | Cost | – | | |
| A10 | Identity | ● | `sudo --as platform`; second approval for `--force` | L9.8 |
| A11 | Defaults | ● | producer hook, strategy, cooling period, GitOps delivery | all in `explain` |
| A12 | Storage | – | | |
| A13 | Networking | – | | |
| A14 | Time | ○ | deferred retirement | the audit line names the condition |
| A15 | Audit | ● | the incident record | every step is a receipt with identity and time |
| A16 | Durability | ● | the condition survives the session | the deferred retire is a session-side object, not a shell loop |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | `reload: restart` on a GitOps-managed Deployment | a rollout restart is an annotation change that self-heal reverts | **L9.3** |
| P2 | POSIX cannot hold | rotation needs both versions valid for a window | producers differ | **L9.2** |
| P3 | Orphaned state | v7 kept forever if dev-sandbox never returns | deferred retire never fires | L9.8 |
| P4 | Identity leak | the leaked value is still in the log that leaked it | rotation does not scrub | L9.7 |
| P5 | Invisible cost | none | | ok |
| P6 | Lying cache | none | | ok |
| P7 | Silent failure | HUP to pid 1 that is a shell wrapper | the signal is swallowed, the app keeps v7 | **L9.6** |
| P8 | Partial fleet | `fuser … --cluster=all \| wc -l` on a degraded listing | exit 0 and the flag is lost in the pipe | **L9.5** |
| P9 | Created elsewhere | pods with the password in an env literal, baked into an image, or read from the vault API directly | invisible to the reference index | **L9.4** |
| P10 | Multi-party | the database must accept the new password; a second approver for `--force` | `secret rotate` writes the vault only | **L9.1**, L9.8 |
| P11 | Data at rest | versions in the vault | vault policy | ok |
| P12 | Inheritance | the HUP loop runs under per-cluster identities | mount identities | ok |

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| producer hook missing | `secret rotate` plan time | 3 | secret has no producer; `--manual` required | `secret rotate --manual` with the prompt audited |
| reload leaves a unit unhealthy | probes after HUP | 1, loop stops | unit, version it holds | rollout undo; the unit still has v7, which is still valid |
| HUP swallowed by the entrypoint | probe shows old version in use (L9.6) | 1 | entrypoint claim false | `kill -HUP --as-restart` |
| GitOps reverts the restart | drift (L9.3) | would be silent | | intent-written restart |
| `fuser` empty because consumers are invisible | nothing, today | 0 | | two-phase retire (L9.4) |
| dev-sandbox never returns | deferred retire never fires | n/a | deferred since, by whom | `secret retire --force` with a second approval |

## 8. Acceptance criteria

- **AC-9.1** Given `fuser -v /secrets/<name> --cluster=all` with one cluster unreachable, then the listing names the missing cluster, exits 0 with the degraded flag, and a consumer in the same pipeline can detect the degradation.
- **AC-9.2** Given a secret with a declared producer, when `secret rotate` runs, then the producer hook runs first, the new version is written only after it succeeds, and both versions are valid for the declared window; a secret without a producer requires `--manual`.
- **AC-9.3** Given `kill -HUP` on a unit with `reload: hup`, then the signal reaches the application process and the probe confirms the new version is in use; on `reload: restart` under GitOps, then the restart is written as intent and reconciled, never as a direct annotation patch.
- **AC-9.4** Given `secret retire <name> v7`, then the version is first disabled with reads denied and logged for the cooling period, and deleted only afterwards; any denied read during the period reopens the step.
- **AC-9.5** Given `secret retire --when=<condition>`, then the condition is evaluated by the session from the index, not from piped text, and the deferred object survives the session's restart.
- **AC-9.6** Given a `--force` retirement while a cluster is unreachable, then a second identity must approve and both are in the receipt.

## 9. Persona lenses

- **SEC**: wants "nothing still uses v7" to be a claim the system can defend, including consumers created outside Convoy; refuses if `fuser` completeness is assumed.
- **PE**: owns producers, reload declarations and the index; wants the GitOps interaction explicit.
- **SRE**: wants HUP to be verifiable by a probe, not assumed from delivery.
- **DO**: wants `reload: restart` to be a commit, like every other change to a managed Deployment.
- **DBA**: wants the producer side of rotation to be the operator's job, with the dual-validity window stated.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L9.1 | P10 | A secret has a producer. Writing v8 to the vault does nothing until the database user has that password; the walkthrough rotates the vault only. | **amend** (applied: ADR 0007 a3) ADR 0007 amendment 1: a secret declares its `producer` (a package hook, e.g. the database operator); `secret rotate` runs the hook and writes the version only on success; secrets without a producer require `--manual`, with the prompt and the operator's identity audited | ADR 0007, annotations.yaml |
| L9.2 | P2 | Consumers reload over minutes; the producer must accept both versions meanwhile, and many cannot. | **rule** (applied: ADR 0007 rules): the producer declares `rotation: dual` (both valid for a window) or `cutover` (rotate and reload all consumers as one coordinated group in index order, with a declared blackout); `explain` prints the strategy and the blackout | ADR 0007, ADR 0008 amendment 1 |
| L9.3 | P1 | `reload: restart` on a GitOps-managed Deployment is a rollout-restart annotation that self-heal reverts; same misroute as US-03 L3.2. | **amend** (applied: ADR 0010 a1) ADR 0010 (with L3.2): restart of a managed unit is written as intent (a version annotation bump in the repository); `explain` prints the delivery path | ADR 0010, ADR 0008 amendment 2 |
| L9.4 | P9 | The reference index sees mounts Convoy made and pod specs it can read; it cannot see env literals, values baked into images, or direct vault reads. An empty `fuser` is not proof. | **rule** (applied: ADR 0007 rules; roadmap M7): `secret retire` is two-phase: disable (version kept, reads denied and logged) for a cooling period, then delete; `fuser --unknown` lists Secret objects referenced by any pod path (env, envFrom, volumes, CSI) that the index did not create; denied reads reopen the retirement | ADR 0007 amendment 1, ROADMAP M7 |
| L9.5 | P8 | The degraded flag has no POSIX channel: `$?` is 0, so `fuser … \| wc -l` or any script reads a partial fleet as a complete one. | **amend** (applied: ADR 0004 a2) ADR 0004: degraded status is written to `/sys/proc/<pid>/degraded` and exported as `CONVOY_DEGRADED` to the next stage; a session option `set -o nodegraded` turns degraded into exit 1 for scripts, on by default in non-interactive `batch`; destructive verbs (`retire`, `rm`, `rmvol`) refuse list input whose producer was degraded | ADR 0004, ADR 0005 |
| L9.6 | P7 | HUP is delivered to pid 1 of the container; under a shell wrapper entrypoint it is swallowed and the unit keeps the old version while reporting delivery. | **rule** (applied: ADR 0008 rules): `reload: hup` is honoured only if the package declares its entrypoint handles HUP (the toolbox init forwards it); otherwise HUP is treated as `restart`; success is confirmed by the probe or by a version check, not by delivery | ADR 0008 amendment 2, annotations.yaml |
| L9.7 | P4 | The leaked value remains in the retained log and in any artifact that captured it; rotation does not scrub. | **open**: a `secret scrub <name> <version>` that quarantines artifacts and logs matching the vault's value hash (see US-07 L7.1); detection is best effort | ADR 0007, ADR 0009 |
| L9.8 | P3, P10 | A cluster that never returns leaves a deferred retirement open forever; forcing it is a two-person decision. | **rule** (applied: ADR 0007 rules): `secret retire --force` requires a second approval through the approvals object (US-03 L3.1); the deferred object shows its age in `ps` | ADR 0007, identity design (open) |
