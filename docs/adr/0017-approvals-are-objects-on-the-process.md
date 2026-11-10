# ADR 0017: An approval is an object on the process; it approves a pinned plan, from another person's session

**Status:** Draft · 2026-11-10
**Supersedes / relates to:** ADR 0004 (completion contract; `/sys/proc/<pid>/`); ADR 0005 (session is server-side; rule L6.7, prompts in non-interactive sessions); ADR 0006 (detach is not destroy; amendment 2, a hold extends to inputs); ADR 0008 (amendment 4, fencing while powered on); ADR 0010 (platform defaults; amendment 1, intent writes); ADR 0013 (`approve`, `release`, `expose`, `secret retire`, `wait`, uniform refusals); ADR 0016 (decisions 5, 6 and 7; rule L17.3); user-story findings L3.1, L6.2, L9.8, L13.1, L17.3

## Context

Seven places in the design already require "a second approval" and none says what one is:

| Where | What is gated | Source |
|---|---|---|
| `release --to prod-*` | a production rollout | US-03, L3.1 |
| `rm -r` on a namespace with volumes | data loss | ADR 0006 rule L6.2 |
| `secret retire --force` | retiring a version while a reference may still be live | ADR 0007 rule L9.8 |
| `rm` on a topic above a byte threshold | data loss | ADR 0012 rule L13.1 |
| `expose --public` | a public address | ADR 0013 decision 7 |
| `sudo` to `svc:root` | root for one command or session | ADR 0016 decision 6 |
| `fenced` while a node is powered on; `kill --force --node` past a PDB | two writers on a volume | ADR 0008 amendment 4, US-16 |

US-03 found the structural fault (L3.1): the walkthrough put the approval in a `read -p`, and the only stdin in a session is the author's, so nothing the author types can be a second person's consent. ADR 0016 then settled who approves (a member of a policy-named group other than the author, with a fallback group, never root, never a service identity) and left the object open.

Facts that constrain the design:

- A Kubernetes Job can be created with `suspend: true` and resumed by clearing the field; the pods do not exist until then. A custom resource can hold the approval's fields, and a CEL validating admission policy can refuse a status write whose requester is the author or is outside the group, so the two-person rule is enforced by the API server and not only by the shell.
- Sessions detach and reattach (ADR 0006). An approver may act hours after the request from their own session, and the author's session may be gone.
- Receipts are the audit trail (ADR 0009). An approval that is not in the receipt chain is reconstructed by hand after an incident, which is what US-03 was written to end.
- What the approver is shown matters: approving the sentence "release 1.20 to prod" approves nothing if the digest, the targets or the inputs change between the request and the rollout. `explain` already produces the complete plan with artifact ids, targets and cost band (ADR 0013 decision 3), and a pipe is a content-addressed artifact (ADR 0001).
- Prompts in non-interactive sessions are refusals with the required flag (ADR 0005 rule L6.7). `--yes` answers a prompt; it must never answer a policy.
- Agents are sessions under a service identity (ADR 0016 decision 5). Two agents must not be able to approve each other.

## Decision

1. **An approval is an object on the process, created only by a gate.** When a plan reaches a stage a policy gates, the session creates `/sys/proc/<pid>/approval/<n>` with: the pid and stage, the verb and its arguments, the policy name, the approving group and fallback, the quorum, the requesting actor and session, the request time, the hash of the plan at the gate and the artifact id of its pinned `explain` output, the expiry, the state (`pending`, `approved`, `denied`, `expired`, `void`, `cancelled`), and the answers (identity, session, time, verdict, reason). The object belongs to the process's group (ADR 0016 decision 3) and is readable by the approving group. Nothing else creates one: not `read`, not a flag the author types, not a model (ADR 0014 decision 5).

