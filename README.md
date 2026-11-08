# Convoy Fleet Manager

**A POSIX-like shell whose primitives are Kubernetes objects.** `ps` lists Jobs, Deployments and Workflows as processes. `kill` scales, suspends or freezes them. `batch` queues work through Kueue. A pipeline compiles into a cached, partitioned DAG that runs the real `grep`, `sort` and `awk` over the cluster, without rewriting them.

> **Status: design phase.** This repository holds the architecture, the decisions and the reviewed research. There is no runnable code yet. The first milestone is the toolbox image and the command annotation database; see [ROADMAP.md](ROADMAP.md).

---

## Why

Kubernetes won the infrastructure layer and lost the user interface. Running a thousand copies of a script, training a model on 64 GPUs, or scheduling a nightly Spark job all mean writing YAML, learning a framework's execution model, or both. Machine-learning teams still choose Slurm over Kubernetes for exactly one reason: `sbatch` is a shell command and a PyTorchJob is not.

The shell is the most successful composition interface ever built. Convoy puts it back on top of the cluster that teams already own.

The idea is not new. Plan 9 gave every process its own namespace, Slurm gave batch its verbs, Modal turned functions into containers, Dagger put content-addressed caching under a shell syntax, and PaSh showed how to parallelize unmodified Unix pipelines. Convoy's contribution is combining them on Kubernetes with the leaks named and handled rather than hidden.

## What Convoy is

- **A session.** A server-side object, one per user, that parses commands, plans them, runs small ones itself, and records every command and every receipt. The laptop runs a thin terminal. Disconnecting changes nothing.
- **A planner.** Pipelines, loops and `xargs` compile to a DAG. Each stage runs in one of three backends, in order: locally in the session, in one pod split across cores, or distributed across Indexed Jobs or Argo Workflows. The input size, which is known, decides. The output size, which is not, is never guessed.
- **A toolbox.** One image carrying the real POSIX utilities, a real init, and artifact helpers. Every command carries an annotation that says whether it can run over partitions and how its outputs merge. Unknown commands run on one node, and `explain` says why.
- **A contract.** A process does not exit, it converges or it does not. Every plan carries a best case, a floor, a deadline and a budget. Exit codes are derived from the contract, and a preempted pod never counts as a failure.
- **A ledger.** Cost is a band at dry-run, a column in `ps` at runtime, and a receipt at the end. The receipt is the run history that tightens the next estimate.

## What Convoy is not

- Not a translation of `grep`, `awk` or `sort` into Spark or SQL. The binaries run unchanged; results are identical to Linux by construction.
- Not a stream processor. A pipe is a staged, content-addressed artifact. Streams are opt-in and lose caching, and the planner says so.
- Not a platform-as-a-service. The platform team owns the defaults, the storage classes, the queues and the packages. Convoy makes those defaults visible through `explain`, not invisible.
- Not an emulation of the kernel. `fork`, `ptrace`, shared memory and real-time scheduling do not cross pod boundaries. Where POSIX semantics cannot hold, the difference is surfaced.

## The three definitions

| Term | Convoy definition | Why it matters |
|---|---|---|
| **Pipe** | A staged, typed, content-addressed artifact between two stages. | Reruns skip unchanged stages. A restart downstream loses nothing. |
| **Command** | A verb with a declared class: stateless, aggregate, pattern, limit, fan-in, fan-out, typed, or unknown. | The class is the optimizer's algebra. Without it there is nothing to reason about. |
| **Session** | A server-side process that owns the mount table, the env, the cwd (cluster, namespace, path), and the audit stream. | Every other abstraction stores its state here. |

## A session, end to end

```sh
$ cd /clusters/prod-east/teams/data
$ ls /s3/logs/2026-11/ | wc -l
1648

$ explain 'cat /s3/logs/2026-11/*.log | grep error | awk "{print $1}" | sort | uniq -c | sort -rn | head -n 20'
plan 7f3a…  backend: distributed   input: 412 GiB (1 648 objects)   partitions: 1 648
  stage 1  map     [grep | awk | sort]       stateless+partial   1 648 × 256 MiB   cached: 0
  stage 2  merge   [sort -m | uniq -c]       global order         64-way tree
  stage 3  tail    [sort -rn | head -n 20]   topk_rewrite at stage 1 ✓
exit contract: floor 100 %   deadline: none   budget: none
cost band: $3.10 – $5.40   (compute $2.9–4.8, S3 GET 1 648 × $0.0004)
requires: argo ✓  artifact store ✓  toolbox 1.4 ✓

$ batch --min-success=95% --budget=10 'cat /s3/logs/2026-11/*.log | grep error | … | head -n 20' > top-errors.txt
[5678] queued in team-data (position 2)

$ ps
PID   TYPE      STAGES  PARTS  DONE   COST    STATE
5678  pipeline  3       1648   1203   $2.14   running
5001  training  1       64gpu  -      $412    running (preemptible, ckpt 12 min ago)

$ wait 5678; echo $?
0
$ time 5678
wall 11m02s   cpu 41h12m   partitions 1648 (1648 ok)   cache hits 0   cost $4.31

$ kill -STOP 5001        # cgroup freeze, memory intact, still billed
$ kill 5001              # scale to zero, checkpoint and volumes retained
```

