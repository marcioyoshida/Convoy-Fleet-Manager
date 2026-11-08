# How should POSIX process signals like SIGTERM, SI

> Source note (DOCX). AI-generated take collected during the Convoy design debate; converted to text verbatim. The critical review lives in `docs/debate-review.md`.

In a traditional POSIX operating system, signals (SIGTERM, SIGKILL, SIGSTOP) are asynchronous notifications sent by the kernel to processes to control their lifecycle, execution state, or cleanup routines.

In a Kubernetes-backed shell, these local signals must be intercepted by the shell translation layer and mapped to Kubernetes pod lifecycles, termination grace periods, and container checkpointing APIs.

## 1. SIGTERM (Graceful Termination & Cleanup)

POSIX Meaning: "Please terminate gracefully. Save your state, close your database connections, and exit cleanly."

Kubernetes Mapping: When a user issues a standard kill <process-id> or closes their terminal session, the shell translates this into a graceful Kubernetes pod deletion request or a signal forwarded via kubectl exec.

How it works: Kubernetes sends a SIGTERM signal to PID 1 inside the target container and starts a countdown defined by terminationGracePeriodSeconds (default: 30 seconds). The application has this window to flush buffers and shut down cleanly before the platform intervenes.

## 2. SIGKILL (Force Termination)

POSIX Meaning: "Terminate immediately. No cleanup, no finalizers—the kernel is killing you right now."

Kubernetes Mapping: Maps to either an immediate force-deletion command (kubectl delete pod --grace-period=0 --force) or the expiration of the terminationGracePeriodSeconds timer.

How it works: If a process ignores SIGTERM or the user explicitly executes kill -9, the container runtime (e.g., containerd) on the worker node drops the hammer, sending a kernel-level SIGKILL to instantly wipe the container namespace and release its resources.

## 3. SIGSTOP and SIGCONT (Suspend and Resume)

POSIX Meaning: SIGSTOP pauses a process's execution without killing it (leaving its memory intact), and SIGCONT resumes it.

Kubernetes Mapping: Kubernetes containers do not natively expose a simple POSIX pause/resume signal interface out of the box. However, modern container runtimes support CRIU (Checkpoint/Restore In Userspace), which bridges this gap:

Freezing State (SIGSTOP): The shell triggers a container checkpoint, snapshotting the process memory and CPU state to a persistent volume.

Resuming State (SIGCONT): The container can later be restored from that exact memory snapshot, even on a different physical node.

## Summary: Signal Translation Matrix

| POSIX Signal | Kubernetes / Cloud-Native Primitive | Operational Behavior |

| SIGTERM | terminationGracePeriodSeconds | Triggers graceful shutdown; container has a timed window to exit cleanly. |

| SIGKILL (-9) | Force Deletion (--force) | Bypasses grace period; instantly terminates container runtime namespaces. |

| SIGSTOP | CRIU Checkpoint / Freeze | Pauses execution and dumps container memory state to persistent storage. |

| SIGCONT | CRIU Restore | Restores and resumes a frozen container from its saved memory checkpoint. |
