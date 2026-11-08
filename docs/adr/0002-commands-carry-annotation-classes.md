# ADR 0002: Commands carry annotation classes; text tools are never translated

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/pipeline-compiler.md §3, §8.3, §8.4; toolbox/annotations.yaml

## Context

SQL optimizers work because relational algebra is closed. A shell runs arbitrary binaries. One note proposed translating grep, awk and sort into Spark DataFrame operators, which changes results (CSV quoting, output order, Turing-complete awk).

## Decision

Each command has an entry in an annotation database with a class: stateless, aggregate (partial + merge program), pattern (stateless for a recognized subset), limit, fan-in, fan-out, typed, or unknown. The planner parallelizes only what the class allows and runs the real binary over partitions, PaSh-style. Unknown commands run on one node with the reason printed. Only typed verbs over typed data map to engine operators.

## Consequences

Results are identical to Linux by construction. Coverage grows by adding entries, not by inferring behaviour. `tool add` must ship an annotation or the tool is non-parallel. Merge programs ship in the toolbox image and are versioned with the database.
