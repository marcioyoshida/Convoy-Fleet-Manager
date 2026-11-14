# ADR 0017: An approval is an object on the process; it approves a pinned plan, from another person's session

**Status:** Draft · 2026-11-10 · revised 2026-11-10 with user-story findings L18.1 and L18.2, and 2026-11-13 with L31.5 (decision 5) and L31.9 (decision 7), marked inline
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

4. **At the gate the process suspends, holds its inputs and keeps its receipt.** The gated stage's object is created suspended (a Job with `suspend: true`, a rollout not yet written, a `sudo` not yet assumed), `ps` shows state `approval` with the group and the age, and `/sys/proc/<pid>/approval/<n>` becomes pending. The stage's inputs are held for the approval's lifetime (ADR 0006 amendment 2) so the approver sees exactly what will run and it cannot expire underneath. The gate belongs to the controller, not to either session: the `Approval` resource's status drives the suspended object, the stage resumes under the group's service identity (ADR 0016 decision 5) whether the requesting session is attached, detached or ended at token expiry, the receipt records the session's end and the resume, and an attached session only reports (revised 2026-11-10, L18.2). If the policy names a `page:` route, the group is paged with the object's path (`caps: alerting`). The contract's deadline keeps counting unless the policy says `deadline-from=approval`, which `explain` prints.

5. **The approver approves the pinned plan, from their own session.** `ls /sys/approvals/<group>/` is the group's inbox: every pending object that names the group or names it as fallback, with age, actor and verb. `cat` on one prints the pinned `explain` output, the plan hash and the policy. `approve <pid>[.<n>] [-m <note>]` and `deny <pid>[.<n>] -m <reason>` are platform verbs (ADR 0013 decision 9) run in the approver's session; the identity is the session's login identity. An answer may also come from a notification channel whose driver declares `act=approve` (ADR 0027 decision 8): the channel maps the account to the identity provider's user, as a code-owner review counts under L28.1, and re-authenticates the user for the answer, which is then that user's answer and records the channel in the receipt; every other channel carries only a link to the object, and a bot's token never answers (revised 2026-11-13, L31.5). A valid answer is from a user (never a service identity, so agents cannot approve agents), who is a member of the group or of its fallback, who is not the requesting actor, from a session other than the requesting one. The CEL admission policy on the approval resource refuses anything else at the API server; the shell refuses first, with the rule and the group (ADR 0013 decision 10).

6. **The plan's shape is the thing approved** (revised 2026-11-10, L18.1). The plan hash covers the shape: verb and arguments, digests, targets, route weights, contract flags and the gating policy. The input artifact ids are not in the hash; they are listed beside it in the pinned `explain` and in the receipt and held for the object's life (decision 4), so the approver sees both and approves the shape. If the shape changes between request and answer, or the object cannot stand (rule L18.3 below), it becomes `void` with the reason and a new one is created; the receipt links both. An approved shape that repeats within the policy's reuse window (the same digest released to a second region the same morning; a nightly housekeeping run whose inputs differ each night) is covered without a new answer, and the second receipt links the first; the default window is none, so every run asks. A policy may name shape fields the window ignores, such as a canary weight, and `explain` prints them (L18.7).

7. **Deny wins, quorum counts people, timeout cancels.** One `deny` ends the object as `denied` and the process exits 3 with the reason in the receipt. `approved` needs `quorum` distinct valid answers; the same person counts once. The timeout counts from the first successful delivery of the request to the approving group's route (ADR 0027 decision 7), bounded by the process's remaining deadline, and the receipt records the creation, delivery and answer times (revised 2026-11-13, L31.9). At the timeout the object is `expired`, the page route fires if named, and the process exits 3 `approval not given`. The requesting actor may `kill` the suspended process, which marks the object `cancelled`. Root never approves (L17.3); `wheel` approving a root assumption is the `wheel` group answering, as any group does.

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
- [US-18](../user-stories/US-18-approval-under-pressure.md) probes this record under the DevOps and Security personas with a release answered after the author's session ended, a policy edit under a pending gate, a denial, an agent at a gate and a cron entry gated nightly. It raised seven findings; two were applied inline the same day: L18.1 (the plan hash is a shape, with the inputs listed beside it, or no reuse window can ever match; decision 6) and L18.2 (the controller owns the gate and resumes the stage, not either session; decision 4). Four are rules, written below and in ROADMAP M2 and the command map: the void reasons enumerated with re-pricing at resume (L18.3), identical pending requests sharing one object (L18.4), answers visible to the requesting actor (L18.5) and a bounded timeout on every policy (L18.6). L18.7 accepts that a canary and its promotion are two gates.


Note · 2026-11-13: the draft [ADR 0027](0027-convoy-tells-a-group-about-its-own-objects.md) gives this record's routes their channel: `page:<route>` is the `page` urgency of a group's route in `/etc/notify.d/<group>`, `caps: alerting` is an `incident` channel, and an `approval.requested` event goes to the approving group's route when the object is created (decisions 2 to 4); answers from channels are decision 8.
## Rules · 2026-11-10 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L18.3** (US-18) the reasons an object becomes `void` are enumerated: the gating policy edited (group, fallback, quorum, timeout, window), a hold lost, a target device removed or disconnected, the approving group no longer able to answer; each is printed, re-creates the object under the current state and re-sends the page; at resume the planner re-prices the plan, the receipt records the pinned and the settled band, and `--budget` still binds.
- **L18.4** (US-18) pending objects are deduplicated per shape hash and group: a request whose shape matches a pending object joins it as a waiter, `ls` on the inbox shows the waiter count, one answer resumes every waiter, each receipt links the object, and the inputs are held once per object.
- **L18.5** (US-18) the answers on an approval object (identity, group, time, reason) are visible to the requesting actor and the owning group regardless of rule L17.6, because an answer is an act toward that process; the inbox stays visible to the approving group only.
- **L18.6** (US-18) every policy has a timeout and the platform default is bounded (24 h unless set); `explain` prints it; at expiry the holds are released, the page route fires and the process exits 3; `df` lists held bytes per pending object.
- **L28.3** (US-28) `plan changed` joins the enumerated void reasons of rule L18.3; the new request shows the diff between plans (full rule in ADR 0024).
- **L29.14** (US-29) on a computer run by an external operator, node verbs become requests on the operator's channel (full rule in [ADR 0025](0025-a-computer-is-a-device-kubernetes-is-one-driver.md)).
- **L30.8, L30.13** (US-30) a release gate reads scan freshness; a vulnerability exception is an approval pinned to a digest and a vulnerability, with an expiry (full rules in [ADR 0026](0026-a-registry-is-a-device-a-tag-is-a-name-for-a-digest.md)).
- **L31.7, L31.12, L31.15** (US-31) a message about an approval is updated as its state changes; requests go to the approving group's on-call, one further answer at a time; a request pages only with less than `page_before` left (full rules in [ADR 0027](0027-convoy-tells-a-group-about-its-own-objects.md)).
