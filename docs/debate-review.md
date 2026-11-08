# Convoy — Builder review of the `research/notes/` takes

**Date:** 2026-11-07. **Scope:** the twelve notes in `research/notes/`. Three were reviewed earlier and are only cross-referenced here: pipes (superseded by `pipeline-compiler.md`), disk operations, and the "umount" reply. The nine others are scrutinized below.

**Method.** For each take: what holds as written, what is factually or operationally wrong against current Kubernetes (1.31–1.34), and what I would build instead. Verdicts: **Feasible** (build as written), **Adjust** (direction right, mechanism wrong), **Wrong** (different design needed).

## Summary matrix

| Take | Verdict | The one fix that matters |
|---|---|---|
| Exit codes, stdout, stderr | Adjust | stdout is an artifact, stderr is a log. The log API cannot separate the two and rotates at 10 MiB |
| Signals | Adjust | PID 1 ignores signals without a handler; `kill -STOP` is a cgroup freeze, not CRIU; Kubernetes has no restore API |
| Secrets and env | Adjust | External Secrets Operator writes to etcd; use the Secrets Store CSI driver. Secrets are references in the session, never values |
| Multi-cluster routing | Adjust | `cd` is a kubeconfig context switch, no federation needed. Object storage is not a FIFO |
| Autoscaling and quotas | Adjust | Pre-check quota at plan time; Fargate is not a cold-start fix; batch pods need do-not-disrupt |
| Priority and preemption | Feasible | Preemption counts as a Job failure unless `podFailurePolicy` ignores `DisruptionTarget` |
| Compliance and audit | Adjust | The API audit log never sees the local backend and attributes secret reads to the kubelet |
| Cost allocation | Feasible | Object-store request costs are invisible to OpenCost; define the cost model once |
| Package management | Adjust | Two tiers (platform vs user) and two verbs (`pkg` for services, `tool` for commands) |
| Pipes | Superseded | see `pipeline-compiler.md` |
| Disk operations | Wrong verbs | see the earlier review: `umount` ≠ delete, no hot mount, RWX ≠ atomic |
| "umount" reply | Partly | fixes 1–2 hold; 3–4 leak (RWX locking, StatefulSet auto-conversion) |

---

## 1. Exit codes, stdout, stderr — **Adjust**

**Holds.** CRI log files under `/var/log/pods` carry a stream tag per line. Exit code lives in `containerStatuses[].state.terminated.exitCode`; 137 with `reason: OOMKilled` distinguishes OOM from a plain SIGKILL. Interactive sessions use `exec`/`attach` with a pty; WebSocket transport is the default since 1.31.

**Wrong.**
- The `pods/{pod}/log` endpoint returns one merged stream. There is no parameter to request stdout only or stderr only, so "client-side demultiplexing" of the log stream is not possible. Separation exists only on `exec`/`attach` channels.
- Routing data through logs is lossy. Kubelet rotates at `containerLogMaxSize` (default 10 Mi) × `containerLogMaxFiles` (default 5). A stage producing 1 GiB of stdout loses 98 % of it. The log path also funnels through the API server.
- Exit codes disappear with the pod. `ttlSecondsAfterFinished` and node GC delete the status object.

**Build instead.**
- **stdout → artifact, stderr → log.** Every remote stage writes stdout to its content-addressed output (see `pipeline-compiler.md` §4) and stderr to the container log. `2>&1` in the user's command is honoured inside the container; it does not change the transport.
- **Exit code → receipt.** The session watches (not polls) the Job/Pod and persists `exitCode`, `reason`, `startedAt`, `finishedAt` into the receipt before the object can be collected. `$?` and `status pid` read the receipt, never the API.
- **Interactive = bare Pod** with `stdin: true`, `tty: true`, `stdinOnce: true`, attached over WebSocket; resize on channel 4. Never a Job.
- **Terminal feedback for pipelines** streams stderr live and the final stage's artifact to the terminal, honouring early close (`head`).

## 2. Signals — **Adjust**

**Holds.** SIGTERM maps to deletion with `terminationGracePeriodSeconds`; SIGKILL to grace 0. CRIU is the only way to move process memory.