2. **Gates come from policy, and an author may only add gates, never remove them.** A policy is a platform default (ADR 0010) bound to a verb and a predicate: `release` to `prod-*`, `rm` with retained bytes above a threshold, `secret retire --force`, `sudo` to `svc:root`, `fenced` while powered on, `expose --public`, `kill --force --node`, `chown -R` across groups. It names the approving group, an optional fallback group (L17.3), a quorum (default 1), a timeout (default the process's remaining deadline), what happens at timeout (`cancel` or `page:<route>` then cancel) and a reuse window (default none). `--approve-by=<group>` on any creating verb adds a gate for that run with the same shape, since stricter is always allowed (as `--strict`). `--yes`, `--force` and `--with-data` answer prompts and pre-flights; none of them answers a gate.

3. **`explain` prints every gate before anything runs, and refuses when a gate cannot be answered.** The ten sections of `explain` (ADR 0013 decision 3) gain a line per gated stage: the policy, the group, the quorum, the timeout and `suspends here`. When the approving group cannot supply a member other than the author, and no fallback can either, `explain` and the verb refuse at plan time with the group and its size (ADR 0016 decision 7). An author who is not a member of the group is told who is, by group name, never by listing members.

4. **At the gate the process suspends, holds its inputs and keeps its receipt.** The gated stage's object is created suspended (a Job with `suspend: true`, a rollout not yet written, a `sudo` not yet assumed), `ps` shows state `approval` with the group and the age, and `/sys/proc/<pid>/approval/<n>` becomes pending. The stage's inputs are held for the approval's lifetime (ADR 0006 amendment 2) so the approver sees exactly what will run and it cannot expire underneath. The session may detach; the approval does not need it. If the policy names a `page:` route, the group is paged with the object's path (`caps: alerting`). The contract's deadline keeps counting unless the policy says `deadline-from=approval`, which `explain` prints.

5. **The approver approves the pinned plan, from their own session.** `ls /sys/approvals/<group>/` is the group's inbox: every pending object that names the group or names it as fallback, with age, actor and verb. `cat` on one prints the pinned `explain` output, the plan hash and the policy. `approve <pid>[.<n>] [-m <note>]` and `deny <pid>[.<n>] -m <reason>` are platform verbs (ADR 0013 decision 9) run in the approver's session; the identity is the session's login identity. A valid answer is from a user (never a service identity, so agents cannot approve agents), who is a member of the group or of its fallback, who is not the requesting actor, from a session other than the requesting one. The CEL admission policy on the approval resource refuses anything else at the API server; the shell refuses first, with the rule and the group (ADR 0013 decision 10).

6. **The plan hash is the thing approved.** If the plan changes between request and answer (an input artifact id, a digest, a target, a contract flag), the object becomes `void` with the reason and a new one is created; the receipt links both. An approved plan whose hash repeats within the policy's reuse window (the same digest released to a second region the same morning) is covered without a new answer, and the second receipt links the first; the default window is none, so every run asks.

7. **Deny wins, quorum counts people, timeout cancels.** One `deny` ends the object as `denied` and the process exits 3 with the reason in the receipt. `approved` needs `quorum` distinct valid answers; the same person counts once. At the timeout the object is `expired`, the page route fires if named, and the process exits 3 `approval not given`. The requesting actor may `kill` the suspended process, which marks the object `cancelled`. Root never approves (L17.3); `wheel` approving a root assumption is the `wheel` group answering, as any group does.

8. **The receipt carries the approval, and so does `wait`.** The process receipt records every approval object in full: policy, group, fallback used and the primary group's size at the time, answers with identities, sessions and times, the plan hash, elapsed time at the gate, the pinned `explain` artifact id. `wait <pid>` waits through the gate and returns the contract code; `wait --approval <pid>` returns when the object leaves `pending`, with 0 approved, 3 otherwise, for scripts that want to branch (ADR 0013 decision 5 gains this subject). `explain <pid>` prints the approval state and age. `status <pid>` lists the objects.

9. **One object, every gate.** `release`, `rm` with data, `secret retire --force`, `expose --public`, `sudo` to root, `fenced`, `kill --force --node` and `chown -R` across groups use this object and nothing else; a verb that needs a second person and does not go through decision 1 is a defect in its ADR 0013 entry. The Kubernetes shape is the same everywhere: a suspended object plus an `Approval` custom resource in the session's namespace, which GitOps, `kubectl` and the audit log can all see.

10. **Non-interactive is the same as interactive.** A batch, a cron run or an agent reaches the gate, suspends and waits, exactly as a terminal does; nothing prompts (ADR 0005 rule L6.7). A cron entry gated every run is a smell `explain` names: the fix is a policy reuse window or a `--approve-by` on the cron entry's first run with a window, both visible in `cron list`.

## Declined

| Proposal | Why not | What replaces it |
|---|---|---|
| `read -p` as the approval step | the only stdin is the author's (L3.1) | the object; `read` stays the author's own input |
| approving from a chat button or an e-mail link | the identity is the bot's or the link holder's, not a session's login; no receipt in the chain | the page route carries the path; the answer is `approve` in a session |
| an approval of the command text | the plan can change under the text | the plan hash and the pinned `explain` |
| standing pre-approval ("the payments team may release without asking") | that is a policy with no gate, which the platform can write; an approval without a plan approves nothing | policies and reuse windows |
| `--yes` or `--force` satisfying a gate | a prompt answer by the author | only another person's `approve` |
| root as the approver of last resort | one person, no second | the fallback group (L17.3); a policy change by the platform |
| a service identity answering | agents approving agents | users only (decision 5) |
| a separate approvals service with its own store and identity | a second audit trail to reconcile with receipts | an object on the process, in the session's state, in the receipt |
| approving by replying in the author's session (`approve` typed by a colleague at the author's terminal) | the session's identity is the author's | the answer must come from the approver's own session |

## Consequences

- L3.1 closes: the object, its path, its verbs, its admission rule and its place in the receipt are decided; the "approver" half was already ADR 0016 decision 7. L6.2, L9.8, L13.1 and the `expose --public` and `sudo` gates now point at one definition.
- The approver reads a plan, not a sentence, which is the same output the author read. Nothing the author can type changes what the approver approved.
- Cost: an `Approval` custom resource and its CEL admission policy in M2, the suspended-object path per gated verb, the inbox view under `/sys/approvals/`, `deny` and `wait --approval` in ADR 0013, the pinned `explain` artifact per gate (small; retained with the receipt), the page route through `caps: alerting`, and a held input set for the life of every pending approval, which `df` reports.
- A gate on a long contract keeps the deadline counting by default, so an approval that takes a day can cancel a job; the policy's `deadline-from=approval` is the opt-out and `explain` prints which applies.
- The residency of the pinned `explain` artifact is the session's; an approver in another residency must be able to read it, which ADR 0011 amendment 3 and misroute S13 govern. A policy that names a group outside the residency is refused at plan time.
- The chart's session box gains `approvals (ADR 0017)`. ROADMAP M2 gains the resource and the admission policy; M7 keeps the groups.
- Open, to be settled against this ADR: the exact `--json` schema of the object; whether a page route may carry the pinned plan or only the path; quorum above one for `wheel`; whether a denied plan may be resubmitted unchanged without a cooling period.
- A user story should probe this record under the DevOps and Security personas with a release approved after the author's session ended, a plan that changes under a pending approval, a denial, an agent that tries to approve, and a cron entry that hits a gate every night.
