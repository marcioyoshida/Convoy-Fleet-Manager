# Convoy — Compiling Unix pipelines into distributed DAGs

**Status:** design, supersedes *How would Unix-style pipes compile into distributed DAGs* (2-page note, 2026-11-07).
**Scope:** the data-flow half of the shell: what `|`, `&&`, loops and `xargs` compile to, how stages exchange data, how the planner picks an engine, and what failure, cost and exit codes mean for a pipeline.
**Depends on:** the command annotation database (§3), content-addressed artifacts (§4), the completion contract (§7). Capability discovery (`caps`) reports which backends a cluster supports.

## 0. What changed from the superseded note

| Superseded note said | This document says | Why |
|---|---|---|
| Compiler decides at compile time whether intermediate data is "small enough" for stdout or needs an artifact repository | Every stage output is an artifact. Input sizes are known at plan time; output sizes never are. Inlining small outputs is a runtime optimization | A size heuristic on an unknown quantity is the same trap as "commands under 30 s become Jobs" |
| `grep`, `awk`, `sed`, `sort` are translated into Spark DataFrame operators | The real binaries run over partitions and their outputs are merged. Nothing is rewritten | `awk -F','` ≠ a CSV reader on quoted commas; `sort \| uniq -c` ≠ `groupBy().count()` on output order; awk and sed are Turing-complete. Translation changes results |
| Two backends: Argo and Spark | Three backends in order: local, parallel-local, distributed. Local is the default | Most pipelines are small. The fast path is the session itself |
| The query-optimizer analogy | The analogy holds only once commands carry an algebra. The annotation database is that algebra | SQL optimizers work because every operator's properties are known. Arbitrary binaries have none until declared |
| Linear pipe → linear DAG | Width comes from `for`, `xargs -P`, `tee`, `fork`; depth from `\|`; joins from `cat`, `sort -m`, `wait` | A chain is a degenerate DAG. The note never showed where fan-out comes from |
| Trade-off table: execution model, failure, latency | Adds caching, ordering, `pipefail`, limit pushdown, records vs lines, cost | These are the items that decide whether a rerun is cheap and whether results are correct |
| Argo example starts at `grep`, omits `sort`/`uniq`, shows no artifact passing | Complete examples with artifacts, fan-out and merge (§9) | The hard part must be in the example or the example proves nothing |

Prior art this document builds on, which the note did not cite: **PaSh** (MIT/UPenn, EuroSys 2021), **POSH** (Stanford, USENIX ATC 2020), **DiSh** (distributed PaSh, NSDI 2023), **Hadoop Streaming** (2008). All four run unmodified Unix binaries over partitions and merge, instead of translating them. **Dagger** for content-addressed stage caching. **Argo Workflows** and **Kubernetes Indexed Jobs** as compiled targets. **Nushell** for the structured path.

## 1. Definitions used here

- **Pipe.** A staged, typed, content-addressed artifact between two stages. Not a byte stream. Streams are opt-in (§8.5).
- **Command.** A verb with a declared class from the annotation database. Unknown commands get the most conservative class and a warning.
- **Stage.** One or more fused commands that run inside a single container or local process on one partition.
- **Plan.** The DAG the planner emits: stages, partitions, merge points, the backend per stage, and the estimated cost band.
- **Session.** The local process that parses, plans, runs the local backend, and records every plan and receipt so a transcript can be replayed as a workflow.

## 2. Parsing: from a command line to an AST

The parser is a POSIX shell parser (reuse an existing one, e.g. mvdan/sh or the PaSh parser). It yields an AST where the planner distinguishes:

| Syntax | Meaning in the plan |
|---|---|
| `a \| b` | Data edge a→b |
| `a && b` | Control edge: b starts after a exits 0 |
| `a \|\| b` | Control edge: b starts after a exits non-zero (fallback policy) |
| `a ; b` | Control edge, unconditional |
| `a &` … `wait` | Detached branch; `wait` is a join |
| `for x in …; do a; done` | Fan-out: one branch per item (map) |
| `xargs -P n a` | Fan-out with bounded parallelism |
| `tee >(a) >(b)` | Fan-out of one artifact to two branches |
| `wait` (no operands) | The DAG join: barrier on every background branch of the previous command; `wait $!` joins one branch (walkthroughs.md, gap G1) |
| `cat f1 f2`, `sort -m`, `join`, `paste` | Fan-in |
| `a < f`, `a > f` | Artifact resolution: `f` becomes an input or output artifact id |
| `set -o pipefail` | Pipeline exit code = first non-zero stage (default **on**, unlike POSIX) |
| `fork N a`, `scale pid N` | Convoy verbs: Indexed Job fan-out; Deployment replicas |

Variables, globs and subshells are resolved in the session before planning. Globs over `/s3/...` and `/hdfs/...` mounts resolve to lists of artifact ids.

## 3. The command annotation database (the algebra)

The planner can only parallelize what it understands. Each command carries an entry. Entries are versioned with the toolbox image so a plan is reproducible.

```yaml
grep:
  class: stateless           # f(concat(parts)) == concat(f(parts))
  split: lines
  merge: concat
  order: preserved-within-partition
  limit_pushdown: true

tr: {class: stateless, split: lines, merge: concat}
cut: {class: stateless, split: lines, merge: concat}

sort:
  class: aggregate
  split: lines
  partial: sort              # run on each partition
  merge: "sort -m"           # k-way merge of sorted partitions
  order: global
  topk_rewrite: true         # sort | head -n K  →  per-partition top-K, then merge

uniq -c:
  class: aggregate
  requires: sorted-input
  partial: "uniq -c"
  merge: sum-by-key          # merge program from the toolbox image

wc -l: {class: aggregate, partial: "wc -l", merge: sum}
wc -c: {class: aggregate, partial: "wc -c", merge: sum}

head -n:
  class: limit
  pushdown: cancel-upstream  # the SIGPIPE equivalent (§5.1)

awk:
  class: pattern              # stateless only for a recognized subset
  stateless_if: "program has no BEGIN/END, no user variables, no getline, no NR/FNR"
  else: non-parallelizable

sed:
  class: pattern
  stateless_if: "no hold space (h/H/g/G/x), no N/D/P, no line-range addresses"
  else: non-parallelizable

cat: {class: fan-in, merge: concat}
tee: {class: fan-out}

# Typed verbs for the structured path (§8.4). These DO map to engine operators.
select: {class: typed, engine: dataframe, op: select}
filter: {class: typed, engine: dataframe, op: filter}
```

Rules:

1. **Unknown command → non-parallelizable.** It runs on one node and `explain` prints the reason. There is no silent guess.
2. **Side effects block fusion across partitions.** A command declared `side_effects: true` (writes outside its stdout, e.g. `curl -X POST`) never runs in more than one partition without an explicit `fork`.
3. **Order is a property, not an afterthought.** A stage downstream of an `order: global` command receives the merged, globally ordered artifact. A downstream `uniq` after `sort` is therefore correct.
4. **Merge programs ship in the toolbox image** (`sum-by-key`, `sort -m`, `concat`). They are tiny, tested, and versioned with the database.

## 4. Artifacts

