# ADR 0003: Three backends, local first, chosen from known input size

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/pipeline-compiler.md §5, §6; docs/debate-review.md §5

## Context

Making every command a pod costs seconds of scheduling and image pull. The notes answered with warm pools and serverless runtimes, which raise baseline cost and still start in tens of seconds.

## Decision

The planner tries, in order: local (in the session), parallel-local (one toolbox pod, split across cores on tmpfs), distributed (Indexed Job or Argo Workflow over partitions). Thresholds come from the platform profile; users override with `--engine` or `@remote`. Partition count derives from input size, the only size the planner may use.

## Consequences

Most commands never create a pod. Cold start is avoided rather than mitigated. Quota is checked at plan time from partitions × requests.

## Amendment 1 · 2026-11-09 · placement by device locality (user story US-10, L10.7)

**Context.** The working directory chooses the cluster a stage runs in. A stage that reads a device local to one site from a working directory on the central cluster would pull the raw bytes over the site's link, which is the one thing an edge deployment cannot afford, and nothing in the decision above places a stage where its input is.

**Decision.** Among the clusters the session has mounted, a stage runs on the cluster where its largest input device is local, provided that cluster's `caps` satisfy the stage; otherwise on the cwd's cluster with the egress line priced. Locality is read from the topology of the device path (ADR 0011 decision 2). `explain` prints `placement: <cluster>` with the reason for every stage, and only the stage's outputs cross the link. The user may pin a stage with `--engine` or `@<mount>` as before, and the pin is printed with its egress.

**Consequences.** A site needs the capabilities a stage requires, evaluated per device (`caps` per target, ADR 0013 decision 8); a one-node site without Argo runs linear scripts only and `explain` says so. The input-size rule is unchanged: size still decides the backend, locality decides the cluster.

## Rules · 2026-11-09 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L10.9** (US-10) `caps` is evaluated per target device at plan time; a backend the site lacks is a plan-time refusal naming the capability and the site.
- **L16.6** (US-16) cordoned and fenced nodes are excluded from allocatable in `/sys/k8s/<id>/` and from the plan-time check (ADR 0003); the node receipt prices cordoned hours and `df`-style output for the pool shows them.
- **L24.5** (US-24) a source-repository mount (ADR 0011 amendment 2) takes `image=<ref>|devcontainer` (the repository's `.devcontainer` definition); local stages whose cwd is under that mount run in that image as a parallel-local backend pod, and `explain` names the image of every stage, `toolbox <digest>` included.
- **L25.13** (US-25) a local stage whose cwd is under a worktree mount (ADR 0022) gets a per-process writable overlay, discarded at exit, and `explain` prints `overlay: discarded` (full rule in ADR 0022).