Everything above is specified in [docs/pipeline-compiler.md](docs/pipeline-compiler.md) and [docs/debate-review.md](docs/debate-review.md). None of it runs yet.

## Architecture

```
 terminal (thin client)
     │  WebSocket attach
     ▼
 ┌──────────────────────── session pod (one per user) ─────────────────────────┐
 │ parser → AST → planner ─► plan (stages, partitions, backend, contract, cost)│
 │ mount table  /dev /clusters /pvc /s3 /artifacts env table    cwd            │
 │ local backend (real binaries, real pipes)      audit stream  receipts       │
 └───────────┬───────────────────────┬───────────────────────┬─────────────────┘
             │                       │                       │
   parallel-local pod        Indexed Job (map+merge)    Argo Workflow (DAG)
   toolbox image, tmpfs      Kueue queue, podFailure-   fan-out, conditionals,
   PaSh-style split/merge    Policy, floors             multiple merges
             │                       │                       │
             └───────────────────────┴───────────────────────┘
                                     ▼
                  content-addressed artifact store (S3/GCS/MinIO)
                  id = sha256(cmd, toolbox, env, input ids, partition)
```

Platform components Convoy expects and reports through `caps`: an artifact store, Kueue, Argo Workflows, volume snapshots, a GPU device plugin or DRA, a node agent for cgroup freeze, Secrets Store CSI, OpenCost. Each verb declares what it requires, and a missing requirement fails at pre-flight with the capability's name, never at runtime with a Kubernetes error.

## Design principles

1. **Expose emulation, not illusion.** Where semantics differ from Linux, `explain` says so. Nothing is hidden behind a heuristic.
2. **Never decide from a number you do not have.** Input size is known at plan time; output size is not. Plans use the first and never the second.
3. **Local first.** Most commands are small. The fastest backend is the session itself. Pods are an escalation, not a default.
4. **The binary is the semantics.** Text tools run unchanged over partitions. Translation into engine operators is reserved for typed verbs over typed data.
5. **Detach is not destroy.** `umount`, `kill` and `rm` never delete data. Destruction has its own verbs, confirmation, and a snapshot first.
6. **Secrets are references.** A value never reaches the client. Users may use secrets they cannot read.
7. **Preemption is a stop, not a kill.** Disruption never burns retries or floors, and checkpointable processes get the grace period they need.
8. **The platform team owns the defaults.** Storage classes, queues, priority tiers, packages and audit sinks are theirs. Convoy makes them legible.
9. **Cost is part of the contract.** A band at dry-run, a column at runtime, a receipt at the end. The budget binds to the upper bound.
10. **The session is the audit source.** The API server's log is joined to it, not the other way round.

## Repository layout

```
README.md                 this file
ROADMAP.md                build order and milestones
CONTRIBUTING.md           how to propose and review a decision
LICENSE                   Apache-2.0
docs/
  README.md               index of design documents
  pipeline-compiler.md    how |, &&, xargs, for and tee compile; annotations; backends; contract
  debate-review.md        builder review of the nine research takes; cross-cutting decisions
  posix-command-map.md    all 155 POSIX utilities mapped to backing, approach, feasibility, pull
  stories.md              user stories the research missed, and abstractions still to design
  adr/                    architecture decision records
toolbox/
  Dockerfile              toolbox image skeleton (init, coreutils, helpers)
  annotations.yaml        the command annotation database seed
examples/
  *.yaml                  compiled targets from the design docs, validated YAML
research/
  README.md               provenance of the research takes
  notes/                  the AI-generated takes, converted to text, kept for the record
```

## Where the design stands

| Area | State |
|---|---|
| Pipeline compiler, annotation classes, three backends, caching | Specified |
| Completion contract and exit codes | Specified |
| Signals, stdout/stderr, secrets, scheduling, cost, packages | Reviewed and corrected in `debate-review.md` |
| POSIX command map | Complete for the 155 utilities |
| Mount namespace and fileserver | Reviewed, superseding design pending |
| Session placement, identity and ownership, networking | Identified as the three architecture-changing gaps, not yet designed |
| Code | None |

## Prior art

PaSh and DiSh (MIT/UPenn), POSH (Stanford), Hadoop Streaming, Plan 9, Slurm, Modal, Dagger, Argo Workflows, Kueue, Nushell, k9s. The design documents cite each where it applies.

## Contributing

Read [CONTRIBUTING.md](CONTRIBUTING.md). The short version: propose changes as decision records, and any claim about Kubernetes behaviour must name the version it holds for.

## License

Apache License 2.0. See [LICENSE](LICENSE).
