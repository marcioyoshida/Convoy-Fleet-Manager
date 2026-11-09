# US-04 · Go from notebook to a sweep to a served model without leaving the shell

| Field | Value |
|---|---|
| Primary persona | DS (Data Scientist) |
| Secondary personas | ML, FIN, PE |
| Workload class | interactive, then batch fan-out, then service |
| Walkthrough | [walkthroughs.md §4](../walkthroughs.md) |
| Gaps already closed there | none |
| Milestones needed | M1, M2, M3, M4, M5, M11 |
| Findings in this document | L4.1 – L4.6 |

## 1. Story

**As a** data scientist, **I want** to explore in a notebook with a GPU, build features from parquet with ordinary commands, run sixteen training configs in parallel, pick the best by a metric and serve it, **so that** the path from an idea to a served model is a shell history I can hand to a colleague.

**Context.** Churn models are retrained monthly. Today the notebook lives on a shared VM, the sweep is a hand-edited YAML per config, and nobody can say which data produced the model in production.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | the data scientist | personal identity on `/clusters/ml-training`, `ns=churn` | interactive |
| Approves | none | | serving to `--internal` needs no approval; a public `expose` would |
| Pays | team churn | | the idle notebook is the surprise line (L4.3) |
| Audits | the ML engineer who takes the model to production | follows `history --receipts` from the served model back to raw data | needs the chain to survive `cp` into the registry (L4.1) |
| Operates the platform | platform team | GPU device plugin, object-store registry mount, serving package | sets the idle policy |

## 3. Preconditions

- **Capabilities**: `gpu` (device plugin or DRA), `indexed-job`, `artifact-store`, `objstore` for `/models`, `service-unit`.
- **Packages**: `jupyter`, `train`, `serve` as user tools or packages with annotations.
- **Mounts**: `/s3/datasets` read-only; `/artifacts`; `/models` on an object-store device that stores metadata; `/home` for the notebook.
- **Annotations**: `select/filter/group/tjoin: typed`; `train: non-parallel, gpu 1, metrics via /sys/receipt` (L4.2); `serve: unit`.
- **Platform defaults** `explain` must print: GPU idle timeout for units, `xargs -P` fail-fast behaviour, content-id method for `/s3` objects.

## 4. Main path

