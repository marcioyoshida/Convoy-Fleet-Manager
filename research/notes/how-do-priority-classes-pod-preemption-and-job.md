# How do priority classes, pod preemption, and job

> Source note (DOCX). AI-generated take collected during the Convoy design debate; converted to text verbatim. The critical review lives in `docs/debate-review.md`.

When multiple users simultaneously launch heavy pipelines in a shared Kubernetes-backed shell, the cluster faces a classic operating system dilemma: resource contention and process starvation.

In a traditional POSIX kernel, the CPU scheduler uses "nice" values, process priorities, and time-slicing to juggle competing tasks. In a distributed Kubernetes cluster, resource spikes are managed through three integrated layers: PriorityClasses, Pod Preemption, and Advanced Batch Schedulers.

## 1. PriorityClasses: The "Nice" Values of the Cluster

Just as a Linux administrator uses nice or renice to give background compilation tasks lower CPU priority than an interactive shell session, Kubernetes uses PriorityClasses to assign numeric weights to different types of workloads.

How it works: The shell maps different user intents to specific Kubernetes PriorityClass objects:

Interactive Shell Sessions / REPLs: Assigned a high priority (e.g., value: 100000) so that a user typing commands never experiences lag.

Standard User Pipelines: Assigned medium priority (value: 50000).

Background Batch Jobs / Heavy ML Training: Assigned low priority (value: 1000).

The Result: When the cluster is fully saturated, the scheduler always evaluates and places higher-priority pods before lower-priority ones.

## 2. Pod Preemption: Making Room by Evicting the Lowly

What happens when the cluster is running at 100% capacity, and a user launches a high-priority interactive pipeline or an urgent system task?

Without preemption, the new pods would sit in a Pending state indefinitely. With Pod Preemption enabled, Kubernetes actively clears the path:

Eviction Trigger: When a high-priority pod cannot be scheduled due to lack of resources, the scheduler identifies lower-priority pods running on the cluster nodes.

Graceful Termination: The scheduler selects a lower-priority pod (preferably one that minimizes disruption) and initiates a graceful shutdown (SIGTERM), giving it time to flush buffers.

Taking Over: Once the low-priority pod terminates and releases its CPU/memory, the high-priority pod steps into its place and starts executing immediately.

Rescheduling: The evicted low-priority job is pushed back into the queue to resume later when resources free up.

## 3. Batch Queuing & Fair-Share Scheduling (Kueue & Volcano)

Standard Kubernetes priority and preemption are designed for microservices, not complex multi-tenant batch pipelines (like heavy Spark jobs or multi-stage Argo DAGs). If left unchecked, multiple users submitting massive pipelines can trigger a preemption war, where users constantly evict each other's jobs in an infinite loop.

To prevent this, the shell architecture integrates batch queuing engines like Kueue or Volcano:

Cohort Quotas: Users or teams are grouped into "cohorts" with guaranteed minimum resource allocations and strict borrowing limits.

Gang Scheduling (All-or-Nothing): If a distributed pipeline requires 50 worker pods to function, a standard scheduler might spin up 25 and deadlock the cluster. Gang scheduling ensures that either all pods in the pipeline get scheduled together, or none do, preventing partial resource lockups.

FIFO / Fair-Share Queuing: Instead of chaotic preemption, heavy pipelines enter structured queues. The scheduler ensures that users get their fair share of cluster resources over time, maintaining order during peak utilization spikes.

## Summary: Resource Contention Resolution

| Traditional OS Mechanism | Kubernetes-Backed Shell Equivalent | How It Handles Spikes |

| Process Nice Values | PriorityClass | Weights workloads so interactive sessions outrank background batch scripts. |

| CPU Time-Slicing & OOM Killer | Pod Preemption (PreemptLowerPriority) | Evicts low-priority batch tasks gracefully to make room for urgent workloads. |

| Batch Job Queues (Slurm/PBS) | Kueue / Volcano Batch Schedulers | Manages multi-user pipeline queues, fair-share limits, and gang-scheduling. |
