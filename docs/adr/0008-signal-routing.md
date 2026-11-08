# ADR 0008: Signals route by backing object; STOP is a cgroup freeze, pause is CRIU

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/debate-review.md §2, §6

## Context

Notes mapped SIGSTOP variously to scale-to-zero (kills the process), to CRIU (releases the pod), and relied on `kill -TERM 1`, which the kernel does not deliver to PID 1 without a handler. Kubernetes has a checkpoint API (beta 1.30) and no restore API.

## Decision

The toolbox entrypoint is a real init and commands run under `exec`. TERM acts on the owning object (suspend a queued Job, delete pods with grace, scale a Deployment to zero). KILL uses grace 1 s without `--force` unless the node is gone. STOP/CONT freeze and thaw the pod cgroup through a node agent, keeping memory and resources. A new verb `pause`/`resume` uses CRIU where the runtime supports restore. Grace period comes from the annotation `checkpoint_seconds`.

## Consequences

STOP without the node agent is refused with the reason. Preempted training jobs get up to 300 s to write a checkpoint. Established TCP connections do not survive `resume` on another node and the receipt says so.

## Amendment 1 · 2026-11-08 · coordinated groups (walkthroughs 2, 5, 6)

**Context.** The decision above freezes "the pod cgroup". A Spark driver frozen alone stops heart-beating and Spark kills its executors; one frozen torchrun rank stalls the collective until the others time out; one frozen database quorum member is a lost heartbeat to its peers. Freezing a single member of a coordinated group is never what the user meant.

**Decision.** The annotation database gains a `group` field on executables (`spark`, `torchrun`, `mpi`, `quorum`, or a package-declared name). A process with a group is shown with its members in `ps -T`. `kill -STOP` on any member is refused with the reason unless `--group` is given; `kill -STOP --group` freezes every member within one scheduling tick through the node agents on each member's node, and `-CONT --group` thaws them in the same way. `TERM` on a group runs the group's own graceful stop (Spark stop, torchrun elastic shutdown, operator-driven switchover) rather than deleting pods. `KILL` deletes all members with grace 1 s.

**Consequences.** The node agent needs a multi-node barrier, which is a Lease per group and a timestamp the agents agree on; if any agent misses the tick the whole freeze is rolled back and the command exits 1. A group whose package declares no graceful stop gets `TERM` = `KILL` and `type` says so. See docs/walkthroughs.md, gap G2.

## Amendment 2 · 2026-11-08 · HUP is reload (walkthrough 9)

**Context.** The decision defines TERM, KILL, STOP and CONT. Secret rotation and configuration changes need the Unix meaning of HUP: re-read your configuration without dying.

**Decision.** `kill -HUP` on a process whose unit declares `reload: hup` delivers SIGHUP to the process and waits for the readiness probe. On `reload: restart`, or with no declaration, it becomes a rolling restart of the owning object with the unit's rollout settings, and `type` says which. On a batch Job, HUP is refused with the reason. `--cluster=<id>` scopes the pids; the `all` union is never a write target.

**Consequences.** A reload that fails its probe is rolled back by the rollout, and the exit code is 1. See docs/walkthroughs.md, gap G10.
