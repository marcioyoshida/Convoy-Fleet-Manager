# US-08 · Close the month in-region, on time, and prove it two years later

| Field | Value |
|---|---|
| Primary persona | SEC (Security & Compliance) |
| Secondary personas | DE, FIN, SRE |
| Workload class | regulated batch |
| Walkthrough | [walkthroughs.md §8](../walkthroughs.md) |
| Gaps already closed there | G9 |
| Milestones needed | M1, M2, M3, M4, M8 |
| Findings in this document | L8.1 – L8.7 |

## 1. Story

**As a** compliance officer, **I want** the EU month-end close to be provably in-region from input to output, to page someone if it will miss 06:00, and to be re-executable by an auditor years later with a yes/no answer, **so that** the evidence is a receipt rather than a slide deck.

**Context.** The regulator can ask for the computation behind a figure for seven years. The team's current evidence is a wiki page and the memory of whoever ran it. Data residency is a contractual obligation with per-incident penalties.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | the finance data engineer | `finance-batch` through the `/clusters/prod-eu` mount, `residency=eu` (L8.2) | |
| Approves | none; the residency policy is the gate | | |
| Pays | finance | euros, from `/sys` price rate of the EU region | |
| Audits | an external auditor, up to seven years later | `replay <pid> --verify` under their own identity (L8.5) | |
| Operates the platform | platform team | EU artifact store device (L8.1), held registry, alerting package | |

## 3. Preconditions

- **Capabilities**: `artifact-store` in `eu-west-1`, `alerting`, `registry-hold`, `object-lock` or copy-on-hold for inputs (L8.3).
- **Packages**: `extract`, `fx-normalize`, `close` as user tools with annotations; the alerting package providing `page:`.
- **Mounts**: `/clusters/prod-eu` with `residency=eu`; `/s3/eu/ledger`, `/s3/eu/ref`; `/artifacts` on `/dev/obj/by-id/cas-eu-west-1`.
- **Annotations**: standard classes; nothing exotic.
- **Platform defaults** `explain` must print: the artifact store chosen and its region, the time zone the deadline is read in, the hold mechanism for inputs and the toolbox image, the alerting route.

## 4. Main path

