# ADR 0003: Three backends, local first, chosen from known input size

**Status:** Proposed · 2026-11-08 · revised 2026-11-13 with user-story findings L29.3 (the decision), L29.7 and L30.6 (amendment 1), marked inline
**Supersedes / relates to:** docs/pipeline-compiler.md §5, §6; docs/debate-review.md §5

## Context

Making every command a pod costs seconds of scheduling and image pull. The notes answered with warm pools and serverless runtimes, which raise baseline cost and still start in tens of seconds.

## Decision

The planner tries, in order: local (in the session), parallel-local (one toolbox pod, split across cores on tmpfs), distributed (Indexed Job or Argo Workflow over partitions). Thresholds come from the platform profile; users override with `--engine` or `@remote`. Partition count derives from input size, the only size the planner may use, and, on a target with a maximum run time (`max_duration` in its `caps`, ADR 0025 decision 2), from that bound and the executable's declared throughput, so that every partition fits it; a stage whose partitions cannot fit is a plan-time refusal naming the bound and the next candidate (revised 2026-11-13, L29.3).

## Consequences

Most commands never create a pod. Cold start is avoided rather than mitigated. Quota is checked at plan time from partitions × requests.

Note · 2026-11-12: the draft [ADR 0025](0025-a-computer-is-a-device-kubernetes-is-one-driver.md) extends the distributed tier to every computer whose driver supports the stage's shape (Slurm job arrays, function invocations, batch array jobs, VM pools), and places by one ordered filter: residency, then `caps` and bounds, then the locality of amendment 1, ranked by cost and estimated start (decision 4). On Kubernetes the tier is still Indexed Job or Argo Workflow.

## Amendment 1 · 2026-11-09 · placement by device locality (user story US-10, L10.7)

**Context.** The working directory chooses the cluster a stage runs in. A stage that reads a device local to one site from a working directory on the central cluster would pull the raw bytes over the site's link, which is the one thing an edge deployment cannot afford, and nothing in the decision above places a stage where its input is.

**Decision.** Among the computers the session has mounted (clusters and, under ADR 0025, every other kind), placement is one ordered filter (revised 2026-11-13, L29.7): computers outside the residency of any input, or whose log and metric sinks lie outside it, are removed; then those whose `caps` or bounds cannot run the stage; then the stage runs on the computer where its largest input device is local, the stage's image counting as one of its inputs and pulled from the nearest replica or mirror the registry device lists, with the pull's bytes per node and price printed by `explain` and a missing replica in the target region counted as a candidate's cost, not a refusal (revised 2026-11-13, L30.6); what is left is ranked by price band and estimated start (`/sys/<class>/<id>/queue`), and `--deadline` removes a candidate whose estimated start cannot meet it. With no local candidate, the stage runs on the cwd's computer with the egress line priced. Locality is read from the topology of the device path (ADR 0011 decision 2). `explain` prints `placement: <cluster>` with the reason for every stage, and every candidate with the step that kept or removed it and its estimated start, and only the stage's outputs cross the link. The user may pin a stage with `--engine` or `@<mount>` as before, and the pin is printed with its egress.

**Consequences.** A site needs the capabilities a stage requires, evaluated per device (`caps` per target, ADR 0013 decision 8); a one-node site without Argo runs linear scripts only and `explain` says so. The input-size rule is unchanged: size still decides the backend, locality decides the cluster.


Note · 2026-11-14: the draft [ADR 0029](0029-a-computers-file-systems-are-in-its-contract-staging-is-a-planned-copy.md) makes amendment 1's locality a choice per input between reading in place and staging, printed by `explain` with its reason, and counts the staging time and price in ranking (decision 2).
## Rules · 2026-11-09 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L10.9** (US-10) `caps` is evaluated per target device at plan time; a backend the site lacks is a plan-time refusal naming the capability and the site.
- **L16.6** (US-16) cordoned and fenced nodes are excluded from allocatable in `/sys/k8s/<id>/` and from the plan-time check (ADR 0003); the node receipt prices cordoned hours and `df`-style output for the pool shows them.
- **L24.5** (US-24) a source-repository mount (ADR 0011 amendment 2) takes `image=<ref>|devcontainer` (the repository's `.devcontainer` definition); local stages whose cwd is under that mount run in that image as a parallel-local backend pod, and `explain` names the image of every stage, `toolbox <digest>` included.
- **L25.13** (US-25) a local stage whose cwd is under a worktree mount (ADR 0022) gets a per-process writable overlay, discarded at exit, and `explain` prints `overlay: discarded` (full rule in ADR 0022).
- **L27.6** (US-27) a cross-provider copy above the platform's `transfer_min` runs as a transfer executable from a package (Storage Transfer Service, DataSync, AzCopy) where `caps` lists one, with egress and requests priced on both sides, and the receipt verifies object counts and checksums.
- **L29.15, L29.18** (US-29) the plan-time quota check uses what remains of account-wide limits after what others hold; a stage over inputs in several topologies is split per topology, and only outputs cross (full rules in [ADR 0025](0025-a-computer-is-a-device-kubernetes-is-one-driver.md)).
- **L32.16** (US-32) `explain` of a Spark or dataframe stage prints its size on disk and in memory, its aggregate memory, the predicted spill and the spill device; spill to a root disk is refused (full rule in [ADR 0028](0028-memory-has-a-distance-and-a-domain-is-claimed-whole.md)).
- **L33.14, L33.15** (US-33) placement drops computers whose file systems cannot hold a stage's data; each input is read in place unless staging is cheaper or faster, and `explain` says why (full rules in [ADR 0029](0029-a-computers-file-systems-are-in-its-contract-staging-is-a-planned-copy.md)).
