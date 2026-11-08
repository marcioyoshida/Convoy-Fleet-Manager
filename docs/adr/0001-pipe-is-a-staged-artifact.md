# ADR 0001: A pipe is a staged, content-addressed artifact

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/pipeline-compiler.md §1, §4, §8.5; ADR 0009

## Context

A Unix pipe is a kernel buffer with backpressure between co-located processes. Between pods there is no kernel. The research notes proposed event streams, object-store FIFOs and mesh sockets at different points.

## Decision

Every stage output is an artifact whose id is `sha256(command, toolbox version, env subset, ordered input ids, partition index)`. Stages read artifacts and write artifacts. Live streams are opt-in (`|=`), lose caching and replay, and `explain` prints that.

## Consequences

Reruns skip unchanged stages. A downstream restart loses nothing. Small outputs are inlined at runtime, never decided at compile time. Object storage is never treated as a FIFO: an object is invisible until its upload completes.

## Amendment 1 · 2026-11-08 · non-deterministic commands (walkthrough 7)

**Context.** The artifact id is a function of command, toolbox, env subset, input ids and partition. An LLM call, a random sample, or a command that reads a clock produces different output from identical inputs. A cache hit on such a stage would silently replay a previous run.

**Decision.** The annotation database gains `deterministic: false`. The planner salts the id of such a stage with the pid, so the artifact is stored and the receipt records its lineage, but it is never served as a cache hit. `explain` marks the stage `no-cache`. A user may force reuse of a specific prior artifact by naming its id as an input, which is an explicit choice recorded in the receipt.

**Consequences.** Downstream deterministic stages still cache against the salted id, so a rerun of the pipeline recomputes the non-deterministic stage and everything after it, and nothing before it. See docs/walkthroughs.md, gap G7.
