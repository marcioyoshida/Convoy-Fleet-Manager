# ADR 0004: A process converges or it does not; exit codes come from a completion contract

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/pipeline-compiler.md §7; docs/debate-review.md §6

## Context

Kubernetes objects do not exit. Distributed work ends partly done. No research note defined what happens when 3 of 1000 partitions fail or when a budget runs out.

## Decision

Every plan carries a contract: best case, floor (`--min-success`), retries per partition, deadline, budget. Exit codes: 0 best case; 0 with a degraded flag when only the floor was met; 1 floor unreachable, with fail-fast cancellation of remaining work; 2 deadline or budget exceeded after the declared action ran; 3 policy denied. `pipefail` is on by default. `&&` requires code 0 without the degraded flag unless `--accept-degraded`.

## Consequences

Maps to Job `successPolicy`, `backoffLimitPerIndex`, `activeDeadlineSeconds` (Kubernetes 1.31+). Fail-fast is done by the session; Kubernetes does not do it. Every emitted Job ignores `DisruptionTarget` in its `podFailurePolicy` so preemption never burns the floor.
