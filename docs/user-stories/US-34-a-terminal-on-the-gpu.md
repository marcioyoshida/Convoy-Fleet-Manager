# US-34 · A terminal on the GPU: notebooks and interactive allocations that hold scarce hardware while a person thinks

| Field | Value |
|---|---|
| Primary persona | DS |
| Secondary personas | ML, DE, PE, FIN, SEC |
| Workload class | interactive work on scarce compute: a notebook on a GPU node of the GB200 cluster, Jupyter inside a batch job to get more GPUs, and an interactive allocation (`salloc`, `srun --pty`) on the university Slurm cluster against staged data; the person thinks, types, leaves for lunch and closes the laptop, and the hardware stays held |
| Walkthrough | written against [ADR 0025](../adr/0025-a-computer-is-a-device-kubernetes-is-one-driver.md) decisions 2, 3, 4, 8 and 9 and its open item on interactive work, [ADR 0020](../adr/0020-a-session-is-a-journal-not-a-pod.md) decisions 4, 5, 6 and 7, [ADR 0019](../adr/0019-a-unit-has-an-address-reach-is-a-grant.md) decisions 2, 3 and 7, [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md) decisions 4, 5, 7 and 11, [ADR 0028](../adr/0028-memory-has-a-distance-and-a-domain-is-claimed-whole.md) decision 8, [ADR 0029](../adr/0029-a-computers-file-systems-are-in-its-contract-staging-is-a-planned-copy.md) decision 4, [ADR 0027](../adr/0027-convoy-tells-a-group-about-its-own-objects.md) decisions 3 and 4, [ADR 0004](../adr/0004-completion-contract-and-exit-codes.md) amendment 3, [ADR 0001](../adr/0001-pipe-is-a-staged-artifact.md) amendment 1, [ADR 0010](../adr/0010-platform-defaults-and-two-package-tiers.md) rule L4.3, accepted limitations L20.11 and L24.11, US-04's notebook unit, and the command map rows for `fg`, `bg`, `tty` and `nice`; it follows US-33 |
| Gaps already closed there | none |
| Milestones needed | M2, M3, M6, M7, M8, M11 |
| Findings in this document | L34.1 – L34.19 |

## 1. Story

**As a** data scientist, **I want** to open a terminal or a notebook on a GPU, or on a university cluster next to my staged data, as easily as I open a shell, and to have it end, warn me, or survive in ways I can predict, **so that** I am not choosing between holding an expensive GPU for three days "just in case" and losing six hours of kernel state to a drain nobody told me about, and so that what I compute interactively can be traced to the data I read and to me.

**Context.** After US-33 the platform runs ADRs 0001 to 0029. A notebook today is US-04's pattern, `run --gpu=1 --unit <name> jupyter lab` and `expose <name> --internal`: a Deployment with one GPU, scaled to zero by the unit idle timeout of L4.3 when no request arrives. There is no interactive allocation on any other kind of computer. Sessions never end remote processes (ADR 0020 decision 7), nobody may `exec` into a pod (L21.9), and `fg` and `bg` attach to pods created with a terminal. The platform team also runs JupyterHub on the GB200 cluster through GitOps, with its own spawner, profiles and idle culler, which nobody has described to Convoy.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Explores in a notebook | `noor` | user `noor`, group `research` | São Paulo; the cluster is in us-east-1 |
| Wants eight GPUs interactively | `yuki` | user `yuki`, group `llm` | |
| Debugs on the university cluster | `mei` | user `mei`, group `genomics` | the lab's allocation on `hpc-sp` |
| Helps `mei` debug | `tomas` | user `tomas`, group `genomics` | |
| Runs JupyterHub | platform team | `svc:platform` | GitOps; culler at 1 h idle |
| Operates `hpc-sp` | the university's HPC team | none | interactive partition, 8 h limit |
| Pays | FinOps | group `finops` | |

## 3. Preconditions

