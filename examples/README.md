# Examples

Compiled targets taken verbatim from `docs/pipeline-compiler.md` §9. They show what the planner emits; they are not meant to be applied until the toolbox helpers exist.

| File | What it is |
|---|---|
| `indexed-job-map-merge.yaml` | Map stage as an Indexed Job (1 648 partitions, 95 % floor via `successPolicy`, `backoffLimitPerIndex`) plus the merge Job. Requires Kubernetes 1.31+. |
| `argo-fan-out-merge.yaml` | The same pipeline as an Argo Workflow with `withSequence` fan-out and artifact keys shown explicitly. |

Every file parses as YAML; a check runs in CI.
