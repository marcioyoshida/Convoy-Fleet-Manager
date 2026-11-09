# Architecture, layer by layer

The chart below is the layered view of Convoy. Every box names the decision record that settles it, so a reader can go from a box to the reasoning in one hop. The ASCII sketch in the root `README.md` is the same picture reduced to the data path of one pipeline.

```mermaid
graph TB
    subgraph User["👤 User perspective (virtual)"]
        direction TB
        Terminal["Terminal, thin client<br/>WebSocket attach, detach, reattach<br/>works from any device"]
        Shell["POSIX-like shell<br/>ps · top · kill · wait · time<br/>cd · ls · df · mount · export<br/>pipelines · batch · run · scale"]
        Scripts["Existing scripts and tools<br/>bash · make · xargs · CI pipelines<br/>agent orchestration"]
        Mental["Familiar mental model<br/>processes · files · env<br/>crontabs · pipes · devices"]
    end

    subgraph Session["⚙️ Session pod, one per user (ADR 0005)"]
        direction TB
        Parser["Parser<br/>POSIX grammar to AST<br/>reuses mvdan/sh"]
        Planner["Planner and pipeline compiler<br/>stages · partitions · backend choice<br/>completion contract · cost band<br/>explain (ADR 0001, 0003, 0004, 0013)"]
        Annot["Annotation database<br/>stateless · aggregate · pattern · limit<br/>fan-in · fan-out · typed · non-parallel<br/>(ADR 0002)"]
        Mounts["Mount table and device tree<br/>/dev/k8s · /dev/queue · /dev/table by-id and views · /sys<br/>/clusters · /pvc · /s3 · /queues · /tables · /artifacts<br/>(ADR 0011, 0012, 0015)"]
        State["Env table · cwd (cluster, namespace, path)<br/>secret references (ADR 0007)<br/>signal router (ADR 0008)"]
        Audit["Audit stream · receipts<br/>stdout is an artifact, stderr a log,<br/>exit code a receipt (ADR 0009)"]
    end

    subgraph Backends["🚀 Execution backends, local first (ADR 0003)"]
        direction LR
        Local["Local<br/>real binaries, real pipes<br/>inside the session pod"]
        ParLocal["Parallel-local<br/>one toolbox pod<br/>tmpfs split and merge"]
        Indexed["Distributed<br/>Indexed Job, map and merge<br/>successPolicy · backoffLimitPerIndex"]
        Argo["Distributed DAG<br/>Argo Workflow<br/>for · tee · conditionals · fan-out"]
    end

    subgraph K8s["☸️ Kubernetes API servers, one per cluster, no federation"]
        direction TB
        Workloads["Workloads<br/>Pod · Job · CronJob · Deployment<br/>podFailurePolicy · do-not-disrupt"]
        Storage["Storage<br/>PVC · StorageClass · VolumeSnapshot<br/>CSI · ReadWriteOncePod"]
        Identity["Identity and config<br/>ServiceAccount · workload identity<br/>Secrets Store CSI · ConfigMap · Lease<br/>owner labels · CEL admission"]
        Platform["Platform components, reported by caps<br/>Kueue · Argo Workflows · node agent (cgroup freeze)<br/>DRA · Gateway · NetworkPolicy · OpenCost<br/>Strimzi · Flink operator · packages via GitOps (ADR 0010)"]
        Inventory["Fleet inventory, the udev of /dev/k8s<br/>Argo CD cluster secrets or Cluster API<br/>by-id · views · permissions · hotplug events"]
    end

    subgraph Infra["🏗️ Underlying infrastructure"]
        direction LR
        Clusters["Clusters as devices<br/>EKS · AKS · GKE · on-prem<br/>edge · constellation"]
        Runtime["Container runtime<br/>CNI · CSI drivers<br/>CRI-O for checkpoint, restore"]
        CAS["Content-addressed artifact store<br/>S3 · GCS · Blob · MinIO<br/>id = sha256(cmd, toolbox, env, inputs, partition)"]
        Queues["Logs and queues as devices<br/>Kafka · Redpanda · NATS · SQS, in or outside a cluster<br/>bounded read = file, follow = stream, write = side effect (ADR 0012)"]
        Tables["Tables and catalogs as devices<br/>Iceberg · Delta · Hudi · BigQuery · Snowflake<br/>a read pins a snapshot, a write is one commit (ADR 0015)"]
        Cloud["Cloud identity and pricing<br/>STS · Entra ID · GCP SA<br/>price lists for the cost band"]
    end

    Mental --> Shell
    Shell --> Terminal
    Scripts --> Terminal
    Terminal --> Parser

    Parser --> Planner
    Annot --> Planner
    Mounts --> Planner
    State --> Planner
    Planner --> Audit

    Planner --> Local
    Planner --> ParLocal
    Planner --> Indexed
    Planner --> Argo

    ParLocal --> Workloads
    Indexed --> Workloads
    Argo --> Platform
    Platform --> Workloads
    Local -.-> Audit

    Mounts --> Inventory
    Mounts --> Storage
    State --> Identity
    Inventory --> Clusters

    ParLocal --> CAS
    Indexed --> CAS
    Argo --> CAS
    Mounts --> CAS
    Mounts --> Queues
    Mounts --> Tables

    Workloads --> Clusters
    Storage --> Clusters
    Identity --> Cloud
    Clusters --> Runtime
    Planner -.->|"/sys price rate, capacity"| Cloud

    classDef virtual fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#000
    classDef session fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000
    classDef backend fill:#fce4ec,stroke:#ad1457,stroke-width:2px,color:#000
    classDef k8s fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#000
    classDef infra fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px,color:#000

    class Terminal,Shell,Scripts,Mental virtual
    class Parser,Planner,Annot,Mounts,State,Audit session
    class Local,ParLocal,Indexed,Argo backend
    class Workloads,Storage,Identity,Platform,Inventory k8s
    class Clusters,Runtime,CAS,Queues,Tables,Cloud infra
```

