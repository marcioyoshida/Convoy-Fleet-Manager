# US-03 · Release a service through staging and two production regions with a canary and a second approver

| Field | Value |
|---|---|
| Primary persona | DO (DevOps Engineer) |
| Secondary personas | SRE, SEC, PE |
| Workload class | release |
| Walkthrough | [walkthroughs.md §3](../walkthroughs.md) |
| Gaps already closed there | G3 |
| Milestones needed | M2, M4, M7, M8, M11 |
| Findings in this document | L3.1 – L3.5 |

## 1. Story

**As a** DevOps engineer, **I want** the release of the payments service to be a script that builds, deploys to staging, waits for health, asks a second person, then canaries and promotes in each production region, **so that** the release is one auditable transcript rather than six pipeline stages in three tools.

**Context.** Payments releases happen twice a week and need a second approver by policy. Today the chain is a CI pipeline, an Argo CD sync wave and a Slack approval bot, and the audit trail is reconstructed by hand after incidents.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | the release engineer | personal identity on `/clusters/ci`, `/clusters/staging`, `/clusters/prod-east-1`, `/clusters/prod-eu`, each mount under its own role | four identities in one session |
| Approves | a second engineer | a different identity, from a different session | the policy says approver ≠ author; see L3.1 |
| Pays | team payments | | small |
| Audits | security, after incidents | reads the receipt chain: commit hashes, ReplicaSet hashes, approver, timestamps | needs the approval to be attributable |
| Operates the platform | platform team | sets `release_path=git`, runs Argo CD, the Gateway and the metrics stack | the GitOps repository is theirs |

## 3. Preconditions

- **Capabilities**: `gitops` (Argo CD or Flux), `gateway-api`, `metrics` (for `--error-rate-below`), `argo` (for the suspend node).
- **Packages**: payments as a service unit (M11) with its rollout strategy; the GitOps repository as a device (L3.3).
- **Mounts**: the four cluster mounts; the intent repository mount.
- **Annotations**: `release: verb, writes intent, policy approver≠author for prod-*`.
- **Platform defaults** `explain` must print: `release_path=git`, the repository and branch written, that the route weight is also written as intent (L3.2), the health probe source.

## 4. Main path

