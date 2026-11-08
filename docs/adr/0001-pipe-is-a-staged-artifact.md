# ADR 0001: A pipe is a staged, content-addressed artifact

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/pipeline-compiler.md §1, §4, §8.5; ADR 0009

## Context

A Unix pipe is a kernel buffer with backpressure between co-located processes. Between pods there is no kernel. The research notes proposed event streams, object-store FIFOs and mesh sockets at different points.

## Decision

Every stage output is an artifact whose id is `sha256(command, toolbox version, env subset, ordered input ids, partition index)`. Stages read artifacts and write artifacts. Live streams are opt-in (`|=`), lose caching and replay, and `explain` prints that.

## Consequences

Reruns skip unchanged stages. A downstream restart loses nothing. Small outputs are inlined at runtime, never decided at compile time. Object storage is never treated as a FIFO: an object is invisible until its upload completes.
