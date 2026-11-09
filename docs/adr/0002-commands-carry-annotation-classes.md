# ADR 0002: Commands carry annotation classes; text tools are never translated

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/pipeline-compiler.md §3, §8.3, §8.4; toolbox/annotations.yaml

## Context

SQL optimizers work because relational algebra is closed. A shell runs arbitrary binaries. One note proposed translating grep, awk and sort into Spark DataFrame operators, which changes results (CSV quoting, output order, Turing-complete awk).

## Decision

Each command has an entry in an annotation database with a class: stateless, aggregate (partial + merge program), pattern (stateless for a recognized subset), limit, fan-in, fan-out, typed, or unknown. The planner parallelizes only what the class allows and runs the real binary over partitions, PaSh-style. Unknown commands run on one node with the reason printed. Only typed verbs over typed data map to engine operators.

## Consequences

Results are identical to Linux by construction. Coverage grows by adding entries, not by inferring behaviour. `tool add` must ship an annotation or the tool is non-parallel. Merge programs ship in the toolbox image and are versioned with the database.

## Amendment 1 · 2026-11-09 · side effects and idempotency (user story US-01, L1.2)

**Context.** A stage marked `side_effects: true` is never a cache hit, so a rerun after a half-written `load` writes its rows twice. The class says what parallelizes; it says nothing about what a retry may repeat.

**Decision.** The annotation database gains `idempotent: true | false` beside `side_effects`. A side-effecting stage receives the content id of its input artifact as `CONVOY_IDEMPOTENCY_KEY` in its environment, so a command that can key its writes does. Automatic retry of a side-effecting stage (per-partition retries, `retry --failed`, a group restart) is refused unless the entry says `idempotent: true`; the refusal is printed by `explain` before the run and recorded in the receipt when a retry would have happened. `tool add` for a side-effecting tool must state the field, like the class.

**Consequences.** `curl`, `agent-run` and any loader without a declared key run with `--retries=0` in effect and say so. A loader that uses the key (an upsert keyed by it, a transactional producer with it as transactional id, ADR 0012 decision 2) is declared idempotent and retries like any stage. Queue writes get their key from this field and ADR 0012 refines it for follows.

## Rules · 2026-11-09 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L12.2** (US-12) on a follow the planner refuses aggregate and fan-in classes and `order: global`, and prints the two alternatives: a bounded read on a schedule, or an engine package with a window; `limit` (`head`) stays allowed and closes the follow.
