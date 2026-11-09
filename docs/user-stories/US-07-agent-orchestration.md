# US-07 · Run a triage agent over a ticket queue with a token budget and no secret in sight

| Field | Value |
|---|---|
| Primary persona | AI (AI Engineer) |
| Secondary personas | SEC, FIN, SRE |
| Workload class | agentic, fan-out, external spend |
| Walkthrough | [walkthroughs.md §7](../walkthroughs.md) |
| Gaps already closed there | G7, G8 |
| Milestones needed | M1, M2, M3, M7, M9 |
| Findings in this document | L7.1 – L7.7 |

## 1. Story

**As an** AI engineer, **I want** to run a support-triage agent over every open ticket in parallel, with the LLM and helpdesk credentials mounted as references, a dollar budget that includes tokens, and a human queue for the cases it cannot close, **so that** the agent is a process with a bill, not a cron job with a credit card.

**Context.** Token spend is invoiced by the provider and shows up nowhere in the cluster's cost tools. The last agent incident was a key copied into a log. The team is one engineer and a security reviewer.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | the AI engineer | personal identity on `/clusters/prod-east-1`, `ns=support` | the agent runs under the team service account |
| Approves | security, once, for the egress list in the package | | |
| Pays | team support | compute plus metered tokens | L7.2 |
| Audits | security, after any incident | artifacts and logs of processes that had secrets mounted | **L7.1** |
| Operates the platform | platform team | Secrets Store CSI, FQDN egress policy (L7.6), the queue package | |

## 3. Preconditions

- **Capabilities**: `secrets-csi`, `fqdn-policy` (or an egress proxy), `queue` (named queues), `indexed-job`, `metered-cost`.
- **Packages**: `agent-run` with `deterministic: false, side_effects: true, egress: [api.anthropic.com, *.zendesk.com], metered: {source: /sys/cost/$pid/metered, unit: tokens}, transient_exit_codes` (L7.4).
- **Mounts**: `/s3/tickets` read-only; `/secrets` reference-only; `/queue/human-review`.
- **Annotations**: as above.
- **Platform defaults** `explain` must print: the egress mechanism, the metered price list version, the sensitive retention class applied to this process's artifacts (L7.1), budget inheritance to children (L7.3).

## 4. Main path