**Wrong.**
- **PID 1 trap.** The kernel does not deliver default-action signals to PID 1 of a pid namespace. `kubectl exec … kill -TERM 1` is a no-op for any program without an explicit handler, and `sh -c "a | b"` as PID 1 never forwards signals to its children. Both docs in this folder build on `kill -TERM 1`.
- **Wrong target.** A process backed by a Deployment or Job is not its pod. Deleting the pod makes the controller create another. SIGTERM must act on the owning object.
- **SIGSTOP ≠ CRIU.** Checkpointing writes memory to disk and releases the pod; that is `pause`, not stop. Kubernetes exposes a kubelet checkpoint API (beta since 1.30, behind `ContainerCheckpoint`) that writes a tar on the node, and **no restore API**. Restore means building an OCI image from the tar and starting a new pod; CRI-O supports it, containerd support is partial. GPU state needs `cuda-checkpoint`. Established TCP connections do not survive a node change.
- Force deletion (`--grace-period=0 --force`) removes the API object before the kubelet confirms the kill; with StatefulSets this can produce two pods with the same identity.

**Build instead.**
- **Toolbox entrypoint is a real init** (`tini` or `dumb-init`) and user commands run under `exec`, so TERM/HUP/USR1 reach the process. The annotation database records whether a command installs handlers.
- **Signal routing table by backing object:**

| Signal | Bare Pod | Job | Deployment |
|---|---|---|---|
| TERM | delete pod, grace G | `spec.suspend: true` if not started; else delete pods, grace G, mark cancelled in receipt | scale to 0, keep object and volumes |
| KILL | grace 1 s, no `--force` unless node is gone | same + delete Job | scale to 0, grace 1 s |
| STOP | cgroup v2 `cgroup.freeze=1` via node agent (keeps resources, memory intact) | same per pod | same per pod |
| CONT | `cgroup.freeze=0` | same | same |
| HUP | exec `kill -HUP` to the process (not PID 1) | n/a | `rollout restart` |
| `pause pid` (new verb) | CRIU checkpoint → artifact, release pod | per index | n/a |
| `resume pid` | new pod from checkpoint image, same node pool | per index | n/a |

