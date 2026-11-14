# US-31 · Nobody was told: a weekend of approvals, finished jobs, expiries and node faults, and who hears about each, where, and how fast

| Field | Value |
|---|---|
| Primary persona | DO |
| Secondary personas | SRE, SEC, ML, FIN, PE, DE |
| Workload class | fleet operation: events from Convoy's own objects (approval requests, process completions and failures, degraded runs, budget thresholds, expiries, disconnections, node health actions) delivered to people through the platform's chat, mail, incident and webhook channels, across residencies and an edge site |
| Walkthrough | written against [ADR 0017](../adr/0017-approvals-are-objects-on-the-process.md) decisions 2, 4, 5 and 7, [ADR 0021](../adr/0021-a-node-is-tested-before-it-is-trusted.md) decisions 5 and 8, [ADR 0023](../adr/0023-telemetry-is-a-device-the-shell-reads.md) decision 6, [ADR 0020](../adr/0020-a-session-is-a-journal-not-a-pod.md) decisions 1 and 7, [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md) decisions 5, 7 and 9, [ADR 0004](../adr/0004-completion-contract-and-exit-codes.md) and its amendment 2, [ADR 0006](../adr/0006-detach-is-not-destroy.md), [ADR 0007](../adr/0007-secrets-are-references.md) amendment 2, [ADR 0009](../adr/0009-stdout-artifact-stderr-log.md) amendment 1, [ADR 0026](../adr/0026-a-registry-is-a-device-a-tag-is-a-name-for-a-digest.md) decision 8, and the command map rows for `mailx`, `write`, `wall` and `mesg`; it follows US-30 |
| Gaps already closed there | none |
| Milestones needed | M2, M4, M7, M8, M10 |
| Findings in this document | L31.1 – L31.17 |

## 1. Story

**As a** release engineer, **I want** the people who must act on something Convoy is waiting for, or must know that something finished, failed or is about to expire, to hear about it once, in the channel their team actually watches, at the urgency it deserves, **so that** a Friday release does not sit unapproved all weekend, a training run that died on Saturday is not discovered on Monday, an exception does not lapse silently into a blocked release, and nobody is woken at 3 a.m. for a job that merely finished.

**Context.** After US-30 the platform runs ADRs 0017, 0021, 0025 and 0026. The organisation has:
- **Chat:** a Slack workspace, hosted in the US, where each team has channels.
- **Incidents:** PagerDuty, where SRE owns the schedules and escalation policies, and some services' routes are written in an Alertmanager configuration in the GitOps repository.
- **Mail:** a mail relay on SES in us-east-1.
- **Webhooks:** for the ML team's own bot.

The approval policy on `release` to `prod-*` names the group `release-approvers` with quorum 2 and `page:release` at timeout. The health controller of ADR 0021 "pages" with no route named. `kofi` (user `kofi`, group `release`) starts the Friday release at 17:40 and leaves. `ines` (group `vision`, ML) starts a 60-hour training run under `nohup`. The vulnerability exception of US-30 for `payments-legacy` expires at 00:00 on Saturday.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Starts the release | `kofi` | user `kofi`, group `release` | detached at 17:45 |
| Approves | `release-approvers` | members; quorum 2 | three members, one on holiday |
| On call | SRE | the PagerDuty schedule for `platform` | not a Convoy object |
| Starts the training run | `ines` | user `ines`, group `vision` | leaves the company on Monday |
| Owns the exception | `security-leads` | members | |
| Edge site | `site-ops` | group `site-ops` | link down Saturday night |
| Operates the channels | platform team | `svc:platform` | the Slack app, the mail relay, the PagerDuty integration |

## 3. Preconditions

- **Capabilities** (`caps`): `alerting` (the PagerDuty integration), the Slack app, mail through the relay; on `sp-01` none of them, only the receipt spool of ADR 0009 amendment 1.
- **Packages**: none; the notifier is described in the command map as "platform notifier (SES/SNS)".
- **Platform defaults** `explain` must print: the route of every event the plan can raise, the channel behind each route, its residency, and its last successful delivery.

## 4. Main path

