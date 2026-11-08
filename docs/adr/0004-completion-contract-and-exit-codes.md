# ADR 0004: A process converges or it does not; exit codes come from a completion contract

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/pipeline-compiler.md §7; docs/debate-review.md §6

## Context

Kubernetes objects do not exit. Distributed work ends partly done. No research note defined what happens when 3 of 1000 partitions fail or when a budget runs out.

## Decision

Every plan carries a contract: best case, floor (`--min-success`), retries per partition, deadline, budget. Exit codes: 0 best case; 0 with a degraded flag when only the floor was met; 1 floor unreachable, with fail-fast cancellation of remaining work; 2 deadline or budget exceeded after the declared action ran; 3 policy denied. `pipefail` is on by default. `&&` requires code 0 without the degraded flag unless `--accept-degraded`.

## Consequences

Maps to Job `successPolicy`, `backoffLimitPerIndex`, `activeDeadlineSeconds` (Kubernetes 1.31+). Fail-fast is done by the session; Kubernetes does not do it. Every emitted Job ignores `DisruptionTarget` in its `podFailurePolicy` so preemption never burns the floor.

## Amendment 1 · 2026-11-08 · gang contract (walkthrough 5)

**Context.** The contract above assumes partitions are independent: a floor below 100 % is meaningful and a failed index can be retried alone. Distributed training (torchrun, MPI), and any workload whose members share a collective, are all-or-nothing: one dead rank stalls every other rank, and recovery is a restart of the whole group from the last checkpoint.

**Decision.** `--gang` is a contract flag. It sets the floor to 100 % and refuses `--min-success`; it replaces per-index retry with restart-the-group-from-checkpoint, counted against `--retries` as one; it requires all-or-nothing queue admission (Kueue). Preemption of one member still does not count against retries (`DisruptionTarget` ignored), but it triggers a group restart rather than a single-index restart. Deadline and budget are unchanged.

**Consequences.** Maps to a JobSet with `failurePolicy` restart-all and `successPolicy` all, or to an Indexed Job with `podFailurePolicy` action `FailJob` plus a session-driven relaunch from the checkpoint path. Exit codes are unchanged; a gang that exhausts `--retries` exits 1 with the last failing rank in the receipt. See docs/walkthroughs.md, gap G4.
