# US-18 · Get a release approved after I have gone home, be told why it was denied, and never let an agent approve an agent

| Field | Value |
|---|---|
| Primary persona | DO |
| Secondary personas | SEC, AI, SRE, FIN, PE |
| Workload class | approvals: a gated release across sessions, a policy edit under a pending gate, a denial, an agent at a gate, a cron entry gated nightly, a timeout |
| Walkthrough | written against [ADR 0017](../adr/0017-approvals-are-objects-on-the-process.md) with [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md) (decisions 5 and 7, rules L17.3 and L17.6), [ADR 0013](../adr/0013-native-verbs-have-one-shape.md) (`release`, `approve`, `deny`, `wait`), [ADR 0006](../adr/0006-detach-is-not-destroy.md) amendment 2 and [ADR 0004](../adr/0004-completion-contract-and-exit-codes.md); it reuses the release of US-03, the triage agent of US-07 and the topic housekeeping of US-13 |
| Gaps already closed there | L3.1 (the object), L17.3 (the fallback group) |
| Milestones needed | M2, M4, M7, M11 |
| Findings in this document | L18.1 – L18.7 |

## 1. Story

**As a** DevOps engineer, **I want** to submit a production release at the end of my day, have the approver read exactly what will run and answer from wherever they are, see the reason when they say no, and know that nothing but a person in the right group can say yes, **so that** approvals stop being a chat bot and a screenshot and become a line in the receipt that security can read without asking anyone.

**Context.** Payments releases need a second person (US-03). Chris submits `ledger-api` v1.21 to both production regions with a canary at 10:06 and shuts the laptop at 10:10; their token expires at 12:06. Ben approves at 13:20 from a train. The same week the platform team tightens the release policy to a quorum of two while a release is pending, Lee denies a namespace deletion, the triage agent of US-07 hits the topic-deletion gate forty times in one night and then tries to approve itself, and the nightly housekeeping cron entry of US-13 has been waking an approver every night for a month.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | `chris`, DevOps | user `chris`, groups `payments-de` | requests the gates; never answers one |
| Approves | `ben` and `lee` | users in `payments-release`; `sec-release` is the policy's fallback | from their own sessions, hours later |
| Denies | `lee` | `payments-release` | with a reason the author must see |
| Requests as an agent | the triage agent | session logged in as `svc:triage` (ADR 0016 decision 5) | may request, may never answer |
| Pays | cost centre `payments-de` | held inputs while a gate is pending | L18.6 |
| Audits | security | the receipt chain: policy, plan hash, answers, elapsed | |
| Operates the platform | the platform team | policies, timeouts, reuse windows, the page route | edits a policy under a pending gate |

## 3. Preconditions

- **Capabilities**: `alerting` (the page route), `gateway` (canary routes, US-03), `job-suspend` (the suspended Job and the `Approval` resource with its admission policy, M2).
- **Packages**: none beyond US-03.
- **Mounts**: `/clusters/staging/payments`, `/clusters/prod-east-1/payments`, `/clusters/prod-eu/payments` from the group profile; `/queues/acme` for the agent (US-13).
- **Annotations**: `release: verb, writes intent`; policy `prod-release: release --to prod-*, group payments-release, fallback sec-release, quorum 1, timeout 24 h, page: #payments-oncall`; policy `data-loss: rm with retained bytes > 1 GiB, group payments-release, quorum 1, timeout 4 h`.
- **Platform defaults** `explain` must print: the policy per gate with group, fallback, quorum, timeout, page route and reuse window; `deadline-from=approval` or not; the held-input retention while pending; the dedup rule for repeated requests.

## 4. Main path

