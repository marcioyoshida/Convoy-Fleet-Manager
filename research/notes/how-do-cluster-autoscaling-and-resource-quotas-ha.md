# How do cluster autoscaling and resource quotas ha

> Source note (DOCX). AI-generated take collected during the Convoy design debate; converted to text verbatim. The critical review lives in `docs/debate-review.md`.

In a traditional operating system, when a user launches a massive, resource-intensive pipeline, the local kernel handles the spike using virtual memory paging, swapping, or invoking the Out-Of-Memory (OOM) killer if resources run dry.

In a Kubernetes-backed shell architecture, handling sudden spikes requires balancing strict multi-tenant guardrails (Resource Quotas) with elastic infrastructure scaling (Cluster Autoscaling). Because worker nodes are finite, a sudden surge from a heavy pipeline triggers an orchestrated dance between API admission control and cloud-provider infrastructure provisioning.

## 1. Resource Quotas and LimitRanges (The Safety Guardrails)

Before a pipeline even begins executing, Kubernetes checks whether the requested resources comply with administrative boundaries. Without these, a single runaway shell command could starve the entire cluster.

Namespace Resource Quotas (ResourceQuota): Each user or tenant session is bound to a specific Kubernetes namespace with hard caps on total CPU, memory, and persistent storage. If a user runs a massive pipeline that requests more than their allocated quota, the API server immediately rejects the request with an ExceededQuota error before a single container is spun up.

Default Limits (LimitRange): If a user executes a pipeline command without specifying resource requirements, LimitRanges automatically inject default CPU and memory requests and limits. This ensures that a poorly written awk or sort script cannot consume infinite memory and trigger a cluster-wide node crash.

## 2. Cluster Autoscaling (The Elasticity Engine)

When a heavy pipeline demands more compute power than the current worker nodes can provide, the system scales out automatically through a two-tier autoscaling model:

Pod-Level Autoscaling (HPA / KEDA): If a pipeline stage processes a massive stream of data, Kubernetes can dynamically scale the number of parallel worker pods using custom metrics (such as queue length in a message broker or CPU load).

Node-Level Autoscaling (Cluster Autoscaler / Karpenter):

When the shell compiler generates dozens of pods for a parallelized pipeline, Kubernetes scheduler evaluates if they fit on existing worker nodes.

If nodes lack sufficient CPU or memory, the pods enter a Pending state.

The Cluster Autoscaler detects these pending pods, calculates the exact compute deficit, and instructs the cloud provider (AWS, GCP, Azure) to provision brand-new virtual machine instances. Once those nodes join the cluster, the pending pipeline pods are automatically scheduled.

## 3. The Cold-Start Bottleneck: Cloud Latency vs. Local POSIX

While cloud autoscaling provides infinite theoretical ceiling, it introduces a physical challenge that traditional operating systems never face: latency.

The Local OS Reality: Spawning a process in Linux takes microseconds.

The Cloud-Native Reality: Spawning a heavy distributed pipeline that requires scaling up new cluster nodes involves provisioning VMs (30–90 seconds), initializing the container runtime, and pulling large container images over the network.

How the Shell Architecture Mitigates This:

Pre-Warmed Node Pools: The platform maintains a pool of "warm" idle nodes configured for burstable workloads.

Serverless Container Runtimes (e.g., Virtual Kubelet / Fargate): For bursty, short-lived pipeline steps, the shell compiler can route pods to serverless container backends that bypass node provisioning delays entirely, launching containers in seconds.

| Scenario | Traditional OS Reaction | Kubernetes-Backed Shell Reaction |

| High Memory Spike | Swaps to disk or triggers local OOM Killer | Container hits memory limit, gets OOMKilled, and exits with code 137. |

| Sudden CPU Surge | Kernel CPU scheduler time-slices processes | Kubernetes throttles CPU based on limits or triggers horizontal/vertical pod scaling. |

| Exhausted Compute | Machine freezes / becomes unresponsive | Pods enter Pending state; Cluster Autoscaler provisions new cloud nodes dynamically. |