- The freeze path needs a privileged DaemonSet (node agent) and is reported by `caps`. Without it, STOP is refused with the reason, not emulated by scale-to-zero (the first document's mapping, which kills the process).
- Grace period is a per-process property set from the annotation (`checkpoint_seconds`) so a training job gets 300 s to write its last checkpoint, not 30.

## 3. Secrets and environment — **Adjust**

**Holds.** Kubernetes Secrets are base64, not encrypted, until etcd encryption with a KMS provider is on. Secret volumes are tmpfs on the node. Namespace RBAC separates tenants.

**Wrong.**
- **External Secrets Operator writes Secrets into etcd.** The note says it avoids etcd "altogether"; the opposite is true. The component that avoids etcd is the **Secrets Store CSI driver** (mounts directly from Vault/ASM/Key Vault into the pod's tmpfs) or a Vault Agent sidecar.
- **`export` → ConfigMap per session** is over-engineering. Env is captured at pod creation; literal `env:` entries in the pod spec do the job. A ConfigMap is only needed when several pods must share a large config.
- **Env vars are the worst place for a secret.** They leak into child processes, crash dumps, `/proc/<pid>/environ` inside the same container, and application logs. "Containers cannot inspect neighbour processes" is true across pods, not within one.
- **Scrubbing secrets from history is heuristic.** If the value ever reaches the client, it has already leaked.
- **Missing the most important mechanism:** projected service-account tokens and workload identity (EKS Pod Identity / IRSA, GKE Workload Identity, AKS Workload Identity) give a process cloud credentials with no stored secret at all.

**Build instead.**
- **Secrets are references, never values, in the session.** `secret set DB_PASS` reads from a prompt or stdin and writes the Secret (or the external store) server-side. In commands the user writes `$DB_PASS`, the session compiles it to a `secretKeyRef` or a CSI mount; the value never exists client-side, so there is nothing to scrub.
- **Files by default, env on request.** `$DB_PASS` mounts at `/run/secrets/DB_PASS`; `--env` opts into an env var and prints a warning once.
- **Reference-only RBAC.** Users get `create pods` but not `get secrets` in their namespace. The kubelet reads the secret on their behalf. They can use what they cannot read.
- **Cloud credentials via workload identity** per process: `run --as=role/reporting` compiles to a service account bound to that role with a projected token (audience, 1 h expiry). This is the "ephemeral credentials per process" story from the capabilities table, and it needs no secret store.
- **Platform defaults:** etcd KMS encryption on, Secrets Store CSI installed, both reported by `caps`.

## 4. Multi-cluster routing — **Adjust**

**Holds.** `cd /clusters/eu-west-1/production/ns-a` is the right UX (story 1 of the missing stories). Move compute to data to avoid egress. Partition errors should look like network errors.

**Wrong.**
- **KubeFed is archived (2023).** Karmada is the CNCF project; KubeAdmiral and Open Cluster Management exist. Cluster API provisions clusters, it does not route to them.
- **No federation is needed for `cd`.** Switching cluster is a kubeconfig context change in the session. Federation matters only for a single process spanning clusters, which is rare and should be explicit.
- **Object storage is not a FIFO.** An S3 object is invisible until its upload completes; a reader cannot consume it "concurrently" while the writer streams. S3 has no multi-region bucket (Multi-Region Access Points plus asynchronous replication); GCS does. The cross-cloud pipe is the staged artifact with partition files from `pipeline-compiler.md`, not a stream.
- **"As if in the same rack" is false.** Cilium Cluster Mesh or Istio multi-primary over WireGuard give a flat network, not low latency; 10–200 ms and WAN bandwidth-delay limits remain. Mesh is for service traffic, not for data pipes.
- **Egress numbers belong in the plan.** AWS→GCP egress is on the order of $0.05–0.09/GB; a 1 TiB cross-cloud pipe is roughly $50–90 before compute. The planner must price it in the dry-run band and prefer moving the stage to the data.

**Build instead.**
- **Session cwd = (cluster, namespace).** `pwd` prints both; `ps` is scoped; `ps -A` fans out across contexts the user can reach. Fleet inventory from a simple list (Argo CD cluster secrets or Cluster API objects) populates `/clusters`.
- **Cross-cluster stage boundary = artifact in a bucket reachable from both sides**, with the egress cost in the plan and a warning when a cheaper placement exists (`explain` shows the alternative).
- **Mesh is optional and per-service**, enabled by the platform team, surfaced by `caps`, used for `run api-server --expose=global`, never by the pipeline compiler.
- **Disconnected mode** (from the "follow the energy" story): commands against an unreachable cluster queue in the session with a visible count, or fail with `EHOSTUNREACH`, by policy.

## 5. Autoscaling and quotas — **Adjust**

**Holds.** ResourceQuota and LimitRange are the guardrails. Pending pods drive Cluster Autoscaler/Karpenter. OOM → 137. CPU limits → CFS throttling.

**Wrong.**
- **Quota does not fail a Job fast.** The Job controller creates pods progressively; when quota is hit it logs events and retries indefinitely. The user sees a stuck pipeline, not `ExceededQuota`.
- **LimitRange defaults × fan-out = autoscaler storm.** A default 500 m CPU request applied to 1 648 `grep` partitions asks for 824 cores and provisions nodes to match.
- **Fargate is not a cold-start solution.** Pod start is 30–60 s, no GPUs, no DaemonSets, image pull dominates. Virtual Kubelet backends vary. The genuine cold-start fix is to not create a pod: the planner's local and parallel-local backends.
- **HPA does not apply to Jobs.** KEDA `ScaledJob` does, but the planner already fixes parallelism from input size; what it needs is mutable `spec.parallelism`, which Jobs support, plus elastic Indexed Jobs (1.27+).
- **Autoscaler scale-down kills long batch pods.** Consolidation evicts a 2-hour `sort` partition unless the pod carries `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` or Karpenter's `karpenter.sh/do-not-disrupt: "true"`.

**Build instead.**
- **Quota pre-check at plan time.** The plan knows partitions × requests; compare with the namespace quota and fail at dry-run with the number that does not fit (exit 3).
- **Requests per annotation class**, not from LimitRange: stateless text stages 100 m / 128 Mi; aggregates sized from input; typed engines from their own profiles. LimitRange stays as a ceiling only.
- **No CPU limits on batch stages** (requests only) to avoid throttling; memory limits stay.
- **Do-not-disrupt on every stage expected to exceed 10 minutes**, PodDisruptionBudget on interactive pods.
- **OOM policy:** 137 + `OOMKilled` triggers one retry at 2× memory for that partition, recorded in the receipt, and the next plan for the same hash starts from the larger size. This is the first useful "self-optimizing" behaviour and it is cheap.

## 6. Priority and preemption — **Feasible**, with one critical fix

**Holds.** PriorityClass tiers (interactive > pipelines > batch), preemption respects grace, Kueue/Volcano for cohorts, borrowing and gang scheduling. User class values must stay below 1e9 (system classes are 2e9).

**Fix.**
- **A preempted or evicted pod counts as a failure** against `backoffLimit`, so an interactive user starting a REPL can burn a training job's retries and its floor. Every Job the shell emits needs:

```yaml
podFailurePolicy:
  rules:
  - action: Ignore
    onPodConditions: [{type: DisruptionTarget}]
  - action: FailJob
    onExitCodes: {operator: In, values: [3]}       # policy denied inside the stage
  - action: Count
    onExitCodes: {operator: NotIn, values: [0]}
```

- **Preemption must be a stop with checkpoint, never a kill.** Kubernetes gives only the grace period, so: processes with `checkpoint_seconds` get a grace period long enough for a final checkpoint (up to 300 s), and `resume` restarts from it. Without checkpoint support the shell lowers the process's preemptibility instead (it runs in a Kueue queue that is not borrowed from).
- **Job suspension is the queue-level STOP.** Kueue admits by flipping `spec.suspend`; a queued-but-not-started job is already "stopped" for free.
- **Preemption wars** are handled by putting all pipeline work through Kueue local queues; raw PriorityClass preemption is reserved for interactive pods against batch pods, never batch against batch.
- `nice` maps to the workload's priority inside its Kueue queue; `renice` edits it (Kueue supports workload priority classes).

## 7. Compliance, audit, monitoring — **Adjust**

**Holds.** API audit log + runtime eBPF (Falco or Tetragon) + Fluent Bit/Vector to WORM object storage is the right stack. Secret values are not logged when the audit policy keeps Secrets at `Metadata` level.

**Wrong.**
- **The audit log does not see the local backend.** Most small commands run in the session and never touch the API. An audit trail built on the API server misses them by design.
- **Secret reads are attributed to the kubelet**, not to the user. The audit entry for a mounted secret shows the node's identity; attribution to a person goes through the pod's owner labels.
- **The audit log records manifests, not commands.** Nothing in a Job object says `grep error | awk`. Without a join, "what did Alice run" is unanswerable.
- **Managed clusters fix the audit policy.** On EKS/GKE/AKS you consume the provider's policy and sink; you cannot customize levels per resource. Design for that.
- **"Cryptographic certainty" is wrong.** Audit entries are not signed. Object Lock gives immutability after landing, not provenance.

**Build instead.**
- **The session is an audit source.** Every command, local or remote, is written to the same immutable stream with session id, user, cwd (cluster, namespace), the redacted command text, the plan hash and, when remote, the object names. `history` is a query on this stream, so users get cross-session history for free.
- **Command annotation on every object:** `convoy.io/cmd` (redacted), `convoy.io/session`, `convoy.io/pid`, `convoy.io/plan`. This is the join key between the API audit log and the session log.
- **Secret use is attributed from the pod spec**, not from the kubelet read: "pid 5678 (alice) mounted secret DB_PASS" is emitted by the session at launch.
- **Tetragon over Falco where enforcement is wanted** (in-kernel kill on policy), Falco where alerting is enough. Both are platform components reported by `caps`.
- **Retention is a policy per stream** (commands 1 y, stderr logs 90 d, receipts 7 y where chargeback requires), stated in the privacy policy, as done for the other products.

## 8. Cost allocation and chargeback — **Feasible**, with scope fixes

**Holds.** Labels on every object plus OpenCost/Kubecost give per-process compute and storage cost. Idle reservation and shared overhead can be allocated. Budget approaching → alerts/approvals.

**Wrong or missing.**
- **Object-store costs are invisible to OpenCost.** `GET`/`PUT` request charges and cross-region egress for artifacts are not Kubernetes resources. For data pipelines they are frequently the largest line.
- **"Automatically generate invoice line items mapped to AWS cost allocation tags"** needs a bridge: pods are not AWS resources. The bridge exists: **AWS Split Cost Allocation Data for EKS** (pod-level in the CUR by namespace/labels) and **GKE cost allocation** to BigQuery. Use the provider's pod-level allocation as the authoritative source where available, OpenCost elsewhere.
- **No cost model is defined.** Requests-based share, usage-based, or max(requests, usage)? Nodes bill regardless of usage. Pick one and print it on every receipt.

**Build instead.**
- **Cost model: max(requests, usage) share of node price**, OpenCost's default, stated on the receipt.
- **Artifact-store metering in the session:** the toolbox's `convoy-get`/`convoy-put` count bytes and requests per plan; the receipt adds them at the provider's list price. Egress is priced when source and destination regions differ.
- **Budget is enforced by the session**, not by Kubernetes (no native object): dry-run refuses plans whose upper band exceeds the remaining budget; runtime applies the declared action at the limit.
- **Showback by default, chargeback by export:** a daily job writes receipts to the finance sink with team/cost-centre labels.

## 9. Package management — **Adjust**

**Holds.** Day-0 infrastructure (clusters, VPCs, IAM, storage backends) is Terraform/OpenTofu's job; day-1/2 workloads are Helm/OCI; operators for stateful services; devcontainers for language dependencies.

**Wrong.**
- **One `pkg install` cannot do both tiers.** Operators and CRDs are cluster-scoped and need cluster-admin; a user's namespace cannot install CloudNativePG. The note blurs platform packages and user packages.
- **Helm does not resolve dependencies against the cluster.** `dependencies:` bundles subcharts; nothing checks "requires cert-manager". The "will create / requires" manifest from the missing stories has to be built.
- **Helm uninstall leaves StatefulSet claims behind.** "Handles persistent storage according to retention flags" is a promise the shell must implement itself (same orphan problem as `umount`).
- **The original meaning of install is gone.** Adding `jq` or `rg` to the commands available in a pipeline is an image-layer operation (the first document's kaniko/buildkit path), and this note dropped it.

**Build instead.**
- **Two tiers.** Platform packages (operators, CRDs, controllers) are installed by the platform team via GitOps and appear in `caps`. User packages are namespaced releases of pre-approved charts, installed by `pkg install` and reconciled by Argo CD/Flux so drift is visible and the audit trail is Git.
- **Two verbs.** `pkg` for services in the cluster (`pkg install postgresql-16`). `tool` for commands in the session's toolbox (`tool add jq`): builds a new toolbox layer, pins the version in the session, and **requires an annotation entry**, otherwise the tool is marked non-parallelizable and `explain` says so.
- **Package manifest:** `provides`, `requires` (checked against `caps`), `creates` (object kinds), `cost_estimate`, `retains` (which claims survive removal), `signature` (cosign). `pkg remove` prints what is retained and how to destroy it.

---

## Cross-cutting inconsistencies to settle once

1. **SIGSTOP** is scale-to-zero in the capabilities document, CRIU here, and cgroup freeze in this review. Decide: freeze for STOP, CRIU for the new `pause` verb, scale-to-zero never.
2. **Cross-cluster pipes** are an object-store FIFO in the routing note, a mesh socket in the same note, and a staged artifact in `pipeline-compiler.md`. Decide: staged artifact; sockets only for services.
3. **stdout** is a log stream in the exit-code note and an artifact in the pipeline compiler. Decide: artifact.
4. **Secrets** are env vars in the secrets note and files everywhere secure. Decide: files, env on request.
5. **The audit source** is the API server in the compliance note; it must be the session, with the API log as a secondary join.

## Suggested build order from this review

1. Toolbox image with init, `exec`, artifact helpers, metering, and the annotation database (unblocks pipes, signals, cost).
2. Session audit stream and object annotations (unblocks compliance and `history`).
3. Job template with `podFailurePolicy`, requests per class, do-not-disrupt, suspend-aware TERM (unblocks priority, autoscaling, signals).
4. Secrets as references + reference-only RBAC + workload identity.
5. `cd` across contexts with a fleet list.
6. `pkg`/`tool` with the manifest schema.
7. Node agent for cgroup freeze; CRIU `pause`/`resume` only where CRI-O is present.