```bash
$ ls /s3/tickets/open/ | xargs -P 8 -n 1 \
    batch --budget=$40 --deadline=30m --secret=anthropic-api-key --secret=zendesk-token \
    agent-run --policy=triage-v3 --max-turns=12 --escalate-to=/queue/human-review
$ top -p 7610                       # TURNS, TOKENS, SPEND
$ wait 7610                         # exit 0 degraded: 7/8 converged, 1 escalated
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `--secret=` | reference mounts | SecretProviderClass, tmpfs in the pod | the vault |
| `xargs -P 8 batch agent-run` | process 7610, 8 indexes | Indexed Job, NetworkPolicy from the package egress list | artifacts (salted ids), receipts |
| `/sys/cost/7610/metered` | metered spend | file written by the package | session cost watcher |
| `--escalate-to` | queue write | queue message with the ticket and the agent's notes | the queue |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ● | `deterministic: false` (G7): stored, never a hit | a rerun executes all eight again and the receipts link both runs |
| A2 | Command | ● | the richest non-GPU annotation | `type agent-run` prints every field |
| A3 | Session | ○ | the session never holds a value | `env` in the session shows no secret |
| A4 | Backend | ○ | Indexed Job | |
| A5 | Contract | ● | budget on compute plus tokens; `--max-turns` as a per-index convergence limit; degraded exit | AC-7.2 |
| A6 | Device tree | ○ | `/s3`, `/queue` | |
| A7 | Signals | ○ | `kill` cancels eight indexes mid-conversation | in-flight tokens are billed; receipt says so |
| A8 | Secrets | ● | references into the pod only | `cat /secrets/anthropic-api-key` in the session is refused |
| A9 | Cost | ● | metered source summed with compute (G8); self-reported (L7.2) | receipt shows compute, metered declared, metered settled when available |
| A10 | Identity | ○ | team service account; egress approved in the package | |
| A11 | Defaults | ● | egress mechanism, price list, retention class | all in `explain` |
| A12 | Storage | – | none | |
| A13 | Networking | ● | egress allow-list by hostname | a tool call to any other host is refused at the network (L7.6) |
| A14 | Time | ○ | deadline 30 m | |
| A15 | Audit | ● | artifacts of a process with secrets mounted | sensitive class, scanned on put (L7.1) |
| A16 | Durability | ○ | an index restarted after a node loss repeats its tokens | billed twice; receipt shows two attempts |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | none | | ok |
| P2 | POSIX cannot hold | provider 429 or 5xx: failure or retry? | not classified | **L7.4** |
| P3 | Orphaned state | tickets half-updated in the helpdesk on cancel | side effects outside Convoy | accepted, stated |
| P4 | Identity leak | the agent can print its mounted key to stdout, which is an artifact | ADR 0007 protects the session, not the store | **L7.1** |
| P5 | Invisible cost | tokens are self-reported by the package | budget binds on declared numbers | **L7.2** |
| P6 | Lying cache | G7 | salted id | ok |
| P7 | Silent failure | hostname egress on a CNI that only does CIDR | undefined | **L7.6** |
| P8 | Partial fleet | provider outage | every index fails the same way; floor unreachable; exit 1 | ok |
| P9 | Created elsewhere | none | | ok |
| P10 | Multi-party | the human queue | `read` from another session drains it | ok |
| P11 | Data at rest | queue items carry ticket PII | retention class on the queue | L7.7 |
| P12 | Inheritance | an agent tool call that runs `batch` | the child has its own budget; the parent's is not a cap | **L7.3** |

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| index 5 hits `--max-turns` | the agent | 0 degraded | escalated to the queue | human drains the queue |
| provider 429 storm | transient codes (L7.4) | continues | attempts per index | automatic |
| budget hit at $40 | metered watcher | 2 | tokens so far, indexes cancelled | rerun on the remainder |
| key printed to stdout | scan on `convoy-put` (L7.1) | 1 | artifact quarantined, security notified | `secret rotate` (US-09) |
| tool call to an unlisted host | NetworkPolicy | the agent sees a refused connection | logged | add to the package egress list with approval |
| child `batch` exceeds parent budget | inheritance (L7.3) | child 2 | capped by parent remaining | |

## 8. Acceptance criteria

- **AC-7.1** Given `--secret=anthropic-api-key`, then the value exists only in the pod's tmpfs, the session's `env` and audit stream never contain it, and `ls -l /secrets` shows who may mount it.
- **AC-7.2** Given eight indexes of which one hits `--max-turns` with `--escalate-to` set, then the exit is 0 with the degraded flag, the queue holds one item, and `&&` after it does not proceed without `--accept-degraded`.
- **AC-7.3** Given metered spend written by the package, then the budget binds on compute plus declared tokens, and the receipt later shows the settled figure when the provider export is ingested.
- **AC-7.4** Given a run with identical inputs on two days, then both execute and the second receipt links to the first as a prior run, never as a cache hit.
- **AC-7.5** Given an artifact containing a mounted secret's value, then `convoy-put` quarantines it with class `sensitive-quarantine`, the stage exits 1, and the receipt names the detector.
- **AC-7.6** Given the agent runs `batch` as a tool call, then the child's budget and deadline are capped by the parent's remaining amounts and `ps --tree` sums them.

## 9. Persona lenses

- **AI**: wants tokens in the same budget as compute and a transcript of what the agent did; refuses if a rerun could silently replay yesterday.
- **SEC**: wants the artifact store treated as a place a secret can land, with detection and quarantine, and egress enforced at the network.
- **FIN**: wants declared versus settled spend, because the package is grading its own homework.
- **SRE**: wants a 429 storm to look like a transient, not like eight failures that trip fail-fast.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L7.1 | P4, P11 | Secrets as references protect the session, not the artifact store: a process with a mounted secret can write its value into stdout, which becomes a retained artifact, or into stderr, which becomes a retained log. | **amend** (applied: ADR 0007 a2) ADR 0007: artifacts and logs of processes with mounted secrets carry retention class `sensitive`; `convoy-put` and the log shipper run secret-pattern detection with the vault's value hashes and quarantine on a match; detection is best effort and the receipt says so | ADR 0007, ADR 0009, ROADMAP M1 |
| L7.2 | P5 | Metered spend is written by the package being budgeted. | **rule**: metered entries are labelled `declared`; a later `settled` source (provider billing export) reconciles; FinOps sees both; a package whose declared spend is under settled by a platform threshold is flagged in `pkg list` | ROADMAP M2, ADR 0010 |
| L7.3 | P12 | Budgets and deadlines do not propagate to processes a process spawns; an agent calling `batch` bypasses its own budget. | **amend** (applied: ADR 0004 a3) ADR 0004: budget and deadline are inherited like cgroup limits: a child's are capped by the parent's remaining; `ps --tree` and the receipt sum the tree; `--budget` on a child above the cap is refused | ADR 0004, ADR 0005 |
| L7.4 | P2 | External transient failures (429, 5xx, connection reset) are neither preemption nor failure in the contract. | **rule**: the annotation `transient_exit_codes` (or the package's declaration) marks attempts that do not burn the floor but do burn deadline and budget; the receipt counts attempts per index | ADR 0004, annotations.yaml |
| L7.5 | P8 | The degraded flag is invisible inside a pipeline; see US-09 L9.5, which generalizes it. | see L9.5 | ADR 0004 |
| L7.6 | P7 | Egress by hostname needs FQDN-aware policy; a plain NetworkPolicy is CIDR-based and would silently allow or block the wrong thing. | **rule**: `egress:` with hostnames requires `caps: fqdn-policy`; without it the planner refuses or routes through the platform's egress proxy package; `explain` names the mechanism | ADR 0010, ROADMAP M7 |
| L7.7 | P11 | Queue items carry ticket contents and the agent's notes; they are data at rest with no stated retention or residency. | **rule**: named queues carry a retention class and sit on a device with a topology; `mkfifo` prints both | stories.md (named pipes), ADR 0011 |