```bash
# Friday 17:40
[kofi] $ release /units/payments --image=sha256:44ab…
gate: release to prod-us needs 2 of release-approvers (policy prod-release), timeout 72h
#  where does the request go? the policy names page:release for the timeout, nothing for the
#  request itself; ls /sys/approvals/release-approvers/ is an inbox nobody opens at the weekend  (L31.1)
[kofi] $ logout
#  the gate holds; nobody is told

# Friday 23:10 the health controller cordons two GPU nodes and "pages"
#  the route it uses is a Slack webhook to #platform-alerts, archived three weeks ago; every
#  page since then has returned 404 and gone nowhere                                             (L31.8)

# Saturday 00:00 the exception for payments-legacy expires (US-30)
#  nothing warned security-leads; Monday's patch release will be refused                        (L31.17)

# Saturday 04:12 ines's training run fails: rank 17 out of memory, retries exhausted
#  ines asked for "notify me" with mailx in a wrapper script; 1 900 partition-level messages
#  from the evaluation fan-out arrive first; the one that matters is buried                      (L31.6, L31.14)
#  the mail includes the last 50 lines of stderr: a customer's e-mail address from a
#  validation sample, now in a US mail relay and in 1 900 inboxes                                 (L31.13)

# Saturday 06:00 someone wires the approval request to #release by hand; the message says
#  "release payments: awaiting approval"
[maria] clicks "Approve" in Slack
#  which identity approved? the Slack app's bot token; ADR 0017 decision 5 says a valid answer
#  comes from the approver's own session                                                          (L31.5)
#  10:30 the release is approved and rolled out; at 11:00 it is rolled back by the canary;
#  #release still says "awaiting approval", and the rollback says nothing                         (L31.7)

# Saturday 22:00 the edge site's link goes down; a unit there is degraded
#  the notifier runs in us-east-1; the site cannot reach it; on reconnect Sunday 14:00 nothing
#  is sent, or everything is sent at once with no sign that it is 16 hours old                    (L31.10)

# Sunday the 72-hour approval timeout is counting from Friday 17:40, although the request
# reached a person only on Saturday 06:00                                                        (L31.9)

# Monday: the audit
#  the PagerDuty incident "payments 5xx" from Alertmanager, and Convoy's "release payments
#  rolled back" page, are two incidents for one event                                            (L31.11)
#  the training run's notification went to ines, who left this morning; nobody in vision knew   (L31.4)
#  the release page woke all three approvers at 03:00, including the one on holiday, for a
#  gate with 70 hours left                                                                        (L31.12, L31.15)
#  someone added a PagerDuty escalation step through a Convoy route file; SRE's schedule in
#  PagerDuty was edited from two places                                                           (L31.2)
#  write kofi "the gate is waiting" from another session: kofi is detached; the message is lost (L31.3)
#  the SMS fallback sent 340 messages on Saturday at $0.05 each, charged to no group              (L31.6)
```

| Step | Convoy object | Channel object | Where the state lives |
|---|---|---|---|
| an approval request | `/sys/approvals/<group>/<id>` | none today | the controller |
| a page at timeout | `page:<route>` | a PagerDuty event | PagerDuty |
| a health controller page | none | a Slack webhook | Slack |
| a finished process | a receipt | none today, or `mailx` in a script | the mail relay |
| an expiry | an exception, a hold, a TTL | none today | |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe: staged artifact, content id, cache | – | | |
| A2 | Command: class, annotation, executable fields | ○ | `mailx` in a wrapper script | |
| A3 | Session: mount table, env, cwd, attach and detach | ● | `write` to a detached session (L31.3) | AC-31.3 |
| A4 | Backend selection | – | | |
| A5 | Completion contract | ● | a failure, a degraded run and a fan-out of partition events (L31.6, L31.14) | AC-31.14 |
| A6 | Device tree | ● | no channel class (L31.1); a channel that fails (L31.8) | AC-31.8 |
| A7 | Signals and lifecycle | ○ | a rollback after an approval | |
| A8 | Secrets as references | ● | stderr in a message body (L31.13) | AC-31.13 |
| A9 | Cost | ● | SMS and voice per message; a storm (L31.6) | AC-31.6 |
| A10 | Identity and authorization | ● | an approval from a chat button (L31.5); a notification to a person who left (L31.4) | AC-31.4, AC-31.5 |
| A11 | Platform defaults and `caps` | ● | routes, channels and their owners (L31.2) | AC-31.2 |
| A12 | Storage | – | | |
| A13 | Networking | ○ | an edge site without its link (L31.10) | AC-31.10 |
| A14 | Time and queueing | ● | urgency and quiet hours; timeouts from delivery; expiry warnings (L31.9, L31.15, L31.17) | AC-31.9, AC-31.15, AC-31.17 |
| A15 | Audit, receipts, retention and `replay` | ● | messages as data at rest; the object as the record (L31.7, L31.13) | AC-31.7 |
| A16 | Durability | ○ | at-least-once delivery (L31.16) | |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative verb, declarative owner | schedules and escalation policies owned by the incident tool or by an Alertmanager configuration in GitOps | ADR 0023 decision 6: Convoy never writes the monitoring configuration | **L31.2** |
| P2 | POSIX semantics that cannot hold | `write` to a detached session; `mailx` exit 0 | the command map maps `write` to a session terminal and `mailx` to a notifier | **L31.3** |
| P3 | Orphaned or lost state | a notification addressed to a person who has left | ADR 0016 decision 5: unattended work belongs to the group | **L31.4** |
| P4 | Identity leak or escalation | an approval answered from a chat button | ADR 0017 decision 5: from the approver's own session | **L31.5** |
| P5 | Invisible cost | SMS and voice per message; a storm | none | **L31.6** |
| P6 | Lying cache | a message whose state is stale an hour later | none | **L31.7** |
| P7 | Silent failure | a route whose channel fails for three weeks | none; ADR 0021 says "pages" | **L31.8** |
| P8 | Partial fleet | a site that cannot reach the notifier | ADR 0009 amendment 1 spools receipts, not events | **L31.10** |
| P9 | Created elsewhere | the monitoring system pages for the same event | ADR 0023 decision 6 keeps alerting outside Convoy | **L31.11** |
| P10 | Multi-party action | a quorum of two paged at 3 a.m. | ADR 0017 decision 4 pages the group | **L31.12** |
| P11 | Data at rest | stderr and arguments in a message body, through a channel in another residency | ADR 0007 amendment 2 scrubs artifacts, not messages | **L31.13** |
| P12 | Process-tree inheritance | every partition of a fan-out notifies on its own | none | **L31.14** |

