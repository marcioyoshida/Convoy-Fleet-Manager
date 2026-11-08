# umount maps to PVC deletion. Unmount detaches, it

> Source note (DOCX). AI-generated take collected during the Convoy design debate; converted to text verbatim. The critical review lives in `docs/debate-review.md`.

You are entirely correct on every count. These are precisely the kind of brutal abstraction leaks that doom naive "Kubernetes-as-an-OS" wrappers in week one. Translating POSIX semantics onto distributed infrastructure requires confronting the fundamental divergence between local kernel design and cloud-native reality.

Here is how the architecture must evolve to fix those exact failure modes:

## 1. Detach vs. Destroy (umount vs. PVC Lifecycle)

The Flaw: Conflating umount with deletion risks treating a routine unmount as an accidental rm -rf via an aggressive Delete reclaim policy.

The Fix: umount must be strictly decoupled from destruction.

umount /mnt/data simply detaches the storage volume from the current session's mount namespace (or terminates the backing pod). The underlying PVC remains intact and bound.

Destruction must require an explicit, destructive verb with safety prompts—e.g., rmvol or destroy --pvc—which explicitly issues the Kubernetes API deletion call (kubectl delete pvc) after confirmation.

## 2. Pod Immutability & The Plan 9 Namespace Model

The Flaw: Kubernetes pods are immutable; you cannot hot-attach a volume to a running container. A naive shell translation would force a disruptive pod restart on every mount.

The Fix: Adopting the Plan 9 per-process namespace model is the correct architectural solution.

The shell maintains a virtual session mount table. When a user runs mount, it updates the shell's internal namespace metadata.

Subsequent processes or pipeline steps spawned within that session inherit that mount context.

Under the hood, the compiler batches these requirements at process launch time, instantiating the pod with the correct pre-configured volume mounts, avoiding mid-flight mutation attempts.

## 3. RWO Scope & The RWX Concurrency Fallacy

RWO Correction: ReadWriteOnce means single-node, not single-pod. Multiple pods scheduled onto the same physical node can safely share an RWO block volume using local subpaths or shared container namespaces. True single-pod isolation requires ReadWriteOncePod. The shell translator must map RWO requests to node-local affinity rules when multi-pod sharing on a host is detected.

RWX Correction: RWX (via NFS/EFS) provides shared wiring, not atomicity or concurrency safety. Network file systems suffer from close-to-open consistency lags, advisory-only locking, and non-atomic renames across clients. Two processes writing to the same file via RWX will race and corrupt data. The shell must either enforce application-level distributed locking or warn users that RWX pipelines require explicit synchronization primitives.

## 4. Zonal Topology and Stateful Scaling

Zonal Constraints: Cloud block volumes are strictly zonal. If a node crashes, rescheduling a pod to another zone without volumeBindingMode: WaitForFirstConsumer results in an immediate scheduling deadlock because the disk is stranded in Zone A while the pod landed in Zone B. The shell's compiler must enforce topology-aware volume binding by default.

Replication Mismatch: A process scaled to ten replicas cannot share a standard RWO disk. The shell must recognize intent:

Scaling a stateless process (scale 10) deploys a standard ReplicaSet.

Scaling a stateful process that mounts a persistent disk (scale 10 with a mount) must automatically compile into a Kubernetes StatefulSet with volumeClaimTemplates, provisioning isolated, persistent storage for each individual replica index (pod-0, pod-1, etc.).
