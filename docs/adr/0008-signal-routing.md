# ADR 0008: Signals route by backing object; STOP is a cgroup freeze, pause is CRIU

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/debate-review.md §2, §6

## Context

Notes mapped SIGSTOP variously to scale-to-zero (kills the process), to CRIU (releases the pod), and relied on `kill -TERM 1`, which the kernel does not deliver to PID 1 without a handler. Kubernetes has a checkpoint API (beta 1.30) and no restore API.

## Decision

The toolbox entrypoint is a real init and commands run under `exec`. TERM acts on the owning object (suspend a queued Job, delete pods with grace, scale a Deployment to zero). KILL uses grace 1 s without `--force` unless the node is gone. STOP/CONT freeze and thaw the pod cgroup through a node agent, keeping memory and resources. A new verb `pause`/`resume` uses CRIU where the runtime supports restore. Grace period comes from the annotation `checkpoint_seconds`.

## Consequences

STOP without the node agent is refused with the reason. Preempted training jobs get up to 300 s to write a checkpoint. Established TCP connections do not survive `resume` on another node and the receipt says so.
