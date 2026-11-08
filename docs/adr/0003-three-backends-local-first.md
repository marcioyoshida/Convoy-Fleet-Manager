# ADR 0003: Three backends, local first, chosen from known input size

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/pipeline-compiler.md §5, §6; docs/debate-review.md §5

## Context

Making every command a pod costs seconds of scheduling and image pull. The notes answered with warm pools and serverless runtimes, which raise baseline cost and still start in tens of seconds.

## Decision

The planner tries, in order: local (in the session), parallel-local (one toolbox pod, split across cores on tmpfs), distributed (Indexed Job or Argo Workflow over partitions). Thresholds come from the platform profile; users override with `--engine` or `@remote`. Partition count derives from input size, the only size the planner may use.

## Consequences

Most commands never create a pod. Cold start is avoided rather than mitigated. Quota is checked at plan time from partitions × requests.
