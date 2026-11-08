# US-NN · <title, a verb phrase the persona would say>

> **How to use this template.** One document per workload. Fill every section; where a section does not apply, write *not touched* and say why, because an aspect that is never touched by any story is an aspect nobody has tested. The point of the document is sections 5 and 6: section 5 walks every aspect of the abstraction and asks what this story expects of it, section 6 runs the fixed list of loophole probes. A probe that finds nothing is still recorded, so a reader can tell "checked, fine" from "not checked". Findings get an id `L<story>.<n>` and a proposed disposition; they are collected in [README.md](README.md). Nothing in a story document changes an ADR by itself.

| Field | Value |
|---|---|
| Primary persona | <one of the ids in README.md> |
| Secondary personas | <ids> |
| Workload class | <batch DAG · coordinated group · release · interactive · stateful service · agentic · regulated batch · fleet operation> |
| Walkthrough | [walkthroughs.md §N](../walkthroughs.md) |
| Gaps already closed there | <G-ids or none> |
| Milestones needed | <M-ids from ROADMAP.md> |
| Findings in this document | <L-ids> |

## 1. Story

**As a** <persona>, **I want** <goal in the persona's words, no Convoy vocabulary>, **so that** <business outcome>.

**Context.** Two or three sentences on the corporate situation: who is waiting on the result, what it costs when it is late or wrong, what the team does today without Convoy.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | | | |
| Approves | | | *none* if no approval step |
| Pays | | | the cost centre the receipt is charged to |
| Audits | | | who reads the receipt later and what they need from it |
| Operates the platform | | | what the platform team had to install or decide first |

## 3. Preconditions

- **Capabilities** (`caps`): <controllers, CRDs, agents the story needs>
- **Packages** (`pkg list`): <platform and user packages>
- **Mounts**: <what the session's mount table must contain, with options>
- **Annotations**: <entries the annotation database must hold, with class and fields>
- **Platform defaults** the story relies on and `explain` must print: <list>

## 4. Main path

A condensed session, then a table from each line to the objects it becomes.

```bash
$ …
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| | | | |

## 5. Abstraction coverage

One row per aspect. *Touched* is `●` when the story stresses it, `○` when it is used without stress, `–` when not used. *What to verify* is a sentence a future test can check.

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe: staged artifact, content id, cache | | | |
| A2 | Command: class, annotation, executable fields | | | |
| A3 | Session: mount table, env, cwd, attach and detach | | | |
| A4 | Backend selection: local, parallel-local, Indexed Job, JobSet, Argo | | | |
| A5 | Completion contract: floor, retries, deadline, budget, gang, exit codes | | | |
| A6 | Device tree: `/dev`, mounts and options, `self`, `all`, `/sys` | | | |
| A7 | Signals and lifecycle: TERM, KILL, STOP, CONT, HUP, groups | | | |
| A8 | Secrets as references: mount, versions, `fuser` | | | |
| A9 | Cost: band, `ps` column, receipt, metered sources | | | |
| A10 | Identity and authorization: `whoami`, `sudo --as`, owner labels, approvals | | | |
| A11 | Platform defaults and `caps`, made visible by `explain` | | | |
| A12 | Storage: volumes, snapshots, `df`, `rmvol`, `umount` | | | |
| A13 | Networking: `expose`, egress policy, DNS names | | | |
| A14 | Time and queueing: cron, deadline, Kueue, priority | | | |
| A15 | Audit, receipts, retention and `replay` | | | |
| A16 | Durability: what survives a session, node or cluster loss | | | |

## 6. Loophole probes

The same twelve probes in every story. Each is a question the design must answer for this workload. *Finding* is `ok` with the reference that answers it, or an `L` id.

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative verb, declarative owner | Does any verb mutate an object a controller or GitOps also owns, so the change is reverted or fought over? | | |
| P2 | POSIX semantics that cannot hold | Which POSIX expectation (exit code, `wait`, `rmdir`, signal, pause) does the cluster not honour, and is the difference surfaced? | | |
| P3 | Orphaned or lost state | Can any step leave data, volumes or partial artifacts that nothing references or that look complete? | | |
| P4 | Identity leak or escalation | Can a value, token or permission reach a place it should not (an artifact, a log, another tenant, a child process)? | | |
| P5 | Invisible cost | Is there spend the band, `ps` or the receipt cannot see, or that is under the workload's own control? | | |
| P6 | Lying cache | Can a cache hit return a result that is stale, non-deterministic or built from under-declared inputs? | | |
| P7 | Silent failure | Can a step succeed in Convoy's eyes and do nothing in the cluster, or fail without a receipt? | | |
| P8 | Partial fleet | What happens when a cluster, zone or dependency is unreachable: degraded, refused, or wrong? | | |
| P9 | Created elsewhere | What does the story depend on seeing that was not created through Convoy (pods, secrets, volumes, routes)? | | |
| P10 | Multi-party action | Does any step need a second person, a producer and a consumer, or a hand-off, and is the shell's single actor enough? | | |
| P11 | Data at rest | Where do artifacts, receipts, logs and queue items physically live, for how long, and under which policy? | | |
| P12 | Process-tree inheritance | Do budget, deadline, identity, residency and signals propagate to children, group members and retries? | | |

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| | | | | |

## 8. Acceptance criteria

Numbered `AC-N.n`, each a Given / When / Then that an implementation can be tested against.

- **AC-N.1** Given … when … then …

## 9. Persona lenses

One line per persona that touches the story: what they must be able to see or do, and what would make them refuse to adopt.

- **<id>**: …

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L N.1 | P? | | amend · rule · open · accepted | ADR / document |

*amend* = an ADR amendment is proposed; *rule* = a sentence in an existing document (command map, compiler, roadmap, defaults); *open* = needs design work that does not exist yet; *accepted* = a documented limitation, surfaced to the user, not fixed.