- **Capabilities** (`caps`): on `gb200`: `dra`, `kueue`, `mig` on the inference pool only, JupyterHub at `hub.ml.internal`; on `hpc-sp`: `slurm:23.11`, partition `interactive` with an 8-hour limit, `identity=shared`, login nodes reachable over SSH by the lab's members.
- **Packages**: `jupyter` and `code-server` in the annotation database as executables; JupyterHub installed by GitOps, not by a Convoy module.
- **Platform defaults** `explain` must print: for an interactive request, where it will run, what it holds, its price per hour, how long it may wait to start, when it ends and why (wall time, idle, detach), what is warned before that, and what survives the end.

## 4. Main path

```bash
# Monday 09:00 · noor, group research, from São Paulo
$ run --gpu=1 --unit nb-noor jupyter lab && expose nb-noor --internal
#  the pod gets a whole B200 on a node of rack nvl-a, a member of a 72-GPU domain   (L34.14)
#  the platform's JupyterHub, with its profiles and culler, is not involved           (L34.13)
#  every keystroke takes 140 ms to echo                                                (L34.19)
#  noor pastes the Jupyter URL with its token into the team chat for a colleague      (L34.5)
#  a cell prints 20 rows of /tables/research/claims (class=restricted); the notebook is
#  saved with its outputs to /home and pushed to a Git host outside the residency      (L34.6)

# Monday to Thursday · the kernel holds 61 GB of GPU memory; GPU use averages 3 %;
#  the browser tab keeps the server's websocket open, so the unit is never idle (L4.3)
#  and costs $310 for nothing                                                            (L34.2)

# Tuesday · from a notebook cell: !batch xargs -P 16 python sweep.py --cfg {} < cfgs.txt
#  the sweep runs as svc:research with no person on its receipt and no budget,
#  since the kernel holds no user token and is not a process with a contract           (L34.12)
# Tuesday 23:10 · the health controller drains noor's node for a GPU fault (ADR 0021)
#  the pod is evicted; six hours of kernel state are gone; nothing told noor             (L34.8, L34.16)

# Wednesday · noor trains in the notebook, writes /models/churn-v7, and releases it
#  the model's lineage says svc:research and no inputs; the cells ran out of order;
#  the table changed on Tuesday and the kernel's dataframe is Monday's                 (L34.7, L34.18)

# Wednesday · yuki, group llm: units are capped at one GPU, so
$ batch --gpu=8 jupyter lab --ip=0.0.0.0
#  a batch stage with no attach path, no idle rule and no end but its deadline;
#  port-forward is refused (L21.9), so yuki tunnels through a debug pod               (L34.17)

# Thursday · mei, group genomics, wants a shell next to the staged BAMs on hpc-sp
#  Convoy has no verb for it                                                            (L34.1)
#  mei SSHes to the university's login node and runs salloc under the lab's allocation;
#  Convoy's allocation balance falls by work it never saw                              (L34.10)
#  salloc waits 40 minutes in the queue with the terminal blocked                      (L34.9)
#  inside srun --pty, mei presses ^Z to "pause and free the GPU"; the job stops and the
#  allocation keeps counting                                                            (L34.4)
#  tomas is asked to look; mei shares the university password                         (L34.11)
#  mei closes the laptop at 18:00; the allocation runs, idle, until its 8-hour limit (L34.3)
#  next day, a second allocation ends at its limit in the middle of a long command,
#  with no warning                                                                       (L34.15)
```