```bash
$ batch ci.sh --deadline=20m                                   # US-01 shape
$ release payments 1.4.2 --to staging && wait --healthy payments --timeout=10m \
  && read -p "promote? " ok && test "$ok" = y \
  && for env in prod-east-1 prod-eu; do
       release payments 1.4.2 --to $env --canary=10% \
       && wait --healthy payments --timeout=15m --error-rate-below=0.5% \
       && release payments --promote --to $env
     done
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `release --to staging` | intent write | commit to the GitOps repository; Argo CD sync; new ReplicaSet | the repository, then the cluster |
| `wait --healthy` | condition wait | rollout status, readiness, Gateway error rate from `/sys` | API server, metrics |
| `read -p` | suspend | Argo suspend node; resumes on input | audit stream |
| `--canary=10%` | intent write | HTTPRoute weight 10/90 as a commit (L3.2) | repository |
| `--promote` | intent write | weight 100/0; old ReplicaSet retained | repository, cluster |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ○ | `ci.sh` artifacts; the image digest is the artifact the release pins | the receipt names the digest, not the tag |
| A2 | Command | ○ | `release`, `wait --healthy` are verbs of the unit family | `type release` shows it writes intent |
| A3 | Session | ● | four cluster mounts under four identities, one cwd at a time | `mount` lists four entries with distinct `role=` |
| A4 | Backend | ○ | the chain is an Argo Workflow because of `read` | `explain` shows the suspend node |
| A5 | Contract | ● | `&&` requires non-degraded 0; `read` times out as exit 2; policy exit 3 | a canary breach exits 1 and weights return to 0/100 |
| A6 | Device tree | ● | environments are mounts, the repository is a device (L3.3), `all` refuses writes | `release --to all` is refused |
| A7 | Signals | ○ | `kill` on the Workflow during suspend cancels the release | nothing was written to prod |
| A8 | Secrets | ○ | the Git write credential is a reference bound to the repository mount | no token in the session env |
| A9 | Cost | – | negligible | |
| A10 | Identity | ● | approver ≠ author | **the author's `read` cannot be the approval** (L3.1) |
| A11 | Defaults | ● | `release_path`, route-weight path, probe source | all in `explain` |
| A12 | Storage | – | none | |
| A13 | Networking | ● | canary is a Gateway route weight | the weight is reconciled, not patched |
| A14 | Time | ○ | timeouts on `wait`, 24h on `read` | exit 2 after 24h with nothing promoted |
| A15 | Audit | ● | the chain of receipts is the release record | each receipt has commit hash and ReplicaSet hash |
| A16 | Durability | ● | the release engineer closes the laptop during `read` | the suspend node waits server-side |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | `release` writes the image as intent (G3), but the canary weight? | walkthrough says "Gateway HTTPRoute weight 10/90", a direct patch | **L3.2** |
| P2 | POSIX cannot hold | `read` in a detached batch | Argo suspend, resumes on session input | ok, but see L3.1 |
| P3 | Orphaned state | `&&` chain stops after prod-east-1 promoted | environments diverge | L3.4 |
| P4 | Identity leak | the Git credential | reference bound to the repository mount | ok once L3.3 exists |
| P5 | Invisible cost | none | | ok |
| P6 | Lying cache | `ci.sh` build cache keyed on source ids | ADR 0001 | ok |
| P7 | Silent failure | `wait --healthy --error-rate-below` with no metrics package | undefined | **L3.5** |
| P8 | Partial fleet | prod-eu unreachable at promote time | the loop stops at `&&`; receipt names the environment | ok; L3.4 states the limit |
| P9 | Created elsewhere | the Deployment and HTTPRoute are created by Argo CD from the repository | Convoy reads them from the API server, writes them through the repository | ok once L3.2 |
| P10 | Multi-party | a second approver | `read -p` reads the author's own stdin | **L3.1** |
| P11 | Data at rest | the repository holds the release history | outside Convoy, referenced by hash | ok |
| P12 | Inheritance | the identity used per `release` is the mount's role | per-mount identity from ADR 0011 | ok |

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| staging unhealthy | rollout status | 1 | ReplicaSet hash, failing probe | fix and rerun; staging left on 1.4.2 |
| canary error rate 0.8 % | metrics condition | 1 | weight returned to 0/100, commit of the revert | none needed |
| approver is the author | policy at `release --to prod-*` | 3 | rule and who may approve | second person runs `approve` (L3.1) |
| nobody answers for 24h | suspend timeout | 2 | nothing promoted | rerun |
| Argo CD reverts the route weight | drift detection on the object | would be silent | | L3.2 closes it |
| prod-eu unreachable | mount open fails | 1 | prod-east-1 at 1.4.2, prod-eu at 1.4.1 | `release payments 1.4.2 --to prod-eu` later |

## 8. Acceptance criteria

- **AC-3.1** Given `release_path=git`, when `release payments 1.4.2 --to staging` runs, then a commit exists in the repository before any object changes, and the receipt holds the commit hash and the resulting ReplicaSet hash.
- **AC-3.2** Given `--canary=10%` on a GitOps-managed route, then the weight change is also a commit and Argo CD reports the application in sync after it.
- **AC-3.3** Given a policy requiring a second approver, when the author answers the `read`, then the promote is refused with exit 3; when a second identity runs `approve <pid>`, then it proceeds and the receipt names both identities.
- **AC-3.4** Given `release --to all`, then it is refused with the reason that the union is read-only.
- **AC-3.5** Given no metrics capability, when `wait --healthy --error-rate-below` runs, then it is refused at plan time naming `caps: metrics`, not silently reduced to rollout status.
- **AC-3.6** Given the session detached during `read`, when the approver acts an hour later, then the chain resumes and the receipt records the elapsed time.

## 9. Persona lenses

- **DO**: wants one script and one transcript; refuses if a canary weight can be reverted by the GitOps controller behind the script.
- **SRE**: wants `wait --healthy` to fail loudly when its signal source is missing, and a rollback that is itself a commit.
- **SEC**: wants the approval attributable to a second identity and present in the receipt chain.
- **PE**: owns the repository and the defaults; wants `release` to never bypass the repository in production namespaces.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L3.1 | P10 | A `read -p` in the author's session can never satisfy "approver differs from author": the only stdin is the author's. The walkthrough's approval step is structurally in the wrong place. | **open** (the approver is settled as a member of the approving group other than the author, ADR 0016 decision 7; the object is drafted in ADR 0017 on 2026-11-10): approvals are objects, `/queue/approvals/<pid>`, created by a policy-gated verb and answered with `approve <pid>` or `deny <pid>` from another identity's session; `read -p` in a batch that a policy marks as two-person compiles to that queue and the suspend node; the receipt names both identities | ADR 0005, ADR 0016, ADR 0017, ROADMAP M7 and M11 |
| L3.2 | P1 | The canary weight is patched on an HTTPRoute that GitOps owns; self-heal reverts it. G3 closed `release` for the image only. | **amend** (applied: ADR 0010 a1) ADR 0010: every mutation of an object under GitOps management (detected from the tracking label or annotation) is written as intent, route weights and scale included; `explain` prints `managed-by: gitops → via commit`; direct patches to managed objects are refused | ADR 0010, ROADMAP M11 |
| L3.3 | P4, P11 | The GitOps repository is written by `release` but is not in the device tree: no identity, no mount options, no `df`. | **amend** (applied: ADR 0011 a2) ADR 0011: source repositories are devices under `/dev/scm/by-id/`, mounted with a write identity and a branch, so `release` has a place to write and the credential is a reference | ADR 0011, ADR 0007 |
| L3.4 | P3, P8 | There is no transaction across clusters; a chain that stops midway leaves regions on different versions. | **accepted**: the union is read-only by design; `release status payments --cluster=all` shows per-environment version and the receipt chain shows where the chain stopped | walkthroughs.md, ROADMAP M11 |
| L3.5 | P7 | `wait --healthy --error-rate-below` depends on a metrics source that may not be installed; falling back to rollout status would be a silent weakening. | **rule** (applied: command map; roadmap M11): `caps: metrics` is required for any metric condition; absence is a plan-time refusal, never a fallback | ROADMAP M11, posix-command-map.md (`wait`) |