```bash
$ explain close.sh --deadline=06:00 --on-deadline=page:finance-oncall     # residency from the mount
convoy: stage 2 reads /dev/obj/by-id/s3-us-ref (us-east-1) outside residency eu (exit 3)
$ cp /s3/ref/fx-rates /s3/eu/ref/fx-rates && sed -i 's#/s3/ref/#/s3/eu/ref/#' close.sh
$ batch close.sh --deadline=06:00 --on-deadline=page:finance-oncall --retain=7y
$ wait 7702 && ls -l /artifacts/receipts/7702
(two years later)  $ replay 7702 --verify
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `explain` with residency | plan-time policy check | nothing created | audit stream |
| `cp` across regions | copy process with egress | copy Job | the EU bucket; **the cross-region transfer is itself an audited event** |
| `batch --retain=7y` | process 7702 with a hold | Indexed Jobs in `prod-eu`; artifacts in the EU store; hold records | store, registry, receipts |
| `replay --verify` | new process linked to 7702 | same toolbox digest, held inputs | new receipt |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ● | artifacts are the evidence; **and they are data at rest with a location** | every artifact of 7702 lives in the EU store (L8.1) |
| A2 | Command | ○ | ordinary classes | |
| A3 | Session | ○ | the session pod's own region | the session for an EU mount runs in the EU or holds no data (L8.1) |
| A4 | Backend | ○ | Indexed Jobs | |
| A5 | Contract | ● | deadline with a paging action; exit 3 on residency | AC-8.2 |
| A6 | Device tree | ● | topology drives residency; `residency=eu` as a mount option (L8.2) | `mount` shows the option; every command under the mount is checked |
| A7 | Signals | – | | |
| A8 | Secrets | – | | |
| A9 | Cost | ○ | euros from the EU price rate | the band is in the mount's currency |
| A10 | Identity | ● | `finance-batch` then, the auditor later | `replay` runs under the auditor and links to the original (L8.5) |
| A11 | Defaults | ● | store, time zone, hold mechanism, alert route | all in `explain` |
| A12 | Storage | ○ | object stores only | |
| A13 | Networking | – | | |
| A14 | Time | ● | a wall-clock deadline in a time zone (L8.4) | the receipt stores UTC and the zone |
| A15 | Audit | ● | the whole story | receipt fields: toolbox digest, input ids, env subset hash, identity, store, hold |
| A16 | Durability | ● | seven years | the held closure survives bucket lifecycle rules, registry pruning and team turnover (L8.3, L8.6) |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | none | | ok |
| P2 | POSIX cannot hold | `--deadline=06:00`: whose clock? | unstated | **L8.4** |
| P3 | Orphaned state | none | holds are the opposite problem | ok |
| P4 | Identity leak | the auditor's replay needs read on held inputs only | identity design open | L8.5 |
| P5 | Invisible cost | seven years of held storage | the receipt prices the hold per month | ok |
| P6 | Lying cache | a rerun after the fix is a cache hit on stages that passed | by design; the hit is itself evidence | ok |
| P7 | Silent failure | `page:` with no alerting package | undefined | **L8.7** |
| P8 | Partial fleet | one cluster | n/a | ok |
| P9 | Created elsewhere | the ledger objects in S3 are written by another system | they are inputs by id; their later change is why holds must copy or lock them | **L8.3** |
| P10 | Multi-party | none | | ok |
| P11 | Data at rest | intermediate artifacts, receipts and logs: where? | the design never placed the artifact store | **L8.1** |
| P12 | Inheritance | residency as a flag on `explain` and `batch` | the manual `cp` ran without it and moved data across regions | **L8.2** |

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| a stage reads a US device | plan-time check | 3 | device, region, nothing ran | mount an EU copy |
| projected finish after 06:00 | deadline watcher | 2 at 06:00 | page sent, partials kept | rerun after fix; cache hits |
| artifact store outside the EU | plan-time check (L8.1) | 3 | store id and region | platform provides an EU store |
| input object deleted after two years | `replay` | 1 | missing input id | none; this is why L8.3 copies or locks |
| toolbox image pruned from the registry | `replay` | 1 | missing digest | none; this is why L8.6 holds a copy |
| alerting package missing | plan time (L8.7) | 3 | `caps: alerting` | platform installs it |

## 8. Acceptance criteria

- **AC-8.1** Given a mount with `residency=eu`, when any command under it reads or writes a device outside an `eu-*` topology, then it is refused with exit 3 naming the device and region, before anything runs; `cp` from a US device into the mount is refused the same way unless `--allow-cross-region` is given and audited.
- **AC-8.2** Given `--deadline=06:00`, then `explain` prints the time zone used; at the deadline the paging action runs, exit is 2 and partial artifacts are kept.
- **AC-8.3** Given `--retain=7y`, then the receipt, every artifact, every input object (copied or locked) and the toolbox image digest are held; `rm`, `rmvol`, GC and registry pruning refuse to break the hold, and the hold's monthly cost is on the receipt.
- **AC-8.4** Given `replay 7702 --verify` two years later under a different identity, then it runs on the held inputs with the held toolbox digest, reports whether output ids are identical, and its receipt links to 7702.
- **AC-8.5** Given an artifact store outside the EU as the only store, then `batch` under the EU mount is refused at plan time naming the store.
- **AC-8.6** Given no alerting capability, then `--on-deadline=page:` is refused at plan time, not silently dropped.

## 9. Persona lenses

- **SEC**: wants residency enforced on the mount for every command and evidence that includes the intermediate data's location; refuses if a flag can be forgotten.
- **DE**: wants the residency failure at `explain`, with the fix spelled out.
- **FIN**: wants the hold priced for seven years at submission time.
- **SRE**: wants the page to fire on the projected miss, not only at 06:00.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L8.1 | P11 | The artifact store is where every pipe, receipt and log lives, and the design never placed it. A single global store breaks residency for every intermediate artifact. | **amend** ADR 0001 and ADR 0011: the artifact store is a device per topology (`/dev/obj/by-id/cas-<region>`); the planner selects a store within the stage's residency and refuses when none exists; receipts name the store; the session pod for a residency mount runs in-region or holds no data | ADR 0001, ADR 0011, ROADMAP M4 |
| L8.2 | P12, P4 | Residency as a per-command flag is bypassable by forgetting it; the walkthrough's own `cp` moved data across regions without a check. | **amend** ADR 0011: `residency=` is a mount option, enforced for every command touching the mount, with `--allow-cross-region` as an audited per-command override; ADR 0010 lets the platform set it per namespace | ADR 0011, ADR 0010 |
| L8.3 | P9 | Inputs live in external stores and may change or vanish; a hold on outputs alone does not make `replay` possible in year three. | **amend** ADR 0006 amendment 1: `--retain` extends to inputs by copying them into the held store or by setting the device's object lock, and the receipt records which | ADR 0006 |
| L8.4 | P2 | `--deadline=06:00` has no stated time zone. | **rule**: wall-clock deadlines are read in the session's `TZ`; receipts store UTC and the zone; `crontab` sets `timeZone` | ADR 0004, posix-command-map.md (`crontab`, `at`) |
| L8.5 | P4 | `replay` years later runs under whose identity, with what rights on the held data? | **rule**: `replay` runs under the caller's identity, which needs read on the held closure only; the replay receipt links to the original and records both identities | ADR 0009, identity design (open) |
| L8.6 | P7 | A held receipt pins an image digest, but the registry is external and prunes. | **rule**: holds copy the image by digest into a platform-held registry device (`/dev/registry/by-id/held`), or refuse when none exists | ADR 0006 amendment 1, ROADMAP M9 |
| L8.7 | P7 | `--on-deadline=page:` depends on an alerting package that may be absent. | **rule**: `caps: alerting` is required for `page:` actions; absence is a plan-time refusal | ADR 0004, ADR 0010 |
