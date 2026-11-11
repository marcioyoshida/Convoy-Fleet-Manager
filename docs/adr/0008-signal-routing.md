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

## Amendment 3 · 2026-11-09 · member signals and the freeze bound (user stories US-02 and US-05, L2.3 and L5.2)

**Context.** Amendment 1 defines signals to a group and refuses `STOP` on one member. It leaves two things undefined: a signal addressed to one member of a group (`kill -TERM 7301.7`, an executor or a rank), and how long a group freeze may last. A freeze longer than the collective's timeout (NCCL, c10d, Spark heartbeats) kills the job on `CONT`, which turns a pause into a loss.

**Decision.** Signals to a single member are refused unless the executable's annotation lists the signal under `member_signals`; Spark lists `TERM`, because the driver handles executor loss, and torchrun and MPI list none. The annotation also carries `freeze_max_seconds`, derived from the package's collective timeout. `kill -STOP --group` prints the bound when it freezes; when the bound is reached the freeze is converted to stop-with-checkpoint (the group's graceful stop with a checkpoint, as on preemption) and the receipt says `freeze bound reached`. `-CONT --group` before the bound thaws as before.

**Consequences.** A frozen training job never dies on resume; it stops cleanly instead and can be relaunched from its checkpoint. The node agent's barrier needs a timer per group. The annotation database gains `member_signals` and `freeze_max_seconds` on grouped executables.

## Amendment 4 · 2026-11-09 · a process in kernel wait is unterminatable; fencing is a node verb (user story US-16, L16.7)

**Context.** The decision says KILL uses grace 1 s. A process blocked in uninterruptible I/O cannot be ended by TERM or KILL, on Linux or here, so the statement is false for exactly the case a hung disk produces. Force-deleting the pod removes the object while the kernel still holds the volume, and a second node then attaches the same disk: two writers, which is how a database was corrupted.

**Decision.** `kill` on a process whose pod is in uninterruptible wait reports `unterminatable: blocked in kernel I/O on <device>`, does not force-delete the pod, and exits 1. A drain (`kill -TERM --node <name>`) routes per backing object as a pid does: group stop-with-checkpoint counting one retry (ADR 0004 amendment 1), operator switchover for units with `scale: delegate`, the eviction API for everything else so PodDisruptionBudgets hold, processes created elsewhere included and listed; it completes without the unterminatable processes and names them. The honest verb for those is at the node: writing `1` to `/sys/k8s/<id>/nodes/<name>/fenced` (ADR 0011 amendment 5) sets the out-of-service taint, after which volumes detach and the partitions reschedule. Fencing is refused while the node is powered on, naming the volumes at risk, unless a second identity approves; after power-off, by the pool manager on `rm` or by hand at the cloud, fencing proceeds. The node receipt records who fenced, when, and whether the node was off.

**Consequences.** A stuck partition costs its retry and the time to power the node off, never a corrupted volume. `kill -KILL` keeps its grace-1 s meaning for every process the kernel can end. The eviction path respects budgets, so a drain can take as long as a budget allows and the receipt says which unit held it. See docs/user-stories/US-16-node-fault.md.

Note · 2026-11-10: the draft [ADR 0021](0021-a-node-is-tested-before-it-is-trusted.md) automates the steps before this amendment's: it excludes components and cordons on its own, but a drain is automatic only for the classes a pool lists in `auto_drain`, and fencing is never automatic.

## Rules · 2026-11-09 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L9.6** (US-09) `reload: hup` is honoured only if the package declares its entrypoint handles HUP (the toolbox init forwards it); otherwise HUP is treated as `restart`; success is confirmed by the probe or by a version check, not by delivery.
- **L16.3** (US-16) `kill -TERM --node <name>` routes per backing object as ADR 0008 does for a pid: group stop-with-checkpoint counting one retry (ADR 0004 a1), operator switchover for `scale: delegate`, the eviction API for everything else so PDBs hold; owner `?` processes are included and listed; the drain receipt names each process's path and the ones it could not terminate.
