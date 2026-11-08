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
