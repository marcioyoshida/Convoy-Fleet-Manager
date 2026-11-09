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

## Amendment 2 · 2026-11-09 · the artifact store is a device per topology (user story US-08, L8.1)

**Context.** Every non-local pipe, every receipt and every log lives in the artifact store, and the decision above never placed it. A single global store moves every intermediate artifact out of the region a regulated workload is confined to, which breaks residency for the pipeline even when its inputs and outputs are in-region.

**Decision.** The artifact store is a device class under `/dev/obj/by-id/cas-<topology>` (ADR 0011 amendment 2), one device per topology the platform serves. The planner selects, for every stage, a store within the stage's residency (the intersection of the `residency=` options of the mounts the stage touches, ADR 0011 amendment 3) and refuses the plan when no such store exists, naming the stores it considered. The receipt names the store of every artifact. A session pod serving a residency-bound mount runs in that region or holds no artifact data itself; which of the two the platform chose is printed by `explain` as a default.

**Consequences.** Content ids are unchanged, so a cache hit is still a cache hit within a store; the same id in two stores is two copies, and a cross-topology stage reads from the store its input is in and writes to the store its residency permits, with the egress line priced. ROADMAP M4 gains the per-topology store and M8 the device entries.
