# ADR 0007: Secrets are references in the session, never values

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/debate-review.md §3

## Context

A note mapped secrets to env vars, proposed scrubbing them from history, and claimed External Secrets Operator avoids etcd. ESO writes Secrets into etcd; env vars leak into child processes, dumps and logs; a value that reaches the client has already leaked.

## Decision

`secret set NAME` writes server-side from a prompt or stdin. In commands, `$NAME` compiles to a `secretKeyRef` or a Secrets Store CSI mount at `/run/secrets/NAME`; env delivery is opt-in with a warning. Users get `create pods` without `get secrets` (reference-only RBAC). Cloud credentials come from workload identity with projected tokens, not stored secrets.

## Consequences

Nothing to scrub. Attribution of secret use is emitted by the session at launch, since the API audit log attributes the read to the kubelet. Platform must provide etcd KMS encryption and the CSI driver; both appear in `caps`.

## Amendment 1 · 2026-11-08 · versions, references and rotation (walkthrough 9)

**Context.** A reference-only model needs an answer to "who uses this secret right now?" and "how do I replace it everywhere without handling the value?". The decision above gives neither.

**Decision.** Secrets are versioned at the vault. The session maintains a reference index per cluster: which process mounts which secret at which version, from the mounts it created plus the fleet inventory for processes created elsewhere. `fuser /secrets/<name> [--version=vN] [--cluster=…|all]` reads it, partial and degraded on unreachable clusters (ADR 0011). `secret rotate <name>` writes a new version and retains the old; `secret retire <name> vN` is the destructive step, refused while any reachable reference exists and deferrable with `--when` until the condition is provably true. No verb ever prints a value.

**Consequences.** Rotation is a loop of `kill -HUP` per cluster (ADR 0008 amendment 2) and a retire. The index is only as complete as the inventory; a cluster the inventory cannot see is reported, never assumed empty. See docs/walkthroughs.md, gap G10.

## Amendment 2 · 2026-11-09 · the artifact store is a secret sink (user story US-07, L7.1)

**Context.** References protect the session, not what a process does once it holds the value: a process with a mounted secret can print it to stdout, which becomes a retained artifact, or to stderr, which becomes a retained log. Both outlive the secret's version.

**Decision.** Artifacts and logs of a process that mounted any secret carry the retention class `sensitive` by default, which the platform maps to a shorter retention and a narrower reader set. The artifact writer (`convoy-put`) and the log shipper run secret-pattern detection using the vault's value hashes and shape rules; a match quarantines the artifact or log segment under class `sensitive-quarantine`, fails the stage with exit 1 and names the secret in the receipt without its value. Detection is best effort and the receipt says `scrub: best-effort, ruleset vN`. The same scrub applies to any context that leaves the session for a model (ADR 0014 decision 5).

**Consequences.** A false positive fails a stage, which is the cheaper error. The vault must expose value hashes to the scrubber, which is a platform capability (`caps: secret-scrub`) printed by `explain` when absent, in which case the class is still `sensitive` and the receipt says no scrub ran.

## Amendment 3 · 2026-11-09 · a secret has a producer (user story US-09, L9.1)

**Context.** `secret rotate` writes a new version to the vault. For a database password that does nothing: the database user still has the old one until something changes it there. The walkthrough rotated the vault only.

**Decision.** A secret declares its `producer`, a package hook that makes the new value true at the system of record (the database operator's user reconciliation, a cloud key rotation API, a certificate issuer). `secret rotate <name>` runs the hook first and writes the version only on the hook's success; on failure nothing is written and the exit is 1 with the hook's reason. A secret with no producer requires `secret rotate --manual`, which prompts for the value server-side (never echoing it) and records the operator's identity and the prompt in the receipt. The producer also declares `rotation: dual | cutover` so the session knows whether two versions may be valid during reload.

**Consequences.** Rotation becomes a transaction across the system of record and the vault, with the hook as the first phase. The annotation database gains `producer` and `rotation` on secret entries. `fuser` and `retire` are unchanged.

## Amendment 4 · 2026-11-09 · a mount declares its secret source (user story US-10, L10.6)

**Context.** References resolve against a central vault. A pod at a site whose link is down cannot resolve them and sits in creation forever, which is a stuck pod rather than an answer.

**Decision.** A mount declares where its references resolve: `secrets=central` (the default) or `secrets=site-replica`, naming a replica the platform runs at the site with a visible sync age under `/sys/<class>/<id>/secrets`. `explain` prints the source and the age for every reference a plan uses. A reference with no reachable source at plan time is exit 3 with the reference and the source named, never a pending pod. Replicas receive versions by the platform's sync, and `fuser` reports a replica that lags as a holder of the old version.

**Consequences.** Rotation at a disconnected site completes when the link returns, and the reference index knows it has not. `secret retire` of a version a replica still holds is refused like any reachable reference (amendment 1).
