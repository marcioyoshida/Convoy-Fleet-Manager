# US-14 · Let the shell finish my line in a regulated namespace and at a cut-off site, without it leaking, paying or acting for me

| Field | Value |
|---|---|
| Primary persona | AI (AI Engineer) |
| Secondary personas | SEC, FIN, PE, DE |
| Workload class | interactive, assisted; regulated and disconnected settings |
| Walkthrough | none; written against [ADR 0013](../adr/0013-native-verbs-have-one-shape.md) decisions 3 and 10 and [ADR 0014](../adr/0014-assisted-completion-the-model-proposes.md), in the settings of US-08 (residency) and US-10 (disconnected sites) |
| Gaps already closed there | none |
| Milestones needed | M2, M5, M7, M8 |
| Findings in this document | L14.1 – L14.6 |

## 1. Story

**As an** AI engineer, **I want** the shell to offer the next word with a tab and to turn a comment into a line I can read before I run it, **so that** I write correct pipelines without memorizing the verbs, and nothing I did not type ever runs, pays or sees more than I can see.

**Context.** Twelve engineers run the triage agents of US-07 in `prod-eu`, a namespace under residency rules, and maintain the anomaly units at the forty sites of US-10. Today they paste commands from a wiki; last quarter a pasted flag deleted a topic. The platform team will attach a model only if it can show the regulator what left the region, show finance what each tab costs, and show itself that the model never acted.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | the AI engineer | personal identity in `prod-eu`; `sudo --as platform` for one step | L14.5 |
| Approves | the platform team, for the model mount | `completion_device` in the profile | |
| Pays | the team's cost centre | per call, under the session | L14.3 |
| Audits | security, on request | what context left the session, to which device, under which residency | L14.2, L14.4 |
| Operates the platform | the same team | the model device in the inventory, its price and retention files, the scrub rules | |

## 3. Preconditions

- **Capabilities**: `model:bedrock` driver for the device; `queue:kafka`; `artifact-store` in `eu-west-1`; `metrics` is not required.
- **Packages**: `agent-run` (US-07), `enrich` (US-12).
- **Mounts**: `/dev/model/self` from `/dev/model/by-id/bedrock-eu-west-1-…` with `identity=secret:completion-reader,residency=eu,scope=rank,translate,cap=2.00`; `/queues/orders` and `/queues/eu` as in US-12; `/sites/17` as in US-10.
- **Annotations**: the whole database, since the grammar reads every class; `enrich: stateless`; `agent-run` as in US-07.
- **Platform defaults** `explain` must print and completion must honour: `completion_device`, `completion_timeout_ms`, the scrub rule set applied to outbound context, the cap window, the residency requirement for `/dev/model` mounts in regulated profiles.

## 4. Main path