## What changed from the original layered sketch

The first version of this chart, drawn before the design documents, had four layers: user, translation layer, Kubernetes control planes with their "abstracted counterparts", and infrastructure. Five things moved.

| Original box | Now | Why |
|---|---|---|
| Translation layer as a local shell runtime | **Session pod**, one per user, server-side, with a thin terminal client | A laptop sleeping must not kill a pipeline; the audit stream must not be client-controlled ([ADR 0005](adr/0005-session-is-a-server-side-object.md)) |
| Primitive mapper | **Planner** driven by the **annotation database** | A command is not mapped to one object. Its annotation class decides whether it runs locally, in one toolbox pod, or as an Indexed Job, and what the completion contract is ([ADR 0002](adr/0002-commands-carry-annotation-classes.md), [ADR 0003](adr/0003-three-backends-local-first.md)) |
| Pipeline compiler feeding orchestration engines (Argo, Tekton, SparkApplication, Kubeflow) | A separate **execution backends** layer: local, parallel-local, Indexed Job, Argo Workflow | Text tools are never translated to Spark or SQL. Real binaries run over partitions. Spark, Kubeflow and the rest are packages the platform installs, not translation targets ([ADR 0001](adr/0001-pipe-is-a-staged-artifact.md), [ADR 0010](adr/0010-platform-defaults-and-two-package-tiers.md)) |
| Multi-cluster resolver feeding Federation (Karmada, KubeFed) | **Mount table and device tree** talking to one API server per cluster, with the fleet inventory as the udev of `/dev/k8s` | No federation. A cluster is a device, mounted by the session with identity and scope options. `all` is a read-only union ([ADR 0011](adr/0011-device-tree-and-mount-table.md)) |
| Configuration (Secret, ConfigMap) as a counterpart of `export` | **Identity and config**: secret references, workload identity, Secrets Store CSI, owner labels | A secret value never reaches the client; users may use secrets they cannot read ([ADR 0007](adr/0007-secrets-are-references.md)) |

Two boxes are new. The **content-addressed artifact store** is the transport of every non-local pipe, so it sits in infrastructure next to the clusters rather than being hidden inside a workflow engine. **Cloud identity and pricing** is where device open performs its token exchange and where the cost band reads its rates.

The **logs and queues** box was added with [ADR 0012](adr/0012-logs-and-queues-are-devices.md): Kafka and its relatives are devices under `/dev/queue`, mounted by the session, whether or not a broker runs inside a cluster. There is deliberately no streaming backend box; a continuous pipeline is a service unit and Flink is a package. The **tables and catalogs** box follows with [ADR 0015](adr/0015-tables-and-catalogs-are-devices.md): lakehouse catalogs and warehouses are devices under `/dev/table`, a read pins a snapshot at plan time, a write is one commit keyed by its input, and a warehouse is never a backend, only a device whose reads are priced queries.

The **planner** box's `explain` is specified in [ADR 0013](adr/0013-native-verbs-have-one-shape.md), which also closes the set of native verbs and gives each the same seven fields. [ADR 0014](adr/0014-assisted-completion-the-model-proposes.md) adds nothing to the chart on purpose: completion is computed in the session from the annotation database and the device tree, and a language model, where one is attached, is one more device class under `/dev/model` in the mount table, not a box of its own.

## Reading the edges

- **Solid edges** are the data and control path of a command: terminal to parser, parser to planner, planner to one of four backends, backends to Kubernetes objects, objects to clusters.
- **Dotted edges** are reads the planner performs before anything runs: capacity and price rate from `/sys`, which produce the cost band in `explain`.
- **Nothing crosses from one API server to another.** Cross-cluster work is two stages with an artifact in the store between them and the egress priced at dry-run.
- **The audit stream is fed by the planner and by the local backend**, not by the API server. The API server's log is joined to the session's stream, never the other way round ([ADR 0009](adr/0009-stdout-artifact-stderr-log.md)).

## What is still missing from the chart

Networking as sockets (`expose`, Gateway routes, NetworkPolicy as the default-deny) is shown only as platform components because the verbs are not designed yet. Session placement, meaning which cluster hosts the session pod and what happens when it is lost, is also undecided. Both are listed in [stories.md](stories.md) and on the [roadmap](../ROADMAP.md) as M11 and M2.
