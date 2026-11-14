# ADR 0004: A process converges or it does not; exit codes come from a completion contract

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/pipeline-compiler.md §7; docs/debate-review.md §6

## Context

Kubernetes objects do not exit. Distributed work ends partly done. No research note defined what happens when 3 of 1000 partitions fail or when a budget runs out.

## Decision

Every plan carries a contract: best case, floor (`--min-success`), retries per partition, deadline, budget. Exit codes: 0 best case; 0 with a degraded flag when only the floor was met; 1 floor unreachable, with fail-fast cancellation of remaining work; 2 deadline or budget exceeded after the declared action ran; 3 policy denied. `pipefail` is on by default. `&&` requires code 0 without the degraded flag unless `--accept-degraded`.

## Consequences

Maps to Job `successPolicy`, `backoffLimitPerIndex`, `activeDeadlineSeconds` (Kubernetes 1.31+). Fail-fast is done by the session; Kubernetes does not do it. Every emitted Job ignores `DisruptionTarget` in its `podFailurePolicy` so preemption never burns the floor.

Amendment note 2026-11-09: the complete table of contract flags, shared by `run`, `batch`, `explain` and `cron`, and the three subjects of `wait`, are drafted in ADR 0013 decisions 4 and 5. The degraded channel (L9.5) and contract inheritance to child processes (L7.3) are decided in amendments 2 and 3 below.

## Amendment 1 · 2026-11-08 · gang contract (walkthrough 5)

**Context.** The contract above assumes partitions are independent: a floor below 100 % is meaningful and a failed index can be retried alone. Distributed training (torchrun, MPI), and any workload whose members share a collective, are all-or-nothing: one dead rank stalls every other rank, and recovery is a restart of the whole group from the last checkpoint.

**Decision.** `--gang` is a contract flag. It sets the floor to 100 % and refuses `--min-success`; it replaces per-index retry with restart-the-group-from-checkpoint, counted against `--retries` as one; it requires all-or-nothing queue admission (Kueue). Preemption of one member still does not count against retries (`DisruptionTarget` ignored), but it triggers a group restart rather than a single-index restart. Deadline and budget are unchanged.

**Consequences.** Maps to a JobSet with `failurePolicy` restart-all and `successPolicy` all, or to an Indexed Job with `podFailurePolicy` action `FailJob` plus a session-driven relaunch from the checkpoint path. Exit codes are unchanged; a gang that exhausts `--retries` exits 1 with the last failing rank in the receipt. See docs/walkthroughs.md, gap G4.

## Amendment 2 · 2026-11-09 · the degraded channel (user story US-09, L9.5)

**Context.** The decision gives exit 0 with a degraded flag when only the floor was met, and `&&` honours the flag. Nothing else does: `$?` is 0, so `fuser … | wc -l` on a partial fleet listing, or any script, reads a partial result as a complete one. The flag has no POSIX channel.

**Decision.** Degraded status is written to `/sys/proc/<pid>/degraded` (the partitions or members that failed, and why) and exported to the next stage of a pipeline and to the shell as `CONVOY_DEGRADED=<pid>`. A session option `set -o nodegraded` turns a degraded 0 into exit 1; it is on by default in non-interactive `batch` and `cron` and off at an interactive prompt. Destructive verbs (`secret retire`, `rm`, `rmvol`, `seek`, `kill` with a list argument) refuse list input whose producer exited degraded, and print the producer's pid and the `nodegraded` spelling. `wait` returns the flag the same way.

**Consequences.** A script written without knowing about the flag fails closed under `batch` instead of acting on a partial list. Interactive users keep POSIX `$?` and see the flag in the prompt and in `ps`. ADR 0013 decision 5 carries the spelling.

## Amendment 3 · 2026-11-09 · the contract is inherited (user story US-07, L7.3)

**Context.** Budget and deadline bind the process that declared them and no other. An agent running under `--budget=50` that calls `batch` spawns a child with no budget and spends past its parent's. The process tree does not inherit the contract.

**Decision.** Budget and deadline are inherited like cgroup limits. A child process's budget and deadline are capped by the parent's remaining budget and remaining time at the moment the child is created; a child that asks for more is refused with exit 3 and the parent's remaining figures. `ps --tree` and the parent's receipt sum the tree, so the parent's budget binds on its own spend plus its descendants'. Retries and group restarts are children for this purpose. Identity and residency inherit the same way (ADR 0011 amendment 3).

**Consequences.** Metered spend declared by a child (ADR 0004 budget over compute plus declared metered spend) counts against the parent at the moment it is declared. A parent that is cancelled on budget cancels its tree with the same fail-fast rule as partitions. `explain` prints the inherited ceiling for any line run from inside a process.

## Rules · 2026-11-09 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L1.3** (US-01) the planner computes the effective floor per stage from downstream classes and refuses a `--min-success` that a fan-in cannot honour (exit 3 at plan time).
- **L2.6** (US-02) annotation `checkpoint: none` makes `--on-budget=pause` and `--on-deadline=pause` a plan-time refusal; the default for such executables is `cancel` and `explain` says so.
- **L4.4** (US-04) `explain` warns when fail-fast can cancel independent siblings; `--keep-going` runs every index and exits 1 with the failed list; platforms may set it as the `xargs -P` default.
- **L5.1** (US-05) `--gang` requires `--checkpoint` on a zone-independent device (object store or RWX); the planner refuses a per-replica PVC path for checkpoints; `explain` prints the rule.
- **L5.5** (US-05) the deadline counts from submission by default, queue and pause time included; `--deadline-from=start` opts out; `explain` prints the choice and the queue estimate.
- **L7.4** (US-07) the annotation `transient_exit_codes` (or the package's declaration) marks attempts that do not burn the floor but do burn deadline and budget; the receipt counts attempts per index.
- **L8.4** (US-08) wall-clock deadlines are read in the session's `TZ`; receipts store UTC and the zone; `crontab` sets `timeZone`.
- **L8.7** (US-08) `caps: alerting` is required for `page:` actions; absence is a plan-time refusal.
- **L23.11** (US-23) a restart caused by a node health action (ADR 0021) or by the platform's node repairer counts as an infrastructure retry, separate from `--retries`, within the platform's `infra_retries` bound; `explain` prints both bounds and the receipt lists the faults that spent each infrastructure retry.
- **L26.10** (US-26) a telemetry read whose result spans stores names the stores it could not read and is degraded (full rule in ADR 0023).
- **L29.4, L29.10, L29.12, L29.13** (US-29) a scheduler's requeue is a new attempt, never a retry; the remaining deadline compiles into an external scheduler's time limit; throttled and dropped function invocations count as failures; a process on an unreachable computer is `unknown`, never `failed` (full rules in [ADR 0025](0025-a-computer-is-a-device-kubernetes-is-one-driver.md)).
- **L31.3, L31.14** (US-31) a notify verb exits 0 on the channel's acceptance, not on delivery; `--notify` on a process applies to its tree and children roll up into the root's event (full rules in [ADR 0027](0027-convoy-tells-a-group-about-its-own-objects.md)).