| Step | Convoy object | Computer's object | Where the state lives |
|---|---|---|---|
| a notebook | a unit (US-04) | Deployment, Service, a GPU | the pod's memory |
| a notebook in a batch stage | a batch process | Job with a GPU | the pod's memory |
| JupyterHub's notebooks | none | the hub's spawned pods | the hub |
| an interactive allocation on Slurm | none | `salloc` job, `srun --pty` step | the scheduler |
| a kernel's state | none | process memory | the node |
| a notebook file | a file on `/home` | a volume | the volume, then a Git host |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe: staged artifact, content id, cache | ● | outputs of interactive work with no reproducible lineage (L34.18) | AC-34.18 |
| A2 | Command: class, annotation, executable fields | ● | `jupyter` under `batch` (L34.17) | AC-34.17 |
| A3 | Session: mount table, env, cwd, attach and detach | ● | an allocation bound to attach (L34.3); a second person attached (L34.11) | AC-34.3, AC-34.11 |
| A4 | Backend selection | ● | GPU shares and domain pools (L34.14) | AC-34.14 |
| A5 | Completion contract | ● | children launched from a kernel (L34.12) | AC-34.12 |
| A6 | Device tree | ○ | `/home` as a device with residency (L34.6) | AC-34.6 |
| A7 | Signals and lifecycle | ● | `^Z` in a remote terminal (L34.4); a drain under a kernel (L34.8) | AC-34.4, AC-34.8 |
| A8 | Secrets as references | ○ | a token in a URL (L34.5) | AC-34.5 |
| A9 | Cost | ● | an idle GPU (L34.2); allocations used outside Convoy (L34.10) | AC-34.2, AC-34.10 |
| A10 | Identity and authorization | ● | who acts inside a kernel (L34.5, L34.12); shared passwords (L34.11) | AC-34.5, AC-34.12 |
| A11 | Platform defaults and `caps` | ● | `interactive_idle`, `detach_grace`, `interactive_max`, `interactive_warn` | AC-34.2, AC-34.15 |
| A12 | Storage | ○ | notebooks saved with outputs (L34.6) | |
| A13 | Networking | ● | reaching a browser app through the gateway (L34.5); latency (L34.19) | AC-34.5 |
| A14 | Time and queueing | ● | a wait to start (L34.9); an end with no warning (L34.15) | AC-34.9, AC-34.15 |
| A15 | Audit, receipts, retention and `replay` | ● | inputs read by a kernel (L34.7); who attached (L34.11) | AC-34.7 |
| A16 | Durability | ○ | kernel memory (L34.16) | |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative verb, declarative owner | the platform's JupyterHub spawns and culls notebooks by its own configuration | ADR 0010 amendment 1: managed objects are intent | **L34.13** |
| P2 | POSIX semantics that cannot hold | `^Z` in a remote terminal holding a GPU | the command map maps `bg` and `fg` to attach | **L34.4** |
| P3 | Orphaned or lost state | an allocation left running after the laptop closed | ADR 0020 decision 7: a session never ends a remote process | **L34.3** |
| P4 | Identity leak or escalation | a Jupyter token in a chat message | ADR 0019 decision 7: reach through the gateway | **L34.5** |
| P5 | Invisible cost | a GPU at 3 % for three days, never idle by the unit rule | L4.3 idles units that receive no requests | **L34.2** |
| P6 | Lying cache | a kernel's dataframe from Monday's snapshot, written out on Wednesday | ADR 0029 decision 4 pins stage inputs, not a kernel's | **L34.7** |
| P7 | Silent failure | a drain evicts a kernel with six hours of state | ADR 0021 drains nodes; nobody attached is told | **L34.8** |
| P8 | Partial fleet | an interactive request queued 40 minutes with the terminal blocked | ADR 0025 decision 2's `queue` estimate, for stages | **L34.9** |
| P9 | Created elsewhere | `salloc` from the university's login node under the same allocation | ADR 0025 decision 9 settles allocations from the scheduler | **L34.10** |
| P10 | Multi-party action | a colleague looks at a terminal | L20.11: a notebook is the group's; ADR 0020 decision 6 | **L34.11** |
| P11 | Data at rest | a notebook saved with outputs of a restricted table | ADR 0007 amendment 5 scrubs artifacts | **L34.6** |
| P12 | Process-tree inheritance | a sweep launched from a notebook cell | ADR 0004 amendment 3: children inherit the contract | **L34.12** |