Four more questions without a probe: is there a notification channel at all (**L31.1**, open); does a gate's timeout count from creation or from the moment someone could know (**L31.9**, amend); which events deserve a page and which can wait for the morning (**L31.15**, rule); and who is warned before something expires (**L31.17**, rule). Delivery that is at least once and best effort is **L31.16**, accepted.

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| channel fails | the notifier (L31.8) | none; the event goes to the fallback route | the route, the error, the fallback used | fix the route |
| no route for an event | plan (L31.1, L31.4) | 3 under `notify=required` | the event and the group without a route | add the group's route |
| answer from an unmapped chat account | the gate (L31.5) | none; the answer is refused | the channel account and the reason | `approve` in a session |
| storm | the notifier (L31.6, L31.14) | none | the count rolled up | none |
| event raised during a disconnection | the site spool (L31.10) | none | raised and delivered times | none |
| expiry warning not delivered | the notifier (L31.17) | none | the object and its expiry | none |

## 8. Acceptance criteria

- **AC-31.2** Given a route that names an incident service or an on-call schedule, then Convoy sends events to it and never edits schedules or escalation policies; a route defined in a GitOps configuration is changed as intent there.
- **AC-31.3** Given `write` to a user with no attached terminal, then the message is kept in the user's session journal and shown at the next attach, and `write` says so; `mailx` exits 0 on acceptance by the channel and `/sys/notify/<id>/` shows delivery.
- **AC-31.4** Given any event, then it is addressed to a group through the group's route, never to a person; a group with no route uses the platform's default route, and no event is dropped for want of a route.
- **AC-31.5** Given an answer to an approval from a channel, then it counts only where the channel's driver maps the account to the identity provider's user and re-authenticates the user for the answer; the receipt records the channel; otherwise the message carries only a link to the object.
- **AC-31.6** Given events from one process, then they are rolled up per process and event class, each route has a rate limit, and every message is priced to the group that owns the object.
- **AC-31.7** Given an object whose state changes after a message about it, then the message is updated in place where the channel supports edits, or a superseding message names the earlier one; every message carries the object's path and the time of its state.
- **AC-31.8** Given a delivery failure, then the event goes to the platform's fallback route, the channel shows `failing` with its last success, and `explain` of any verb whose events use it prints that.
- **AC-31.9** Given an approval request, then its timeout counts from the first successful delivery to the approving group's route, bounded by the process's deadline.
- **AC-31.10** Given a site without its link, then events raised there are spooled with the receipts and delivered on reconnect marked with their raise time, and events about the site's disconnection are raised by the central controller.
- **AC-31.11** Given an event about a unit or process, then it carries a deduplication key derived from the object's path, so the incident tool groups it with alerts about the same object.
- **AC-31.12** Given a gate with a quorum, then the request goes to the approving group's route, which resolves to whoever is on call; a second answer is requested only after the first, and the author is never notified as an approver.
- **AC-31.13** Given a message, then its body is the object's path, the event class, a status line and the scrubbed `explain` summary, never stderr or argument values; a channel carries a residency, and an event about an object outside it is reduced to the path.
- **AC-31.14** Given a process tree, then `--notify` on the root applies to the tree, and child processes and partitions raise no events of their own.
- **AC-31.15** Given an event, then it has an urgency class: `page` only for gates with less than the platform's `page_before` left, unattended work refused or out of budget, and health actions; `notify` for completions and failures; `digest` for the rest, sent in the group's working hours.
- **AC-31.17** Given an object with an expiry (an exception, a hold, a branch unit, a lend, a grant), then its owning group is notified at the platform's `expiry_warn` lead time.

