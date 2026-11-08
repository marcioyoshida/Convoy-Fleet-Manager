# Design documents

| Document | What it settles |
|---|---|
| [pipeline-compiler.md](pipeline-compiler.md) | How `\|`, `&&`, `xargs`, `for` and `tee` compile; the annotation database; artifacts and caching; the three backends; optimizations; the completion contract; complete Indexed Job and Argo examples |
| [debate-review.md](debate-review.md) | Builder review of the research takes on exit codes, signals, secrets, multi-cluster routing, autoscaling, priority, audit, cost and packages; five cross-cutting decisions; build order |
| [posix-command-map.md](posix-command-map.md) | All 155 POSIX utilities mapped to Kubernetes backing, approach, feasibility and pull, with the findings that changed the design |
| [stories.md](stories.md) | User stories the research missed, each with failure behaviour, and the abstractions still to design |
| [adr/](adr/) | Architecture decision records, numbered |

## Reading order

1. The three definitions and principles in the root `README.md`.
2. `pipeline-compiler.md` §0 (what changed from the research) and §3 (the annotation database).
3. `debate-review.md` summary matrix and the cross-cutting decisions at the end.
4. The ADRs, which are the same decisions in a form that can be reversed one at a time.
5. `posix-command-map.md` when you need to know what a specific command does on Convoy.

## Open design work

Three abstractions change the architecture and are not yet designed: session placement (ADR 0005 states the decision, not the design), identity and ownership, and networking as sockets. The mount namespace and fileserver design that supersedes the disk-operations research note is also pending; it will be written against [ADR 0011](adr/0011-device-tree-and-mount-table.md), which settles how clusters and other cloud resources appear as devices under `/dev` and are mounted by the session. See `stories.md` for the full list.