Seven more questions without a probe: is there an interactive allocation at all (**L34.1**, open); does a notebook need a whole GPU, and in which pool (**L34.14**, rule); is the end of an allocation warned (**L34.15**, rule); can a kernel's memory survive a drain or an end (**L34.16**, accepted); what is a notebook server under `batch` (**L34.17**, rule); what is the lineage of what a kernel writes (**L34.18**, rule); and should an allocation be placed near the person or near the data (**L34.19**, accepted).

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| idle past `interactive_idle` | the controller from telemetry (L34.2) | none; released after warnings | held resources, idle time, price | start again |
| nobody attached past `detach_grace` | the controller (L34.3) | none; released after warnings | last detach, price | start again, or `--keep` next time |
| drain or preemption | the computer's notice (L34.8) | the process ends with its reason | the notice time and who was told | start again |
| no start within `--wait` | the plan (L34.9) | 3 | the scheduler's estimate | a later time, another computer |
| end of wall time | the scheduler (L34.15) | the process ends at its limit | warnings sent and when | start again |
| notebook server under `batch` | the plan (L34.17) | 3 | the executable's class and the `run -it` spelling | `run -it` |
| output of interactive work promoted | `release` (L34.18) | 3 | the interactive lineage | rerun as a batch stage |

## 8. Acceptance criteria

- **AC-34.2** Given an interactive process, then idle is measured on each resource it holds (accelerator use from telemetry, terminal input, kernel activity), not on requests; after the platform default `interactive_idle` it is warned and then released, and `ps` and the prompt show its price per hour while it runs.
- **AC-34.3** Given an interactive process with no attached terminal or browser for `detach_grace`, then it is warned and released, unless it was started with `--keep`, which is bounded by `interactive_max`.
- **AC-34.4** Given a remote terminal in an interactive process, then job-control keys go to the remote foreground job; `^Z` stops it and `jobs` says the allocation still counts; detaching is `bg`, and only `exit` or `kill` releases the allocation.
- **AC-34.5** Given a browser or terminal application in an interactive process, then it is reached only through the platform's gateway with the login of ADR 0016; its own tokens are never handed out; other people attach only under AC-34.11.
- **AC-34.6** Given a home device used by interactive processes, then it carries a residency and is mounted only on computers in that residency; a notebook saved by a process that read a `class=restricted` mount is marked restricted, and leaving the device with it runs the scrub of ADR 0007 amendment 5 and strips outputs.
- **AC-34.7** Given an interactive process, then each input it reads is recorded with its snapshot and time in `/sys/proc/<pid>/inputs`, and anything it writes carries those as lineage.
- **AC-34.8** Given a drain, a preemption notice or an out-of-memory kill under an interactive process, then the attached people are told in their terminals and through the session channel at once, with the time left; interactive processes are never placed on preemptible capacity unless asked.
- **AC-34.9** Given an interactive request, then `explain` and the request print the scheduler's estimated start; `--wait=<duration>` bounds the wait, after which the request ends with exit 3 and the estimate; the session's shell is never blocked without a message.
- **AC-34.10** Given a computer with an allocation, then work under that allocation that Convoy did not submit is read from the scheduler and shown in `cost` as `outside Convoy`, by user where the scheduler names one.
- **AC-34.11** Given an interactive process, then a second person attaches only with the owner's grant, read-only unless the grant says `write`; `who` and the receipt list everyone attached and when; passwords are never shared.
- **AC-34.12** Given a process launched from inside an interactive process, then it is launched through the attached user's session with that user as the actor, inherits the interactive process's contract, and outlives it as an ordinary process; with nobody attached, a launch is refused with exit 3.
- **AC-34.13** Given a platform notebook hub, then it is the driver for browser interactive work on its computer; Convoy asks it to spawn, reads its profiles and culler as the computer's interactive policy, and never runs a second spawner.
- **AC-34.14** Given an interactive request, then it gets the smallest accelerator share the computer offers (a MIG slice, a time-sliced share) in a pool without domains (ADR 0028 decision 8); a whole accelerator or a domain member needs `--gpu=whole`.
- **AC-34.15** Given an interactive process with an end (wall time, `interactive_max`, idle, detach), then `expiring` is sent to the attached people's terminals and sessions at `interactive_warn` before it, and the end runs the declared state policy (save open notebooks to the home device).
- **AC-34.17** Given an executable annotated `interactive` (a notebook server, an editor server, a shell with a terminal) under `batch`, then the plan is refused with exit 3 and the `run -it` spelling.
- **AC-34.18** Given anything written by an interactive process, then its lineage carries `interactive`, the person attached and the inputs of AC-34.7; it is never a cache hit, and `release` refuses it until a batch stage reproduces it.