```bash
$ cd /clusters/prod-eu/finance-close
$ cat /s3/close/2026-10/*.csv | sort -t, -k1 | <TAB>
  uniq -c    aggregate, accepts sorted input     1 648 partitions, $3.10–5.40, 0 cached
  join       fan-in, needs a second sorted input
  wc -l      aggregate                           same partitions, $2.90–4.10
  (tail -f   refused: a follow after a bounded pipe)
[12 candidates from the grammar, 3 glossed · rank: bedrock-eu-west-1, $0.0004, 140 ms]

$ # reprocess yesterday's orders into the eu enriched topic
$ <TAB>
draft: cat --from=@2026-11-08T00:00Z --to=@2026-11-09T00:00Z /queues/orders/orders.placed \
       | enrich --geo /s3/ref/geo.parquet | tee /queues/eu/orders.enriched
explain: 48 partitions, coalesced to 16 · band $9–14 · side effect on orders.enriched
warning: orders.enriched already holds output for this window from pid 7718 (a rerun appends)
[translate: bedrock-eu-west-1, $0.003 · the draft is at the prompt, nothing ran]

$ rm /queues/acme/<TAB>
  acme.events    38 GiB  7d  (rm on a topic needs --with-data and an approval; never pre-filled)

$ ls /secrets/<TAB>
  payments-db-password   triage-api-key          (names only; no value exists in the session)

$ cd /sites/17 && cat queue/<TAB>
  telemetry.raw   stale 41 min, site-17 disconnected        alerts   stale 41 min
[grammar order · model not called: listing is stale]

$ cd /clusters/prod-us/ops && cat /queues/us/<TAB>
  orders  returns  shipments
[grammar order · model not consulted: names under prod-us would leave residency eu]

$ cat /sys/cost/session/completion
calls 212  grammar-only 180  model-calls 32  spent $0.41 of cap $2.00 (rolling 24 h)
dropped: outside-set 3  type-check 1  destructive-proposed 2  scrubbed 1

$ batch nightly.sh        # a comment inside the script is a comment; no model is consulted
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `mount /dev/model/…` | model device in the mount table | a Bedrock endpoint opened under a role; nothing in the cluster | session |
| tab after `\|` | candidate set from the annotation database, filtered by identity, glossed by `explain --json` | none | session |
| rank | one inference | the provider's request log | provider, under its retention (L14.4) |
| translate | a draft line at the prompt | none; `explain` of the draft | session |
| tab on a path | metadata cache entries with age | none | session cache |
| `/sys/cost/session/completion` | ledger lines and counters | none | session, then the ledger |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ○ | the gloss shows cache hits the plan would get | the gloss matches a later `explain` |
| A2 | Command | ● | the annotation classes are the grammar | after a sorted pipe, aggregates rank above follows; a follow is refused in the list |
| A3 | Session | ● | the session owns every input to completion | two terminals on one session see the same candidates and the same counters |
| A4 | Backend | ○ | the gloss names the backend the plan would use | |
| A5 | Contract | ○ | the gloss prints the contract that would apply | |
| A6 | Device tree | ● | the model is a device; a disconnected site contributes a stale listing | `ls /dev/model/by-id`, `/sys/model/<id>/price`, `stale` marks |
| A7 | Signals | – | | |
| A8 | Secrets | ● | secret names complete, values never exist | no candidate or gloss ever contains a value; a comment containing a secret shape is scrubbed (L14.2) |
| A9 | Cost | ● | every call a ledger line, a cap per mount | `/sys/cost/session/completion` sums to the ledger; at the cap, grammar order with a notice |
| A10 | Identity | ● | candidates filtered by the effective identity | after `sudo --as platform`, candidates widen and narrow back on exit (L14.5) |
| A11 | Defaults | ● | timeout, scrub rules, cap window, residency requirement | all in `explain`; a mount without `residency` on a regulated profile is refused |
| A12 | Storage | – | | |
| A13 | Networking | – | | |
| A14 | Time | ○ | the timeout and the cap window | the fallback is marked (L14.1); the window is rolling, not midnight in an unstated zone |
| A15 | Audit | ● | suggestions are not actions; refused proposals are counted | the audit stream has no suggestion events; the counters exist |
| A16 | Durability | ○ | counters and spend survive a reattach | |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | the model mount comes from the platform profile | a default mount, not a managed object (ADR 0010) | ok |
| P2 | POSIX cannot hold | bash completion is local and instant; this one may call a network | `completion_timeout_ms` then grammar order, but is the fallback visible? | **L14.1** |
| P3 | Orphaned state | a draft line, a dropped proposal | neither is state; counters are | ok |
| P4 | Identity leak | the comment sent for translation is free text the user typed | ADR 0014 d5 scrubs values that cannot exist, not text that does | **L14.2** |
| P5 | Invisible cost | a gloss per candidate is a plan per candidate, and a plan may price a listing | `explain` reads the cache only (ADR 0013 d3), but how many plans per tab? | **L14.3** |
| P6 | Lying cache | a stale listing offers a path that is gone | age shown; `stale` on a disconnected device (ADR 0014 d7) | ok |
| P7 | Silent failure | a proposal outside the set is dropped | counted under `/sys/session/<id>/completion/` (d8) | ok |
| P8 | Partial fleet | tab under `/clusters/all/` fans out to every mount | per-mount cache, each stale mark separate (ADR 0011 d6) | ok |
| P9 | Created elsewhere | a topic or unit created outside Convoy appears as a candidate | owner `?` in the gloss (S9) | ok |
| P10 | Multi-party | `approve <TAB>` lists pids awaiting approval | only those the identity may approve; never pre-filled | ok once L3.1 exists |
| P11 | Data at rest | the provider keeps the request; residency says where, not for how long | nothing in ADR 0014 names provider-side retention | **L14.4** |
| P12 | Inheritance | `sudo --as`, and a line that names paths under two residencies | the identity and residency in force at the keystroke are undefined | **L14.5** |

A thirteenth question specific to this story: two identical tabs can rank differently. The ranking is a model output, non-deterministic by nature. **L14.6**.

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| model device unreachable or slow | `completion_timeout_ms` | n/a | counter `timeouts`; list marked `grammar order` (L14.1) | none needed |
| daily cap reached | the ledger | n/a | notice on the list; counter | wait, or raise `cap` on the mount |
| comment contains a secret shape | the scrub rule | translation refused | counter `scrubbed` | rewrite the comment |
| path names leave the mount's residency | residency intersection (L14.5) | n/a | `model not consulted` with the reason | a model mounted for that region |
| site disconnected | device state | n/a | `stale` with age | none; the listing is the last known |
| proposal outside the candidate set | the session | n/a | counter `outside-set` | none |
| mount without `residency` on a regulated profile | `mount` pre-flight | 3 | the profile rule | add the option |

## 8. Acceptance criteria

- **AC-14.1** Given a cursor after `sort |`, then every candidate is a command whose input class accepts a sorted stream, a follow is listed as refused, and with the model device unmounted the same set appears in grammar order.
- **AC-14.2** Given a comment at the prompt and a tab, then a draft line appears at the prompt, `explain` of it is shown including its warnings and side effects, and no object, receipt or ledger line other than the inference exists afterwards.
- **AC-14.3** Given a model call, then a ledger line with the device's price exists under the session, the counters increase by one, and at the cap the list is in grammar order with a notice.
- **AC-14.4** Given a path under a device in `disconnected` state, then candidates come from the cache with their age, the model is not called, and the tab returns within the platform's timeout.
- **AC-14.5** Given a comment that matches a secret shape, then translation is refused, the comment is not sent, and the `scrubbed` counter increases.
- **AC-14.6** Given a line that names a path on a device outside the model mount's residency, then the model is not consulted and the notice names the device and the residency.
- **AC-14.7** Given `sudo --as platform` and a tab, then the candidate set is the platform identity's and returns to the personal set on exit; given a script run by `batch`, then no model call appears in the ledger.
- **AC-14.8** Given `/sys/model/<id>/retention`, then a regulated profile refuses to mount a device whose retention is not zero, and `explain` prints the retention of the mounted device.

## 9. Persona lenses

- **AI**: wants the grammar to do most of the work and the model to explain, not decide; refuses if a tab can run something.
- **SEC**: wants a list of what left the region and for how long the provider keeps it; refuses on free-text comments going out unscrubbed.
- **FIN**: wants each tab's price on a line they can sum, and a cap that cannot be exceeded by typing faster.
- **PE**: wants the model to be one more device in the inventory with the same `caps`, price and hotplug rules, not a setting in each user's dotfiles.
- **DE**: wants the draft of a reprocess to carry the same warnings `explain` would give them, so the comment-to-line path is not a way around L11.2.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L14.1 | P2 | When the model times out or the cap is reached, the list falls back to grammar order, and a user who believes it is ranked reads the first candidate as the recommended one. | **rule**: every list states its source, `ranked by <device>` or `grammar order`, and the reason for the latter (timeout, cap, residency, stale listing); the notice is the same line that appears in `/sys/cost/session/completion` counters | ADR 0014 d6, d7 |
| L14.2 | P4 | Decision 5 scrubs what cannot exist in the session (secret values) but the comment a user types for translation is free text and may contain a password, a token or a customer identifier. | **amend** (applied: ADR 0014 d5 revised) ADR 0014 d5: outbound context passes the same scrub that `convoy-put` applies to artifacts (L7.1); a match refuses the translation, names the rule, and counts it; the rule set is a platform default and `explain` prints its version | ADR 0014, ADR 0009 (L7.1) |
| L14.3 | P5 | A gloss per candidate is an `explain --json` per candidate; a tab with twelve candidates is twelve plans, each reading the metadata cache, and the first `explain` of a large listing may be the one that populates the cache at a price. | **rule**: completion glosses at most the platform's `gloss_top_k` candidates after grammar ranking, never populates a priced listing (a cold path shows `size unknown` instead), and the plan it shows is the one `explain` would print from cache | ADR 0013 d3, ADR 0014 d10 |
| L14.4 | P11 | `residency` says where the context goes, not how long the provider keeps it. The request is data at rest at the provider under the provider's retention, which a regulated profile must be able to require to be zero. | **amend** (applied: ADR 0014 d3 revised) ADR 0014 d3: `/sys/model/<id>/retention` states provider-side retention from the inventory; mount option `retention=0` is required by regulated profiles and the mount is refused otherwise; the audit record of a mount includes it | ADR 0014, ADR 0011 d7 |
| L14.5 | P12 | The identity and residency in force at a keystroke are undefined: `sudo --as` widens the candidate set, and a line that names paths on devices in two topologies has no single residency. | **rule**: candidates are filtered by the effective identity at the keystroke and re-filtered when it changes; the residency of outbound context is the intersection of the model mount's `residency` and the topology of every device the line names, and an empty intersection means the model is not consulted, with the notice of L14.1 | ADR 0014 d4, d5; ADR 0011 d2 |
| L14.6 | none | Two identical tabs can rank differently because ranking is a model output. | **accepted**: ranking is interactive and non-deterministic by design; scripts never see it (d9); a user who wants a stable list sets `completion=grammar` on the session; the `/sys` counters make drift visible to the platform | ADR 0014 d2, d9 |