```bash
# Monday 10:06, chris
$ release ledger-api v1.21 --to /clusters/prod-east-1/payments --to /clusters/prod-eu/payments --canary=10
explain: stage 1 staging rollout … stage 2 gate prod-release: group payments-release (2 members, 2 can answer), fallback sec-release,
         quorum 1, timeout 24 h, page #payments-oncall, suspends here; deadline keeps counting (deadline-from=start)
pid 8102   stage 1 done 10:09   stage 2 approval pending   object 8102.1   plan 9e4f…   explain pinned a71c…   inputs held 1.2 GiB
$ ps
8102  release  payments-de  chris  svc:payments-de  approval 1 min (payments-release)
# 10:10 chris closes the laptop; the session detaches, then ends at token expiry 12:06

# Monday 13:18, ben, on a train
[ben] $ ls /sys/approvals/payments-release/
8102.1  release ledger-api v1.21 to prod-east-1, prod-eu, canary 10   requested by chris   age 3 h 12 m   plan 9e4f…
[ben] $ cat /sys/approvals/payments-release/8102.1
policy prod-release   quorum 1   expires Tue 10:06   requesting session ended 12:06 (token expiry)
plan 9e4f…  digest sha256:4c1d…  targets prod-east-1, prod-eu  route weight 10  contract --deadline=4h from start
inputs 3 ids held   cost band $0.40 to $0.70   explain a71c… (full text follows)
[ben] $ approve 8102.1 -m "digest matches CI run 4471"
approved 13:20 by ben (payments-release) from session s-71b2
8102 resumes under svc:payments-de; the requesting session is gone, the controller resumed it                       (L18.2)
stage 2: canary 10% in prod-east-1 13:21, prod-eu 13:22   exit 0
receipt 8102: gate 8102.1 requested 10:06 by chris, answered 13:20 by ben, elapsed 3 h 14 m, plan 9e4f…, session ended 12:06

# Tuesday 09:30, chris promotes the canary; a second plan, a second gate (accepted, L18.7)
$ release ledger-api v1.21 --to /clusters/prod-east-1/payments --to /clusters/prod-eu/payments
explain: gate prod-release … reuse window none: a new answer is required; plan 2b77… differs from 9e4f… in route weight
pid 8130   approval pending   object 8130.1
# 09:40 the platform team edits prod-release to quorum 2
8130.1 void: policy prod-release changed (quorum 1 to 2) at 09:40   new object 8130.2, page re-sent                   (L18.3)
[ben] $ approve 8130.2        # 1 of 2
[lee] $ approve 8130.2        # 2 of 2, approved 10:02   8130 resumes   exit 0

# Tuesday 11:00, chris cleans up the old staging namespace
$ rm -r /clusters/prod-eu/payments-staging --with-data
explain: gate data-loss: 3 volumes, 41 GiB retained, group payments-release, quorum 1, timeout 4 h, suspends here
pid 8141   approval pending   object 8141.1   inputs held: none, 3 volumes snapshotted first (ADR 0006)
[lee] $ deny 8141.1 -m "pvc-audit-2026-10 holds the October audit copy; move it first"
denied 11:12 by lee (payments-release)
8141 exit 3: denied by lee (payments-release): pvc-audit-2026-10 holds the October audit copy; move it first           (L18.5)
$ status 8141
gate 8141.1 denied 11:12 by lee, reason above, elapsed 12 min; the snapshots are retained 7 d

# Tuesday night, the triage agent (svc:triage) hits the data-loss gate on every ticket it closes
[svc:triage] $ rm /queues/acme/acme.tickets-done --with-data        # 40 times between 00:00 and 06:00
explain: gate data-loss … an identical pending request exists (8150.1, shape f0a3…): joining it, 1 of 40 waiting       (L18.4)
[svc:triage] $ approve 8150.1
refused (exit 3): service identities do not answer approvals (ADR 0017 decision 5); the inbox is /sys/approvals/payments-release/
[ben] $ ls /sys/approvals/payments-release/
8150.1  rm acme.tickets-done (38 GiB)   requested by svc:triage (actor: triage agent, cron ticket-sweep)   age 6 h   40 pids waiting
[ben] $ approve 8150.1 -m "retention policy says 7 d; fine"
approved 07:05   40 processes resume, one answer, 40 receipts link 8150.1

# Wednesday, the nightly housekeeping entry that has woken an approver every night
$ cron list payments-de
topic-housekeeping   svc:payments-de   nightly 02:00   gated every run by data-loss (30 of 30 runs waited, median 5 h 40 m)   (L18.1)
$ explain cron topic-housekeeping
gate data-loss on `rm --with-data` of partitions older than 7 d; the plan's shape f0a3… repeats nightly, only the inputs differ;
a reuse window on the policy would cover it; today the window is none, so every run asks
# the platform sets data-loss: reuse 7 d on shape; Thursday's run is covered by Wednesday's answer and its receipt links it

# the one that nobody answered
$ ls /sys/approvals/sec-release/
8102.1 (fallback) … answered   8199.1  expose ledger-admin --public   requested by chris   age 23 h 58 m   held 0 B
# 24 h: 8199.1 expired, #payments-oncall paged, 8199 exit 3 approval not given, nothing held                           (L18.6)
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `release` reaches the gate | `/sys/proc/8102/approval/1` pending; inputs held | `Approval` resource in the session namespace; the rollout Job created with `suspend: true` | session state, artifact store (the pinned `explain`) |
| session detaches, then ends | nothing on the object | the session pod ends at token expiry (ADR 0016 d9) | the controller owns the suspended object |
| `ls`, `cat` on the inbox | the group's pending objects; the pinned plan | reads of `Approval` resources the group may read | `/sys/approvals/<group>/` view |
| `approve`, `deny` | an answer with identity, session, time, reason | a status write on the `Approval`, admitted by the CEL policy | receipt |
| resume | the stage runs as `svc:payments-de` | `suspend: false`; the intent commit; the Gateway weight | receipts |
| policy edit under a pending gate | the object `void`, a new one `pending` | the policy ConfigMap changed; a new `Approval` | receipt links both |
| repeated identical requests | one pending object, many waiters | one `Approval`, forty suspended Jobs | the object's waiter list |
| timeout | `expired`, holds released, the page route fired | the Job deleted; the alert | receipt |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ○ | inputs held while pending; released at expiry (L18.6) | `df` shows held bytes per pending object |
| A2 | Command | – | | |
| A3 | Session | ● | the requesting session ends before the answer; the approver answers from their own (L18.2) | AC-18.1 |
| A4 | Backend | – | | |
| A5 | Contract | ● | the deadline keeps counting at the gate unless the policy says otherwise; timeout is a bounded default (L18.6) | `explain` prints both |
| A6 | Device tree | ○ | the inbox is a view under `/sys`; a target device removed voids the object (L18.3) | |
| A7 | Signals | ○ | `kill` by the actor cancels the request | the object reads `cancelled` |
| A8 | Secrets | – | | |
| A9 | Cost | ○ | held bytes per pending gate; 40 waiters hold once (L18.4) | `df` |
| A10 | Identity | ● | users only, never the actor, never a service identity, never root; fallback group; answers visible to the actor (L18.5) | AC-18.3, AC-18.4 |
| A11 | Defaults | ● | policy fields, timeout, reuse window, dedup, page route, `deadline-from` | all in `explain` |
| A12 | Storage | ○ | snapshots before a gated `rm -r` (ADR 0006) | |
| A13 | Networking | ○ | the canary weight is part of the plan's shape (L18.7) | |
| A14 | Time | ● | age in the inbox, elapsed in the receipt, timeout, reuse window (L18.1, L18.6) | AC-18.6 |
| A15 | Audit | ● | the receipt carries policy, plan hash, answers, sessions, elapsed, void reasons | AC-18.7 |
| A16 | Durability | ○ | the gate survives the requesting session; forty waiters survive one answer | |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | the platform edits a policy while a gate is pending; which policy applies? | the object is `void` with the reason and re-created under the new policy | **L18.3** |
| P2 | POSIX cannot hold | `wait` on a suspended pid blocks for hours; a script needs to branch | `wait --approval` returns when the object leaves `pending` (ADR 0017 d8) | ok |
| P3 | Orphaned state | the requesting session ended; who resumes the stage when the answer lands? | not the sessions: the controller, from the resource's status | **L18.2** |
| P4 | Identity leak | the author is outside the approving group; L17.6 hides actors across groups, so can the author see who denied them? | answers on an approval are visible to the requesting actor | **L18.5** |
| P5 | Invisible cost | inputs held for a pending gate that nobody answers; forty agents holding forty copies | a bounded timeout releases holds; identical requests join one object | **L18.4**, **L18.6** |
| P6 | Lying cache | the pinned `explain` priced the plan at 10:06; it runs at 13:20 | the shape is what was approved; the planner re-prices at resume and the receipt records both; `--budget` still binds | **L18.3** (part) |
| P7 | Silent failure | a denial while the author is away | exit 3 with the reason on stderr and in `status`, as every refusal (ADR 0013 d10) | ok |
| P8 | Partial fleet | one target region disconnected while pending | the hold on that device fails; the object is `void` naming the device; `release` is not run half-way | **L18.3** (part) |
| P9 | Created elsewhere | an `Approval` resource written by `kubectl` | the CEL policy admits only answers from a session identity that satisfies decision 5; a hand-made resource gates nothing | ok |
| P10 | Multi-party | an agent requests; an agent answers; the same person answers twice under quorum 2 | users only; one person counts once | ok, AC-18.4 |
| P11 | Data at rest | the pinned `explain` artifact contains inputs' ids and cost; who can read it | the owning group and the approving group; retention with the receipt | ok |
| P12 | Inheritance | a cron entry whose plan hash changes every night because the inputs do | the plan hash must be a shape, with inputs listed beside it, or no reuse window can ever match | **L18.1** |

A thirteenth question for this story: the canary and its promotion are two plans and two gates. Is that a defect? No: the second changes the production route weight to 100, which is the decision the policy exists for. **L18.7**, accepted.

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| group cannot answer (one member, the author) | `explain`, plan time (ADR 0016 d7) | 3 | group and size; the fallback | `--approver=<fallback>` (L17.3) |
| policy edited under a pending gate | the controller | pending again | `void: policy changed`, the new object | none needed; the page is re-sent |
| target device disconnected while pending | the hold | pending then void | `void: hold lost on <device>` | re-run when the device is back |
| denial | `deny` | 3 | the denier, the reason, elapsed | fix and resubmit; a denied plan unchanged is a policy question (open) |
| timeout | the controller | 3 | `expired`, page sent, holds released | resubmit |
| service identity answers | `approve` pre-flight and the CEL policy | 3 | the rule | a user answers |
| the actor answers, or the same person twice | pre-flight and the CEL policy | 3 | the rule | another member |
| the requesting session ended | nothing fails | n/a | `session ended <time>`, resumed by the controller | none |
| forty identical requests | dedup | pending | `joined 8150.1`, waiters | one answer |

## 8. Acceptance criteria

- **AC-18.1** Given a pending gate whose requesting session has ended, when a valid answer lands, then the stage resumes under the group's service identity, and the receipt records the session's end and that the controller resumed it.
- **AC-18.2** Given `cat` on an inbox object, then it prints the policy, quorum, expiry, the pinned `explain` and the plan hash, and `approve` records the hash it approved.
- **AC-18.3** Given `deny -m <reason>`, then the process exits 3 with the denier's identity, group and reason on stderr and in `status`, visible to the requesting actor whatever their groups.
- **AC-18.4** Given `approve` by a service identity, by the actor, from the requesting session, by a non-member, or a second time by the same person under quorum 2, then it is refused with the rule, by the shell and by the admission policy.
- **AC-18.5** Given a policy edit, a lost hold or a removed target while a gate is pending, then the object is `void` with the reason, a new object is created under the current state, the page route fires again, and the receipt links both.
- **AC-18.6** Given a gate with no answer at the policy's timeout, then the object is `expired`, every hold it took is released, the page route fires, and the process exits 3; given a repeated plan shape within the reuse window, then no new answer is asked and the receipt links the answered object.
- **AC-18.7** Given identical pending requests (same shape, same group), then they share one object with a waiter count, one answer resumes all of them, and each receipt links the object.

## 9. Persona lenses

- **DO**: wants to submit and go home, read the reason for a no, and never re-type a release because an approval pointed at a sentence.
- **SEC**: wants users only, the actor excluded by the API server and not just the shell, the plan hash in the receipt, and a denial that is attributable.
- **AI**: wants the agent to queue at the gate like anyone and to be told plainly that it cannot answer.
- **SRE**: wants the on-call page to carry the object's path and a timeout that frees the held inputs when nobody answers.
- **FIN**: wants one hold for forty waiters and held bytes visible in `df`.
- **PE**: wants to edit a policy without wondering which version a pending gate is under, and a reuse window that makes a nightly cron entry stop paging.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L18.1 | P12 | ADR 0017 decision 6 makes the plan hash cover the input artifact ids, so a nightly cron entry never repeats a hash and a reuse window can never match; decision 10's own remedy cannot work. | **amend** ADR 0017 decision 6: the plan hash is the plan's shape (verb, arguments, digests, targets, route weights, contract flags, policy); the input artifact ids are listed beside it in the pinned `explain` and the receipt; a reuse window matches the shape; the approver sees the inputs and the shape, and approves the shape | ADR 0017 d6, d10 |
| L18.2 | P3 | Decision 4 says the session may detach, but under ADR 0016 decision 9 a session also ends at token expiry, and nothing says who clears the suspended object when the answer lands after that. | **amend** ADR 0017 decision 4: the gate is owned by the controller, not by either session; the `Approval` resource's status drives the suspended object, the stage resumes under the group's service identity (L17.1), and the receipt records the requesting session's end and the resume; an attached session only reports | ADR 0017 d4, ADR 0016 d5, d9 |
| L18.3 | P1, P6, P8 | Decision 6 voids the object when "the plan changes" but, with inputs held and the digest resolved at plan time, the plan cannot change; what can change is the policy, the holds and the devices, and the price. | **rule**: the void reasons are enumerated: the gating policy edited (group, fallback, quorum, timeout, window), a hold lost, a target device removed or disconnected, the approving group no longer able to answer; each is printed and re-creates the object under the current state with the page re-sent; at resume the planner re-prices and the receipt records the pinned and the settled band, with `--budget` still binding | ADR 0017 d6, ADR 0006 a2, ADR 0011 a4 |
| L18.4 | P5, P10 | An agent in a loop creates a pending object per attempt: forty gates, forty holds, forty pages for one decision. | **rule**: pending objects are deduplicated per (shape hash, group): a request whose shape matches a pending object joins it as a waiter, `ls` shows the waiter count, one answer resumes every waiter, each receipt links the object, and the inputs are held once per object | ADR 0017 d1, d4 |
| L18.5 | P4 | Rule L17.6 hides actor names outside the owning group; an approver is usually in another group than the author, so the author could not see who denied them. | **rule**: the answers on an approval object (identity, group, time, reason) are visible to the requesting actor and the owning group regardless of L17.6, because an answer is an act toward that process; the inbox stays visible to the approving group only | ADR 0017 d8, ADR 0016 rule L17.6 |
| L18.6 | P5 | Decision 2 defaults the timeout to the process's remaining deadline; an interactive verb with no deadline then has no timeout, and its held inputs are held forever. | **rule**: every policy has a timeout and the platform default is bounded (24 h unless set); `explain` prints it; at expiry the holds are released, the page route fires and the process exits 3; `df` lists held bytes per pending object | ADR 0017 d2, d7; ADR 0010 |
| L18.7 | none | A canary and its promotion are two plans and two gates, which doubles the approver's work for one release. | **accepted**: the promotion changes the production route weight, which is the decision the policy guards; a platform that wants one answer sets a reuse window that excludes the weight from the shape, and `explain` prints which fields the window ignores | ADR 0017 d6 |