- **Every stage output is an artifact.** Id = `sha256(command text, toolbox version, env subset, ordered input ids, partition index)`. Two runs with the same id share one object. This is the cache.
- **Input size is known at plan time.** Inputs are either artifacts (size recorded) or files on a resolvable mount (`stat`). Output size is unknown and is never used by the planner.
- **Small outputs are inlined at runtime**, not at compile time: the executor records outputs under a threshold (default 256 KiB, Argo's result limit) alongside the artifact so the next stage can skip a fetch.
- **Local files become artifacts on first use.** `cat data.csv` where `data.csv` is in the session's working directory uploads it to the artifact store under its hash. A second run with an unchanged file uploads nothing.
- **Final stdout goes back to the terminal.** The last stage's artifact is streamed to the session, with `head`-style early close honoured (§5.1).
- **Retention follows the session's policy**, default 7 days for intermediates, keep-forever for anything referenced by a receipt the user pinned.

## 5. The planner

Order of attempts, first that fits wins. Thresholds are platform-profile defaults; the user overrides with `--engine` or a `@remote` marker.

| Backend | When | Where it runs |
|---|---|---|
| **local** | total input < 256 MiB and every command is available locally | the session process |
| **parallel-local** | input fits one node (profile: 64 GiB) and ≥ 1 stage is `stateless` or `aggregate` | one pod with the toolbox image, PaSh-style split/merge on tmpfs |
| **distributed** | input larger than a node, or the user asked, or a stage is a Convoy verb (`fork`, `train`) | Argo Workflow or Indexed Job over `ceil(input / 256 MiB)` partitions, capped by quota |

Partition count is a plan-time number derived from the known input size, which is the only size the planner may use.

### 5.1 Optimizations

- **Stage fusion.** Adjacent stateless commands fuse into one stage (`grep error | awk '{print $1}'` is one container). A `partial` aggregate fuses onto the end of a fused stateless chain (`… | sort` per partition).
- **Limit pushdown.** `head -n K` cancels upstream once K lines are produced. In the distributed backend the executor stops partitions early and the plan annotates the stage as `early-close`. With `topk_rewrite`, `sort | head -n K` becomes per-partition top-K then merge.
- **Filter-before-shuffle.** A stateless stage is always placed before an aggregate's merge, never after.
- **Cache hits.** Any stage whose output id already exists is skipped. `explain` marks it `cached`.
- **Merge trees.** More than 64 partitions feeding one `merge` form a two-level tree so a single merge pod is never the bottleneck.

### 5.2 `explain`

```
$ explain 'cat /s3/logs/2026-11/*.log | grep error | awk "{print $1}" | sort | uniq -c | sort -rn | head -n 20'
plan 7f3a…  backend: distributed   input: 412 GiB (1 648 objects)   partitions: 1 648
  stage 1  map     [grep | awk | sort]          stateless+partial   1 648 × 256 MiB   cached: 0
  stage 2  merge   [sort -m | uniq -c]          global order         64-way tree
  stage 3  tail    [sort -rn | head -n 20]      topk_rewrite: per-partition top-20 at stage 1 ✓
exit contract: floor 100 % (no --min-success given)   deadline: none   budget: none
cost band: $3.10 – $5.40   (compute $2.9–4.8, S3 GET 1 648 × $0.0004, egress 0)
requires: argo ✓   artifact store ✓   toolbox 1.4 ✓
```

## 6. Backends

### 6.1 local
The session forks the real binaries with real pipes. Nothing is staged except the final artifact if the user redirected to a mount. Latency is POSIX latency.

### 6.2 parallel-local
One pod, toolbox image, input split on tmpfs into `n = min(cores, input / 64 MiB)` partitions, PaSh-style. Output merged in the pod. Used when the data fits a node but the work benefits from cores. Cheapest way to make `sort` on 20 GiB fast.

### 6.3 distributed
Two targets, chosen by shape:

- **Indexed Job** for a single map stage with a merge (the common case). `JOB_COMPLETION_INDEX` is the partition key. Requires Kubernetes 1.31+ for `successPolicy` and `backoffLimitPerIndex`.
- **Argo Workflow** when the DAG has more than one merge point, conditional edges (`&&`, `||`), or fan-out from `for`/`tee`.

Both carry labels `convoy.io/pid`, `convoy.io/session`, `convoy.io/plan` so `ps`, `cost` and the audit log can find them without a session-side state store.

## 7. Failure semantics: the completion contract

A pipeline does not "exit"; its stages converge or not. Each plan carries a contract:

| Field | Default | Source |
|---|---|---|
| floor | 100 % of partitions | `--min-success=95%` |
| retries per partition | 2 | `--retries` |
| deadline | none | `--deadline=2h` |
| budget | none | `--budget=50` |

Exit codes derived from the contract:

| Code | Meaning |
|---|---|
| 0 | best case: every partition succeeded |
| 0 + `degraded` flag | floor met, some partitions failed; `status pid` lists them |
| 1 | floor unreachable. **Fail fast:** the moment failures exceed `partitions − floor`, remaining partitions are cancelled |
| 2 | deadline or budget exceeded; the budget action (pause-with-checkpoint by default, kill if declared) has run |
| 3 | policy denied (quota, admission, unknown command in a `--strict` plan) |

- `pipefail` is on by default. The pipeline's code is the first non-zero stage's code, after contract evaluation.
- `a && b`: b runs only on code 0 **without** the degraded flag unless `--accept-degraded` is set.
- `a || b`: b runs on 1, 2 or 3.
- `retry pid --failed` reruns only the failed partition indexes; everything else is a cache hit.
- **SIGPIPE equivalent.** When a downstream limit closes, upstream partitions receive cancel, not a failure. They do not count against the floor.

Mapping to primitives: floor → `successPolicy.rules[].succeededCount`; retries → `backoffLimitPerIndex`; deadline → `activeDeadlineSeconds`; fail-fast → the session deletes the Job once the floor is provably unreachable (Kubernetes does not do this on its own).

## 8. Semantics that must be preserved or declared

### 8.1 Ordering
Unix pipelines are ordered. The plan preserves order within a partition (stateless) and restores global order at every `order: global` merge. A plan never reorders silently; a stage that would require reordering for parallelism (e.g. `uniq` without a preceding `sort`) runs non-parallel and `explain` says why.

### 8.2 Lines vs records
Default split is on `\n`. Binary or record-oriented inputs (Parquet, Avro, gzip) are **not split by lines**. The annotation `split: file` keeps each object whole; splitting inside such files is only possible on the structured path (§8.4).

### 8.3 Text tools on typed data
`grep` on a Parquet file is allowed, runs, and matches bytes, exactly as on Linux. The planner does not "upgrade" it. If the user wants column semantics they use the typed verbs.

### 8.4 The structured path
Typed verbs (`select`, `filter`, `group`, `join` over Parquet, CSV-with-schema, Iceberg tables) carry `engine: dataframe` in the database. Only these compile to Spark, DuckDB or Polars operators, and the engine is chosen like any other backend: DuckDB in-session for small inputs, DuckDB in a pod for one-node inputs, Spark for larger. The mapping is exact because the operators are declared, not inferred from awk programs.

### 8.5 Streams, opt-in
`a |= b` (or `--stream`) requests a live channel instead of an artifact: a sidecar relay with bounded buffer and backpressure. The stage pair then loses caching and replay, and `explain` prints that. Use for `tail -f`-style sources and interactive feedback only.

## 9. Compiled examples

Pipeline used throughout:

```sh
cat /s3/logs/2026-11/*.log | grep error | awk '{print $1}' | sort | uniq -c
```

### 9.1 Indexed Job (map stage) + merge Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pid-5678-map
  labels: {convoy.io/pid: "5678", convoy.io/session: "sid-1234", convoy.io/plan: "7f3a", convoy.io/stage: "1"}
spec:
  completionMode: Indexed
  completions: 1648
  parallelism: 256                 # quota-capped
  backoffLimitPerIndex: 2
  successPolicy:
    rules: [{succeededCount: 1566}] # floor 95 %
  activeDeadlineSeconds: 7200
  ttlSecondsAfterFinished: 86400
  template:
    metadata:
      labels: {convoy.io/pid: "5678"}
    spec:
      restartPolicy: Never
      serviceAccountName: user-marcio
      containers:
      - name: stage
        image: registry.example.com/convoy/toolbox:1.4
        command: [sh, -o, pipefail, -c]
        args:
        - >
          convoy-get "$IN_PREFIX/part-$JOB_COMPLETION_INDEX"
          | grep error | awk '{print $1}' | sort
          | convoy-put "$OUT_PREFIX/part-$JOB_COMPLETION_INDEX"
        env:
        - {name: IN_PREFIX,  value: "cas/plan-7f3a/stage-0"}
        - {name: OUT_PREFIX, value: "cas/plan-7f3a/stage-1"}
        resources:
          requests: {cpu: "1", memory: "1Gi"}
          limits:   {cpu: "2", memory: "2Gi"}
---
apiVersion: batch/v1
kind: Job
metadata:
  name: pid-5678-merge
  labels: {convoy.io/pid: "5678", convoy.io/plan: "7f3a", convoy.io/stage: "2"}
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: stage
        image: registry.example.com/convoy/toolbox:1.4
        command: [sh, -o, pipefail, -c]
        args:
        - >
          convoy-get-all "cas/plan-7f3a/stage-1/" --as-files /work
          && sort -m /work/part-* | uniq -c
          | convoy-put "cas/plan-7f3a/stage-2/out"
```

`convoy-get`, `convoy-put`, `convoy-get-all` are the toolbox's artifact helpers; they verify hashes and honour inlined small outputs. The session creates the merge Job only when the map Job's status satisfies the contract; on floor-unreachable it deletes the map Job (fail fast).

### 9.2 Argo Workflow (same pipeline, fan-out + merge, with artifacts shown)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: pid-5678-
  labels: {convoy.io/pid: "5678", convoy.io/session: "sid-1234", convoy.io/plan: "7f3a"}
spec:
  entrypoint: pipeline
  serviceAccountName: user-marcio
  activeDeadlineSeconds: 7200
  arguments:
    parameters:
    - {name: parts, value: "1648"}
  artifactRepositoryRef: {configMap: convoy-artifacts, key: s3}
  templates:
  - name: pipeline
    dag:
      tasks:
      - name: map
        template: stage
        withSequence: {count: "{{workflow.parameters.parts}}"}
        arguments:
          parameters:
          - {name: cmd,  value: "grep error | awk '{print $1}' | sort"}
          - {name: part, value: "{{item}}"}
          artifacts:
          - name: in
            s3: {key: "cas/plan-7f3a/stage-0/part-{{item}}"}
      - name: merge
        dependencies: [map]
        template: merge-stage
        arguments:
          parameters:
          - {name: cmd, value: "sort -m /in/part-* | uniq -c"}
          artifacts:
          - name: in
            s3: {key: "cas/plan-7f3a/stage-1/"}   # directory key = all map outputs

  - name: stage
    inputs:
      parameters: [{name: cmd}, {name: part}]
      artifacts: [{name: in, path: /in/data}]
    outputs:
      artifacts:
      - name: out
        path: /out/data
        s3: {key: "cas/plan-7f3a/stage-1/part-{{inputs.parameters.part}}"}
    container:
      image: registry.example.com/convoy/toolbox:1.4
      command: [sh, -o, pipefail, -c]
      args: ["{{inputs.parameters.cmd}} < /in/data > /out/data"]
      resources: {requests: {cpu: "1", memory: "1Gi"}}

  - name: merge-stage
    inputs:
      parameters: [{name: cmd}]
      artifacts: [{name: in, path: /in}]
    outputs:
      artifacts:
      - name: out
        path: /out/data
        s3: {key: "cas/plan-7f3a/stage-2/out"}
    container:
      image: registry.example.com/convoy/toolbox:1.4
      command: [sh, -o, pipefail, -c]
      args: ["{{inputs.parameters.cmd}} > /out/data"]
```

The 95 % floor is not expressible in Argo's DAG directly; the session enforces it from the fan-out task statuses and marks the workflow degraded or cancels it. Argo is used for shapes with conditionals and multiple merges; for the single map+merge shape §9.1 is cheaper.

### 9.3 Structured path (typed verbs, exact engine mapping)

```sh
cat /s3/sales/2026/*.parquet | filter 'region == "BR"' | select cnpj, amount | group cnpj sum(amount) | head -n 20
```

Plan: input 38 GiB Parquet → engine `dataframe` → DuckDB in one pod (fits a node) → `SELECT cnpj, SUM(amount) FROM read_parquet(...) WHERE region = 'BR' GROUP BY cnpj LIMIT 20`. The limit is pushed into the query. Had the input been 4 TiB the same plan would target Spark with the identical logical plan. No text tool was rewritten.

## 10. Cost

- **Dry-run:** a band, not a number. Lower bound = requests × spot price × estimated duration from history (or a default per class); upper bound = limits × on-demand × deadline. Includes object-store request charges (`GET` per partition), egress if the artifact store and the cluster are in different regions, and storage of intermediates for the retention window.
- **Runtime:** `ps` shows `COST` so far and `BURN/H` from OpenCost allocation by `convoy.io/pid`.
- **End:** a receipt in the audit log with actual cost by stage. Receipts are the run history that tightens the next estimate for the same plan hash.
- **Budget binds to the upper bound.** If the band's upper bound exceeds the budget, the plan is refused at dry-run, not killed at runtime.

## 11. Non-goals

- Translating `grep`, `awk`, `sed`, `sort` or any text tool into Spark, Flink or SQL.
- Deciding anything from output size at plan time.
- Live streams as the default channel between stages.
- Auto-upgrading a text pipeline to a cluster engine because the input "looks big"; the planner scales the *same* commands, it does not swap them.

## 12. Open questions

1. Annotation coverage: how many of coreutils, plus common tools (`jq`, `rg`, `csvkit`), need entries before users stop hitting "non-parallelizable"? PaSh's library covers ~60 commands; start there.
2. Checkpointing inside a long aggregate (a 2 h `sort` partition): accept loss and retry, or periodic spill to the artifact store?
3. Top-K rewrite correctness with `sort -u` and locale-dependent collation: pin `LC_ALL=C` in the toolbox or expose collation as a plan parameter.
4. Argo-side enforcement of floors: wait for a native `successPolicy` equivalent, or keep it in the session as designed here.
5. Whether `fork N cmd` (explicit) and the planner's own partitioning (implicit) should be the same mechanism with different defaults. Current answer: yes, both emit §9.1.

## 13. Rules from the user-story probes · 2026-11-09

The stories in `docs/user-stories/` decided the following for the compiler and the planner. Each is also recorded under the ADR it touches.

- **L1.1** annotated commands hash their declared `env` list; `unknown` commands and executables hash the whole environment minus a platform-declared volatile list (`PWD`, `OLDPWD`, `CONVOY_PID`, `_`, `SHLVL`); `type` and `explain --cache` print the subset; a cache hit names the subset in the receipt.
- **L1.3** the planner computes the effective floor per stage from downstream classes and refuses a `--min-success` that a fan-in cannot honour (exit 3 at plan time).
- **L2.2** bands for executables are tagged `declared` in `explain`; the receipt records declared versus observed; a package may provide an estimator hook that reads the job config.
- **L5.3** `/sys/k8s/<id>/preemption_rate` per node group (ADR 0011 decision 7); the band widens by expected restart loss on spot; the receipt separates productive from lost time.
- **L11.3** queue reads carry `order: per-partition` as a stage property, like partitioned file reads; the compiler warns when an order-sensitive class follows without a global sort.
- **L12.2** on a follow the planner refuses aggregate and fan-in classes and `order: global`, and prints the two alternatives: a bounded read on a schedule, or an engine package with a window; `limit` (`head`) stays allowed and closes the follow.