## 9. Persona lenses

- **DO**: wants a release gate to reach the approvers the moment it opens, and the outcome of a release to reach the channel that asked.
- **SRE**: wants Convoy to feed the incident tool they already run, with one incident per event, schedules left alone, and no 3 a.m. pages for things that can wait.
- **SEC**: wants approvals answered only by people the identity provider vouches for, and no stderr in chat.
- **ML**: wants to hear once that a 60-hour run failed, not 1 900 times, and wants the team to hear it if they have left.
- **FIN**: wants SMS and voice charged to the group that caused them.
- **PE**: wants routes to be devices with owners, residency and health, and a fallback that is never silent.
- **DE**: wants a degraded load to be noticed on the day, not at the next month-end.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L31.1 | none | Convoy raises events that need people (approval requests, failures, degraded runs, budget exhaustion, expiries, disconnections, health actions), but only names routes (`page:<route>`, `caps: alerting`) that nothing defines; ADR 0021 "pages" with no route; `mailx` maps to an unnamed notifier. There is no channel, no route, no subscription and no event model, and ADR 0023 decision 6 says Convoy never alerts without saying whether its own events are alerts. | **open** (drafted 2026-11-13 in [ADR 0027](../adr/0027-convoy-tells-a-group-about-its-own-objects.md): channels are devices under `/dev/notify/`; a group's route in `/etc/notify.d/<group>` maps `page`, `notify` and `digest` to channels; a closed set of event classes raised only by Convoy's own objects; events go to groups, roll up per process tree, and are pointers to objects; delivery belongs to the controller and a failure is itself an event). The record asked for: notification channels as devices (chat, mail, incident tools, webhooks), routes owned by groups, a closed set of event classes raised by Convoy's own objects with an urgency each, subscriptions on process trees and objects; Convoy notifies about its own objects and never alerts on telemetry | ADR 0017 d2, d4; ADR 0021 d5, d8; ADR 0023 d6; ADR 0011 d9; command map `mailx` |
| L31.2 | P1 | A route was used to add an escalation step, so SRE's schedule in the incident tool was edited from two places. | **rule** (ADR 0023 rules; ADR 0010 rules): a route names a service, a channel or a webhook the platform has; Convoy sends events to it and never creates or edits schedules, escalation policies or alert rules; a route whose definition lives in a GitOps configuration is changed as intent there | ADR 0023 d6; ADR 0010 a1 |
| L31.3 | P2 | `write` to a detached user lost the message, and `mailx` exit 0 meant the relay accepted it, not that anyone received it. | **rule** (ADR 0020 rules; command map): `write` to a user with no attached terminal keeps the message in the user's session journal (ADR 0020 decision 2) and shows it at the next attach, saying so; `mailx` and other notify verbs exit 0 on acceptance by the channel, and delivery is a state in `/sys/notify/<id>/` | ADR 0020 d2; ADR 0004 |
| L31.4 | P3 | A failure notification went to `ines`, who had left; her group never heard. | **rule** (ADR 0016 rules): events are addressed to the owning group through its route, never to a person; a person may subscribe in addition; a group without a route uses the platform's default route, and no event is dropped for want of one; offboarding never silences a group's events (ADR 0016 decision 9) | ADR 0016 d5, d9 |
| L31.5 | P4 | An approval was answered by a chat button under the chat app's bot token. | **amend** ADR 0017 decision 5: an answer may come from a channel only where the channel's driver maps the account to the identity provider's user (as a code-owner review counts under L28.1) and re-authenticates the user for the answer; the receipt records the channel; any other channel carries only a link to the object, and the answer is given in a session | ADR 0017 d5; ADR 0024 d4; ADR 0016 d4 |
| L31.6 | P5 | A fan-out raised 1 900 messages, and an SMS fallback sent 340 messages charged to nobody. | **rule** (ADR 0010 rules; ADR 0009 rules): events are rolled up per process and event class; each route has a rate limit (`notify_max`), past which events go to a digest; channels have a price file, and each message is charged to the group that owns the object | ADR 0010; ADR 0009 |
| L31.7 | P6 | Chat still said "awaiting approval" after the release was approved, rolled out and rolled back. | **rule** (ADR 0017 rules; ADR 0013 rules): every message carries the object's path and the time of the state it reports; a later state of the same object updates the message in place where the channel supports edits, or sends a superseding message that names the earlier one; the object, never the message, is the record | ADR 0017 d8; ADR 0013 d5 |
| L31.8 | P7 | The health controller's route had failed for three weeks; every page returned 404. | **rule** (ADR 0021 rules; ADR 0011 rules): a delivery failure is itself an event, sent to the platform's fallback route; the channel shows `failing` with its last success; `explain` of a verb whose events use a failing route prints it, and the inventory checks each route daily with a test event | ADR 0021 d5, d8; ADR 0011 a4 |
| L31.9 | none | The 72-hour approval timeout counted from Friday 17:40, though no approver could know before Saturday 06:00. | **amend** ADR 0017 decision 7: an approval's timeout counts from the first successful delivery of its request to the approving group's route, bounded by the process's remaining deadline; the receipt records creation, delivery and answer times | ADR 0017 d2, d7; ADR 0004 |
| L31.10 | P8 | Events raised at an edge site without its link were lost, or would arrive all at once with no age. | **rule** (ADR 0009 rules; ADR 0011 rules): events raised where the notifier cannot be reached are spooled with the receipts (ADR 0009 amendment 1) and delivered on reconnect marked with their raise time; the disconnection itself is raised by the central controller, which can see it | ADR 0009 a1; ADR 0011 a4 |
| L31.11 | P9 | Alertmanager's incident and Convoy's page for the same rollback were two incidents. | **rule** (ADR 0023 rules): every event carries a deduplication key derived from the object's path (`unit/payments`, `proc/9101`), which the incident tool uses to group it with alerts that carry the same labels (ADR 0018); Convoy notifies only about its own objects and never alerts on telemetry | ADR 0023 d6; ADR 0018 |
| L31.12 | P10 | The release gate paged all three approvers at 3 a.m., one of them on holiday. | **rule** (ADR 0017 rules): an approval request goes to the approving group's route, which the incident tool resolves to whoever is on call; for a quorum above one, a further answer is requested only after each answer; the author is never notified as an approver | ADR 0017 d2, d4, d7; ADR 0016 d7 |
| L31.13 | P11 | A failure mail carried 50 lines of stderr with a customer's e-mail address, through a US relay. | **rule** (ADR 0007 rules; ADR 0011 rules): a message body is the object's path, the event class, a status line and the scrubbed `explain` summary, never stderr or argument values; each channel carries a residency and a retention, and an event about an object outside the channel's residency is reduced to its path | ADR 0007 a2; ADR 0011 a3; ADR 0016 d11 |
| L31.14 | P12 | Each partition of a fan-out notified on its own, burying the training run's failure. | **rule** (ADR 0004 rules): `--notify=<route>[:<events>]` on a process applies to its tree; child processes and partitions raise no events of their own, and the root's event summarises them (failed partitions, degraded floor) | ADR 0004 a2, a3 |
| L31.15 | none | Every event had the same urgency; a gate with 70 hours left paged at 3 a.m. | **rule** (ADR 0017 rules; ADR 0010 rules): every event class has an urgency: `page` for gates with less than `page_before` left, unattended work refused or out of budget, and health actions; `notify` for completions, failures and degraded runs; `digest` for the rest, sent in the group's declared working hours | ADR 0017 d4; ADR 0010 |
| L31.16 | none | Channels deliver at least once and on a best-effort basis: a message can be late, duplicated or missed. | **accepted**: the object is the record, the message is a pointer to it; receipts record when each event was raised and delivered | ADR 0009 |
| L31.17 | none | A vulnerability exception expired at midnight with no warning; holds, branch-unit TTLs, lends and grants expire the same way. | **rule** (ADR 0006 rules; ADR 0026 rules): every object with an expiry raises `expiring` to its owning group at the platform's `expiry_warn` lead time, and `expired` when it lapses | ADR 0006; ADR 0026 d8; ADR 0022 d3 |
