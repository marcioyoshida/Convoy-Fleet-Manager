# ADR 0007: Secrets are references in the session, never values

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/debate-review.md §3

## Context

A note mapped secrets to env vars, proposed scrubbing them from history, and claimed External Secrets Operator avoids etcd. ESO writes Secrets into etcd; env vars leak into child processes, dumps and logs; a value that reaches the client has already leaked.

## Decision

`secret set NAME` writes server-side from a prompt or stdin. In commands, `$NAME` compiles to a `secretKeyRef` or a Secrets Store CSI mount at `/run/secrets/NAME`; env delivery is opt-in with a warning. Users get `create pods` without `get secrets` (reference-only RBAC). Cloud credentials come from workload identity with projected tokens, not stored secrets.

## Consequences

Nothing to scrub. Attribution of secret use is emitted by the session at launch, since the API audit log attributes the read to the kubelet. Platform must provide etcd KMS encryption and the CSI driver; both appear in `caps`.