```bash
$ run --gpu=1 --unit notebook jupyter && expose notebook --internal
$ batch features.sh --deadline=1h
$ ls /s3/datasets/churn/configs/*.yaml | xargs -P 16 -n 1 train --features /artifacts/9f3a… --config
$ history --receipts | sort -t, -k cost -n | tail -3
$ cp /artifacts/d02b… /models/churn/1.3
$ run --unit churn-api --gpu=1 --image=serve:1.3 --model=/models/churn/1.3 && scale churn-api 3 && expose churn-api
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `run --unit notebook` | service unit | Deployment 1/1 with a GPU, Service | `/home` volume |
| `batch features.sh` | typed DAG | Indexed Jobs | artifacts |
| `xargs -P 16 train` | fan-out, 16 indexes | Indexed Job, one GPU per index | 16 artifacts and receipts |
| `cp` into `/models` | copy with provenance | object with source-id metadata (L4.1) | object store |
| `run --unit churn-api` | service unit | Deployment, Service | image digest and model id pinned in the unit receipt |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ● | lineage from model to raw data is a chain of ids | `history --receipts --follow /models/churn/1.3` reaches the raw dataset ids |
| A2 | Command | ● | typed verbs, an executable per config, a unit | `type train` shows GPU request and metric channel |
| A3 | Session | ○ | a long interactive session with a notebook attached | the notebook survives a session restart |
| A4 | Backend | ● | typed stages on dataframe operators, sweep on Indexed Job | `explain` names both |
| A5 | Contract | ● | sweep floor default 100 % with fail-fast | one bad config cancels fifteen good runs unless told otherwise (L4.4) |
| A6 | Device tree | ○ | `/s3`, `/artifacts`, `/models` as devices | `mount` shows `/models` type objstore with metadata support |
| A7 | Signals | ○ | `kill` the notebook unit | scale to zero, `/home` retained |
| A8 | Secrets | – | none | |
| A9 | Cost | ● | per-config cost in receipts; idle GPU | `ps` shows IDLE for the notebook after the timeout (L4.3) |
| A10 | Identity | ○ | personal identity | the served unit runs under the team's service account, not the person |
| A11 | Defaults | ● | idle timeout, fail-fast, content-id method | all in `explain` |
| A12 | Storage | ○ | `/home` for the notebook | survives `kill` |
| A13 | Networking | ● | `expose --internal` and `expose` | networking design is open; stated dependency (L4.6) |
| A14 | Time | ○ | deadline on features | |
| A15 | Audit | ● | receipts carry metrics for selection | `auc` in the receipt comes from a defined channel (L4.2) |
| A16 | Durability | ○ | sweep survives the notebook dying | independent processes |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | the serving unit may be GitOps-managed in production | see US-03 L3.2 | ok here (dev namespace) |
| P2 | POSIX cannot hold | `xargs -P` semantics: GNU runs all and exits 123 on any failure; Convoy fail-fast cancels | different by design | **L4.4** |
| P3 | Orphaned state | 15 cancelled training runs' partial checkpoints | `partial` class (US-01 L1.5) | ok |
| P4 | Identity leak | none | | ok |
| P5 | Invisible cost | the idle GPU notebook | no idle policy in the design | **L4.3** |
| P6 | Lying cache | `train --config <s3 path>`: is the id built from the path or the content? | ADR 0001 says input ids; the S3 content id method is unstated | **L4.5** |
| P7 | Silent failure | `auc=0.912` in the receipt: who wrote it? | no channel | **L4.2** |
| P8 | Partial fleet | one cluster | n/a | ok |
| P9 | Created elsewhere | the registry object may be overwritten by another tool | metadata survives only if the other tool keeps it | L4.1 |
| P10 | Multi-party | hand-off from DS to ML engineer | the shell history and receipts | ok |
| P11 | Data at rest | `cp` out of `/artifacts` drops the content id | the copy is a new object under a mutable path | **L4.1** |
| P12 | Inheritance | the sweep budget across 16 indexes | one contract binds all | ok |

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| config 7 fails | index failure | 1 (floor 100 %) | 15 siblings cancelled (L4.4) | rerun with `--keep-going` |
| notebook idle 6h on a GPU | idle timer (L4.3) | n/a | unit stopped, `/home` retained | `kill -CONT notebook` |
| `serve:1.3` tag moved | digest pinned in unit receipt | n/a | digest differs from tag | `release` the new digest |
| registry object overwritten | metadata id mismatch | warning in `ls -l /models` | | `cp` again from the artifact id |
| GPU quota exhausted | Kueue | queued | position, holder | wait or `nice` |

## 8. Acceptance criteria

- **AC-4.1** Given a model served from `/models/churn/1.3`, when `history --receipts --follow` runs on it, then the chain reaches the content ids of the raw datasets and the config file.
- **AC-4.2** Given `train` writes `auc` through the receipt metric channel, then the receipt holds it under `metrics` and marks it as self-reported.
- **AC-4.3** Given a GPU unit idle beyond the platform's idle timeout, then `ps` shows IDLE and the unit is scaled to zero with its volume retained, and the receipt shows the idle cost.
- **AC-4.4** Given `xargs -P 16` with the default contract and one failing index, then `explain` has warned that fail-fast will cancel siblings, and `--keep-going` runs all sixteen and exits 1 listing the failed index.
- **AC-4.5** Given two config files with identical content at different paths, then `train` on them is one cache hit, not two executions.
- **AC-4.6** Given `cp /artifacts/<id> /models/churn/1.3`, then `ls -l /models/churn/1.3` shows the source artifact id and the object's metadata holds it.

## 9. Persona lenses

- **DS**: wants to never write a Job spec; refuses if one bad config kills a sweep without warning.
- **ML**: wants to take a model to production with its full lineage and a pinned digest; refuses if `cp` breaks the chain.
- **FIN**: wants idle GPUs to be visible and stopped by policy, not by an email.
- **PE**: owns the idle timeout, the registry mount and the GPU sharing policy.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L4.1 | P11, P9 | Copying an artifact to a mutable registry path drops its content id; the served model and the lineage are no longer tied. | **rule** (applied: ADR 0011 rules; command map; roadmap M5): object-store mounts record the source artifact id as object metadata on `cp`; `ls -l` shows it; `run --model=` resolves and pins the id in the unit receipt; when the registry is the artifact store itself, `ln -s` is the recommended spelling | ADR 0011 decision 9, posix-command-map.md (`cp`, `ln`), ROADMAP M5 |
| L4.2 | P7 | A command has no channel to put structured fields into its receipt; `auc=0.912` is asserted by the walkthrough without a mechanism. | **rule** (applied: ADR 0009 rules; roadmap M1): the toolbox provides `convoy-metric key=value` writing to `/sys/receipt/$pid/metrics`; receipts carry a `metrics` map flagged self-reported (see US-07 L7.2) | ADR 0009, ROADMAP M1 |
| L4.3 | P5 | An interactive GPU unit idles at full price; nothing in the design stops it. | **rule** (applied: ADR 0010 rules; roadmap M11): platform default `idle_timeout` per unit class with a GPU; idle is scale-to-zero with volumes retained (ADR 0006); `ps` shows IDLE and the receipt the idle cost | ADR 0010, ROADMAP M11 |
| L4.4 | P2 | Floor 100 % plus fail-fast turns one failed config into fifteen cancelled runs; GNU `xargs` would have run them all. | **rule** (applied: ADR 0004 rules; command map): `explain` warns when fail-fast can cancel independent siblings; `--keep-going` runs every index and exits 1 with the failed list; platforms may set it as the `xargs -P` default | ADR 0004, posix-command-map.md (`xargs`) |
| L4.5 | P6 | The content id of an object under `/s3` is not defined; an ETag is not a content hash for multipart uploads. | **rule** (applied: ADR 0001 rules; roadmap M5): the fileserver resolves every path argument to a content id at plan time, by sha256 for small objects and by object version id plus size for large ones, and the receipt says which | ADR 0001, ROADMAP M5 |
| L4.6 | P9 | `expose` depends on the networking abstraction, which is not designed. | **open**: already listed in stories.md as architecture-changing; this story is its first acceptance test | stories.md, ROADMAP M11 |