## 9. Persona lenses

- **DS**: wants a notebook to start where the data is, warn before it ends, and never quietly cost a week of GPU.
- **ML**: wants eight GPUs interactively for an hour without pretending to be a batch job.
- **DE**: wants a shell next to staged data on a cluster Convoy does not run, through Convoy, with the allocation counted.
- **PE**: wants one spawner, the hub's policy respected, and small work off the big domains.
- **FIN**: wants idle interactive hardware ended or charged visibly, and allocation use outside Convoy shown.
- **SEC**: wants no token URLs, no shared passwords, no restricted rows in notebook files that leave the residency, and a person on every action.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L34.1 | none | Interactive work has no shape in the design. A notebook is a unit (US-04), a shell with a terminal is a bare pod (the debate review), and on every other kind of computer there is nothing: no interactive allocation, no attach path to it, no idle rule, no end, no warning, no lineage for what it writes. ADR 0025 leaves it open. | **open**. The record asked for: an `interactive` process class (a process that holds a computer's resources while people attach to it), compiled per kind (a pod with a terminal or a browser app on Kubernetes, `salloc` and `srun --pty` on Slurm, a VM on a pool, `none` on functions and batch), started by `run -it` for a terminal and `run -it --app=<executable>` for a browser application, reached through the session's attach channel and the gateway, with a lifetime bound to attach, idle measured on held resources, warnings before every end, and lineage marked `interactive` | ADR 0025 d2, d3; ADR 0020 d5, d6; ADR 0019 d7; command map `fg`, `bg`, `tty` |
| L34.2 | P5 | A notebook held a GPU at 3 % for three days and was never idle by the unit rule, because the browser kept its connection open. | **rule**: idle for an interactive process is measured on each resource it holds (accelerator use from telemetry, terminal input, kernel activity), never on requests; after the platform default `interactive_idle` it is warned and released; `ps` and the prompt show its price per hour | ADR 0010 rules (L4.3); ADR 0023; ADR 0025 d9 |
| L34.3 | P3 | An allocation ran idle for seven hours after the laptop closed. | **amend** ADR 0020 decision 7: a session never ends a remote process, except an interactive process, whose lifetime is bound to attach; with no attached terminal or browser for `detach_grace` it is warned and released, unless started with `--keep`, bounded by `interactive_max` | ADR 0020 d7; ADR 0006 |
| L34.4 | P2 | `^Z` inside `srun --pty` stopped the job and the allocation kept counting. | **rule**: in an interactive process, job-control keys go to the remote foreground job; `^Z` stops it and `jobs` says the allocation still counts; detaching is `bg`; only `exit` or `kill` releases the allocation | command map `fg`, `bg`; ADR 0008 |
| L34.5 | P4 | A Jupyter URL with its token was pasted into chat; anyone with it could run code as the group. | **rule**: an interactive application is reached only through the platform's gateway with the login of ADR 0016; its own tokens are never handed out; others attach only under L34.11 | ADR 0019 d7; ADR 0016 d5; ADR 0020 d5 |
| L34.6 | P11 | A notebook saved with 20 rows of a restricted table was pushed to a Git host outside the residency. | **rule**: a home device carries a residency and is mounted only on computers in it; a notebook saved by a process that read a `class=restricted` mount is marked restricted, and leaving the device runs the scrub of ADR 0007 amendment 5 and strips outputs | ADR 0011 a3; ADR 0007 a5; ADR 0016 d11 |
| L34.7 | P6 | A kernel's dataframe was Monday's snapshot, and Wednesday's model was written from it with nothing saying so. | **rule**: each input an interactive process reads is recorded with its snapshot and time in `/sys/proc/<pid>/inputs`, and anything it writes carries them as lineage | ADR 0001; ADR 0029 d4 |
| L34.8 | P7 | A drain evicted a notebook with six hours of state, and nobody was told. | **rule**: a drain, a preemption notice or an out-of-memory kill under an interactive process is told at once to the attached people, in their terminals and through the session channel (ADR 0027), with the time left; interactive processes are never placed on preemptible capacity unless asked | ADR 0027 d3, d4; ADR 0021; ADR 0008 |
| L34.9 | P8 | `salloc` waited 40 minutes in the queue with the terminal blocked. | **rule**: an interactive request prints the scheduler's estimated start before it waits; `--wait=<duration>` bounds the wait, then exit 3 with the estimate; the session's shell is never blocked without a message | ADR 0025 d2, d4 |
| L34.10 | P9 | An allocation used from the university's login node lowered Convoy's balance with work it never saw. | **rule**: work under a computer's allocation that Convoy did not submit is read from the scheduler and shown in `cost` as `outside Convoy`, by user where the scheduler names one | ADR 0025 d9; ADR 0024 d8; L19.1 |
| L34.11 | P10 | A colleague was shown a terminal by sharing a password. | **rule**: a second person attaches to an interactive process only with the owner's grant, read-only unless the grant says `write`; `who` and the receipt list everyone attached and when | ADR 0016 d7; ADR 0020 d6; L21.9; L20.11 |
| L34.12 | P12 | A sweep launched from a notebook cell ran as the group's service identity, with no person on its receipt and no budget. | **rule**: a process launched from inside an interactive process goes through the attached user's session, with that user as the actor; it inherits the interactive process's contract (ADR 0004 amendment 3) and outlives it as an ordinary process; with nobody attached, the launch is refused with exit 3 | ADR 0004 a3; ADR 0016 d4, d5; ADR 0020 d4 |
| L34.13 | P1 | The platform's JupyterHub and Convoy's notebook units were two spawners with two idle policies. | **rule**: where the platform runs a notebook hub, it is the driver for browser interactive work on that computer; Convoy asks it to spawn, reads its profiles and culler as the computer's interactive policy, and never runs a second spawner | ADR 0010 a1; ADR 0025 d8; ADR 0024 d8 |
| L34.14 | none | A notebook took a whole B200 inside a 72-GPU NVLink domain. | **rule**: an interactive request gets the smallest accelerator share the computer offers (a MIG slice, a time-sliced share) in a pool without domains (ADR 0028 decision 8); a whole accelerator or a domain member needs `--gpu=whole` | ADR 0028 d8; ADR 0011 d9 |
| L34.15 | none | An allocation ended at its 8-hour limit in the middle of a command, with no warning. | **rule**: every end of an interactive process (wall time, `interactive_max`, idle, detach) sends `expiring` to the attached people's terminals and sessions at `interactive_warn` before it, and the end runs the declared state policy (open notebooks saved to the home device) | ADR 0027 d3; ADR 0025 d2 |
| L34.16 | none | A kernel's memory does not survive a drain, a preemption or the end of an allocation. | **accepted**: Convoy does not checkpoint a kernel's memory; interactive work survives as files, receipts and lineage, and notebooks are saved to the home device by the state policy | ADR 0008; ADR 0006 |
| L34.17 | none | Jupyter ran under `batch` to get eight GPUs, with no attach path, no idle rule and no end but its deadline. | **rule**: an executable annotated `interactive` (a notebook server, an editor server, a shell with a terminal) under `batch` is refused with exit 3 and the `run -it` spelling; interactive limits are set per group, not by disguising work as batch | ADR 0002; ADR 0013 |
| L34.18 | none | A model trained in a notebook with cells run out of order was released with lineage "svc:research, no inputs". | **rule**: anything written by an interactive process carries `interactive`, the person attached and its recorded inputs as lineage; it is never a cache hit, and `release` refuses it until a batch stage reproduces it | ADR 0001 a1; ADR 0002; ADR 0026 |
| L34.19 | none | Keystrokes echoed in 140 ms because the allocation was in us-east-1 and the person in São Paulo. | **accepted**: an interactive process is placed near its data and within its residency, not near the person; `explain` prints the expected round trip from the person's gateway | ADR 0016 d11; L26.4 |
